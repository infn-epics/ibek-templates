# AD Camera IOC Template v2 - Progressive Plugin Chain

## Overview

This is an improved version of the AD Camera template that implements a **linear, progressive plugin chain** architecture. Unlike the original parallel design, this template chains plugins sequentially so that each optional plugin processes data from the previous one.

## Key Design Principles

1. **Sequential Processing**: `Raw → ROI → Proc → Stats → Overlay`
2. **Optional Plugins**: Every plugin except the camera source is optional
3. **Intelligent Chaining**: Each plugin automatically connects to the last enabled plugin
4. **TIFF1 Follows Chain**: The main `TIFF1` writer always connects to the final active plugin
5. **PVA at Every Stage**: Each enabled plugin gets its own PVAccess publisher
6. **Per-Plugin TIFF**: When `tiff_enable=true`, each plugin gets its own TIFF writer

## Plugin Flow Architecture

### Progressive Chain Logic

```
Camera (Raw) ──┬─► [if roi_enable] ──► ROI ──┬─► [if proc_enable] ──► Proc ──┬─► [if stats_enable] ──► Stats ──┬─► [if overlay_enable] ──► Overlay ──┬─► TIFF1
               │                              │                                │                                  │                                        │
               │                              │                                │                                  │                                        └─► (final output)
               │                              │                                │                                  └─► [if tiff_enable] ──► Stats.TIFF
               │                              │                                └─► [if tiff_enable] ──► Proc.TIFF
               │                              └─► [if tiff_enable] ──► ROI.TIFF
               │
               ├─► PVA (always)
               ├─► StdArrays (always)
               └─► [if stream_enable] ──► ffmpegStream (parallel, not in chain)
```

### Data Flow Examples

#### Example 1: All Plugins Enabled
```
Camera → ROI → Proc → Stats → Overlay → TIFF1
   ├─► PVA        ├─► PVA   ├─► PVA    ├─► PVA      ├─► PVA
   ├─► StdArrays  ├─► TIFF  ├─► TIFF   ├─► TIFF     ├─► TIFF
   └─► Stream     └─► StdArrays
```

**TIFF1 Source**: Overlay (final plugin)

#### Example 2: ROI + Proc Only
```
Camera → ROI → Proc → TIFF1
   ├─► PVA    ├─► PVA   ├─► PVA
   ├─► StdArrays  ├─► TIFF  ├─► TIFF
   └─► Stream                └─► StdArrays
```

**TIFF1 Source**: Proc (last enabled plugin)

#### Example 3: Stats Only
```
Camera → Stats → TIFF1
   ├─► PVA      ├─► PVA
   ├─► StdArrays  └─► TIFF
   └─► Stream
```

**TIFF1 Source**: Stats (last enabled plugin)

#### Example 4: No Optional Plugins
```
Camera → TIFF1
   ├─► PVA
   ├─► StdArrays
   └─► Stream
```

**TIFF1 Source**: Camera (raw)

## Enable/Disable Switches

| Switch | Type | Description | Default | In Chain? |
|--------|------|-------------|---------|-----------|
| `roi_enable` | Boolean | Enable Region of Interest | false | Yes (1st) |
| `proc_enable` | Boolean | Enable NDProcess (image processing) | false | Yes (2nd) |
| `stats_enable` | Boolean | Enable NDStats (statistics) | false | Yes (3rd) |
| `overlay_enable` | Boolean | Enable NDOverlay (graphics) | false | Yes (4th) |
| `tiff_enable` | Boolean | Enable per-plugin TIFF writers | false | Parallel |
| `stream_enable` | Boolean | Enable HTTP streaming | false | Parallel |

### Switch Dependencies

**No Dependencies!** Unlike the original template:
- `proc_enable` no longer requires `roi_enable`
- Each plugin independently connects to whatever came before it
- Plugins can be enabled in any combination

## Port Naming Convention

