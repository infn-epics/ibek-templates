# keysight34465A template

Keysight 34465A Truevolt DMM (ibek support `keysight34465A`, module
`multimeter-keysight`). One `devices` entry per DMM; PVs are
`<iocprefix>:<device.name>:<PV>`.

| Key (top level or per device) | Default | |
|---|---|---|
| `server` | — | DMM IP (per-device `server` overrides) |
| `port` | 5025 | SCPI raw socket |
| `nelm` | 1000 | max readings per `READ?` |
| `scan` | `1 second` | acquisition rate |
| `rb_scan` | `5 second` | settings readback rate |
| `err_scan` | `1 second` | SCPI error poll rate |
| `acq_en` | 1 | acquisition enabled at boot |
| `boot_state` | none | state file (e.g. `STATE_1.sta`) restored at every IOC boot, overwriting the DMM setup |
| `conversion.factor` / `.offset` / `.egu` / `.prec` | 1 / 0 / "" / 4 | `CONV_* = reading * factor + offset` |
| `iocinit` (per device) | — | list of `{name, value}` dbpf after boot, e.g. `FUNC` |
