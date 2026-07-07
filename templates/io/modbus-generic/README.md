# Generic Modbus TCP Template

`modbus-generic.yaml.j2` builds an IOC for one or more Modbus TCP devices purely
from data in the IOC's `values.yaml` entry, on top of the generic entities in
`ibek-support/modbus/modbus.ibek.support.yaml` (`modbusPortConfigure`,
`modbusAsynConfigure`, `modbusFloatReadout`/`modbusFloatSetpoint`,
`modbusIntReadout`/`modbusIntSetpoint`, `modbusCoilReadout`/`modbusCoilWrite`).

No device-specific ibek support module is needed — any Modbus TCP
instrument/PLC can be described directly in `values.yaml`.

## Structure

```
connections:                # one entry per physical Modbus TCP link (asyn port)
  - name: <asyn port name>
    server: <IP>            # optional, falls back to top-level "server"
    port: <TCP port>        # optional, falls back to top-level "port" (default 502)
    slaves:                 # one entry per contiguous register/coil block
      - name: <MODBUSPORT name>       # unique
        slave_id: <unit id>           # default 1
        func_code: <1|2|3|4|5|6|15|16># default 4 (read holding registers)
        start_addr: <int>             # default 0
        length: <int>                 # bits for coil func codes, words for register func codes
        data_type: <int>              # modbus module DATA_TYPE, default 7
        poll_delay: <ms>              # default 1000
        registers:                    # PVs within this block
          - name: <suffix used to build R>
            offset: <int>             # relative to start_addr
            type: float | int | coil  # default float
            access: read | write      # default read
            desc: ...
            egu: ...
            hopr / lopr / prec: ...   # float/int only
            scan: ...                 # int/coil readouts only
            pini: ...                 # float/int writes only
            znam / onam: ...          # coil only
```

`P` is built once as `{{iocprefix}}{{":" + iocroot if iocroot is defined}}`.
Each register's PV is `P + R`, where `R` defaults to `:<name>` or can be
overridden explicitly with `r:`.

## Example

Reproduces the electrex power-meter case from
`ibek-support/modbus/README.md` (5 Modbus slaves, each with a power and an
energy register block) — see `tests/modbus-generic-electrex-test.yaml` for
the full working example.

```yaml
iocname: electrex-plc
iocprefix: DT
iocroot: kilo
template: modbus-generic
asset: "..."

connections:
  - name: electrex
    server: 192.168.146.107
    port: 502
    slaves:
      - name: ELECTREX1
        slave_id: 1
        start_addr: 277
        length: 8
        poll_delay: 5000
        registers:
          - name: ced_nd_p
            offset: 0
            type: float
            desc: "Instantaneous Power"
            egu: KW
      - name: ELECTREX12
        slave_id: 1
        start_addr: 413
        length: 8
        poll_delay: 5000
        registers:
          - name: ced_nd_ea
            offset: 0
            type: int
            desc: "Average Energy"
            egu: KWh
```

Read/write registers and coils in the same IOC by mixing `type`/`access` per
register — see `tests/modbus-generic-rw-test.yaml`.