| Plugin | Port Name | NDARRAY_PORT | PV Prefix |
|--------|-----------|--------------|-----------|
| Camera | `{camera.name}` | - | `:` |
| Camera PVA | `{camera.name}.PVA` | `{camera.name}` | `:Pva1:` |
| Camera StdArrays | `{camera.name}.NTD` | `{camera.name}` | `:image1:` |
| Stream | `{camera.name}.STREAM` | `{camera.name}` | `:Stream1:` |
| ROI | `{camera.name}.ROI1` | `{current_source}` | `:Roi1:` |
| ROI PVA | `{camera.name}.ROI.PVA` | `{camera.name}.ROI1` | `:Roi1:Pva1:` |
| ROI TIFF | `{camera.name}.ROI.TIFF` | `{camera.name}.ROI1` | `:Roi1:TIFF:` |
| Proc | `{camera.name}.PROC` | `{current_source}` | `:Proc1:` |
| Proc PVA | `{camera.name}.PROC.PVA` | `{camera.name}.PROC` | `:Proc1:Pva1:` |
| Proc StdArrays | `{camera.name}.PROC.ARR` | `{camera.name}.PROC` | `:Proc1:Image:` |
| Proc TIFF | `{camera.name}.PROC.TIFF` | `{camera.name}.PROC` | `:Proc1:TIFF:` |
| Stats | `{camera.name}.STATS` | `{current_source}` | `:Stats1:` |
| Stats PVA | `{camera.name}.STATS.PVA` | `{camera.name}.STATS` | `:Stats1:Pva1:` |
| Stats TIFF | `{camera.name}.STATS.TIFF` | `{camera.name}.STATS` | `:Stats1:TIFF:` |
| Overlay | `{camera.name}.OVERLAY1` | `{current_source}` | `:Overlay1:` |
| Overlay PVA | `{camera.name}.OVERLAY.PVA` | `{camera.name}.OVERLAY1` | `:Overlay1:Pva1:` |
| Overlay TIFF | `{camera.name}.OVERLAY.TIFF` | `{camera.name}.OVERLAY1` | `:Overlay1:TIFF:` |
| **TIFF1** (main) | `{camera.name}.TIFF1` | `{current_source}` | `:TIFF1:` |

**Note**: `{current_source}` is dynamically determined based on which plugins are enabled.

## PVAccess (PVA) Publishers

Each enabled plugin publishes data via PVAccess:

| Plugin | PVA Port | Published Name | PV Prefix |
|--------|----------|----------------|-----------|
| Camera | `.PVA` | `{iocprefix}:{camera.name}:PVA:OUTPUT` | `:Pva1:` |
| ROI | `.ROI.PVA` | `{iocprefix}:{camera.name}:ROI1:OUTPUT` | `:Roi1:Pva1:` |
| Proc | `.PROC.PVA` | `{iocprefix}:{camera.name}:PROC:OUTPUT` | `:Proc1:Pva1:` |
| Stats | `.STATS.PVA` | `{iocprefix}:{camera.name}:STATS:OUTPUT` | `:Stats1:Pva1:` |
| Overlay | `.OVERLAY.PVA` | `{iocprefix}:{camera.name}:OVERLAY1:OUTPUT` | `:Overlay1:Pva1:` |

## TIFF File Writers

### TIFF1 - Main Writer (Always Present)

- **Port**: `{camera.name}.TIFF1`
- **Source**: Automatically connects to the **last active plugin** in the chain
- **PV Prefix**: `{iocprefix}:{camera.name}:TIFF1:`
- **File Pattern**: `{camera.name}_{number}.tiff`

### Per-Plugin TIFF Writers (tiff_enable=true)

When `tiff_enable=true`, each enabled plugin gets its own TIFF writer:

| Plugin | Port | PV Prefix | File Pattern | Created When |
|--------|------|-----------|--------------|--------------|
| ROI | `.ROI.TIFF` | `:Roi1:TIFF:` | `{camera.name}_roi_{number}.tiff` | `roi_enable=true` |
| Proc | `.PROC.TIFF` | `:Proc1:TIFF:` | `{camera.name}_proc_{number}.tiff` | `proc_enable=true` |
| Stats | `.STATS.TIFF` | `:Stats1:TIFF:` | `{camera.name}_stats_{number}.tiff` | `stats_enable=true` |
| Overlay | `.OVERLAY.TIFF` | `:Overlay1:TIFF:` | `{camera.name}_overlay_{number}.tiff` | `overlay_enable=true` |

**All TIFF writers use FileWriteMode=2** (Stream mode)

## Configuration Matrix

