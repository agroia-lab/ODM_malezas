# ODM Mosaic Processing

Process drone images into georeferenced orthomosaics using OpenDroneMap Docker.

## Arguments

- `$ARGUMENTS` - Project name or path to images folder

## Quick Docker Command (Simplest)

```bash
# Basic orthomosaic CPU (fast, for flat agricultural fields)
docker run --rm \
  -v /home/malezainia1/odm_datasets:/datasets \
  opendronemap/odm \
  --project-path /datasets PROJECT_NAME \
  --fast-orthophoto \
  --orthophoto-resolution 2 \
  --dsm

# With dual RTX 4090 GPU acceleration (RECOMMENDED for 100+ images)
# NOTE: First run `docker context use default` if using Docker Desktop
docker run --rm --gpus all \
  -v /home/malezainia1/odm_datasets:/datasets \
  opendronemap/odm:gpu \
  --project-path /datasets PROJECT_NAME \
  --fast-orthophoto \
  --orthophoto-resolution 2 \
  --feature-type sift \
  --max-concurrency 32 \
  --dsm
```

## Data Structure Required

```
/home/malezainia1/odm_datasets/
└── PROJECT_NAME/
    └── images/
        ├── DJI_0001.JPG
        ├── DJI_0002.JPG
        └── ... (overlapping drone photos with GPS EXIF)
```

## Processing Steps

1. **Create project folder** with images:
   ```bash
   mkdir -p /home/malezainia1/odm_datasets/MY_CAMPO/images
   cp /path/to/drone/photos/*.JPG /home/malezainia1/odm_datasets/MY_CAMPO/images/
   ```

2. **Run ODM**:
   ```bash
   docker run --rm \
     -v /home/malezainia1/odm_datasets:/datasets \
     opendronemap/odm \
     --project-path /datasets MY_CAMPO \
     --fast-orthophoto \
     --orthophoto-resolution 2 \
     --dsm
   ```

3. **View output**:
   ```bash
   qgis /home/malezainia1/odm_datasets/MY_CAMPO/odm_orthophoto/odm_orthophoto.tif
   ```

## Key Parameters for Agriculture

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--fast-orthophoto` | flag | Skip 3D mesh, faster for flat fields |
| `--orthophoto-resolution` | 1-5 | cm/pixel (2 recommended for weed detection) |
| `--dsm` | flag | Generate Digital Surface Model (plant height) |
| `--dtm` | flag | Generate Digital Terrain Model (ground level) |
| `--matcher-neighbors` | 8-12 | Improve alignment for sparse GPS |

## Advanced: Python Script with Progress

```bash
cd /home/malezainia1/dev/INIA_DeepLearning_Ubuntu_mod_lleon

# Analyze flight pattern and select optimal subset
python scripts/mosaic/analyze_flight_pattern.py \
    --input "/path/to/drone/images" \
    --output "/home/malezainia1/odm_datasets/campo_name" \
    --lines 5 --per-line 40

# Run ODM with progress monitoring
python scripts/mosaic/run_odm.py \
    --project "/home/malezainia1/odm_datasets/campo_name" \
    --multispectral
```

## Output Structure

```
PROJECT_NAME/
├── images/                      # Input images
├── odm_orthophoto/
│   └── odm_orthophoto.tif      # Main output - georeferenced mosaic
├── odm_dem/
│   ├── dsm.tif                 # Digital Surface Model
│   └── dtm.tif                 # Digital Terrain Model (if --dtm)
├── odm_georeferencing/
│   └── odm_georeferenced_model.laz  # Point cloud
├── odm_report/
│   └── report.pdf              # Processing report
├── benchmark.txt               # Timing info
└── log.json                    # Detailed log
```

## Integration with YOLO Detection Pipeline

After generating orthomosaic:

```bash
# Run weed detection with SAHI tiling
python /home/malezainia1/dev/INIA_DeepLearning_Ubuntu_mod_lleon/scripts/detection/run_sahi_detection.py \
    --input /home/malezainia1/odm_datasets/MY_CAMPO/odm_orthophoto/odm_orthophoto.tif \
    --model /path/to/yolo_weed_model.pt \
    --tile-size 1280 \
    --overlap 0.2
```

## Multispectral Processing (DJI P4 Multispectral / Mavic 3M)

```bash
docker run --rm \
  -v /home/malezainia1/odm_datasets:/datasets \
  opendronemap/odm \
  --project-path /datasets CAMPO_MULTI \
  --radiometric-calibration camera \
  --orthophoto-resolution 5 \
  --dsm
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Not enough images" | Need 70-80% overlap between images |
| "Poor alignment" | Add `--matcher-neighbors 16` |
| Out of memory | Add `--pc-quality low` or `--resize-to 2048` |
| Slow processing | Use `--fast-orthophoto` for flat terrain |
| No GPS in images | Provide GCP file with `--gcp gcp_list.txt` |

## Resource Estimates

| Images | RAM | Time (CPU) | Output Size |
|--------|-----|------------|-------------|
| 50 | 8 GB | ~15 min | ~2 GB |
| 100 | 16 GB | ~30 min | ~5 GB |
| 200 | 24 GB | ~1 hour | ~10 GB |
| 500 | 32 GB | ~3 hours | ~25 GB |

## Prerequisites Check

```bash
# Verify Docker
docker --version

# Check ODM image
docker images | grep opendronemap

# Pull if needed
docker pull opendronemap/odm
```
