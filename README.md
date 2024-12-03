# Dev

Install [uv](https://github.com/astral-sh/uv), then `uv sync`

# Cellpainting data

Dataset is 64800 images of 6 channels (.tiff) from 5 timepoints with one plate per timepoint.

Data dir:

```
/data/panthera/panthera-cellpainting/operetta_raw_data
```

Images folder structure:

```
   .
   |-operetta_raw_data
   |---241101_oncocross_mcf7
   |-----12h
   |-------oncocross_mcf7_12h__2024-11-01T22_39_14-Measurement 1
   |---------Assaylayout
   |---------Images
   |-----24h
   |-------oncocross_mcf7_24h__2024-11-02T10_28_25-Measurement 1
   |---------Assaylayout
   |---------Images
   |-----48h
   |-------oncocross_mcf7_48h__2024-11-03T11_11_52-Measurement 1
   |---------Assaylayout
   |---------Images
   |-----6h
   |-------oncocross_mcf7_6h__2024-11-01T17_10_48-Measurement 1
   |---------Assaylayout
   |---------Images
   |-----72h
   |-------oncocross_mcf7_72h__2024-11-04T11_40_59-Measurement 1
   |---------Assaylayout
   |---------Images
```

Image filename:

(see also: https://forum.image.sc/t/regular-expression-for-exporting-data-from-perkin-elmer-phenix-opera-images/16389)

```
r09c03f01p01-ch1sk1fk1fl1.tiff
```

r: row, c: column, f: field, p: z-plane, ch: channel, ignore sk, fk, fl (unsure meaning)

Row/col refers to location of well within each plate. 9 fields arranged in 3x3 square were taken from each well.

Only one plane was taken from each field. Each image has 6 channels:

1. Brightfield
2. Mito
3. Hoechst 33342: binds DNA, stains nuclei)
4. Fluor 488 (Concavalin A): binds ER
5. 512 RNA
6. Fluor 568 (Phalloidin), Fluor 555 (WGA): stains actin

## Metadata

- `manifest.tsv` maps each file to plate, time, row, column, field, and channel
- `plate_map.xlsx` maps row/col to drug and dose -- this location mapping is the same across all timepoints


# Task
- [ ] define dataset & collater that loads data from all subdirectories, assigns labels/metadata and combines 6 tiff images into stacked tensor
- [ ] embed images using any pretrained model
    - [ ] imagenet
    - [ ] UNI
- [ ] bonus? train model to predict drug, time, and dose from image labels

Pretrained models expect RGB 3-channel images. There are several ways to make these images compatible:
1. Convert the brightfield channel to 3 channel (grayscale to RGB)
2. Assign fluorescent color to 6 channels, combine into RGB image using [microfilm](https://guiwitz.github.io/microfilm/notebooks/create_plots.html)

```python
from microfilm.colorify import multichannel_to_rgb

multichannel_to_rgb(cellpainting_image)
```
 
3. Simulate H&E stain. Hematoxylin and eosin stain nuclei and cytoplasm respectively, so cell painting channels that 
   stain nuclei and cytoplasm can be used to simulate H&E stain with something like [falsecolor](https://github.com/serrob23/falsecolor). However, there is 
   not a good cytoplasm stain that is equivalent to Eosin, I have tried using actin and mito stains but they are not good.

```python
import falsecolor as fc
from falsecolor.process import ViewImage


def pseudo_he(cellpainting_image, nuc_idx=2, cyto_idx=1):
    """
    :param cellpainting_image: 6 channel tensor
    :param nuc_idx: index of nuclear stain
    :param cyto_idx: index of cyto stain
    :return: false color img
    """
    nuclei = cellpainting_image[nuc_idx].numpy()
    cyto = cellpainting_image[cyto_idx].numpy()

    nuc_bg = fc.getBackgroundLevels(nuclei)[1]
    cyto_bg = fc.getBackgroundLevels(cyto)[1]

    pseudo_he = fc.falseColor(nuclei, cyto, nuc_threshold=nuc_bg, cyto_threshold=cyto_bg)
    ViewImage(pseudo_he)
```