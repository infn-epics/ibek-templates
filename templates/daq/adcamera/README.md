# AD Camera IOC Template - Plugin Flow Documentation

## Overview

This Jinja2 template (`adcamera.yaml.j2`) generates EPICS IOC configurations for Area Detector cameras with a flexible plugin architecture. The template supports both simulated cameras and real Aravis-based GigE Vision cameras (Basler models).

## Camera Types Supported

| Camera Model | Array Elements | Notes |
|--------------|----------------|-------|
| **camerasim** | 325546 (default) | Simulated camera using ADSimDetector |
| **Basler-a2A1920** | 2354176 | 1920x1226 resolution |
| **Basler-scA1400** | 1447680 | 1392x1040 resolution |
| **Basler-a2A2600** | 5616000 | 2592x2160 resolution |
| **Other Aravis** | 325546 (default) | Generic GigE Vision cameras |

## Enable/Disable Feature Switches

The template provides five main boolean switches to control plugin chains:

| Switch | Type | Description | Default |
|--------|------|-------------|---------|
| `stream_enable` | Boolean | Enable HTTP streaming via ffmpegServer | User-defined |
| `roi_enable` | Boolean | Enable Region of Interest processing | User-defined |
| `proc_enable` | Boolean | Enable NDProcess plugin for image processing | User-defined |
| `overlay_enable` | Boolean | Enable NDOverlay for graphics overlay | User-defined |
| `stats_enable` | Boolean | Enable NDStats plugins at various points | User-defined |
| `tiff_enable` | Boolean | Enable NDFileTIFF plugins and related startup initialization | `true` when omitted |

### Additional Configuration Switches

- `iocexit` - If true, IOC exits after initialization (for testing)
- Camera-specific parameters configurable per device:
  - `buffers` - Number of NDArray buffers (default: 100)
  - `memory` - Memory allocation (default: 0)
  - `CAMERA_TYPE` - Data type (default: "Int16")
  - `CAMERA_FTVL` - Field type value (default: "USHORT")
  - `CAMERA_STATS_XSIZE` - Stats X dimension (default: 1024)
  - `CAMERA_STATS_YSIZE` - Stats Y dimension (default: 768)

## Plugin Flow Architecture

### Base Camera Layer (Always Present)

```
┌─────────────────────────────────────┐
│  Camera Source                       │
│  - ADSimDetector (camerasim)        │
│  - ADAravis.aravisCamera (hardware) │
│  PORT: {camera.name}                │
└──────────────┬──────────────────────┘
               │
               ├─► NDPvxsPlugin (PVA)      ─► PVA:OUTPUT
               ├─► NDStdArrays (NTD)       ─► :image1:
               ├─► NDFileTIFF (TIFF)       ─► :TIFF1:
               └─► NDStats (STATS) [if stats_enable]
```

**Always Created Plugins:**
- **NDPvxsPlugin** (PORT: `.PVA`) - PVAccess server publishing camera output
- **NDStdArrays** (PORT: `.NTD`) - Standard array plugin for waveform PVs

**Conditionally Created:**
- **NDFileTIFF** (PORT: `.TIFF`) - TIFF file writer [requires `tiff_enable=true` or omitted]
- **NDStats** (PORT: `.STATS`) - Statistics computation [requires `stats_enable=true`]

---

### Stream Plugin Chain (stream_enable)

```
overlay_enable=false:
Camera ──┬─► ffmpegServer.ffmpegStream
         │   PORT: {camera.name}.STREAM
         │   HTTP_PORT: 8082
         │   R: :Stream1:

overlay_enable=true:
Camera ──┬─► NDOverlay (OVERLAY1) ──► ffmpegServer.ffmpegStream
         │                          PORT: {camera.name}.STREAM
         │                          HTTP_PORT: 8082
         │                          R: :Stream1:
```

**When `stream_enable=true`:**
- **ffmpegServer.ffmpegStream** - HTTP video streaming server
  - Publishes H.264 encoded video stream
  - Accessible via HTTP on port 8082
  - Max resolution: 1600x1200 (configurable)
  - JPEG quality: 100
  - Uses `NDARRAY_PORT={camera.name}.OVERLAY1` when `overlay_enable=true`, otherwise the raw camera port

---

### ROI Plugin Chain (roi_enable)

