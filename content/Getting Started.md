---
title: Getting Started
---


We have gained access to AAU AI Cloud and can begin.

The following is a brief introduction on how to access and run jobs on AI Cloud.

## Tech stack
The way AAU AI Cloud works is typical to many AI-dedicated HPC's. Particularly, they do not employ `Docker` but `Apptainer/Singularity` to run containers. I assume this is because `Docker` does not really have way of controlling user privileges. Luckily for us, `Singularity` is compatible with `Docker` containers, so we can run any we find.

> [!info]
> Containers in this context basically refers to an image of an operating system which has pre-installed software and is pre-configured. As such, we can use a container which already has fine-tuning software installed, so we do not need to do this every time we run a fine-tuning.

## Logging in
Accessing the AI Cloud is done via `ssh`. This can only be achieved if you are connected to AAU network, either phyiscally, via VPN or through first ssh'ing into the ssh gateway: [https://www.en.its.aau.dk/instructions/ssh](https://www.en.its.aau.dk/instructions/ssh).

Here is how:

```
ssh -l <aau email> ai-fe02.srv.aau.dk
```

## Running jobs
The way to run a process on AI Cloud is done through [`Slurm`](https://slurm.schedmd.com). Slurm is a workload manager, which means that its job is to ensure that the server is not running over capacity. It basically queues the processes everyone using AI Cloud wants to run, ensuring that each process gets the resources it needs and is run when those resources become available.

Using Slurm is a relatively straightforward thing to do, and is accomplished through the command `srun`. This is also where we input which container to use. This makes things very easy for us. We do not need to install software that is needed for the fine-tuning process, instead we can simply point it to a container which already has the software installed. It will download the container for us, and then run it.

Here is an example `srun` command:

```
srun --mem 32G singularity pull docker://nvcr.io/nvidia/tensorflow:22.07-tf2-py3
```

Let us break down the arguments:

`--mem XXG`: Defines the memory needed for the process that is to be run.  
`singularity`: Defines which software `srun`  should launch.  
`pull`: Becomes a command for `singularity`, telling it to download a container for us.  
`docker://nvcr.io/nvidia/tensorflow:22.07-tf2-py3` is the link to the container that is to be downloaded.  

This first command will, of course, only download the container. From here we have multiple options, and can either:

### Access a shell
We can open an interactive shell (or, a pseudo-terminal) in the container, so that we can access it as if it was our own system. This is helpful if we need to, for instance, install some more software or other dependencies before executing a script. Shell is exited with command `exit`. Shell is accessed through the following command:

```
srun --pty singularity shell tensorflow_22.07-tf2-py3.sif
```

Let us break down the arguments:

`--pty`: Launches a pseudo-terminal. This, essentially, just means that we can interact with the system in a terminal environment.  
`singularity shell`: Tells `srun` to run `singularity`, and the container to launch a shell so we can interact with the container.  
`tensorflow_22.07-tf2-py3.sif`: This is the filename of our container.  

### Execute a command
Alternatively, we can tell it to not open a shell, but to run a given script. This is what we will do when we are configured and ready for fine-tuning. It is done as such:

```
srun singularity exec tensorflow_22.07-tf2-py3.sif cmd 
```

Let us break it down:

`singularity exec`: Runs a specific command within a container.  
`tensorflow_22.07-tf2-py3.sif`: This is the filename of our container.  
`cmd`: The command to be executed inside the container.  

We will then setup the commands after what we find necessary, to ease the process of fine-tuning.

### Running the default action
And finally, we can set a default action for a container. This might be useful, but I doubt it. In any case, it is performed as follows:

```
srun singularity run tensorflow_22.07-tf2-py3.sif
```

Let us break it down:

`singularity run`: Runs default action of a container.  
`tensorflow_22.07-tf2-py3.sif`: This is the filename of our container.  

And, of course, if the default container action is to open a shell, we need `--pty` to create a pseudo-terminal for us.

## Using GPUs
And finally, to be able to actually do anything meaningful as relates to AI, we need GPUs (graphical processing units). [View available GPUs](https://aicloud-docs.claaudia.aau.dk/introduction/#overview)

GPUs are allocated in the beginning of the `srun` command, but the type of GPU needed is specified at the end. Here is an example command:

```
srun --gres=gpu:a10:1 singularity exec --nv tensorflow_22.07-tf2-py3.sif command
```

Let us break it down:

`--gres=:gpu:a10:1`: This tells `srun` that we need 1 GPU of the type `a10`.  
`singularity exec`: Runs a specific command within a container.  
`--nv`: Tells the container that we need to use Nvidia GPUs. Is required.  
`tensorflow_22.07-tf2-py3.sif`: This is the filename of our container.  
`command`: Specifies what command will be executed.

With this, our container now has access to a GPU.