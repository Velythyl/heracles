# heracles

## Problem

You've got a bunch of runs to do inside of a container, but it's pretty hard to call them through Slurm? 

For example: you've got Hydra setup inside of the container, but you can't use hydra's `multirun`, because, well hydra is INSIDE the container, and the thing that summons tasks is OUTSIDE the container.

## Solution

Slay the hydra 🤪

Basically we just re-implement a very naive version of `multirun` OUTSIDE the container so that SLURM can understand it.

## How it works

```bash
heracles <name of script to modify> <option> <hydra multirun-like sweep>
```

For example let's say I have a simple `sbatch.sh` that looks like this:

```bash
#!/bin/bash
#SBATCH --account=ACCOUNT 
#SBATCH --time=6:00:00
#SBATCH --job-name=WOW  #Job name
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16           # CPU cores/threads
#SBATCH --array=1-90

singularity --quiet exec --nv -H ~/projects/ACCOUNT/USER/PROJ/temp/ \
  --env WANDB_MODE="offline",WANDB_API_KEY="LOL_U_THOUGHT",WANDB_WATCH="false" \
  apptainer.sif /YOUR_PROJECT/main.py
```

You want to run `/YOUR_PROJECT/main.py -m env.num_envs=8,16,32 env.name=HalfCheetah,Reacher`, but since hydra can't launch
runs using sbatch, you're in a bit of a pickle.

Instead, you can call `heracles sbatch.sh -w env.num_envs=8,16,32 env.name=HalfCheetah,Reacher`. This will do a 
cartesian product (just like hydra's multirun does). It will then concatenate the products to copies of `sbatch.sh` that
it will create (these copies follow the `<file>_hydra<id>.sh` scheme).
These files will look like this (one for every product):

```bash
#!/bin/bash
#SBATCH --account=ACCOUNT 
#SBATCH --time=6:00:00
#SBATCH --job-name=WOW  #Job name
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16           # CPU cores/threads
#SBATCH --array=1-90

singularity --quiet exec --nv -H ~/projects/ACCOUNT/USER/PROJ/temp/ \
  --env WANDB_MODE="offline",WANDB_API_KEY="LOL_U_THOUGHT",WANDB_WATCH="false" \
  apptainer.sif /YOUR_PROJECT/main.py env.num_envs=8 env.name=HalfCheetah
```

> Note: Heracles also supports `range`, so `env.num_envs=2,4,6,8` is the same as `env.num_envs=range(2,10,2)`.

### But wait, this just writes down all the runs that I want to run as separate files. How do I actually run them?

Let's look at Heracles' options:

| Option                                              | Function                                                                                                         | Why use it                                                       |   |   |
|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|---|---|
| -w, --write, -n, --noop                             | Writes down all the products                                                                                     | Check that the script writes down the correct runs               |   |   |
| -r=\<s\>, -\-run=\<s\>, -m=\<s\>, -\-multirun=\<s\> | Writes down all the products and subsequently calls `<s> <file>` for all written files. Then, deletes the files. | Run your runs lol                                                |   |   |
| -d, --delete, -c, --clear                           | Writes down all the products and then deletes them                                                               | Clears your directory from the billion files created by heracles |   |   |

So if you want to both write AND run the files, use 
`heracles sbatch.sh -r=sbatch env.num_envs=8,16,32 env.name=HalfCheetah,Reacher`. 

### What about seeds?

Yeah that's ok bby we got seeds. You can do it 2 ways.

#### I want each product to be run for N seed that are identitcal product-to-product

Use slurm's ARRAY_ID thingy. Make your `sbatch.sh` to use the ARRAY_ID as the seed.

```bash
#!/bin/bash
#SBATCH --account=ACCOUNT 
#SBATCH --time=6:00:00
#SBATCH --job-name=WOW  #Job name
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16           # CPU cores/threads
#SBATCH --array=1-90

singularity --quiet exec --nv -H ~/projects/ACCOUNT/USER/PROJ/temp/ \
  --env WANDB_MODE="offline",WANDB_API_KEY="LOL_U_THOUGHT",WANDB_WATCH="false" \
  apptainer.sif /YOUR_PROJECT/main.py seed=$SLURM_ARRAY_TASK_ID
```

Then run Heracles as normal: `heracles sbatch.sh -w env.num_envs=8,16,32 env.name=HalfCheetah,Reacher`

#### I want a unique seed per product

Heracles has a unique little util called `__randperm__`. You can use it like so: 
`heracles sbatch.sh -w env.num_envs=8,16,32 seed=__randseed(90)__ env.name=HalfCheetah,Reacher`. Basically
`__randperm(N)__` -> generate N random ints and add them as elements of the cartesian product. For syntactic sugar reasons,
you can use `__randperm__` instead of `__randperm(1)__`.

## That's very cool Charlie thanks for the show-and-tell. How do I install it?

Literally just copy paste the file into your HPC login node or whatever. Or git clone it. 
There's no dependencies except for python and its standard libs.