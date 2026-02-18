# GeoSAM-Image-Encoder

[![PyPI Version](https://img.shields.io/pypi/v/GeoSAM-Image-Encoder)](https://pypi.org/project/GeoSAM-Image-Encoder/)
[![Downloads](https://static.pepy.tech/badge/GeoSAM-Image-Encoder)](https://pepy.tech/project/GeoSAM-Image-Encoder)
[![Run in Google Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/atsyplenkov/GeoSAM-Image-Encoder/blob/main/examples/Geo-SAM_encoding.ipynb)

This repository is a fork of the Geo-SAM image encoder package. It remains a standalone Python package (no QGIS dependency) for encoding geospatial rasters into SAM features that can be consumed by Geo-SAM workflows.

## Upstream Context

- Upstream project: [coolzhao/Geo-SAM](https://github.com/coolzhao/Geo-SAM)
- This fork is focused on Colab-friendly usage and bug fixes around raster indexing, bounds handling, CRS usage, and output naming.
- License: [LICENSE](LICENSE)

## Fork Changes

Recent updates in this fork include:

- Added a Colab notebook example at `examples/Geo-SAM_encoding.ipynb`.
- Improved sampler/index compatibility handling in `geosam/torchgeo_sam.py`.
- Improved geopandas/index fallback behavior in `geosam/torchgeo_sam.py`.
- Added tuple-resolution handling in sampling/export logic (`geosam/torchgeo_sam.py`, `geosam/image_encoder.py`).
- Ensured source CRS is passed into dataset creation in `geosam/image_encoder.py`.
- Updated feature export naming to include hashed extent/resolution context in `geosam/image_encoder.py`.

## Installation

### Recommended (this fork, pinned)

```bash
pip install -U git+https://github.com/atsyplenkov/GeoSAM-Image-Encoder.git@f55e888
```

### PyPI release

```bash
pip install GeoSAM-Image-Encoder
```

### GPU note

Install a PyTorch build that matches your hardware and CUDA/ROCm setup before installing this package:
<https://pytorch.org/get-started/locally/>

## Google Colab Quickstart

Use the badge above or open:
<https://colab.research.google.com/github/atsyplenkov/GeoSAM-Image-Encoder/blob/main/examples/Geo-SAM_encoding.ipynb>

Typical Colab flow:

1. Install this fork.
2. Download a SAM checkpoint (for example `sam_vit_l_0b3195.pth`).
3. Mount Google Drive and set input/output paths.
4. Run the encoder in Python.

## Python Usage

```python
import geosam
from geosam import ImageEncoder

print(geosam.gpu_available())
```

### Direct parameters

```python
checkpoint_path = "/content/sam_vit_l_0b3195.pth"
image_path = "/content/drive/MyDrive/input.tif"
feature_dir = "/content/drive/MyDrive/features"

encoder = ImageEncoder(
    checkpoint_path=checkpoint_path,
    model_type="vit_l",
    batch_size=1,
    gpu=True,
    gpu_id=0,
)

encoder.encode_image(
    image_path=image_path,
    feature_dir=feature_dir,
    bands=[1, 2, 3],
    stride=512,
    value_range=(0.0, 255.0),
    resolution=0.25,
)
```

### Using `settings.json` exported by Geo-SAM

```python
import geosam
from geosam import ImageEncoder

settings = geosam.parse_settings_file("/content/setting.json")
settings.update({"feature_dir": "/content/features"})

init_settings, encode_settings = geosam.split_settings(settings)

encoder = ImageEncoder(**init_settings)
encoder.encode_image(**encode_settings)
```

## CLI Usage

The recommended CLI entrypoint after installation is module execution:

```bash
python -m geosam.image_encoder -h
```

Direct parameters:

```bash
python -m geosam.image_encoder \
  -i /path/to/image.tif \
  -c /path/to/sam_vit_l_0b3195.pth \
  -f /path/to/output_features
```

Using settings file (with optional overrides):

```bash
python -m geosam.image_encoder \
  -s /path/to/setting.json \
  -f /path/to/output_features \
  --stride 256 \
  --value_range "(10,255)"
```

Supported options are:

- `-s`, `--settings`
- `-i`, `--image_path`
- `-c`, `--checkpoint_path`
- `-f`, `--feature_dir`
- `--model_type`
- `--bands`
- `--stride`
- `--extent`
- `--value_range`
- `--resolution`
- `--batch_size`
- `--gpu_id`

## Input/Output Notes

- Input bands: SAM encoding supports at most 3 bands. If fewer than 3 are provided, bands are repeated to 3 internally.
- Input CRS: the raster must have a valid CRS. Encoding fails for CRS-less rasters.
- Model type: `vit_h`, `vit_l`, and `vit_b` are supported (or integer aliases `0`, `1`, `2`).
- Output layout: features are written under:
  - `<feature_dir>/<image_stem>/sam_feat_<model_type>_bands_<bands>_<extent_hash>/`
  - Per patch: `sam_feat_<model_type>_<bbox_hash>.tif`
  - Per folder index: `<folder_name>.csv`

## Troubleshooting

- `Could not infer the model type from the checkpoint path`: set `model_type` explicitly.
- `Input raster has no CRS`: assign CRS before encoding.
- No GPU available: encoder automatically falls back to CPU (batch size effectively reduced for CPU path).
