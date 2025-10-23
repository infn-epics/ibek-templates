# PPT Modulator Template

This directory contains the Jinja2 template and JSON schema for creating IOC configurations for PPT (Pulse Power Technology) Modulators.

## Files

- **ppt.yaml.j2**: Jinja2 template for generating IBEK IOC YAML configurations
- **ppt.schema.json**: JSON Schema for validating PPT Modulator configuration files

## Usage

Create a YAML configuration file that follows the schema. Example with single modulator:

```yaml
iocname: "ppt-modulator-single"
asset: "https://confluence.infn.it/display/LDCG/PPT-MOD-001"
iocprefix: "PPT"
template: "ppt"
devtype: "ppt"
devgroup: "modulator"

modulators:
  - name: "PPT_MOD_001"
    iocroot: "MOD1"
    server: "192.168.197.111"
    port: 2000
```

Example with multiple modulators:

```yaml
iocname: "ppt-modulator-multi"
iocprefix: "PPT"
template: "ppt"

modulators:
  - name: "PPT_MOD_001"
    iocroot: "MOD1"
    server: "192.168.197.111"
    port: 2000
  - name: "PPT_MOD_002"
    iocroot: "MOD2"
    server: "192.168.197.112"
    port: 2000
```

## Configuration Parameters

### Required Parameters

- **iocname**: Name of the IOC instance
- **iocprefix**: PV prefix for all records (e.g., "PPT")
- **template**: Must be "ppt"
- **modulators**: Array of modulator configurations (at least one required)
  - **name**: Controller/Asyn port name (optional, defaults to "PPT_MOD_1", "PPT_MOD_2", etc.)
  - **server**: IP address of the modulator (can use global default)
  - **port**: TCP port (can use global default, default: 2000)
  - **iocroot**: PV root suffix for this modulator (optional, auto-generated if not specified)

### Optional Parameters

- **server**: Global default IP address for all modulators
- **port**: Global default TCP port for all modulators (default: 2000)
- **priority**: Global default Asyn priority (default: 0)
- **noautoconnect**: Global disable auto-connect (0=auto, 1=no auto, default: 0)
- **noprocesseos**: Global disable EOS processing (0=process, 1=no process, default: 0)
- **aliases**: Array of global PV name aliases
- **iocinit**: Array of global initialization parameters to set at IOC startup
- **iocexit**: Exit IOC after initialization for testing (default: false)

### Per-Modulator Optional Parameters

Each modulator in the array can have:
- **priority**: Override global Asyn priority
- **noautoconnect**: Override global auto-connect setting
- **noprocesseos**: Override global EOS processing setting
- **aliases**: Modulator-specific PV aliases
- **iocinit**: Modulator-specific initialization parameters
- **asset**: Asset documentation link for this modulator

## Template Features

The template generates an IBEK configuration that:

1. Creates `ppt.PptModulatorController` entities for each modulator in the array
2. Loads both monitoring (`ppt.template`) and control (`ppt_control.template`) databases for each
3. Supports global and per-modulator configuration
4. Supports PV aliasing at global and per-modulator level
5. Allows initialization of PV values at IOC startup (global and per-modulator)
6. Auto-generates unique PV roots when multiple modulators are configured

## Example with Aliases and Initialization

```yaml
iocname: "ppt-mod-sparc"
asset: "https://confluence.infn.it/display/SPARC/PPT-SYSTEM"
iocprefix: "SPARC:MOD"
template: "ppt"
devtype: "ppt"
devgroup: "modulator"

# Global defaults
server: "10.0.1.100"
port: 2000

modulators:
  - name: "PPT_RF1"
    iocroot: "RF1"
    server: "10.0.1.101"
    aliases:
      - name: "HeaterVoltage1"
        alias: "SPARC:RF1:HV1"
      - name: "KlystronVoltage"
        alias: "SPARC:RF1:KLYS_V"
    iocinit:
      - name: "HVPS:VoltageSet"
        value: "0.0"
  
  - name: "PPT_RF2"
    iocroot: "RF2"
    server: "10.0.1.102"
    aliases:
      - name: "HeaterVoltage1"
        alias: "SPARC:RF2:HV1"
    iocinit:
      - name: "HVPS:VoltageSet"
        value: "0.0"

# Global initialization
iocinit:
  - name: "RF1:HeaterVoltage1.SCAN"
    value: ".5 second"
```

## Generated PV Structure

The template will generate PVs with the following structure for each modulator:

**Monitoring PVs** (from ppt.template):
- `{iocprefix}:{modulator.iocroot}:HeaterVoltage1` - Heater Voltage 1
- `{iocprefix}:{modulator.iocroot}:KlystronVoltage` - Klystron Voltage
- `{iocprefix}:{modulator.iocroot}:TotalCurrent` - Total Current
- ... and many more monitoring values

**Control PVs** (from ppt_control.template):
- `{iocprefix}:{modulator.iocroot}:HVPS:VoltageSet` - HVPS Voltage Setpoint
- `{iocprefix}:{modulator.iocroot}:Thy:OnCmd` - Thyratron Heater ON Command
- `{iocprefix}:{modulator.iocroot}:Klys:On80Cmd` - Klystron 80% ON Command
- ... and other control commands

### Multiple Modulators Example

For a configuration with two modulators:
- Modulator 1: `PPT:MOD1:HeaterVoltage1`, `PPT:MOD1:KlystronVoltage`, etc.
- Modulator 2: `PPT:MOD2:HeaterVoltage1`, `PPT:MOD2:KlystronVoltage`, etc.

## See Also

- [PPT Modulator IBEK Support](../../../ibek-support-infn/ppt-modulator/)
- [PPT Modulator Repository](https://github.com/infn-epics/ppt-modulator)
- Test configuration: `tests/ppt-modulator-test.yaml`