```
Camera ──┬─► NDROI (ROI1)
         │   PORT: {camera.name}.ROI1
         │   ├─► NDPvxsPlugin (PVA3)    ─► :Roi1:Pva1: → ROI1:OUTPUT
         │   ├─► NDFileTIFF (TIFFREAD)  ─► :Proc1:TIFF: [Note: PV name mismatch]
         │   ├─► NDFileTIFF (TIFF3)     ─► :Roi1:TIFF1:
         │   └─► NDStats (STATS3) [if stats_enable] ─► :Roi1:Stats1:
         │
         └─► [feeds into PROC if proc_enable]
```

**When `roi_enable=true`:**
- **NDROI** (PORT: `.ROI1`) - Region of interest selector
- **NDPvxsPlugin** (PORT: `.PVA3`) - PVAccess for ROI output
- **NDFileTIFF** (PORT: `.TIFFREAD`) - TIFF reader/writer [requires `tiff_enable=true` or omitted] ⚠️ PV prefix shows `:Proc1:TIFF:` (appears to be naming inconsistency)
- **NDFileTIFF** (PORT: `.TIFF3`) - TIFF writer for ROI [requires `tiff_enable=true` or omitted]
- **NDStats** (PORT: `.STATS3`) - Statistics on ROI [requires `stats_enable=true`]

**File Output Pattern:** `{filename}_roi_{number}.tiff`

---

### Process Plugin Chain (proc_enable)

When ROI is disabled, `NDProcess` now reads directly from the base camera port.

```
ROI1 or Camera ──► NDProcess (PROC)
         PORT: {camera.name}.PROC
         ├─► NDPvxsPlugin (PVA2)    ─► :Proc1:Pva1: → PROC:OUTPUT
         ├─► NDStdArrays (NTD2)     ─► :image2:
         ├─► NDFileTIFF (TIFF2)     ─► :Proc1:TIFF1:
         └─► NDStats (STATS2) [if stats_enable] ─► :Proc1:Stats1:
```

**When `proc_enable=true`:**
- **NDProcess** (PORT: `.PROC`) - Image processing (filters, transforms, arithmetic)
  - Input source: `{camera.name}.ROI1` when `roi_enable=true`, otherwise `{camera.name}`
- **NDPvxsPlugin** (PORT: `.PVA2`) - PVAccess for processed output
- **NDStdArrays** (PORT: `.NTD2`) - Array plugin for processed images
- **NDFileTIFF** (PORT: `.TIFF2`) - TIFF writer for processed images [requires `tiff_enable=true` or omitted]
- **NDStats** (PORT: `.STATS2`) - Statistics on processed images [requires `stats_enable=true`]

**File Output Pattern:** `{camera.name}_proc_{number}.tiff`

---

### Overlay Plugin Chain (overlay_enable)

```
Camera ──► NDOverlay (OVERLAY1)
           PORT: {camera.name}.OVERLAY1
           NOverlays: 8
           ├─► NDPvxsPlugin (PVA4)  ─► :Overlay1:Pva1: → OVERLAY1:OUTPUT
           └─► NDFileTIFF (TIFF4)   ─► :Overlay1:TIFF1:
```

**When `overlay_enable=true`:**
- **NDOverlay** (PORT: `.OVERLAY1`) - Graphics overlay (8 overlay objects)
  - Supports shapes: cross, rectangle, etc.
  - Configurable RGB colors
  - Default: White cross (255,255,255), 2x2 pixels
- **NDPvxsPlugin** (PORT: `.PVA4`) - PVAccess for overlay output
- **NDFileTIFF** (PORT: `.TIFF4`) - TIFF writer for overlay images [requires `tiff_enable=true` or omitted]

**File Output Pattern:** `{camera.name}_overlay_{number}.tiff`

---

