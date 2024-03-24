---
title: 3. Setting up LLaMa2Lang
---

Let us first `cd` to `/tools/LLaMa2Lang`. The next step is to install requirements for python:

```
pip install -r requirements.txt
```

And then we can should be able to go right ahead and try to translate OASST using Google's MADLAD. We can use the `translate.py` script for this step.

```
python translate.py madlad --model_size 7b da ./output_da --quant8 --batch_size 5 --max_length 512
```

This should translate, using MADLAD 7B, the OASST dataset into Danish, outputting the resulting files in `/tools/LLaMa2Lang/output_da`.

After a little while, I am told that 

```
==ERROR==
CUDA SETUP: CUDA detection failed! Possible reasons:
```

`CUDA` is a software layer which enables us to communicate with the `GPU`. The container is being run via `Singularity` with access to one Nvidia L40, so this seems to be a software-related problem. The script cannot communicate with the `GPU`.

A little bit of looking through the error messages tells us that the culprit is likely the Python library `bitsandbytes`. A bit of Googling tells me that other people have found success by building it from source. Let us try that.

Building from scratch requires `cmake`, so let's install that first.

```
apt-get update
apt-get install -y build-essential cmake
```

Then we need to download the source code for `bitsandbytes`. Let us clone it into the `/tools` directory.

```
cd ../tools
git clone https://github.com/TimDettmers/bitsandbytes.git && cd bitsandbytes
```

Let us install the necessary Python packages.

```
pip install -r requirements-dev.txt
```

And then run `cmake`.

```
cmake -DCOMPUTE_BACKEND=cuda -S .
```

And then `make`.

```
make
```

After a bit we are told that it

```
[100%] Built target bitsandbytes
```

And so finally we install it via pip so Python can use it.

```
pip install .
```

And a quick run of `python -m bitsandbytes` should tell us whether it can now communicate with the GPU.

```
SUCCESS!
Installation was successful!
```

Hooray! We can now try to translate again. Let us run the same command as before, from `/tools/LLaMa2Lang`.

```
python translate.py madlad --model_size 7b da ./output_da --quant8 --batch_size 5 --max_length 512
```

It takes a while to load the model into memory, and just as it seems like something is happening, it crashes with the following error message:

```
RuntimeError: expected scalar type Float but found Half
```

I am wondering whether we need to actually quantize the model here. We've got plenty of memory. This is just the default command from the `LLaMa2Lang` repo. Let us try it without quantization.

```
python translate.py madlad --model_size 7b da ./output_da --batch_size 5 --max_length 512
```

And success! The translation begins, with an ETA of about 26 hours. That seems a bit long for an L40. I wonder if we could increase the `batch_size` to something like 32. And then, let's add in some checkpoint saving, in case the script crashes, so we don't have to start over.

```
python translate.py madlad --model_size 7b da ./output_da --batch_size 32 --checkpoint_n 400 --max_length 512
```

