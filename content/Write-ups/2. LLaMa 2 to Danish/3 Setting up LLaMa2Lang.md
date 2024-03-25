---
title: 3. Translating to Danish 
---

First, we need to clone `LLaMa2Lang`. The home directory currently contains the following directories:

```
containers
tools
models
scripts
```

Let us clone it, using git, into tools:

```
cd tools
git clone https://github.com/UnderstandLingBV/LLaMa2lang/tree/main
```

The next step is to install requirements for python:

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
python translate.py madlad --model_size 7b da ./output_da --batch_size 12 --checkpoint_n 400 --max_length 512
```

Oh...

```
Exception: Checkpoint N must be a multiple of batch size!
```

Let's try again then, with a batch_size of 20.

```
python translate.py madlad --model_size 7b da ./output_da --batch_size 20 --checkpoint_n 400 --max_length 512
```

```
torch.cuda.OutOfMemoryError: CUDA out of memory.
```

Hmm. Batch size of 20 might have been too high. Let's try 10.

```
python translate.py madlad --model_size 7b da ./output_da --batch_size 20 --checkpoint_n 400 --max_length 512
```

ETA of 14 hours. That's better. Let's just let it finish.

And, 14 hours later, we end up with a bunch `.json`-files like this:

```
<snip>
-rw-rw-r-- 1 xx@id.aau.dk xx@id.aau.dk 547933 Mar 25 02:22 upto_10000.json
-rw-rw-r-- 1 xx@id.aau.dk xx@id.aau.dk 595039 Mar 25 02:30 upto_10400.json
-rw-rw-r-- 1 xx@id.aau.dk xx@id.aau.dk 621759 Mar 25 02:39 upto_10800.json
-rw-rw-r-- 1 xx@id.aau.dk xx@id.aau.dk 609283 Mar 25 02:48 upto_11200.json
<snip>
```

Each of these contain a bunch of translated training data from OASST. Here are some snippets of the translated data:

```
I dette tilfælde er implementeringen af MACD crossover-strategien baseret på en almindeligt anvendt teknisk analyseindikator, som har været meget udbredt i den finansielle industri i mange år.
```

and

```
Du kommer til at have brug for:
* 1 lb hakket kalkun
* 1 æg
* 1/4 kop tern løg
* 1 fed hakket hvidløg
* 1/4 kop tern peberfrugter (valgfrit)
* 1 tsk salt (eller efter smag)
* 1-2 spsk vegetabilsk olie
* krydderier: 1 tsk tørret timian, 1 tsk tørret rosmarin
```

These look pretty good, and we can probably move on the the next stage.