## Complete Plugin Flow Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                         CAMERA SOURCE                             │
│              (ADSimDetector or ADAravis.aravisCamera)             │
│                     PORT: {camera.name}                           │
└───┬──────────────────────────────────────────────────────────────┘
    │
    ├─[stream_enable]──► ffmpegServer (.STREAM) ──► HTTP:8082
    │
    ├─[overlay_enable]─► NDOverlay (.OVERLAY1)
    │                    ├─► NDPvxsPlugin (.PVA4) ──► OVERLAY1:OUTPUT
    │                    └─► NDFileTIFF (.TIFF4)   ──► overlay TIFFs
    │
    ├─[roi_enable]─────► NDROI (.ROI1)
    │                    ├─► NDPvxsPlugin (.PVA3) ──► ROI1:OUTPUT
    │                    ├─[tiff_enable]─► NDFileTIFF (.TIFFREAD)
    │                    ├─[tiff_enable]─► NDFileTIFF (.TIFF3)  ──► roi TIFFs
    │                    ├─[stats_enable]─► NDStats (.STATS3)
    │                    │
    │                    └─[proc_enable]─► NDProcess (.PROC)
    │                                       ├─► NDPvxsPlugin (.PVA2) ──► PROC:OUTPUT
    │                                       ├─► NDStdArrays (.NTD2)
    │                                       ├─[tiff_enable]─► NDFileTIFF (.TIFF2)  ──► proc TIFFs
    │                                       └─[stats_enable]─► NDStats (.STATS2)
    │
    ├─► NDPvxsPlugin (.PVA) ────────────► PVA:OUTPUT (always)
    ├─► NDStdArrays (.NTD) ─────────────► :image1: (always)
    ├─[tiff_enable]─► NDFileTIFF (.TIFF) ─► base TIFFs
    └─[stats_enable]─► NDStats (.STATS) ─► :Stats1:
```

## Port Naming Convention

| Plugin Type | Port Name Pattern | Example |
|-------------|-------------------|---------|
| Camera | `{camera.name}` | `CAM1` |
| Stream | `{camera.name}.STREAM` | `CAM1.STREAM` |
| ROI | `{camera.name}.ROI1` | `CAM1.ROI1` |
| Process | `{camera.name}.PROC` | `CAM1.PROC` |
| Overlay | `{camera.name}.OVERLAY1` | `CAM1.OVERLAY1` |
| PVA (base) | `{camera.name}.PVA` | `CAM1.PVA` |
| PVA (proc) | `{camera.name}.PVA2` | `CAM1.PVA2` |
| PVA (roi) | `{camera.name}.PVA3` | `CAM1.PVA3` |
| PVA (overlay) | `{camera.name}.PVA4` | `CAM1.PVA4` |
| StdArrays (base) | `{camera.name}.NTD` | `CAM1.NTD` |
| StdArrays (proc) | `{camera.name}.NTD2` | `CAM1.NTD2` |
| Stats (base) | `{camera.name}.STATS` | `CAM1.STATS` |
| Stats (proc) | `{camera.name}.STATS2` | `CAM1.STATS2` |
| Stats (roi) | `{camera.name}.STATS3` | `CAM1.STATS3` |
| TIFF (base) | `{camera.name}.TIFF` | `CAM1.TIFF` |
| TIFF (proc) | `{camera.name}.TIFF2` | `CAM1.TIFF2` |
| TIFF (roi) | `{camera.name}.TIFF3` | `CAM1.TIFF3` |
| TIFF (overlay) | `{camera.name}.TIFF4` | `CAM1.TIFF4` |
| TIFF (roi read) | `{camera.name}.TIFFREAD` | `CAM1.TIFFREAD` |

## PV Naming Convention

### Base Camera PVs

- `{iocprefix}:{camera.name}:` - Camera parameters
- `{iocprefix}:{camera.name}:image1:` - NDStdArrays waveform
- `{iocprefix}:{camera.name}:TIFF1:` - Base TIFF writer
- `{iocprefix}:{camera.name}:Stats1:` - Base statistics (if stats_enable)
- `{iocprefix}:{camera.name}:Pva1:` - Base PVA plugin

### Stream PVs (stream_enable)

- `{iocprefix}:{camera.name}:Stream1:` - ffmpeg stream controls
- Access URL: `http://<host>:8082`

### ROI PVs (roi_enable)

- `{iocprefix}:{camera.name}:Roi1:` - ROI controls (position, size)
- `{iocprefix}:{camera.name}:Roi1:TIFF1:` - ROI TIFF writer
- `{iocprefix}:{camera.name}:Roi1:Stats1:` - ROI statistics (if stats_enable)
- `{iocprefix}:{camera.name}:Roi1:Pva1:` - ROI PVA plugin

### Process PVs (proc_enable)

- `{iocprefix}:{camera.name}:Proc1:` - Process controls (filters, operations)
- `{iocprefix}:{camera.name}:image2:` - Processed image waveform
- `{iocprefix}:{camera.name}:Proc1:TIFF1:` - Processed TIFF writer
- `{iocprefix}:{camera.name}:Proc1:Stats1:` - Process statistics (if stats_enable)
- `{iocprefix}:{camera.name}:Proc1:Pva1:` - Process PVA plugin
- `{iocprefix}:{camera.name}:Proc1:TIFF:` - Additional TIFF writer (from TIFFREAD port)

