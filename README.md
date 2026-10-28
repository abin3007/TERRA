# TERRA Node

A tamper-evident, hyper-local weather and ground-conditions logger on an **ATmega328P** (3.3 V, 8 MHz), built as the BACSE324
Embedded Systems Architecture project around the TERRA platform. It samples temperature, humidity, pressure, rainfall and battery
voltage, and writes 32-byte records into an external EEPROM ring. Every record carries an 8-byte HMAC-SHA256 tag chained to the
previous record, so a host-side verifier can tell whether the log was edited, reordered, truncated or torn by a power cut.
A witness button stamps a MARK record so a citizen reporter can say "I was here, at this moment".

> **Status: the firmware has been verified in simulation only** (simavr, cycle-accurate, with models of the I²C parts).
> It has not yet been run on hardware. `docs/bench-plan.md` lists the measurements to take on the real board.

## Layout

| Path | Contents |
|---|---|
| `firmware/` | ATmega328P firmware (C, avr-gcc). `src/`, `include/`, `bench/` (micro-benchmark firmware), `Makefile` |
| `host/` | Host unit and differential tests: SHA-256/HMAC vs Python `hashlib`, BME280 integer compensation vs the datasheet's double-precision reference |
| `sim/` | `harness.c`: simavr harness with models of the BME280, DS3231, 24LC512, reed switch, button, UART host, battery, and fault injection |
| `tools/` | `terra_verify.py` (log verifier), `tamper_test.py` (nine attack types) |
| `analysis/` | Campaign scripts, analysis, power/schedulability model, figure scripts |
| `results/` | Raw run outputs (`runs/`), `results.json`, `model.json`, `microbench.json` |
| `report/` | Report builder, figures, Graphviz sources |
| `docs/` | Viva notes, bench plan, decision log |

## Build and run

Requirements (Debian/Ubuntu): `gcc-avr avr-libc simavr libsimavr-dev libelf-dev python3 graphviz`

```sh
make -C firmware variants           # base, only_<switch>, opt (+ noprof, test-hook and measurement builds)
make -C firmware VARIANT=opt        # the shipping configuration; prints flash and RAM
make -C firmware benchall           # micro-benchmark builds
cd sim && gcc -O2 -w -I/usr/include/simavr -I../firmware/include -I../host -o harness harness.c ../firmware/src/bme280_comp.c -lsimavr -lelf -lm -ldl
./harness ../firmware/build/opt/fw.elf --scenario day --out /tmp/day
python3 tools/terra_verify.py /tmp/day/eeprom.bin --key firmware/include/node_key_list.h
python3 tools/tamper_test.py /tmp/day/eeprom.bin firmware/include/node_key_list.h 200
```

Scenarios: `day`, `tips`, `button`, `dump`, `boot`, `bench`, `fault_sensor`, `fault_store`, `fault_rtc`, `fault_hang` (use the `opt_hooks` build),
`powercut` (use `--pc-prob 0.3` to cut power during 30 % of EEPROM write cycles), `lowbatt`, `lowbatt_cycle`, `uart_wake`, `uart_block`, `power`.

Host tests:
```sh
cd host && python3 gen_vectors.py && gcc -O2 -I../firmware/include -I. -o /tmp/tc test_crypto.c ../firmware/src/sha256.c && /tmp/tc vectors.txt
gcc -O2 -I../firmware/include -I. -o /tmp/tb test_bme.c ../firmware/src/bme280_comp.c -lm && /tmp/tb
```

To regenerate every number in the report: `analysis/run_campaign.sh`, `analysis/run_campaign2.sh`, `analysis/run_campaign3.sh`, `python3 tools/tamper_test.py ... results/tamper.json`, `python3 analysis/microbench.py`,
`python3 analysis/analyze.py`, `python3 analysis/model.py`.

## Record format (32 bytes, little-endian)

`seq u32 · ts u32 (unix) · temp i16 (0.01 °C) · rh u16 (0.01 %) · p u16 (0.1 hPa) · rain_tips u16 · rain_total u16 · vbat_mv u16 · flags u8 · node u8 · crc16 u16 · tag[8]`

Flags: MARK 0x01, STORM 0x02, SENSOR 0x04, LOWBATT 0x08, BOOT 0x10, WDT 0x20, TIMEUNSET 0x40, RECOVERED 0x80.
`tag = HMAC-SHA256(K, prev_tag || bytes 0..23)[0..7]`. CRC is CRC-16/CCITT-FALSE over bytes 0..21.

## Known limits (see report section 11)

* The HMAC key is compiled into flash (`node_key_list.h` holds a **test key**). Anyone who can read the flash can forge a chain.
  Tamper evidence is only as strong as the key and the *anchoring* of the chain head (`seq:tag`) somewhere the attacker cannot rewrite.
* Anchoring in TERRA is future work; the verifier already supports `--anchor seq:tag`.
* Rain-gauge calibration (mm per tip) is nominal.
* Simulation does not model oscillator start-up after power-down, supply droop, or real I²C analog behaviour.
