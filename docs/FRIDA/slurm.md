# Slurm

Reservation and management of FRIDA compute resources is based on Slurm ([Simple Linux Utility for Resource Management](https://slurm.schedmd.com/)), a software package for submitting, scheduling, and monitoring jobs on large compute clusters. Slurm governs access to the cluster's resources through three views; account/user association, partition, and QoS (Quality of Service). Accounts link users into groups (based on research lab, project or other shared properties). Partitions link nodes and QoS allow users to control the priority and limits on a job level. Each of these views can impose limits and affect the submitted job's priority.

## Nodes

FRIDA currently consists of one login node and 3 compute nodes with characteristics listed in the table below. Although the login node has Python 3.10 and venv pre-installed, these are provided only to aid in quick scripting, and the login node is not intended for any intensive processing. Refrain from installing user space additions like conda and others. All computationally intensive tasks must be submitted as Slurm jobs via the corresponding Slurm commands. User accounts that fail to adhere to these guidelines will be subject to suspension.

| NODE  | ROLE    | vCPU | MEM   | nGPU | GPU type       |
|-------|--------:|-----:|------:|-----:|---------------:|
| login | login   |   64 | 256GB |    - |              - |
| ana   | compute |  112 |   1TB |    8 | A100 80GB PCIe |
| axa   | compute |  256 |   2TB |    8 | A100 40GB SXM4 |
| ixh   | compute |  224 |   2TB |    8 | H100 80GB HBM3 |

<!--
| NODE | CPU_BRD | CPU_GEN     | CPU_SKU         | GPU_BRD | GPU_GEN | GPU_MEM | GPU_SKU |
|------|--------:|------------:|----------------:|--------:|--------:|--------:|--------------:|
| ana  | AMD     | ZEN3        | EPYC_7453       | A100    | AMPERE  | 80GB    | A100_80GB_PCIE |
| axa  | AMD     | ZEN2        | EPYC_7742       | A100    | AMPERE  | 40GB    | A100_SXM4_40GB |
| ixh  | INTEL   | GOLDEN_COVE | PLATINUM_8480CL | H100    | HOPPER  | 80GB    | H100_80GB_HBM3 |
-->


## Partitions

Within Slurm subsets of compute nodes are organized into partitions. On FRIDA there are two types of partitions, general and private (available to selected research labs or groups based on their co-funding of FRIDA). Interactive jobs can be run only on `dev` partition. Production runs are not permitted in interactive jobs, `dev` partition is thus intended to be used for code development, testing, and debugging only.

| PARTITION | TYPE    | nodes | default time |     max time |                   available gres types |
|-----------|--------:|------:|-------------:|-------------:|---------------------------------------:|
| frida     | general |   all |           4h |           7d | gpu, gpu:A100, gpu:A100_80GB, gpu:H100 |
| dev       | general |   ana |           2h |          12h | shard, gpu, gpu:A100_80GB              |
| _exp_     | general |     - |           2h |           1d | _Planed for Q1 2024_                   |
| cjvt      | private |   axa |           4h |           4d | gpu, gpu:A100                          |
| psuiis    | private |   ana |           4h |           4d | gpu, gpu:A100_80GB                     |

<!--
| PARTITION | TYPE    | nodes | default time |     max time |                   available gres types |   allowd QoS         |
|-----------|--------:|------:|-------------:|-------------:|---------------------------------------:|---------------------:|
| frida     | general |   all |           4h |           7d | gpu, gpu:A100, gpu:A100_80GB, gpu:H100 | normal, cjvt, psuiis |
| dev       | general |   ana |           2h |          12h | shard, gpu, gpu:A100_80GB              | dev                  |
| _exp_     | general |     - |           2h |           1d | _Planed for Q1 2024_                   | dev                  |
| cjvt      | private |   axa |           4h |           4d | gpu, gpu:A100                          | cjvt                 |
| psuiis    | private |   ana |           4h |           4d | gpu, gpu:A100_80GB                     | psuiis               |

### QoS

As a further tool in affecting job submission, Slurm allows users the ability to provide the desired QoS (Quality of Service). Depending on the user's associations only certain QoS levels may be available, similarly depending on the partition only certain QoS levels will be accepted. In contrast to general QoS levels, private QoS levels provide a higher priority but are limited by a monthly quota. Once the sum of wall-times of all jobs submitted using a QoS exhausts the available quota, jobs that specify this QoS will not be accepted until the quota's next reset (monthly). Similarly, jobs will be rejected if they are not able to finish before the quota's exhaustion.

| QOS       | TYPE    | partition | default time |     max time | priority | monthly quota [d-hh:mm] |
|-----------|--------:|----------:|-------------:|-------------:|---------:|------------------------:|
| normal    | general |       any |           4h |           7d |        1 |                       - |
| dev       | general |       dev |           2h |          12h |       10 |                       - |
| cjvt      | private |       any |           4h |           4d |      100 |                 8-02:42 |
| psuiis    | private |       any |           4h |           4d |      100 |                 4-07:50 |
-->

## Shared storage

All FRIDA nodes have access to a limited amount of shared data storage. For performance reasons, it is served by a raid0 backend. Note that FRIDA does not provide automatic backups, these are entirely in the domain of the end user. Also, as of current settings, FRIDA does not enforce shared storage quotas, but this may change in future upgrades. Access permissions on the user folder are set to user only. Depending on the groups a user is a member of, they may have access to multiple workspace folders. These folders have the group special ([SGID](https://www.redhat.com/sysadmin/suid-sgid-sticky-bit)) bit set, so files created within them will automatically have the correct group ownership. All group members will have read and execute rights. Note, however, that group write access is masked, so users who wish to make their files writable by other group members should change permissions by using the `chmod g+w` command. All other security measures dictated by the nature of your data are the responsibility of the end users.

In addition to access to shared storage, compute nodes provide also an even smaller amount of local storage. The amount varies per node and may change with FRIDA's future updates. Local storage is intended as scratch space, the corresponding path is created on a per-job basis at job start and purged as soon as the job ends.

| TYPE      | backend | backups | access  | location                                  | env        | quota        |
|-----------|--------:|--------:|--------:|------------------------------------------:|-----------:|-------------:|
| shared    | raid0   | no      | user    | `/shared/home/$USER`                      | `$HOME`    |            - |
| shared    | raid0   | no      | group   | `/shared/workspace/$SLURM_JOB_ACCOUNT`    | `$WORK`    |            - |
| scratch   | varies  | no      | job     | `/local/scratch/$USER/$SLURM_JOB_ID`      | `$SCRATCH` |            - |

## Usage

In Slurm there are two principal ways of submitting jobs. Interactive and non-interactive, or batch submission. What follows in the next subsections is a quick review of most typical use cases.

Some useful Slurm commands with their typical use case, notes and corresponding Slurm help are displayed in the table below. In the next sections, we will be using these commands to submit and view jobs.

| CMD  | typical use case | notes | help |
|------|:-----------------|:------|-----:|
| `slurm` | view the current status of all resources | this is a custom, FRIDA alias built on top of the more general `sinfo` Slurm command | see Slurm docs on [`sinfo`](https://slurm.schedmd.com/sinfo.html) |
| `salloc` | resource reservation and allocation for interactive use | intended for interactive jobs; upon successful allocation `srun` commands can be used to open a shell with the allocated resources | see Slurm docs on [`salloc`](https://slurm.schedmd.com/salloc.html) |
| `srun` | resource reservation, allocation and execution of supplied command on these resources | a blocking command; with `--pty` can be used for interactive jobs; by appending `&` followed by a `wait` the command can be turned into non-blocking | see Slurm docs on [`srun`](https://slurm.schedmd.com/srun.html) |
| `stunnel` | resource reservation, allocation and setup of a vscode tunnel to these resources | this is a custom, FRIDA alias built on top of the more general `srun --pty` Slurm command; requires a GitHub or Microsoft account for tunnel registration; intended for dvelopment, testing and debugging | see section [code tunnel](#code-tunnel) |
| `sbatch` | resource reservation, allocation and execution of non-interactive batch job on these resources | asynchronous execution of sbatch script; when combined with `srun` multiple sub-steps become possible | see Slurm docs on [`sbatch`](https://slurm.schedmd.com/sbatch.html) |
| `squeue` | view the current queue status | on large clusters the output can be large; filter can be applied to limit output to specific partition by `-p {partition}` | see Slurm docs on [`squeue`](https://slurm.schedmd.com/squeue.html) |
| `sprio`  | view the current priority status on queue | this is a custom FRIDA alias built on top of the more general `sprio` Slurm command | see Slurm docs on [`sprio`](https://slurm.schedmd.com/sprio.html) |
| `scancel` | cancel a running job | | see Slurm docs on [`scancel`](https://slurm.schedmd.com/scancel.html) |
| `scontrol show job` | show detailed information of a job | | see Slurm docs on [`scontrol`](https://slurm.schedmd.com/scontrol.html) |

<!-- | `slimits` | view limits imposed on current user | this is a custom, FRIDA alias built on top of the more general `sacctmgr` Slurm command | see Slurm docs on [`sacctmgr`](https://slurm.schedmd.com/sacctmgr.html) |
| `sqos` | view the current status of resources | this is a custom, FRIDA alias built on top of the more general `sacctmgr` Slurm command | see Slurm docs on [`sacctmgr`](https://slurm.schedmd.com/sacctmgr.html) | --->

### Interactive sessions

Slurm in general provides two ways of running interactive jobs; either via the `salloc` or via the `srun --pty` command. The former will first reserve the resources, which you then explicitly use via the latter, while the latter can be used to do the reservation and execution in a single step.

For example, the following snippet shows how to allocate resources on `dev` partition with default parameters (2 vCPU, 8GB RAM) and then start a bash shell on the allocated resources. Done that it shows how to check what was allocated, then exits the bash shell and finally releases the allocated resources.
```bash
ilb@login-frida:~$ salloc -p dev
salloc: Granted job allocation 498
salloc: Waiting for resource configuration
salloc: Nodes ana are ready for job

ilb@login-frida:~ [interactive]$ srun --pty bash

ilb@ana-frida:~ [interactive]$ scontrol show job $SLURM_JOB_ID | grep AllocTRES
   AllocTRES=cpu=2,mem=8G,node=1,billing=2

ilb@ana-frida:~ [interactive]$ exit
exit

ilb@login-frida:~ [interactive]$ exit
exit
salloc: Relinquishing job allocation 498
salloc: Job allocation 498 has been revoked.

ilb@login-frida:~$
```

The following snipped, on the other hand, allocates resources on `dev` partition and starts a bash shell in one call, this time requesting for 32 vCPU cores and 32 GB of RAM. It then checks what was allocated, and finally exits from the bash shell jointly releasing the allocated resources.
```bash
ilb@login-frida:~$ srun -p dev -c32 --mem=32G --pty bash

ilb@ana-frida:~ [bash]$ scontrol show job $SLURM_JOB_ID | grep AllocTRES
   AllocTRES=cpu=32,mem=32G,node=1,billing=32

ilb@ana-frida:~ [bash]$ exit
exit

ilb@login-frida:~$
```

Access to the allocated resources (nodes) can be achieved also by invoking `ssh`; however, note that then the [Slurm-defined environment variables](https://slurm.schedmd.com/srun.html#lbAJ) will not be available to you. The auto inclusion of the corresponding environment variables is an added benefit of accessing the allocated resources via `srun --pty`. With multi-node resource allocations use `-w` to specify the node on which you wish to start a shell.
```bash
ilb@login-frida:~$ srun -p dev --pty bash

ilb@ana-frida:~ [bash]$ echo $SLURM_
$SLURM_CLUSTER_NAME       $SLURM_JOB_ID             $SLURM_JOB_QOS            $SLURM_SUBMIT_HOST
$SLURM_CONF               $SLURM_JOB_NAME           $SLURM_NNODES             $SLURM_TASKS_PER_NODE
$SLURM_JOBID              $SLURM_JOB_NODELIST       $SLURM_NODELIST
$SLURM_JOB_ACCOUNT        $SLURM_JOB_NUM_NODES      $SLURM_NODE_ALIASES
$SLURM_JOB_CPUS_PER_NODE  $SLURM_JOB_PARTITION      $SLURM_SUBMIT_DIR

ilb@ana-frida:~ [bash]$ exit
exit

ilb@login-frida:~$
```

Note also that when you have multiple jobs running on a single node `ssh` will always open a shell with the resources allocated by the last submitted job. The `srun` command on the other hand provides two parameters through which one can specify the desire to not start a new job, but open an additional shell in an already running job. For example, assuming a job with id `500` is already running, the next snippet shows how to open a second bash shell in the same job.
```bash
ilb@login-frida:~$ srun --overlap --jobid=500 --pty bash

ilb@ana-frida:~ [bash]$ echo $SLURM_JOB_ID
500

ilb@ana-frida:~ [bash]$ exit
exit

ilb@login-frida:~$
```

#### Running jobs in containers

FRIDA was set up by following [deepops](https://github.com/NVIDIA/deepops), an open-source infrastructure automation tool by NVIDIA. Our setup is focused on supporting ML/AI training tasks, and based on current trends of containerization even in HPC systems, it is also purely container-based (modules are not supported). Here we opted for [Enroot](https://github.com/NVIDIA/enroot) an extremely lightweight container runtime capable of turning traditional container/OS images into unprivileged sandboxes. An added benefit of Enroot is its tight integration into Slurm commands via the [Pyxis](https://github.com/NVIDIA/pyxis) plugin, thus providing a very good user experience.

Regardless if the Enroot/Pyxis bundle turns containers/OS images into unprivileged sandboxes, automatic root remapping allows users to extend and update the imported images with ease. The changes can be retained for the duration of a job or exported to a squashfs file for cross-job retention. Some of the most commonly used Pyxis-provided `srun` parameters are listed in the table below. For further details on all available parameters consult the official [Pyxis documentation](https://github.com/NVIDIA/pyxis/wiki/Usage), while information on how to set up credentials that enable importing containers from private container registries can be found in the official [Enroot documentation](https://github.com/NVIDIA/enroot/blob/master/doc/cmd/import.md).

| <div style="width:190px">srun parameter</div> | <div style="width:290px">format</div> | notes                                                               |
|---------------------------|--------|---------------------------------------------------------------------|
| `--container-image`       | `[USER@][REGISTRY#]IMAGE[:TAG]\|PATH` | container image to load, path must point to a squashfs file |
| `--container-name`        | `NAME` | name the container for easier access to running containters; can be used to change container contents without saving to a squashfs file, but note that non-running named containers are kept only within a `salloc` or `sbatch` allocation |
| `--container-save`        | `PATH` | save the container state to a squashfs file                         |
| `--container-mount-home`  |        | bind mount the user home; not mounted by default to prevent local config collisions |
| `--container-mounts`      | `SRC:DST[:FLAGS][,SRC:DST...]` | bind points to mount into the container; `$SCRATCH` is auto-mounted     |

For example, the following snippet will check the OS release on `dev` partition, node `ana`, then start a bash shell in a CentOS container on the same node and recheck. In this case, the CentOS image tagged latest will be pulled from the official [DockerHub](https://hub.docker.com) page.
``` bash
ilb@login-frida:~$ srun -p dev -w ana bash -c 'echo "$HOSTNAME: `grep PRETTY /etc/os-release`"'
ana: PRETTY_NAME="Ubuntu 22.04.3 LTS"

ilb@login-frida:~$ srun -p dev -w ana --container-image=centos --pty bash
pyxis: importing docker image: centos
pyxis: imported docker image: centos
[root@ana /]# echo "$HOSTNAME: `grep PRETTY /etc/os-release`"
ana: PRETTY_NAME="CentOS Linux 8"

[root@ana /]# exit
exit

ilb@login-frida:~$
```

The following snippet allocates resources on `dev` partition, then starts an `ubuntu:20.04` container and checks if `file` utility exists. The command results in an error indicating that the utility is not present. It then creates a named container that is based on `ubuntu:20.04` and installs the utility. Then it rechecks if the utility exists in the named container. The snippet ends by releasing the resources, which jointly purges the named container.
```bash
ilb@login-frida:~$ salloc -p dev
salloc: Granted job allocation 526
salloc: Waiting for resource configuration
salloc: Nodes ana are ready for job

ilb@login-frida:~ [interactive]$ srun --container-image=ubuntu:20.04 which file
pyxis: importing docker image: ubuntu:20.04
pyxis: imported docker image: ubuntu:20.04
srun: error: ana: task 0: Exited with exit code 1

ilb@login-frida:~ [interactive]$ srun --container-image=ubuntu:20.04 --container-name=myubuntu sh -c 'apt-get update && apt-get install -y file'
pyxis: importing docker image: ubuntu:20.04
pyxis: imported docker image: ubuntu:20.04
Get:1 http://security.ubuntu.com/ubuntu focal-security InRelease [114 kB]
Get:2 http://archive.ubuntu.com/ubuntu focal InRelease [265 kB]
Get:3 http://security.ubuntu.com/ubuntu focal-security/main amd64 Packages [3238 kB]
Get:4 http://security.ubuntu.com/ubuntu focal-security/multiverse amd64 Packages [29.3 kB]
Get:5 http://security.ubuntu.com/ubuntu focal-security/restricted amd64 Packages [3079 kB]
Get:6 http://security.ubuntu.com/ubuntu focal-security/universe amd64 Packages [1143 kB]
Get:7 http://archive.ubuntu.com/ubuntu focal-updates InRelease [114 kB]
Get:8 http://archive.ubuntu.com/ubuntu focal-backports InRelease [108 kB]
Get:9 http://archive.ubuntu.com/ubuntu focal/main amd64 Packages [1275 kB]
Get:10 http://archive.ubuntu.com/ubuntu focal/multiverse amd64 Packages [177 kB]
Get:11 http://archive.ubuntu.com/ubuntu focal/restricted amd64 Packages [33.4 kB]
Get:12 http://archive.ubuntu.com/ubuntu focal/universe amd64 Packages [11.3 MB]
Get:13 http://archive.ubuntu.com/ubuntu focal-updates/main amd64 Packages [3726 kB]
Get:14 http://archive.ubuntu.com/ubuntu focal-updates/restricted amd64 Packages [3228 kB]
Get:15 http://archive.ubuntu.com/ubuntu focal-updates/multiverse amd64 Packages [32.0 kB]
Get:16 http://archive.ubuntu.com/ubuntu focal-updates/universe amd64 Packages [1439 kB]
Get:17 http://archive.ubuntu.com/ubuntu focal-backports/main amd64 Packages [55.2 kB]
Get:18 http://archive.ubuntu.com/ubuntu focal-backports/universe amd64 Packages [28.6 kB]
Fetched 29.4 MB in 3s (9686 kB/s)
Reading package lists...
Reading package lists...
Building dependency tree...
Reading state information...
The following additional packages will be installed:
  libmagic-mgc libmagic1
The following NEW packages will be installed:
  file libmagic-mgc libmagic1
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 317 kB of archives.
After this operation, 6174 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu focal/main amd64 libmagic-mgc amd64 1:5.38-4 [218 kB]
Get:2 http://archive.ubuntu.com/ubuntu focal/main amd64 libmagic1 amd64 1:5.38-4 [75.9 kB]
Get:3 http://archive.ubuntu.com/ubuntu focal/main amd64 file amd64 1:5.38-4 [23.3 kB]
debconf: delaying package configuration, since apt-utils is not installed
Fetched 317 kB in 1s (345 kB/s)
Selecting previously unselected package libmagic-mgc.
(Reading database ... 4124 files and directories currently installed.)
Preparing to unpack .../libmagic-mgc_1%3a5.38-4_amd64.deb ...
Unpacking libmagic-mgc (1:5.38-4) ...
Selecting previously unselected package libmagic1:amd64.
Preparing to unpack .../libmagic1_1%3a5.38-4_amd64.deb ...
Unpacking libmagic1:amd64 (1:5.38-4) ...
Selecting previously unselected package file.
Preparing to unpack .../file_1%3a5.38-4_amd64.deb ...
Unpacking file (1:5.38-4) ...
Setting up libmagic-mgc (1:5.38-4) ...
Setting up libmagic1:amd64 (1:5.38-4) ...
Setting up file (1:5.38-4) ...
Processing triggers for libc-bin (2.31-0ubuntu9.12) ...


ilb@login-frida:~ [interactive]$ srun --container-name=myubuntu which file
/usr/bin/file

ilb@login-frida:~ [interactive]$ exit
exit
salloc: Relinquishing job allocation 526
salloc: Job allocation 526 has been revoked.

ilb@login-frida:~$
```

Using named containers becomes handy on occasions when one needs to open a second shell inside the same container (within the same job with the same resources). Assuming a job with id `527` is already running, that it was created with the parameter `--container-name=myubuntu`, and that it installed the `file` utility. The next snippet shows how to open a second bash shell in the same container.
```bash
ilb@login-frida:~$ srun --overlap --jobid=527 --container-name=myubuntu --pty bash

root@ana:/# which file
/usr/bin/file

root@ana:/# exit
exit

ilb@login-frida:~$
```

Without the use of named containers, the task becomes more challenging as one needs to first start an overlapping shell on the node, find out the PID of the enroot container in question, and then use enroot commands to start a bash shell in that container. The following snippet shows an example.
```bash
ilb@login-frida:~$ srun --overlap --jobid=527 --pty bash

ilb@ana-frida:~ [bash]$ enroot list -f
NAME               PID    COMM  STATE  STARTED  TIME   MNTNS       USERNS      COMMAND
pyxis_527_527.0  11229  bash  Ss+    20:35    00:23  4026538269  4026538268  /usr/bin/bash

ilb@ana-frida:~ [bash]$ enroot exec 11229 bash

root@ana:/# which file
/usr/bin/file

root@ana:/# exit
exit

ilb@login-frida:~$
```

Some containers are big and memory-hungry. One such example is Pytorch, whose latest containers are very large and require large amounts of memory to start. The following snippet shows the output of a failed attempt to start the `pytorch:23.08` container with just 2 vCPU and 8GB of RAM, and a successful one with 32GB.
```bash
ilb@login-frida:~$ srun -p dev --container-image=nvcr.io#nvidia/pytorch:23.08-py3 --pty bash
pyxis: importing docker image: nvcr.io#nvidia/pytorch:23.08-py3
slurmstepd: error: pyxis: child 12455 failed with error code: 137
slurmstepd: error: pyxis: failed to import docker image
slurmstepd: error: pyxis: printing enroot log file:
slurmstepd: error: pyxis:     [INFO] Querying registry for permission grant
slurmstepd: error: pyxis:     [INFO] Authenticating with user: <anonymous>
slurmstepd: error: pyxis:     [INFO] Authentication succeeded
slurmstepd: error: pyxis:     [INFO] Fetching image manifest list
slurmstepd: error: pyxis:     [INFO] Fetching image manifest
slurmstepd: error: pyxis:     [INFO] Found all layers in cache
slurmstepd: error: pyxis:     [INFO] Extracting image layers...
slurmstepd: error: pyxis:     [INFO] Converting whiteouts...
slurmstepd: error: pyxis:     [INFO] Creating squashfs filesystem...
slurmstepd: error: pyxis:     /usr/lib/enroot/docker.sh: line 337: 12773 Killed                  MOUNTPOINT="${PWD}/rootfs" enroot-mksquashovlfs "0:$(seq -s: 1 "${#layers[@]}")" "${filename}" ${timestamp[@]+"${timestamp[@]}"} -all-root ${TTY_OFF+-no-progress} -processors "${ENROOT_MAX_PROCESSORS}" ${ENROOT_SQUASH_OPTIONS} 1>&2
slurmstepd: error: pyxis: couldn't start container
slurmstepd: error: spank: required plugin spank_pyxis.so: task_init() failed with rc=-1
slurmstepd: error: Failed to invoke spank plugin stack
slurmstepd: error: Detected 1 oom_kill event in StepId=1054.0. Some of the step tasks have been OOM Killed.
srun: error: ana: task 0: Out Of Memory

ilb@login-frida:~$ srun -p dev --mem=32G --container-image=nvcr.io#nvidia/pytorch:23.08-py3 --pty bash
pyxis: importing docker image: nvcr.io#nvidia/pytorch:23.08-py3
pyxis: imported docker image: nvcr.io#nvidia/pytorch:23.08-py3

root@ana:/workspace# echo $NVIDIA_PYTORCH_VERSION
23.08

root@ana:/workspace# exit
exit

ilb@login-frida:~$
```

<!--
##### Private container registries

https://github.com/NVIDIA/enroot/blob/fb267cbebceced24556e05c7661fbd9a958f5540/doc/cmd/import.md#description
-->

#### Requesting GPUs

All FRIDA compute nodes provide GPUs, but they differ in provider, architecture, and available memory. In Slurm, a request for allocation of GPU resources can be done using either by specifying the `--gpus=[type:]{count}` or `--gres=gpu[:type][:{count}]` parameter, as shown in the snippet below. The available GPU types depend on the selected partition, whose available GPU types in turn depend on the nodes that constitute the partition (see section [partitions](#partitions)).

```bash
ilb@login-frida:~$ srun -p dev --gres=gpu nvidia-smi -L
GPU 0: NVIDIA A100 80GB PCIe (UUID: GPU-845e3442-e0ca-a376-a3de-50e4cb7fd421)

ilb@login-frida:~$ srun -p dev --gpus=A100_80GB:4 nvidia-smi -L
GPU 0: NVIDIA A100 80GB PCIe (UUID: GPU-845e3442-e0ca-a376-a3de-50e4cb7fd421)
GPU 1: NVIDIA A100 80GB PCIe (UUID: GPU-7f73bd51-df5b-f0ba-2cf2-5dadbfa297e1)
GPU 2: NVIDIA A100 80GB PCIe (UUID: GPU-d666e6d2-5265-29ae-9a6f-d8772807d34f)
GPU 3: NVIDIA A100 80GB PCIe (UUID: GPU-fd552b1f-e767-edd6-5cc0-69514f1748d2)
```

When the nature of a job is such that it does not require exclusive access to a full GPU, the `dev` partition on FRIDA allows also a partial allocation of GPUs. This can be done via the `--gres=shard[:{count}]` parameter, where the value `{count}` can be interpreted as the amount of requested GPU memory in GB. For example, the following snippet allocates a 15GB slice on an otherwise 80GB GPU.

```bash
ilb@login-frida:~$ srun -p dev --gres=shard:15 nvidia-smi -L
GPU 0: NVIDIA A100 80GB PCIe (UUID: GPU-845e3442-e0ca-a376-a3de-50e4cb7fd421)
```

!!! Warning
    Partial GPU allocation via `shard` reserves the GPUs in a non-exclusive mode and as such allows for sharing GPUs. However, currently, Slurm has no viable techniques to enforce these limits, and thus jobs, even if they request only a portion of a GPU, are granted access to the full GPU. Allocation with `shard` then needs to be interpreted as _a promise_ that the job will not consume more GPU memory than what it requested. Failure to do so may result in the failure of the user's job as well as the failure of jobs of other users that happen to share the same GPU. Ensuring that the job does not consume more GPU memory than what it requested for, is thus mandatory, and entirely under the responsibility of the user who submitted the job. In case your job is based on Tensorflow or Pytorch there exist [approaches to GPU memory management](https://wiki.ncsa.illinois.edu/display/ISL20/Managing+GPU+memory+when+using+Tensorflow+and+Pytorch) that you can use to ensure the job does not consume more than what it requested. In addition, Pytorch version 1.8 introduced a special function for limiting process GPU memory, see  [`set_per_proces_memory_fraction`](https://github.com/pytorch/pytorch/blob/e44b2b72bd4ccecf9c2f6c18d09c11eff446b5a3/torch/cuda/memory.py#L75-L99)). For further details on sharding consult the [Slurm official documentation](https://slurm.schedmd.com/gres.html#Sharding). User accounts that fail to adhere to these guidelines will be subject to suspension.


#### Code tunnel

When an interactive session is intended for code development, testing, and/or debugging, many a time it is desirable to work with a suitable IDE. The requirement of resource allocation via Slurm and the lack of any toolsets on bare-metal hosts might seem too much of an added complexity, but in reality, there is a very elegant solution by using Visual Studio Code in combination with the Remote Development Extension (via `code_tunnel`). Running `code_tunnel` will allow you to use VSCode to connect directly to the container that is running on the compute node that was assigned to your job. Combined with root remapping this has the added benefit of a user experience that feels like working with your own VM.

On every run of `code_tunnel` you will need to register the tunnel with your GitHub or Microsoft account; this interaction requires the `srun` command to be run with the parameter `--pty`, and for this reason, a suitable alias command named `stunnel` was setup. Once registered you will be able to access the compute node (while `code_tunnel` is running) either in a browser by visiting [https://vscode.dev](https://vscode.dev) or with a desktop version of VSCode via the Remote Explorer (part of the [Remote Development Extension pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)). In both cases, the 'Remotes (Tunnels/SSH)' list in the Remote Explorer pane should contain a list of tunnels to FRIDA (named `frida-{job-name}`) that you have registered and also denote which of these are currently online (with your job running). In the browser, it is also possible to connect directly to a workspace on the node (on which the `code_tunnel` job is running). Simply visit the URL of the form `https://vscode.dev/tunnel/frida-{job-name}[/{path-to-workspace}]`. If you wish to close the tunnel before the job times out press `Ctrl-C` in the terminal where you started the job.

```bash
ilb@login-frida:~$ stunnel -c32 --mem=48G --gres=shard:20 --container-image=nvcr.io#nvidia/pytorch:23.10-py3 --job-name torch-debug
# srun -p dev -c32 --mem=48G --gres=shard:20 --container-image=nvcr.io#nvidia/pytorch:23.10-py3 --job-name torch-debug --pty code_tunnel
pyxis: importing docker image: nvcr.io#nvidia/pytorch:23.10-py3
pyxis: imported docker image: nvcr.io#nvidia/pytorch:23.10-py3
Tue Nov 28 21:31:40 UTC 2023
*
* Visual Studio Code Server
*
* By using the software, you agree to
* the Visual Studio Code Server License Terms (https://aka.ms/vscode-server-license) and
* the Microsoft Privacy Statement (https://privacy.microsoft.com/en-US/privacystatement).
*
✔ How would you like to log in to Visual Studio Code? · Github Account
To grant access to the server, please log into https://github.com/login/device and use code 17AC-E3AE
[2023-11-28 21:31:15] info Creating tunnel with the name: frida-torch-debug

Open this link in your browser https://vscode.dev/tunnel/frida-torch-debug/workspace

[2023-11-28 21:31:41] info [tunnels::connections::relay_tunnel_host] Opened new client on channel 2
[2023-11-28 21:31:41] info [russh::server] wrote id
[2023-11-28 21:31:41] info [russh::server] read other id
[2023-11-28 21:31:41] info [russh::server] session is running
[2023-11-28 21:31:43] info [rpc.0] Checking /local/scratch/root/590/.vscode/cli/ana/servers/Stable-1a5daa3a0231a0fbba4f14db7ec463cf99d7768e/log.txt and /local/scratch/root/590/.vscode/cli/ana/servers/Stable-1a5daa3a0231a0fbba4f14db7ec463cf99d7768e/pid.txt for a running server...
[2023-11-28 21:31:43] info [rpc.0] Downloading Visual Studio Code server -> /tmp/.tmpbbxB9w/vscode-server-linux-x64.tar.gz
[2023-11-28 21:31:45] info [rpc.0] Starting server...
[2023-11-28 21:31:45] info [rpc.0] Server started
^C[2023-11-28 21:37:45] info Shutting down: Ctrl-C received
[2023-11-28 21:37:45] info [rpc.0] Disposed of connection to running server.
Tue Nov 28 21:37:45 UTC 2023

ilb@login-frida:~$
```

#### Keeping interactive jobs alive

Interactive sessions are typically run in the foreground, require a terminal, and last for the duration of the SSH session, or until the Slurm reservation times out (whichever comes first). On flaky internet connections, this can become a problem, where a lost connection might lead to a stopped job. One remedy is to use `tmux` or `screen` which allows for running interactive jobs in detached mode.

### Batch jobs

Once development, testing, and/or debugging is complete, at least in ML/AI training tasks, the datasets typically become such that jobs last much longer than a normal SSH session. They can also be safely run without any interaction and also without a terminal. In Slurm parlance these types of jobs are termed batch jobs. A batch job is a shell script (typically marked by a `.sbatch` extension), whose header provides Slurm parameters that specify the resources needed to run the job. For details of all available parameters consult the official [Slurm documentation](https://slurm.schedmd.com/sbatch.html). Below is a very brief deep-dive introduction to Slurm batch jobs.

Assume a toy MNIST training script based on Pytorch and Lightning, named `tarin.py`.
``` py title="train.py"
import os
import torch
from torch import nn
import torch.nn.functional as F
from torchvision import transforms
from torchvision.datasets import MNIST
from torch.utils.data import DataLoader, random_split
from torchmetrics.functional import accuracy
import lightning.pytorch as L
from pathlib import Path


class LitMNIST(L.LightningModule):
    def __init__(self, data_dir=os.getcwd(), hidden_size=64, learning_rate=2e-4):
        super().__init__()

        # We hardcode dataset specific stuff here.
        self.data_dir = data_dir
        self.num_classes = 10
        self.dims = (1, 28, 28)
        channels, width, height = self.dims
        self.transform = transforms.Compose(
            [
                transforms.ToTensor(),
                transforms.Normalize((0.1307,), (0.3081,)),
            ]
        )

        self.hidden_size = hidden_size
        self.learning_rate = learning_rate

        # Build model
        self.model = nn.Sequential(
            nn.Flatten(),
            nn.Linear(channels * width * height, hidden_size),
            nn.ReLU(),
            nn.Dropout(0.1),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Dropout(0.1),
            nn.Linear(hidden_size, self.num_classes),
        )

    def forward(self, x):
        x = self.model(x)
        return F.log_softmax(x, dim=1)

    def training_step(self, batch):
        x, y = batch
        logits = self(x)
        loss = F.nll_loss(logits, y)
        return loss

    def validation_step(self, batch, batch_idx):
        x, y = batch
        logits = self(x)
        loss = F.nll_loss(logits, y)
        preds = torch.argmax(logits, dim=1)
        acc = accuracy(preds, y, task="multiclass", num_classes=10)
        self.log("val_loss", loss, prog_bar=True)
        self.log("val_acc", acc, prog_bar=True)

    def configure_optimizers(self):
        optimizer = torch.optim.Adam(self.parameters(), lr=self.learning_rate)
        return optimizer

    def prepare_data(self):
        # download
        MNIST(self.data_dir, train=True, download=True)
        MNIST(self.data_dir, train=False, download=True)

    def setup(self, stage=None):
        # Assign train/val datasets for use in dataloaders
        if stage == "fit" or stage is None:
            mnist_full = MNIST(self.data_dir, train=True, transform=self.transform)
            self.mnist_train, self.mnist_val = random_split(mnist_full, [55000, 5000])

        # Assign test dataset for use in dataloader(s)
        if stage == "test" or stage is None:
            self.mnist_test = MNIST(
                self.data_dir, train=False, transform=self.transform
            )

    def train_dataloader(self):
        return DataLoader(self.mnist_train, batch_size=4)

    def val_dataloader(self):
        return DataLoader(self.mnist_val, batch_size=4)

    def test_dataloader(self):
        return DataLoader(self.mnist_test, batch_size=4)


# Some prints that might be useful
print("SLURM_NTASKS =", os.environ["SLURM_NTASKS"])
print("SLURM_TASKS_PER_NODE =", os.environ["SLURM_TASKS_PER_NODE"])
print("SLURM_GPUS_PER_NODE =", os.environ.get("SLURM_GPUS_PER_NODE", "N/A"))
print("SLURM_CPUS_PER_NODE =", os.environ.get("SLURM_CPUS_PER_NODE", "N/A"))
print("SLURM_NNODES =", os.environ["SLURM_NNODES"])
print("SLURM_NODELIST =", os.environ["SLURM_NODELIST"])
print(
    "PL_TORCH_DISTRIBUTED_BACKEND =",
    os.environ.get("PL_TORCH_DISTRIBUTED_BACKEND", "nccl"),
)


# Explicitly specify the process group backend if you choose to
ddp = L.strategies.DDPStrategy(
    process_group_backend=os.environ.get("PL_TORCH_DISTRIBUTED_BACKEND", "nccl")
)

# setup checkpointing
checkpoint_callback = L.callbacks.ModelCheckpoint(
    dirpath="./checkpoints/",
    filename="mnist_{epoch:02d}",
    every_n_epochs=1,
    save_top_k=-1,
)

# resume from checkpoint
existing_checkpoints = list(Path('./checkpoints/').iterdir()) if Path('./checkpoints/').exists() else []
last_checkpoint = str(sorted(existing_checkpoints, key=os.path.getmtime)[-1]) if len(existing_checkpoints)>0 else None

# model
model = LitMNIST()
# train model
trainer = L.Trainer(
    callbacks=[checkpoint_callback],
    max_epochs=100,
    accelerator="gpu",
    devices=-1,
    num_nodes=int(os.environ["SLURM_NNODES"]),
    strategy=ddp,
)
trainer.fit(model,ckpt_path=last_checkpoint)
```

To run this training script on a single node with a single GPU of type NVIDIA A100 40GB SXM4 one needs to prepare the following script (`train_1xA100.sbatch`). The script will reserve 16 vCPU, 32GB RAM, and 1 NVIDIA A100 40GB SXM4 for 20 minutes. It will also give the job a name, for easier finding when listing the partition queue status. As part of the actual job steps, it will start a `pytorch:23.08-py3` container, mount the current path, install `lightning:2.1.2`, and finally run the training, all with one single `srun` command. Without additional parameters, the `srun` command will use all of the allocated resources. The standard output of the script will be saved into a file named `slurm-{SLURM_JOB_ID}.out`.
``` bash title="train_1xA100.sbatch"
#!/bin/bash
#SBATCH --job-name=mnist-demo
#SBATCH --time=20:00
#SBATCH --gres=gpu:A100
#SBATCH --cpus-per-task=16
#SBATCH --mem=32G

srun \
  --container-image nvcr.io#nvidia/pytorch:23.08-py3 \
  --container-mounts ${PWD}:${PWD} \
  --container-workdir ${PWD} \
  bash -c 'pip install lightning==2.1.2; python3 train.py'
```

The Slurm job is then executed with the following command. It runs in the background without any terminal output, but its progress can be monitored by viewing the corresponding output file.
```bash
ilb@login-frida:~/mnist-demo$ sbatch -p frida train_1x100.sbatch
Submitted batch job 600

ilb@login-frida:~/mnist-demo$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
               600     frida mnist-de      ilb  R       0:35      1 axa

ilb@login-frida:~/mnist-demo-1$ tail -f slurm-600.out
pyxis: importing docker image: nvcr.io#nvidia/pytorch:23.08-py3
pyxis: imported docker image: nvcr.io#nvidia/pytorch:23.08-py3
Looking in indexes: https://pypi.org/simple, https://pypi.ngc.nvidia.com
Collecting lightning==2.1.2
  Obtaining dependency information for lightning==2.1.2 from https://files.pythonhosted.org/packages/9e/8a/9642fdbdac8de47d68464ca3be32baca3f70a432aa374705d6b91da732eb/lightning-2.1.2-py3-none-any.whl.metadata
  Downloading lightning-2.1.2-py3-none-any.whl.metadata (61 kB)
     ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 61.8/61.8 kB 2.8 MB/s eta 0:00:00
...
```

To run the same training script on more GPUs one needs to change the `SBATCH` line with `--gres` and add a line that defines the number of parallel tasks Slurm should start (`--tasks`). For example, to run on 2 GPUs of type NVIDIA H100 80GB HBM3 and this time save the standard output in a file named `{SLURM_JOB_NAME}_{SLURM_JOB_ID}.out`, one needs to prepare the following batch script.
``` bash title="train_2xH100.sbatch"
#!/bin/bash
#SBATCH --job-name=mnist-demo
#SBATCH --partition=frida
#SBATCH --time=20:00
#SBATCH --gres=gpu:H100:2
#SBATCH --tasks=2
#SBATCH --cpus-per-task=16
#SBATCH --mem=32G
#SBATCH --output=%x_%j.out

srun \
  --container-image nvcr.io#nvidia/pytorch:23.08-py3 \
  --container-mounts ${PWD}:${PWD} \
  --container-workdir ${PWD} \
  bash -c 'pip install lightning==2.1.2; python3 train.py'
```

Since the script also preselects the partition on which to run it is not necesary to provide it when executing the `sbatch` command.
```bash
ilb@login-frida:~/mnist-demo$ sbatch train_2xH100.sbatch
Submitted batch job 601

ilb@login-frida:~/mnist-demo$
```

A more advanced script that allows for a custom organization of experiment results, autoarchival of ran scripts, can be run single-node single-GPU or multi-node multi-GPU, modifies the required container on the fly, and is capable of auto-re-queueing itself and continuation, can be seen in the following listing. The process of submitting the job remains the same as before.
``` bash title="train_2x2.sbatch"
#!/bin/bash
#SBATCH --job-name=mnist-demo # provide a job name
#SBATCH --account=lpt # povide the account which determines the resource limits
#SBATCH --partition=frida # provide the partition from where the resources should be allocated
#SBATCH --time=20:00 # provide the time you require the resources for
#SBATCH --nodes=2 # provide the number of nodes you are requesting
#SBATCH --ntasks-per-node=2 # provide the number of tasks to run on a node; slurm runs this many replicas of the subsequent commands
#SBATCH --gpus-per-node=2 # provide the number of GPUs that is required per node; avoid using --gpus-per-tasks as there are issues with allocation as well as inter-gpu communication - see https://github.com/NVIDIA/pyxis/issues/73
#SBATCH --cpus-per-task=4 # provide the number of CPUs that is required per task; be mindful that different nodes have different numbers of available CPUs
#SBATCH --mem-per-cpu=4G # provide the required memory per CPU that is required per task; be mindful that different nodes have different amounts of memory
#SBATCH --output=/dev/null # we do not care for the log of this setup script, we redirect the log of the execution srun
#SBATCH --signal=B:USR2@90 # instruct Slurm to send a notification 90 seconds prior to running out of time
# SBATCH --signal=B:TERM@60 # this is to enable the graceful shutdown of the job, see https://slurm.schedmd.com/sbatch.html#lbAH and lightning auto-resubmit, see https://lightning.ai/docs/pytorch/stable/cloudsy/cluster_advanced.html#enable-auto-wall-time-resubmissions

# get the name of this script
if [ -n "${SLURM_JOB_ID:-}" ] ; then
  SBATCH=$(scontrol show job "$SLURM_JOB_ID" | awk -F= '/Command=/{print $2}')
else
  SBATCH=$(realpath "$0")
fi

# allow passing a version, for autorequeue
if [ $# -gt 1 ] || [[ "$0" == *"help" ]] || [ -z "${SLURM_JOB_ID:-}" ] || [[ "$1" != "--version="* ]]; then
  echo -e "\nUsage: sbatch ${SBATCH##*$PWD/} [--version=<version>] \n"
  exit 1
fi

# convert the --key=value arguments to variables
for argument in "$@"
do
  if [[ $argument == *"="* ]]; then
    key=$(echo $argument | cut -f1 -d=)
    value=$(echo $argument | cut -f2 -d=)
    if [[ $key == *"--"* ]]; then
        v="${key/--/}"
        declare ${v,,}="${value}"
   fi
  fi
done

# setup signal handler for autorequeue
if [ "${version}" ]; then
  sig_handler() {
    echo -e "\n\n$(date): Signal received, requeuing...\n\n" >> ${EXPERIMENT_DIR}/${EXPERIMENT_NAME}/${version}/${RUN_NAME}.1.txt
    scontrol requeue ${SLURM_JOB_ID}
  }
  trap 'sig_handler' SIGUSR2
fi

# time of running script
DATETIME=`date "+%Y%m%d-%H%M"`
version=${version:-$DATETIME} # if version is not set, use DATETIME as default

# work dir
WORK_DIR=${HOME}/mnist

# experiment name
EXPERIMENT_NAME=demo

# experiment dir
EXPERIMENT_DIR=${WORK_DIR}/exp
mkdir -p ${EXPERIMENT_DIR}/${EXPERIMENT_NAME}/${version}

# set run name
if [ "${version}" == "${DATETIME}" ]; then
  RUN_NAME=${version}
else
  RUN_NAME=${version}_R${DATETIME}
fi

# archive this script
cp -rp ${SBATCH} ${EXPERIMENT_DIR}/${EXPERIMENT_NAME}/${version}/${RUN_NAME}.sbatch

# prepare execution script name
SCRIPT=${EXPERIMENT_DIR}/${EXPERIMENT_NAME}/${version}/${RUN_NAME}.sh
touch $SCRIPT
chmod a+x $SCRIPT

IS_DISTRIBUTED=$([ 1 -lt $SLURM_JOB_NUM_NODES ] && echo " distributed over $SLURM_JOB_NUM_NODES nodes" || echo " on 1 node")

# prepare the execution script content
echo """#!/bin/bash

# using `basename $SBATCH` -> $RUN_NAME.sbatch, running $SLURM_NPROCS tasks$IS_DISTRIBUTED

# prepare sub-script for debug outputs
echo -e \"\"\"
# starting at \$(date)
# running process \$SLURM_PROCID on \$SLURMD_NODENAME
\$(nvidia-smi | grep Version | sed -e 's/ *| *//g' -e \"s/   */\n# \${SLURMD_NODENAME}.\${SLURM_PROCID}>   /g\" -e \"s/^/# \${SLURMD_NODENAME}.\${SLURM_PROCID}>   /g\")
\$(nvidia-smi -L | sed -e \"s/^/# \${SLURMD_NODENAME}.\${SLURM_PROCID}>   /g\")
\$(python -c 'import torch; print(f\"torch: {torch.__version__}\")' | sed -e \"s/^/# \${SLURMD_NODENAME}.\${SLURM_PROCID}>   /g\")
\$(python -c 'import torch, torch.utils.collect_env; torch.utils.collect_env.main()' | sed \"s/^/# \${SLURMD_NODENAME}.\${SLURM_PROCID}>   /g\")
\$(env | grep SLURM | sed \"s/^/# \${SLURMD_NODENAME}.\${SLURM_PROCID}>   /g\")
\$(cat /etc/nccl.conf | sed \"s/^/# \${SLURMD_NODENAME}.\${SLURM_PROCID}>   /g\")
\"\"\"

# set unbuffered python for realtime container logging
export PYTHONFAULTHANDLER=1
export NCCL_DEBUG=INFO

# train
python /train/train.py

echo -e \"\"\"
# finished at \$(date)
\"\"\"
""" >> $SCRIPT

# install lightning into container, ensure it is run only once per node
srun \
  --ntasks=${SLURM_JOB_NUM_NODES} \
  --ntasks-per-node=1 \
  --container-image nvcr.io#nvidia/pytorch:23.08-py3 \
  --container-name ${SLURM_JOB_NAME} \
  --output="${EXPERIMENT_DIR}/${EXPERIMENT_NAME}/${version}/${RUN_NAME}.%s.txt" \
  pip install lightning==2.1.2 \
  || exit 1

# run training script, run in background to receive signal notification prior to timout
srun \
  --container-name ${SLURM_JOB_NAME} \
  --container-mounts ${EXPERIMENT_DIR}/${EXPERIMENT_NAME}/${version}:/exp,${WORK_DIR}:/train \
  --output="${EXPERIMENT_DIR}/${EXPERIMENT_NAME}/${version}/${RUN_NAME}.%s.txt" \
  --container-workdir /exp \
  /exp/${RUN_NAME}.sh \
  &
PID=$!
wait $PID;
```

This script can then be run and monitored with the standard commands.
``` bash
ilb@login-frida:~/mnist-demo$ sbatch train_2x2.sbatch --version=v1
Submitted batch job 602

ilb@login-frida:~/mnist-demo$ tail -f exp/demo/v1/v1_20231213-2237.1.txt
...
```
<!--
_TODO: notes ... no requeue support from within container (scontrol not available). send emails on start, error, ..._

#### Auto requeue

`sbatch signal`
`pytorch signal`

### Open a shell into a job running inside a container

```
srun --container-name xxx --jobid
```
-->
<!--
### Useful Slurm commands

Show the current status of resources:
```bash
ilb@login-frida:~$ slurm
PARTITION NODELIST      NODES CPUS(A/I/O/T)     GRES                                                   GRES_USED
frida     ana           1     0/112/0/112       gpu:A100_80GB:8(S:0-1),shard:A100_80GB:640(S:0-1)      gpu:A100_80GB:0(IDX:N/A),shard:A100_80GB:0(0/80,0/80,0/80,0/80,0/80,0/80,0/80,0/80)
frida     axa           1     0/256/0/256       gpu:A100:8(S:0-1)                                      gpu:A100:0(IDX:N/A),shard:0
frida     ixh           1     0/224/0/224       gpu:H100:8(S:0-1)                                      gpu:H100:0(IDX:N/A),shard:0
dev*      ana           1     0/112/0/112       gpu:A100_80GB:8(S:0-1),shard:A100_80GB:640(S:0-1)      gpu:A100_80GB:0(IDX:N/A),shard:A100_80GB:0(0/80,0/80,0/80,0/80,0/80,0/80,0/80,0/80)
```

Show the user/account associated limits:
`slimits`
```bash
$ sacctmgr show association format=account%20,user%30,partition,priority,GrpTRESMins%30,qos user=$USER
   Account       User  Partition   Priority                    GrpTRESMins                  QOS
---------- ---------- ---------- ---------- ------------------------------ --------------------
       lpt        ilb                                                                    normal
```

Show the available QoS:
`sqos`
```bash
$ sacctmgr show qos  format="name%20,preempt,priority,GrpTRES,MaxTresPerJob,MaxJobsPerUser,MaxWall,flags"
                Name    Preempt   Priority       GrpTRES       MaxTRES MaxJobsPU     MaxWall                Flags
-------------------- ---------- ---------- ------------- ------------- --------- ----------- --------------------
              normal                     0
```

Show the current status of queues:
`squeue`


Show current priority
`sprio`


`sattach`
```srun --pty --overlap --jobid 485 --nodelist=ixh enroot```
```enroot list -f```
```enroot exec PID bash```
https://ask.cyberinfrastructure.org/t/how-to-attach-to-a-running-job-to-run-top-on-compute-node/912/3
-->