### Overlay PVs (overlay_enable)

- `{iocprefix}:{camera.name}:Overlay1:` - Overlay controls (8 overlays)
- `{iocprefix}:{camera.name}:Overlay1:TIFF1:` - Overlay TIFF writer
- `{iocprefix}:{camera.name}:Overlay1:Pva1:` - Overlay PVA plugin

## File Output Configuration

All TIFF writers are initialized with post-startup commands:

| Writer | File Path | File Pattern | Mode |
|--------|-----------|--------------|------|
| Base TIFF | `{data_dir}` | `camera_{number}.tiff` | Stream (2) |
| ROI TIFF | `{data_dir}` | `{camera.name}_roi_{number}.tiff` | Stream (2) |
| Process TIFF | `{data_dir}` | `{camera.name}_proc_{number}.tiff` | Stream (2) |
| Overlay TIFF | `{data_dir}` | `{camera.name}_overlay_{number}.tiff` | Stream (2) |

**FileWriteMode = 2**: Stream mode (continuous writing to disk)

## Dependency Matrix

| Feature | Depends On | Notes |
|---------|------------|-------|
| `stream_enable` | None | Independent |
| `roi_enable` | None | Independent |
| `overlay_enable` | None | Independent |
| `proc_enable` | None | Uses ROI output when available, otherwise raw camera |
| `stats_enable` | None | Adds stats at multiple points |

## Configuration Examples

### Minimal Configuration (Base Camera Only)

```yaml
stream_enable: false
roi_enable: false
proc_enable: false
overlay_enable: false
stats_enable: false
tiff_enable: false
```

**Result:** Camera + PVA + StdArrays only

### Full Processing Pipeline

```yaml
stream_enable: true    # Enable HTTP streaming
roi_enable: true       # Enable ROI
proc_enable: true      # Enable image processing (depends on ROI)
overlay_enable: true   # Enable graphics overlay
stats_enable: true     # Enable statistics at all levels
tiff_enable: true      # Enable TIFF writers
```

**Result:** All plugins active, maximum flexibility

### ROI + Statistics Only

```yaml
stream_enable: false
roi_enable: true
proc_enable: false
overlay_enable: false
stats_enable: true
tiff_enable: true
```

**Result:** Camera + ROI with stats + Base stats + TIFF outputs

### Streaming + Overlay

```yaml
stream_enable: true
roi_enable: false
proc_enable: false     # Cannot enable without ROI
overlay_enable: true
stats_enable: false
```

**Result:** Camera + HTTP stream + Overlay graphics + Base outputs

---

### ELI Profile A: ROI + Proc + Stats + Overlay + TIFF

This matches:

```yaml
- name: "overlay_enable"
  value: true
- name: "stats_enable"
  value: true
- name: "proc_enable"
  value: true
- name: "tiff_enable"
  value: true
- name: "roi_enable"
  value: true
```

**Resulting plugin layout (adcamera)**:
- Base path: Camera -> PVA + StdArrays + TIFF + STATS
- ROI path: Camera -> ROI -> ROI PVA + ROI TIFF + ROI STATS
- Proc path: ROI -> PROC -> Proc PVA + Proc StdArrays + Proc TIFF + Proc STATS
- Overlay path: Camera -> OVERLAY -> Overlay PVA + Overlay TIFF
- Stream path (only if enabled): from OVERLAY when `overlay_enable=true`, otherwise from Camera

**Important behavior**:
- In this template, Overlay is not chained after Proc/Stats; it reads directly from Camera
- Stats plugins are created at multiple points (base, ROI, Proc)
- TIFF outputs are produced from multiple branches (base, ROI, Proc, Overlay)

---

### ELI Profile B: Stats + TIFF on Raw Camera

This matches:

```yaml
- name: "overlay_enable"
  value: false
- name: "stats_enable"
  value: true
- name: "proc_enable"
  value: false
- name: "tiff_enable"
  value: true
- name: "roi_enable"
  value: false
```

**Resulting plugin layout (adcamera)**:
- Base path only: Camera -> PVA + StdArrays + TIFF + STATS

**Important behavior**:
- No ROI, Proc, or Overlay plugins are created
- One Stats plugin is active at base camera level (`:Stats1:`)
- Main TIFF writer is base `:TIFF1:` and writes raw camera frames

