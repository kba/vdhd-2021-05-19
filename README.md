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