| roi | proc | stats | overlay | Data Flow | TIFF1 Source |
|-----|------|-------|---------|-----------|--------------|
| ❌ | ❌ | ❌ | ❌ | Camera | Camera |
| ✅ | ❌ | ❌ | ❌ | Camera→ROI | ROI |
| ✅ | ✅ | ❌ | ❌ | Camera→ROI→Proc | Proc |
| ✅ | ✅ | ✅ | ❌ | Camera→ROI→Proc→Stats | Stats |
| ✅ | ✅ | ✅ | ✅ | Camera→ROI→Proc→Stats→Overlay | Overlay |
| ❌ | ✅ | ❌ | ❌ | Camera→Proc | Proc |
| ❌ | ❌ | ✅ | ❌ | Camera→Stats | Stats |
| ❌ | ❌ | ❌ | ✅ | Camera→Overlay | Overlay |
| ❌ | ✅ | ✅ | ❌ | Camera→Proc→Stats | Stats |
| ❌ | ❌ | ✅ | ✅ | Camera→Stats→Overlay | Overlay |

## Plugin Details

### 1. Camera (Raw) - Always Present

**Purpose**: Image acquisition from hardware or simulator

**Plugins Created**:
- Camera driver (ADSimDetector or ADAravis)
- NDPvxsPlugin (`.PVA`) - PVA publisher
- NDStdArrays (`.NTD`) - Waveform arrays

**PVs**:
- `{iocprefix}:{camera.name}:` - Camera controls
- `{iocprefix}:{camera.name}:image1:` - Image waveform
- `{iocprefix}:{camera.name}:Pva1:` - PVA plugin controls

### 2. ROI Plugin (roi_enable)

**Purpose**: Select region of interest, reduce image size

**Input**: Previous plugin output (`{current_source}`)

**Plugins Created**:
- NDROI (`.ROI1`) - ROI selector
- NDPvxsPlugin (`.ROI.PVA`) - PVA publisher
- NDFileTIFF (`.ROI.TIFF`) - TIFF writer [if `tiff_enable`]

**PVs**:
- `{iocprefix}:{camera.name}:Roi1:` - ROI position/size controls
- `{iocprefix}:{camera.name}:Roi1:Pva1:` - PVA controls
- `{iocprefix}:{camera.name}:Roi1:TIFF:` - TIFF writer [if enabled]

### 3. Process Plugin (proc_enable)

**Purpose**: Image processing (filters, transforms, arithmetic)

**Input**: Previous plugin output (`{current_source}`)

**Plugins Created**:
- NDProcess (`.PROC`) - Image processor
- NDPvxsPlugin (`.PROC.PVA`) - PVA publisher
- NDStdArrays (`.PROC.ARR`) - Processed image waveform
- NDFileTIFF (`.PROC.TIFF`) - TIFF writer [if `tiff_enable`]

**PVs**:
- `{iocprefix}:{camera.name}:Proc1:` - Processing controls
- `{iocprefix}:{camera.name}:Proc1:Image:` - Processed image waveform
- `{iocprefix}:{camera.name}:Proc1:Pva1:` - PVA controls
- `{iocprefix}:{camera.name}:Proc1:TIFF:` - TIFF writer [if enabled]

### 4. Stats Plugin (stats_enable)

**Purpose**: Compute image statistics (min, max, mean, sigma, centroid, etc.)

**Input**: Previous plugin output (`{current_source}`)

**Plugins Created**:
- NDStats (`.STATS`) - Statistics calculator
- NDPvxsPlugin (`.STATS.PVA`) - PVA publisher
- NDFileTIFF (`.STATS.TIFF`) - TIFF writer [if `tiff_enable`]

**PVs**:
- `{iocprefix}:{camera.name}:Stats1:` - Statistics values
- `{iocprefix}:{camera.name}:Stats1:Pva1:` - PVA controls
- `{iocprefix}:{camera.name}:Stats1:TIFF:` - TIFF writer [if enabled]

### 5. Overlay Plugin (overlay_enable)

**Purpose**: Add graphics overlays (crosshairs, boxes, text) to images

**Input**: Previous plugin output (`{current_source}`)

**Plugins Created**:
- NDOverlay (`.OVERLAY1`) - Graphics overlay (8 objects)
- NDPvxsPlugin (`.OVERLAY.PVA`) - PVA publisher
- NDFileTIFF (`.OVERLAY.TIFF`) - TIFF writer [if `tiff_enable`]

