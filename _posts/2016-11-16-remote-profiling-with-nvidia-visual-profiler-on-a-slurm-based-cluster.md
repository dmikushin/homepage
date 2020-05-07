---
layout: post
title: "Remote profiling with NVIDIA Visual Profiler on a SLURM-based cluster"
tags:
  - Software Engineering
thumbnail_path: blog/2016-11-16-remote-profiling-with-nvidia-visual-profiler-on-a-slurm-based-cluster/nvvp_results.png
---

GPU-equipped clusters are often managed by SLURM job control system. Essentially, developer logs into the frontend node by SSH, builds the application and then queries SLURM for compute node(s) allocation. Once compute nodes are granted, the application is executed on them. In order to debug or profile an application, developer is allowed to SSH from the frontend node to individual compute nodes granted for execution. Now, how this pipeline can play together with NVIDIA Visual Profiler?

![alt text](\assets\img\blog\2016-11-16-remote-profiling-with-nvidia-visual-profiler-on-a-slurm-based-cluster/nvvp_results.png "Logo Title Text 1")

NVIDIA Visual Profiler offers extremely handy feature of remote profiling. That is, having GPU server available over SSH, developer can profile an application on it, collecting visually rich analysis right on his low-end laptop. However, NNVP supports only direct SSH connection to GPU-enabled compute node, which is not possible in case of SLURM. SSH tunneling can also me limited/tricky due to security settings. We do not want also to run NVVP on cluster's frontend over X forwarding, as the performance is usually terrible. So, what else could be done, if we want to connect NVIDIA Visual Profiler to a SLURM-managed GPU compute node?

We have developed and tested on CUDA 8.0 the following approach. Create a file called "nvprof" in your home folder (in this case - /home/dmikushin) and make it executable:

```

#!/bin/sh
#
# NVPROF wrapper for remote NVVP profiling on SLURM-managed compute node
# (c) 2016 Applied Parallel Computing LLC | http://parallel-computing.pro
#
# TODO Adjust CUDA module setting according to your system
#
MODULE="module load cuda/8.0.44"
#
QUIET=-Q
mkdir -p $HOME/nvvp_$USER
ln -sf $HOME/nvvp_$USER /tmp/nvvp_$USER
if [ -z ${SRUN+x} ]; then
if [ ! -z ${PARTITION+x} ]; then
PARTITION="-p $PARTITION"
fi
ME=$0
export SRUN=1
salloc $QUIET $PARTITION srun $QUIET $ME $@
else
NODE=$(hostname)
DEVICE=$(ssh $NODE "nvidia-smi -q -i 0 | grep \"Product Name\" | sed 's/.*\:\s//g'")
echo "Using $DEVICE on node $NODE"
echo "nvprof $@"
ssh $NODE "$MODULE && nvprof $@"
fi
```

This script basically creates a temporary folder in home directory, visible both on frontend and compute nodes, then it allocates a compute node in a given partition (in default partition, if unspecified) and calls itself once again with SRUN=1 environment variable. SRUN-managed script call reports the device and node names and then calls the real nvprof tool from the CUDA toolkit module on compute node over SSH.

NVVP on developer's machine should be configured as shown on the screenshot. The toolkit path should point to the home directory, where the nvprof script above is located. Note the partition name is specified in PARTITION environment variable (leave unspecified to use the default partition).

![alt text](\assets\img\blog\2016-11-16-remote-profiling-with-nvidia-visual-profiler-on-a-slurm-based-cluster/nvvp_slurm_setup.png "Logo Title Text 1")

The presented profiler workflow has a nice effect on efficiency. NVVP session does not lock the compute node busy all the time. Instead, each time NVVP invokes nvprof (device quiery, timeline generation, examining individual kernels, indivudual kernels profiling, etc.), the script allocates a compute node and releases it afterwards. Repeatable allocation of the same class of GPU is achieved by specifying partition name (e.g. hsw_k80 in this case allows to get Tesla K80 on all profiling phases).
