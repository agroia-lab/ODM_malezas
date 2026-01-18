# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenDroneMap (ODM) is a command-line toolkit that processes aerial drone imagery into:
- Classified Point Clouds
- 3D Textured Models
- Georeferenced Orthorectified Imagery
- Georeferenced Digital Elevation Models

## INIA Integration Context

This ODM instance is **Layer 2** in the AgroIA multi-scale precision agriculture pipeline:

```
Layer 1: Satellite (WorldCereal) → Zone delineation, scouting priorities
    ↓
Layer 2: Drone Mapping (ODM) ← THIS REPOSITORY
    ↓
Layer 3: Detection (YOLO + SAHI) → Weed species ID, density estimation
    ↓
Layer 4: Advisory (AgroIA) → Treatment recommendations
```

**Primary use case**: Generate georeferenced orthomosaics from drone imagery of Chilean agricultural fields (campos) for downstream weed detection with YOLO models.

### Integration Points

- **Input**: Overlapping drone images (JPEGs with GPS EXIF) from campos
- **Output**: GeoTIFF orthomosaic at 1-5cm resolution for SAHI tiling
- **CRS**: Must preserve accurate georeferencing (typically EPSG:32719 for Chile)
- **Downstream consumer**: `/home/malezainia1/dev/INIA_DeepLearning_Ubuntu_mod_lleon` (YOLO + SAHI inference)

### Key Parameters for Agriculture Workflow

```bash
# Fast orthophoto (skip 3D reconstruction, faster for flat agricultural fields)
--fast-orthophoto

# Control resolution (cm/pixel) - balance detail vs processing time
--orthophoto-resolution 2

# Generate DSM for plant height analysis
--dsm

# For multispectral cameras (vegetation indices)
--radiometric-calibration camera
```

### Related Projects

- INIA Detection Pipeline: `/home/malezainia1/dev/INIA_DeepLearning_Ubuntu_mod_lleon`
- Integration Roadmap: `../INIA_DeepLearning_Ubuntu_mod_lleon/docs/INTEGRATION_ROADMAP.md`

## Build and Run Commands

### Docker (Recommended)
```bash
# Pull image
docker pull opendronemap/odm

# Process a dataset
docker run -ti --rm -v /path/to/datasets:/datasets opendronemap/odm --project-path /datasets project_name

# GPU support
docker run -ti --rm -v /path/to/datasets:/datasets --gpus all opendronemap/odm:gpu --project-path /datasets project_name --feature-type sift

# Build from source
docker build -t my_odm_image --no-cache .
```

### Native Linux (Ubuntu 24.04)
```bash
bash configure.sh install        # Initial install
bash configure.sh reinstall      # Update/rebuild
./run.sh /path/to/project        # Process dataset
```

### Development Environment
```bash
DATA=/path/to/datasets ./start-dev-env.sh    # Start dev container
bash configure.sh reinstall                   # Inside container: setup
./run.sh --project-path /datasets mydataset   # Inside container: run
docker stop odmdev                            # Stop container
```

## Testing

```bash
bash test.sh              # Run all tests
bash test.sh camera       # Run specific test module (test_camera.py)
```

Tests are in `tests/` directory using Python unittest framework.

## Architecture

### Pipeline Structure
ODM uses a stage-based processing pipeline defined in `stages/odm_app.py`:

```
dataset → split → merge → opensfm → [openmvs] → filterpoints → meshing → texturing → georeferencing → dem → orthophoto → report → postprocess
```

The `--fast-orthophoto` flag skips OpenMVS for faster processing.

### Key Directories

- **`opendm/`** - Core Python modules
  - `config.py` - CLI argument definitions (100+ parameters) and rerun stage mapping
  - `types.py` - Core data structures (ODM_Tree, ODM_Photo, ODM_Reconstruction, ODM_Stage)
  - `photo.py` - EXIF parsing, camera calibration, band detection
  - `osfm.py` - OpenSfM integration
  - `dem/` - DEM generation with ground rectification
  - `thermal_tools/` - Thermal image processing
  - `skyremoval/` - Sky segmentation filters

- **`stages/`** - Pipeline stage implementations (each inherits from `types.ODM_Stage`)
  - `odm_app.py` - Pipeline orchestration
  - `dataset.py` - Image loading and validation
  - `run_opensfm.py` - Structure from Motion
  - `openmvs.py` - Dense point cloud generation
  - `splitmerge.py` - Large dataset split/merge processing

- **`SuperBuild/`** - CMake-based C++ dependency compilation (OpenSfM, OpenMVS, PDAL, GDAL, etc.)

### Entry Points
- `run.py` - Main entry point (parses args → builds pipeline → executes stages)
- `run.sh` - Wrapper that activates venv and sets library paths

### Output Structure
Processing creates these directories in the project folder:
```
project/
├── images/              # Input images
├── opensfm/             # SfM results
├── odm_meshing/         # 3D mesh (.ply)
├── odm_texturing/       # Textured models (.obj)
├── odm_georeferencing/  # Point cloud (.laz)
├── odm_orthophoto/      # Orthophoto (.tif)
├── log.json             # Execution log
└── options.json         # Run parameters (for smart rerun detection)
```

## Key Patterns

- **Multi-camera support**: Auto-detects RGB + multispectral bands with normalization
- **Smart reruns**: `config.py` maps parameters to stages; changing a parameter triggers rerun from the affected stage
- **Parallel processing**: `opendm.concurrency.parallel_map()` for concurrent operations
- **JSON logging**: Every run outputs `log.json` with per-stage timing and error tracking

## Dependencies

- **Python**: numpy, scipy, GDAL, rasterio, Fiona, Shapely, scikit-learn, Pillow
- **C++**: OpenSfM, OpenMVS, PDAL, GDAL, Entwine, MVS-Texturing, PoissonRecon (built via SuperBuild)

## Resources

- Documentation: https://docs.opendronemap.org
- Community forum: https://community.opendronemap.org
- Parameters reference: https://docs.opendronemap.org/arguments/