**PVs**:
- `{iocprefix}:{camera.name}:Overlay1:` - Overlay controls
- `{iocprefix}:{camera.name}:Overlay1:Pva1:` - PVA controls
- `{iocprefix}:{camera.name}:Overlay1:TIFF:` - TIFF writer [if enabled]

**Default Overlay Settings**:
- Shape: Cross (1)
- Color: White (255, 255, 255)
- Size: 2x2 pixels

### 6. Stream Plugin (stream_enable) - Parallel

**Purpose**: HTTP video streaming (not in processing chain)

**Input**: Always connects to camera (parallel path)

**Plugins Created**:
- ffmpegServer (`.STREAM`) - HTTP streaming server

**PVs**:
- `{iocprefix}:{camera.name}:Stream1:` - Streaming controls
- **HTTP Access**: `http://<host>:8082`

**Settings**:
- Max resolution: 1600x1200
- JPEG quality: 100
- Port: 8082

## Configuration Examples

### Minimal Configuration (Raw Camera Only)

```yaml
roi_enable: false
proc_enable: false
stats_enable: false
overlay_enable: false
tiff_enable: false
stream_enable: false
```

**Chain**: `Camera → TIFF1`
**Plugins**: Camera + PVA + StdArrays + TIFF1
**TIFF1 Source**: Camera

---

### ROI + Statistics

```yaml
roi_enable: true
proc_enable: false
stats_enable: true
overlay_enable: false
tiff_enable: true
stream_enable: false
```

**Chain**: `Camera → ROI → Stats → TIFF1`
**TIFF Writers**:
- `ROI.TIFF` (intermediate)
- `STATS.TIFF` (intermediate)
- `TIFF1` (final, from Stats)

---

### Full Processing Pipeline

```yaml
roi_enable: true
proc_enable: true
stats_enable: true
overlay_enable: true
tiff_enable: true
stream_enable: true
```

**Chain**: `Camera → ROI → Proc → Stats → Overlay → TIFF1`
**Parallel**: `Camera → Stream` (HTTP)
**TIFF Writers**:
- `ROI.TIFF`
- `PROC.TIFF`
- `STATS.TIFF`
- `OVERLAY.TIFF`
- `TIFF1` (final, from Overlay)

**PVA Publishers**: Camera, ROI, Proc, Stats, Overlay (5 total)

---

### Processing Without ROI

```yaml
roi_enable: false
proc_enable: true
stats_enable: true
overlay_enable: false
tiff_enable: false
stream_enable: false
```

**Chain**: `Camera → Proc → Stats → TIFF1`
**TIFF1 Source**: Stats
**Note**: ✅ Works! Unlike original template, proc_enable doesn't require roi_enable

---

### Overlay Only (Visual Feedback)

```yaml
roi_enable: false
proc_enable: false
stats_enable: false
overlay_enable: true
tiff_enable: false
stream_enable: true
```

**Chain**: `Camera → Overlay → TIFF1`
**Parallel**: `Camera → Stream`
**Use Case**: Add visual markers/crosshairs with live streaming

## Template Variables

### Required Variables

| Variable | Type | Description |
|----------|------|-------------|
| `iocname` | string | IOC instance name |
| `iocprefix` | string | PV prefix for all records |
| `devices` | list | Camera configurations |
| `devices[].name` | string | Camera identifier/port name |
| `devices[].devtype` | string | Camera type (camerasim, Basler-*) |
| `devices[].id` | string | Camera ID (Aravis cameras) |
| `data_dir` | string | TIFF file output directory |
| `config_dir` | string | Configuration file directory |

### Optional Boolean Switches

| Variable | Default | Description |
|----------|---------|-------------|
| `roi_enable` | false | Enable ROI plugin |
| `proc_enable` | false | Enable Process plugin |
| `stats_enable` | false | Enable Stats plugin |
| `overlay_enable` | false | Enable Overlay plugin |
| `tiff_enable` | false | Enable per-plugin TIFF writers |
| `stream_enable` | false | Enable HTTP streaming |
| `iocexit` | false | Exit after init (testing) |

