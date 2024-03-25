---
title: sbatch
---

While we can technically run everything we need to using `srun`, it does have a two major drawbacks:

1. Need to type in container name every time
2. Tied to our terminal session
	- Job terminates if we exit terminal session.
3. Unable to notify when ran
	- We have to manually monitor our terminal session to see what goes on.

We can use `sbatch` instead. This command ensures that our jobs are detached from our terminal session, and so once we queue them, we can shut everything off and leave, and it will still do its thing. Let us set it up so that we also receive e-mail notifications when the job begins and finishes. I also have this idea where the output from running the scripts are neatly organized, and end up in `~/output`. This, however, requires a convoluted approach. We will create a Python script `sb`, to which we can simply pass what we want executed in our container. We can also pass in `--gres` and `--mem` to request specific resources.

I came up with the following:

```python
#!/usr/bin/env python3

import sys
from datetime import datetime
import subprocess
import os

def submit_job(*args):
    # Look for --gres argument and extract it if present
    gres_value = None
    remaining_args = []
    for arg in args:
        if arg.startswith("--gres="):
            gres_value = arg
        elif arg.startswith('--mem'):
            gres_value = gres_value + ' ' + arg
        else:
            remaining_args.append(arg)

    if len(remaining_args) < 1:
        print("Usage: sb <script_path> [additional arguments]")
        sys.exit(1)

    script_path = ' '.join(remaining_args)

    script_name = os.path.basename(script_path)
    now = datetime.now().strftime('%H%M%S-%Y-%m-%d')
    output_file = f"/home/id.aau.dk/xx/output/{now}-{script_name}.log"

    sbatch_command = ["sbatch"]

    # Include gres_value if it's specified
    if gres_value:
        sbatch_command.append(gres_value)

    sbatch_command += [
        "--output", output_file,
        "/home/id.aau.dk/xx/scripts/sbatch/base.sh",
        script_path
    ]

    print(' '.join(sbatch_command))
    result = subprocess.run(sbatch_command, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    print(result.stdout.decode())
    if result.stderr:
        print(result.stderr.decode())

if __name__ == "__main__":
    submit_job(*sys.argv[1:])
```

This script generates our `sbatch` command, and tells `sbatch` where to save the output (in `~/output`). The `sbatch` command itself is `base.sh` in `~/scripts/sbatch`. It also makes sure that `--gres`and `--mem` are correctly inserted in the `sbatch` command.

The `base.sh` command is currently unnecessary, I think. I need to make `sbatch`run Python directly, and we eliminate the need for two scripts. But, that's a refactoring for when I have time. In any case, here is `base.sh`:

```bash
#!/bin/bash

# Initialize variables for optional arguments
gres=""
mem=""

# Parse command-line arguments
while [[ "$#" -gt 0 ]]; do
    case $1 in
        --gres) gres="$1 $2"; shift ;;  # Capture --gres and its value
        --mem) mem="$1 $2"; shift ;;    # Capture --mem and its value
        *) break ;;                      # Break on any other argument
    esac
    shift
done

# Construct the base srun command
cmd="srun"

# Add --gres and --mem to the command if they were provided
if [ ! -z "$gres" ]; then
    cmd="$cmd $gres"
fi

if [ ! -z "$mem" ]; then
    cmd="$cmd $mem"
fi

# Construct the singularity command, add --nv if --gres is provided
singularity_cmd="singularity exec"
if [ ! -z "$gres" ]; then
    singularity_cmd="$singularity_cmd --nv"
fi

# Append the rest of the singularity command
singularity_cmd="$singularity_cmd --writable --fakeroot /home/id.aau.dk/xx/containers/kajmunk_pytorch_24.02-py3"

# Combine the commands and execute
full_cmd="$cmd $singularity_cmd $@"
echo $full_cmd
eval $full_cmd
```

When resources are allocated, this is what will be ran. We have hardcoded the sandbox container, with the `--writable` and `--fakeroot` commands.

All of this allows us to simply run:

```
sb python /root/scripts/python/test.py
```

Which results in output file `~/output/043606-2024-03-25-test.py.log`, containing first the `srun` command that was executed, then all output.

```
srun singularity exec --writable --fakeroot /home/id.aau.dk/sklitj17/containers/kajmunk_pytorch_24.02-py3 python /root/scripts/python/test.py
WARNING: Skipping mount /etc/localtime [binds]: /etc/localtime doesn't exist in container
13:4: not a valid test operator: (
13:4: not a valid test operator: 535.161.07
This is a test script.
```
