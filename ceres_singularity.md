# Singularity 3.x Usage on Ceres

Singularity is available on Ceres and allows users to create their own environment and let the users develop and customize their workflow without the need for admin intervention.

As always, email scinet_vrsc@usda.gov if you have questions regarding the use of Singularity.

## Using docker and singularity images from existing container libraries

### List of useful container libraries

#### 1. Docker Based Container Libraries

Docker Hub: <https://hub.docker.com/>

Nvidia GPU-Accelerated Containers (NGC): <https://ngc.nvidia.com/>   (Account Required!)

Quay (Bioinformatics): <https://quay.io/> or <https://biocontainers.pro/#/registry>

#### 2. Singularity Container Library

Singularity Library: <https://cloud.sylabs.io/library>

## Using Singularity

Visit any one of the above listed libraries and search for the container image you need.

**After gaining interactive access to one of the compute nodes**, we first pull (download) the pre-built image

    singularity pull docker://gcc                      # pulls an image from docker hub

This pulls the latest GCC container (gcc v 9.1.0 as of 08/19/2019) and saves the image in your current working directory.

If you prefer to pull a specific GCC version, look at the [available tags](https://hub.docker.com/r/library/gcc/tags/) for the specific container and append the tag version to the end of the container name. For example, if you need to pull the GCC v 5.3.0

    singularity pull docker://gcc:5.3.0

> **Note:** You can pull the images to a directory of your choosing (assuming you have write permission) by setting the variables SINGULARITY_CACHEDIR and SINGULARITY_TMPDIR. For instance,

    export SINGULARITY_CACHEDIR=$TMPDIR
    export SINGULARITY_TMPDIR=$TMPDIR

> **Note on home directory and Singularity**  
While pulling the containers, pay attention to the home directory as the cached image blobs will be saved in ${HOME}/.singularity  
Since the home directory has a limited amount of space, this can fill up quite easily. Users can change where the files will be cached by setting SINGULARITY_CACHEDIR and SINGULARITY_TMPDIR environment variables.

To use the executables within the container

    $ singularity exec gcc.img gcc --version

    gcc (GCC) 9.1.0

    Copyright (C) 2019 Free Software Foundation, Inc.

    This is free software; see the source for copying conditions. There is NO

    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

You can use this to compile your code or run your scripts using the container's executables.

    $ singularity exec gcc.img gcc hello_world.c -o hello_world

    $ singularity exec gcc.img ./hello_world
    Hello, World!

/home/${USER}, /project and ${TMPDIR} will be accessible via the container image.

    $ pwd
    /home/ynanyam
    $ ls

    catkin_ws       cuda       env_before     luarocks        pycuda_test.py

    $ singularity exec gcc.img pwd

    /home/ynanyam

    $  singularity exec gcc.img ls

    catkin_ws       cuda         env_before          luarocks         pycuda_test.py

### Interactive access

To gain interactive access to the container

    $ singularity shell gcc.img

    Singularity gcc.img:~> gcc --version

    gcc (GCC) 9.1.0

    Copyright (C) 2019 Free Software Foundation, Inc.

    This is free software; see the source for copying conditions. There is NO

    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

To access GPUs within a container, use "--nv" option

    singualrity exec --nv <options>

    singualrity shell --nv <options>

## Slurm batch jobs with containers

Here is a example slurm batch script that downloads a container and uses it to run a program.

    #!/bin/bash
    #SBATCH -N1
    #SBATCH -n20
    #SBATCH -t120
    #SBATCH -pbrief-low

    unset XDG_RUNTIME_DIR
    cd $TMPDIR

    wget ftp://ftp.ncbi.nlm.nih.gov/genomes/Bactrocera_oleae/protein/protein.fa.gz

    gunzip protein.fa.gz

    SINGULARITY_TMPDIR=$TMPDIR SINGULARITY_CACHEDIR=$TMPDIR singularity build clustalo.sif docker://quay.io/biocontainers/clustalo:1.2.4--1

    singularity exec clustalo.sif clustalo -i protein.fa -o result.fasta --threads=${SLURM_NPROCS} -v

### Singularity and MPI

**Reference:** <https://sylabs.io/guides/3.3/user-guide/mpi.html>

Singularity supports running MPI applications via a container. In order to use singularity containers with MPI, the MPI application must be made available on both the host and within the container.

This requires the host and container MPI versions to be compatible. One way to make sure the versions match is to use the MPI executables and libraries available on the host and bind it to the containers.

Below is a simple MPI batch script that uses OpenMPI

    #!/bin/bash

    #SBATCH -N1
    #SBATCH -n4
    #SBATCH -t20
    #SBATCH -e slurm-%j.err
    #SBATCH -o slurm-%j.out

    unset XDG_RUNTIME_DIR
    cd $TMPDIR
    SINGULARITY_TMPDIR=$TMPDIR SINGULARITY_CACHEDIR=$TMPDIR singularity build centos-openmpi.sif docker://centos

    export SINGULARITYENV_PREPEND_PATH=/software/7/apps/openmpi/3.0.0/bin
    export SINGULARITYENV_LD_LIBRARY_PATH=/software/7/apps/openmpi/3.0.0/lib

    wget https://raw.githubusercontent.com/wesleykendall/mpitutorial/gh-pages/tutorials/mpi-hello-world/code/mpi_hello_world.c

    module load openmpi/3.0.0
    mpicc mpi_hello_world.c -o hello_world
    mpirun -np ${SLURM_NPROCS} singularity exec --bind /software/7/apps/openmpi centos-openmpi.sif ./hello_world

In the above example script - openmpi is made available with the container using

    export SINGULARITYENV_PREPEND_PATH=/software/7/apps/openmpi/3.0.0/bin  
    export SINGULARITYENV_LD_LIBRARY_PATH=/software/7/apps/openmpi/3.0.0/lib

and then bind mounted to the container. (By default we don't bind mount the software directories)

In most cases, the containers we use don't include the Infiniband drivers, OpenMPI fails through and then falls back to the ethernet interface.

If users are building their own containers which make use of MPI be sure to include the OFED IB driver stack.

### Singularity and Nvidia Container Library

Nvidia provides a container library that works with both Docker and Singularity.

Users must have an account created in NGC and create an API key (save your key in a secure location!!!). Instructions to get started are here: <https://docs.nvidia.com/ngc/ngc-user-guide/singularity.html#singularity>

Once you have an account created, export the required environment variables and download the containers for the library. <https://ngc.nvidia.com/catalog/containers>

> **NOTE:**  
Do not set the environment variables SINGULARITY_DOCKER_USERNAME and SINGULARITY_DOCKER_PASSWORD in your .bashrc file as it prevents pulling containers from libraries other than NGC <https://groups.google.com/a/lbl.gov/forum/#!topic/singularity/9q1aTycZ6CA>

Below is an example script that downloads namd container and then runs an example on GPU node.  <https://ngc.nvidia.com/catalog/containers/hpc:namd>

    #!/bin/bash
    #SBATCH -N1
    #SBATCH -n72
    #SBATCH -t10
    #SBATCH -pgpu-low
  
    unset XDG_RUNTIME_DIR
    cd $TMPDIR
    export SINGULARITY_DOCKER_USERNAME='$oauthtoken'
    export SINGULARITY_DOCKER_PASSWORD=<API Key>
    wget http://www.ks.uiuc.edu/Research/namd/utilities/stmv.tar.gz
    tar xf stmv.tar.gz

    curl -s https://www.ks.uiuc.edu/Research/namd/2.13/benchmarks/stmv_nve_cuda.namd > stmv/stmv_nve_cuda.namd  # constant energy
    curl -s https://www.ks.uiuc.edu/Research/namd/2.13/benchmarks/stmv_npt_cuda.namd > stmv/stmv_npt_cuda.namd  # constant pressure

    nvidia-cuda-mps-server # loads nvidia modules to work with singularity GPU containers

    SINGULARITY_TMPDIR=$TMPDIR SINGULARITY_CACHEDIR=$TMPDIR singularity pull docker://nvcr.io/hpc/namd:2.13-singlenode

    singularity exec --nv --bind $PWD:/host_pwd namd_2.13-singlenode.sif namd2 +ppn ${SLURM_NPROCS} +setcpuaffinity +idlepoll +devices 0,1 /host_pwd/stmv/stmv_nve_cuda.namd # stmv constant energy benchmark

    singularity exec --nv --bind $PWD:/host_pwd namd_2.13-singlenode.sif namd2 +ppn ${SLURM_NPROCS} +setcpuaffinity +idlepoll +devices 0,1 /host_pwd/stmv/stmv_npt_cuda.namd # stmv constant pressure benchmark

## Building Containers

Since docker images are compatible with Singularity, users can write recipes for their containers either as a dockerfile or singularity definition file.

Be aware that regular users do not have privileges to build container images from recipes on the cluster. They either need privileged access to a machine with docker or singularity installed or use docker hub/singularity library to push their recipes and build container images - we recommend the latter.

### Docker

Below are the links to get started on building docker containers -

Dockerfile reference - <https://docs.docker.com/engine/reference/builder/>

Upload your dockerfiles to docker hub - <https://docs.docker.com/docker-hub/>

### Singularity

Create an account at Sylabs to build a container using a definition.  <https://cloud.sylabs.io/builder>

Once logged in we are presented with a text box to start writing the definition.

Detailed documentation for a Singularity definition file is [here](https://sylabs.io/guides/3.3/user-guide/definition_files.html). But below is a simple example to get you started.

    Bootstrap: docker
    From: continuumio/miniconda3

    %labels
    maintainer "Name" <email address>

    %post
    apt-get update && apt-get install -y git

    # Conda install stringtie
    conda install -c bioconda stringtie

The header includes the Bootstrap and the label of the container.

Most of the definition files use docker as their bootstrap as docker library is more robust and well maintained.

This is using an existing docker image for miniconda (Python 3) available from <https://hub.docker.com/r/continuumio/miniconda3>

Post section is any modifications or additions the user can make to the original container - in this case we are adding stringtie package to the container.

## Launching Client/Server applications

### RStudio

The following is the guide for RStudio server on Ceres by Nathan Weeks. The original document can be found on Basecamp "Ceres RStudio Server User Guide"

This can be easily adapted for Jupyter Notebook or any application that requires port forwarding.

0. (If using VPN) Connect to SCINet VPN [see instructions for configuring SCINet VPN ](https://3.basecamp.com/3625179/buckets/5538276/vaults/1070659735)
1. Log into Ceres via SSH
2. Submit the RStudio SLURM job script with the following command

        sbatch /reference/containers/RStudio/3.5.0/rstudio.job

3. After the job has started, view the $HOME/rstudio-JOBID.out file for login information (where JOBID is the SLURM job ID reported by the sbatch command)

       [jane.user@sn-cn-8-1 ~]$ sbatch /reference/containers/RStudio/3.5.0/rstudio.job
       Submitted batch job 214664
       [jane.user@sn-cn-8-1 ~]$ cat ~/rstudio-214664.out
       VPN Users:
       1. Connect to SCINet VPN and point your web browser to http://sn-cn-6-0:57088
       2. log in to RStudio Server using the following credentials:
          user: jane.user
          password: 4wjRJfpIvQDtKdDZpmzY
       SSH users:
       1. SSH tunnel from your workstation using the following command (macOS or Linux only;
          for how to enter this in PuTTY on Windows see the Ceres RStudio User Guide)
          ssh -N -L 8787:sn-cn-6-0:57088 jane.user@login.scinet.science
          and point your web browser to http://localhost:8787
       2. log in to RStudio Server using the following credentials:
          user: jane.user
          password: 4wjRJfpIvQDtKdDZpmzY
       When done using RStudio Server, terminate the job by:
          1. Exit the RStudio Session ("power" button in the top right corner of the RStudio
       window)
          2. On the Ceres command line, issue the command
             scancel -f 214664

4. (If using VPN) Point your web browser to the listed hostname / port (in this example, <http://sn-cn-6-0:57088>), then enter your SCINet user name and the temporary password (valid for this job only; in this example 4wjRJfpIvQDtKdDZpmzY)

5. (If using SSH Port Forwarding instead of VPN) See "SSH Port Forwarding" section below for directions.

#### Stopping RStudio Server

1. Click the Quit Session (“power”) button in the top-right corner of the RStudio window (see picture), or select “File > Quit Session...”
2. After the “R Session has Ended” window appears, cancel the SLURM job from the Ceres command line. E.g., if the job ID is 214664:

               [jane.user@sn-cn-8-1 ~]$ scancel -f 214664

   Be sure to specify the

                scancel -f / --full
   option as demonstrated above
3. (If using SSH Port Forwarding instead of VPN) Close the terminal / PuTTY window in which the SSH
   tunnel was established

#### SSH Port Forwarding (instead of VPN)

##### Windows + PuTTY users

1. Open a new PuTTY window
2. In Session > Host Name, enter: login.scinet.science
3. In the category: Connection > SSH > Tunnels, enter 8787 in Source Port, the Destination
hostname:port listed in the job script output (in this example: sn-cn-6-0:57088), click “Add”,
then click “Open”
4. Point your browser to <http://localhost:8787>. Enter your SCINet user name, and one-time  
password listed in the job script output file

##### macOS / Linux users

1. Open a a new macOS/Linux terminal window and enter the SSH command listed in the job
script output file. In this example:
ssh -N -L 8787:sn-cn-6-0:57088 jane.user@login.scinet.science
There will be no output after logging in. Keep the window / SSH tunnel open for the duration of the
RStudio session
2. Point your browser to <http://localhost:8787>. Enter your SCINet user name, and one-time password
listed in the job script output file

## Things to keep in mind

* By default - project, home, ${TMPDIR} are bind mounted to the container

* User outside the container = User inside the container (This implies that the permissions within the container are the same as on the bare-metal compute node)

* All the networking stack is available from within the container - if the container has Infiniband stack installed it will make use of the network

* Having

        unset XDG_RUNTIME_DIR

  in your slurm script is useful when you have jupyter notebook in the container. Also removes some annoying warnings from your logs

* In the examples above, everything was done within ${TMPDIR} which will be deleted at the end of the job. Make sure you copy the output to your project directory to save it

* Make sure you have

        nvidia-cuda-mps-server

  when using GPU nodes as this loads all the required modules to make them work with Singularity
