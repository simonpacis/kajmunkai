---
title: 2. Setting up LLaMa 2 
---
First up, we need to download a container image. Nvidia provides multiple. According to Meta's own [LLaMa 2 repository](https://github.com/meta-llama/llama) we need PyTorch in our container. Everything else can be installed after the fact. Nvidia, through their NGC Catalog, provides a [container which has PyTorch installed](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch).

## Downloading container
As explained in [[Getting Started]], we use `Slurm` to download container images.

```
srun --mem 32G singularity pull docker://nvcr.io/nvidia/pytorch:24.02-py3
```

Executing this command seems to work, but eventually halts because we ran out of memory.

```
srun: error: a256-t4-03: task 0: Out Of Memory
```

Alright, so the 32 gigabytes we allocated weren't enough. Looking through Meta's and Nvidia's documentation I can't find anything telling me how much memory is needed. But [AI Cloud's documentation tells me of a workaround](https://aicloud-docs.claaudia.aau.dk/singularity/#work-arounds) wherein we create a temporary directory that the process can use. Let's try it.

First we setup a bash shell we can use.

```
srun --pty bash -l
```

Then we create a temporary directory that the download process can use.

```
mkdir /tmp/`whoami`
```

Then we define the temporary directory so `Singularity` can use it.

```
SINGULARITY_TMPDIR=/tmp/`whoami`
```

And finally we try to run the download process again.

```
singularity pull docker://nvcr.io/nvidia/pytorch:24.02-py3
```

After waiting a bit, the script tells me that it was `Killed`. Exiting out of my bash shell, I am told that

```
srun: error: a256-t4-01: task 0: Out Of Memory
```

So that workaround didn't help. Maybe I did it wrong? The easy solution now is to just try and allocate more memory to see if that works. I'll double it, and try 64G.

```
srun --mem 64G singularity pull docker://nvcr.io/nvidia/pytorch:24.02-py3
```

Success! After a while it finishes, and our home directory now looks like this:

```
-rwxrwxr-x 1 xx@xx.aau.dk xx@xx.aau.dk 10211835904 Mar 22 18:24 pytorch_24.02-py3.sif
```

We have our container!

## Creating writable sandbox container 
The containers downloaded are by default in read-only mode, but since we at the very least need to download the model, we need to be able to write to the container. We can do this by creating a sandbox container based on the container we just downloaded. It's very straightforward, just a single command:

```
srun singularity build --sandbox kajmunkai_pytorch_24.02-py3 pytorch_24.02-py3.sif
```

This command creates a sandbox container called `kajmunkai_pytorch_24.02-py3` based on the downloaded container `pytorch_24.02-py3.sif`. Running it gives us:

```
INFO:    Build complete: kajmunkai_pytorch_24.02-py3
```

And our home directory now looks like this:

```
drwxr-xr-x 19 xx@xx.aau.dk xx@xx.aau.dk    1M Mar 22 18:03 kajmunkai_pytorch_24.02-py3
-rwxrwxr-x  1 xx@xx.aau.dk xx@xx.aau.dk 9739M Mar 22 18:24 pytorch_24.02-py3.sif
```

Success!

## Download LLaMa 2
To download LLaMa 2, we first need a unique download link from Meta. This is obtained by filling out [this form](https://llama.meta.com/llama-downloads).

Then we need to clone the [`llama` Github repository](https://github.com/meta-llama/llama). To do this, we first open a shell in our newly created sandbox container:

```
srun --pty singularity shell --writable --fakeroot kajmunkai_pytorch_24.02-py3
```

Check that git is installed:

```
git
```

returns

```
usage: git [--version] [--help] [-C <path>] [-c <name>=<value>]
```

So we're golden! Let us now clone the repository.

```
git clone https://github.com/meta-llama/llama
```

Gives us a directory `llama` in our home directory. In here is a script called `download.sh`, which is run to download the model. We enter the unique download URL Meta sent us via email, and then we have to choose which LLaMa2 model we want to use, whether 7B, 13B, or 70B (and whether it should be the chat-variant or not). 

```
Singularity> ./download.sh
Enter the URL from email: XXX
Enter the list of models to download without spaces (7B,13B,70B,7B-chat,13B-chat,70B-chat), or press Enter for all:
```

At this point I am unsure which of the variants covers our use case, however, we should only need the chat variants. I am assuming that the 70 billion parameter variant is perhaps overkill, but let us download all of them so that we can at least try it out.

```
Enter the list of models [...]: 7B-chat,13B-chat,70B-chat
```

This command begins a rather lengthy download process. Our home directory has 1TB of available space. The 7B variant takes up ~12GB, the 13B takes up ~24GB, and the 70B variant takes up ~112GB. No problems there. And in any case, we can always request more storage if needed.

## Initial testing
Before we do anything else, let us try to see if we can get the model to spin up. The 70B parameter model requires 48GB VRAM, so we need to use the A40 GPU. The following command should get us into a shell with the necessary resources:

```
§ srun --gres=gpu:a40:1 --pty singularity shell --nv --writable --fakeroot kajmunkai_pytorch_24.02-py3/
srun: job 145385 queued and waiting for resources
```

So now we wait for the resources to be allocated... An hour later, and we are still unallocated. Because it is a Friday, a lot of other users of AI Cloud probably queued jobs to run all through the weekend. Let us check `nodesummary`.

```
<snip>

i256-a10-06
   GPUs: 4                             /   4
   CPUs: 42                            /  64
   Mem.: 192                           / 244 GB

i256-a10-07
   GPUs: 4                             /   4
   CPUs: 40                            /  64
   Mem.: 160                           / 244 GB

i256-a10-08
   GPUs: 4                             /   4
   CPUs: 40                            /  64
   Mem.: 160                           / 244 GB

i256-a10-09
   GPUs: 4                             /   4
   CPUs: 40                            /  64
   Mem.: 160                           / 244 GB

i256-a10-10
   GPUs: 4                             /   4
   CPUs: 40                            /  64
   Mem.: 160                           / 244 GB

<snip>
```

Yeah, that's probably it. All GPUs on all nodes are allocated. So, a different route if we want to try it out now. Instead, we will use [`llama.cpp`](https://github.com/ggerganov/llama.cpp), which enables us to run inference on LLaMa 2 entirely using the CPU, without the need for a GPU. It will be incredibly slow, but this is just to test it out after all.

After cloning `llama.cpp`, and `make`ing it, we need to convert the downloaded model so that it can be used with `llama.cpp` – it needs to be in `.gguf` format. The following command, launched from the cloned `llama.cpp` directory does this:

```
python3 convert.py /root/llama/llama-2-7b-chat/
```

After running for a minute, it tells us:

```
Wrote /root/llama/llama-2-7b-chat/ggml-model-f32.gguf
```

Perfect. Now we can try to ask it a question, see what it tells us. Here's what I'll ask it for its first go: "Du må kun svare på dansk. Fortæl mig lidt om hvem Kaj Munk var." Let's see how it does.

```
./main -t 22 -m /root/llama/llama-2-7b-chat/ggml-model-f32.gguf --color -c 4096 --temp 0.7 --repeat_penalty 1.1 -n -1 -p "### Instruction: Du må kun svare på dansk. Fortæl mig lidt om hvem Kaj Munk var.\n### Response:"
```

It takes about three minutes to start working on the prompt. It generates about a character every 30 seconds. I'll let it run for a while, just to see what we get. 

```
Instruction: Du må kun svare på dansk. Fortæl mig lidt om hvem Kaj Munk var.
Response: Kaj Munk was a Danish playwright,
```

I stopped it here. This obviously highlights the need for us to fine-tune the model to be conversational in Danish. It clearly understands my prompt, but thinks that I expect it to reply to me in English.

With that small testing out of the way, let us move on to working with `LLaMa2Lang`.

[[3 Setting up LLaMa2Lang|Continue to "Setting Up LLaMa2Lang"]]