## Common Issues and Notes

### 1. ⚠️ PV Naming Inconsistency

The TIFFREAD plugin under ROI has inconsistent PV naming:
- Port: `{camera.name}.TIFFREAD`
- PV Prefix: `{iocprefix}:{camera.name}:Proc1:TIFF:` (should be `:Roi1:TIFF:`)

This appears to be a template bug or intentional reuse of the Proc1 namespace.

### 2. proc_enable Dependency

Always ensure `roi_enable=true` when using `proc_enable=true`. The template does not validate this dependency automatically.

### 3. stats_enable Global Impact

When `stats_enable=true`, statistics plugins are added at:
- Base camera level (`.STATS`)
- ROI level (`.STATS3`) if `roi_enable=true`
- Process level (`.STATS2`) if `proc_enable=true`

### 4. Memory Configuration

Use `EPICS_CA_MAX_ARRAY_BYTES=10000000` (set via `EpicsCaMaxArrayBytes` entity) to handle large image arrays.

### 5. Multiple Camera Support

The template loops over `devices`, so multiple cameras can be configured in one IOC with independent plugin chains per camera.

## Initialization Flow

1. **Camera Creation** - ADSimDetector or ARAravis camera
2. **Plugin Chain Creation** - Based on enable switches
3. **Database Loading** - Load `infncam.db` for each camera
4. **Alias Setup** - Create PV aliases if defined
5. **Post-Startup Initialization**:
   - Configure TIFF file paths and naming
   - Initialize overlay settings (if enabled)
   - Apply common `iocinit` parameters to all cameras
   - Apply camera-specific `iocinit` parameters
   - Generate PV list or exit (based on `iocexit`)

## Template Variables Reference

### Required Variables

- `iocname` - IOC instance name
- `iocprefix` - PV prefix for all records
- `devices` - List of camera configurations
  - `devices[].name` - Camera identifier/port name
  - `devices[].devtype` - Camera type (camerasim, Basler-*, etc.)
  - `devices[].id` - Camera ID (for Aravis cameras)
- `data_dir` - Directory path for TIFF file output
- `config_dir` - Directory for configuration files (PV list)

### Optional Variables

- `stream_enable` - Enable streaming (default: false)
- `roi_enable` - Enable ROI (default: false)
- `proc_enable` - Enable processing (default: false)
- `overlay_enable` - Enable overlay (default: false)
- `stats_enable` - Enable statistics (default: false)
- `iocexit` - Exit after init (default: false)
- `iocinit` - Common initialization parameters (list)
- `iocroot` - Root name for aliases

### Per-Camera Variables

- `camera.buffers` - Buffer count (default: 100)
- `camera.memory` - Memory allocation (default: 0)
- `camera.pixels` - Override pixel count
- `camera.CAMERA_TYPE` - Data type (default: "Int16")
- `camera.CAMERA_FTVL` - Field type (default: "USHORT")
- `camera.CAMERA_STATS_XSIZE` - Stats width (default: 1024)
- `camera.CAMERA_STATS_YSIZE` - Stats height (default: 768)
- `camera.iocinit` - Camera-specific init parameters (list)
- `camera.aliases` - List of PV aliases (name/alias pairs)

## Best Practices

1. **Start Simple**: Begin with base camera only, then add plugins incrementally
2. **Test Processing Paths**: Verify whether `Proc1` should consume ROI output or the raw camera in your beamline configuration
3. **Monitor Memory**: Large arrays require proper `EPICS_CA_MAX_ARRAY_BYTES` configuration
4. **Use Stats Wisely**: Enable statistics only where needed to reduce CPU load
5. **File Management**: Ensure `data_dir` has sufficient space for TIFF files
6. **Network Bandwidth**: Consider streaming impact on network when using `stream_enable`

## Support and EPICS Modules

This template requires the following EPICS support modules:

- **ADCore** - Area Detector core functionality
- **ADSimDetector** - Simulated camera driver (for camerasim)
- **ADAravis** - Aravis GigE Vision driver (for hardware cameras)
- **ADGenICam** - GenICam support
- **ffmpegServer** - HTTP streaming (if `stream_enable=true`)
- **asyn** - Asynchronous driver support
- **pvxs** - PVAccess server

---

*Generated from template: `adcamera.yaml.j2`*  
*Template version: v1.0.2b4*