### Per-Camera Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `camera.buffers` | 100 | NDArray buffer count |
| `camera.memory` | 0 | Memory allocation |
| `camera.pixels` | (auto) | Override pixel count |
| `camera.CAMERA_TYPE` | "Int16" | Data type |
| `camera.CAMERA_FTVL` | "USHORT" | Field type |
| `camera.CAMERA_STATS_XSIZE` | 1024 | Stats width |
| `camera.CAMERA_STATS_YSIZE` | 768 | Stats height |
| `camera.iocinit` | [] | Specific init parameters |
| `camera.aliases` | [] | PV aliases |

## Advantages Over Original Template

### ✅ Pros

1. **No Hidden Dependencies**: `proc_enable` no longer requires `roi_enable`
2. **Logical Data Flow**: Sequential processing is intuitive
3. **Dynamic TIFF1**: Main file writer always captures final processed output
4. **Consistent PVA**: Every plugin gets PVA publisher
5. **Flexible Combinations**: Any combination of plugins works
6. **Clear Chaining**: Easy to understand data flow
7. **Simplified Naming**: Consistent port naming scheme

### ⚠️ Considerations

1. **Processing Order Fixed**: Chain is always ROI→Proc→Stats→Overlay
2. **Cannot Skip Middle**: If you want Stats from raw camera but also want Overlay, Stats will be before Overlay in chain
3. **Stream is Parallel**: ffmpeg stream always reads from camera, not from processed output

## Use Cases

### Quality Control & Alignment

```yaml
roi_enable: true        # Select beam area
stats_enable: true      # Measure centroid/intensity
overlay_enable: true    # Show crosshair at centroid
stream_enable: true     # Live view
tiff_enable: false      # No intermediate files
```

**Result**: Live stream with ROI stats and visual overlay markers

---

### Multi-Stage Processing with Archiving

```yaml
roi_enable: true        # Crop to region
proc_enable: true       # Apply filters/transforms
stats_enable: true      # Compute statistics
overlay_enable: false   # No graphics needed
tiff_enable: true       # Save all stages
stream_enable: false
```

**Result**: Save raw ROI, processed, and stats images separately, plus final combined

---

### Beam Diagnostics

```yaml
roi_enable: false       # Full frame
proc_enable: true       # Background subtraction
stats_enable: true      # Profile analysis
overlay_enable: false
tiff_enable: true
stream_enable: true
```

**Result**: Process full frame, compute profiles, save stages, stream live

## Implementation Details

### Jinja2 Variable Tracking

The template uses Jinja2 `set` statements to track the current source:

```jinja
{%- set current_source = camera.name %}  {# Start with camera #}

{%- if roi_enable %}
  {# ... create ROI plugin connected to current_source ... #}
  {%- set current_source = camera.name ~ ".ROI1" %}  {# Update source #}
{%- endif %}

{%- if proc_enable %}
  {# ... create Proc plugin connected to current_source ... #}
  {%- set current_source = camera.name ~ ".PROC" %}  {# Update source #}
{%- endif %}

{# ... TIFF1 always uses final current_source ... #}
```

This ensures plugins are automatically chained regardless of which ones are enabled.

## EPICS Support Modules Required

- **ADCore** - Area Detector core
- **ADSimDetector** - Simulated camera (for camerasim)
- **ADAravis** - Aravis GigE Vision driver (for hardware)
- **ADGenICam** - GenICam support
- **ffmpegServer** - HTTP streaming [if `stream_enable`]
- **asyn** - Asynchronous I/O
- **pvxs** - PVAccess server

## Migration from Original Template

### Key Differences

| Aspect | Original | v2 (This Template) |
|--------|----------|-------------------|
| Plugin chain | Parallel branches | Sequential chain |
| proc_enable | Requires roi_enable | Independent |
| stats_enable | Multiple Stats instances | Single Stats in chain |
| TIFF1 source | Always camera | Last active plugin |
| PVA naming | Mixed (.PVA, .PVA2, .PVA3, .PVA4) | Consistent (.PVA, .ROI.PVA, etc.) |
| Port naming | Inconsistent | Systematic |
| TIFF writers | Fixed per plugin | `tiff_enable` switch |

### Breaking Changes

1. **PV Names Changed**: Port names are different (`.ROI.PVA` vs `.PVA3`)
2. **TIFF1 Behavior**: Source is dynamic, not always camera
3. **Stats Plugin**: Only one Stats plugin instead of multiple
4. **Dependency Removal**: `proc_enable` works without `roi_enable`

---

**Template Version**: 2.0
**Based on**: adcamera v1.0.2b4
**Date**: January 2026
