# Hackathon OCR@vDHD

## Jannik Franz' samples

Download the PDF files (`Silcher_Werkverzeichnis_s1.pdf`, `programmheft_10.pdf`) to `jannik.frantz` and change to the directory.

### Import using `ocrd-import`

`ocrd-import` is provided by @bertsky's
[workflow-configuration](https://github.com/bertsky/workflow-configuration)
project (installed with [ocrd_all](https://github.com/OCR-D/ocrd_all/))

```sh
ocrd-import -r 70
```

This tells `ocrd-import` to render the images from the PDF with 70 DPI. This should of course match
the actual DPI of the original as good as possible. Note that with the standard setting (300 DPI) file
sizes will be vastly larger (a few MiB vs. up to a few hundred MiB).

This will extract the images from the PDF and initiate a new workspace with those images added.

### Troubleshooting ImageMagick - not authorized

If no images are created and you see a line like

```
convert-im6.q16: not authorized `programmheft_10.pdf' @ error/constitute.c/ReadImage/412.
```

containing `not authorized`, that means that PDF conversion using ghostscript has been disabled. You need to change the `/etc/ImageMagick-6/policy.xml`:

```diff
78c78
<   <policy domain="coder" rights="none" pattern="" />
---
>   <!--<policy domain="coder" rights="none" pattern="PDF" />-->
```

### Troubleshooting ImageMagick - cache resources exhausted

If you see a line like

```
convert-im6.q16: cache resources exhausted `programmheft_10.pdf' @ error/cache.c/OpenPixelCache/3984.
```

containing `cache resources exhausted`, it means that you hit the memory limit which is by default 1 GiB.  Change the `/etc/ImageMagick-6/policy.xml` to increase it:

```diff
58c58
<   <policy domain="resource" name="disk" value="1GiB"/>
---
>   <policy domain="resource" name="disk" value="8GiB"/>
```


### Run eynollah for segmentation

Let's try the simplest call possible

```sh
ocrd-eynollah-segment -I OCR-D-IMG -O SEG-EYNOLLAH
Traceback (most recent call last):
  File "/home/kba/monorepo/ocrd_all/venv/bin/ocrd-eynollah-segment", line 33, in <module>
    sys.exit(load_entry_point('eynollah', 'console_scripts', 'ocrd-eynollah-segment')())
  File "/home/kba/monorepo/ocrd_all/venv/lib/python3.6/site-packages/click/core.py", line 829, in __call__
    return self.main(*args, **kwargs)
  File "/home/kba/monorepo/ocrd_all/venv/lib/python3.6/site-packages/click/core.py", line 782, in main
    rv = self.invoke(ctx)
  File "/home/kba/monorepo/ocrd_all/venv/lib/python3.6/site-packages/click/core.py", line 1066, in invoke
    return ctx.invoke(self.callback, **ctx.params)
  File "/home/kba/monorepo/ocrd_all/venv/lib/python3.6/site-packages/click/core.py", line 610, in invoke
    return callback(*args, **kwargs)
  File "/home/kba/monorepo/ocrd_all/eynollah/qurator/eynollah/ocrd_cli.py", line 8, in main
    return ocrd_cli_wrap_processor(EynollahProcessor, *args, **kwargs)
  File "/home/kba/monorepo/ocrd_all/venv/lib/python3.6/site-packages/ocrd/decorators/__init__.py", line 91, in ocrd_cli_wrap_processor
    run_processor(processorClass, ocrd_tool, mets, workspace=workspace, **kwargs)
  File "/home/kba/monorepo/ocrd_all/venv/lib/python3.6/site-packages/ocrd/processor/helpers.py", line 63, in run_processor
    parameter=parameter
  File "/home/kba/monorepo/ocrd_all/eynollah/qurator/eynollah/processor.py", line 30, in __init__
    super().__init__(*args, **kwargs)
  File "/home/kba/monorepo/ocrd_all/venv/lib/python3.6/site-packages/ocrd/processor/base.py", line 112, in __init__
    raise Exception("Invalid parameters %s" % report.errors)
Exception: Invalid parameters ["[] 'models' is a required property"]
```

That was too simple :) Let's find out which models are available:

```sh
ocrd resmgr list-available -e ocrd-eynollah-segment
ocrd-eynollah-segment
- default  (https://qurator-data.de/eynollah/models_eynollah.tar.gz)
  models for eynollah
```

Let's download it

```sh
ocrd resmgr download ocrd-eynollah-segment default
```

and call eynollah again with the `models` parameter set:


```sh
ocrd-eynollah-segment -I OCR-D-IMG -O SEG-EYNOLLAH -P models default
```

**NOTE:** Until [v0.0.5](https://github.com/qurator-spk/eynollah/releases/tag/v0.0.5) of eynollah, the `allow_enhancement` parameter was by default set to `true`, which led to inconsistent coordinates. If you are using an older version of eynollah, always explicitly disable that parameter, e.g.


```sh
ocrd-eynollah-segment -I OCR-D-IMG -O SEG-EYNOLLAH -P models default -P allow_enhancement false
```

This will take a few minutes, depending on whether a GPU is available.

### Segmentation results

Page 1:

![](./jannik.franz/segmentation-p0001.png)

Page 2:

![](./jannik.franz/segmentation-p0002.png)

Note that while the segmentation does a good job detecting the cells of the table, it does not produce
a readily usable tabular structure, i.e. you do need to postprocess the results with heuristics
to transform it into a true table.

## Stefan von der Heide's samples



### `ocrd-import` the images

Create directory `OCR-D-IMG`, put the images there, import the files

```sh
mkdir OCR-D-IMG
cp -t OCR-D-IMG BadScan-01.jpg DeWarp-Example-01.tif NewsPaperWithAdsLayout-01.tif
ocrd-import
```

### Run workflow

```sh
ocrd process \
  "cis-ocropy-binarize -I OCR-D-IMG -O OCR-D-BIN" \
  "anybaseocr-crop -I OCR-D-BIN -O OCR-D-CROP" \
  "skimage-denoise -I OCR-D-BIN -O OCR-D-BIN-DENOISE -P level-of-operation page" \
  "tesserocr-deskew -I OCR-D-BIN-DENOISE -O OCR-D-BIN-DENOISE-DESKEW -P operation_level page" \
  "tesserocr-segment -I OCR-D-BIN-DENOISE-DESKEW -O OCR-D-SEG -P shrink_polygons true" \
  "cis-ocropy-dewarp -I OCR-D-SEG -O OCR-D-SEG-DEWARP" \
  "tesserocr-recognize -I OCR-D-SEG-DEWARP -O OCR-D-OCR -P textequiv_level glyph -P overwrite_segments true -P model Fraktur_GT4HistOCR"
```

### Binarize the images

```sh
ocrd-olena-binarize -I OCR-D-IMG -O BIN
```

### Try different segmentations

```sh
ocrd-cis-ocropy-segment -I BIN -O SEG-OCRO
ocrd-tesserocr-segment -I BIN -O SEG-TESS
ocrd-eynollah-segment -I OCR-D-IMG -O SEG-EYNOLLAH
```
