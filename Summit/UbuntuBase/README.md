# Base container
The base container example provides a basis on which to build your own Singularity containers suitable for deployment on Summit. To facilitate building such containers a helper script, `SummitPrep.sh`, has been created which will automatically create the necessary bind points and setup environment variables. The container is still responsible for installing the `CUDA toolkit`. What follows is a basic `Ubuntu Zesty(17.4)` container build capable of supporting `MPI` and `CUDA` applications. It is assumed that you are building the container on a Linux system in which you have root privilege on or through a service such as `Singularity Hub`.

## Definition walkthrough
```sh
BootStrap: docker
From: ppc64le/ubuntu:zesty

%post
# Set PATH and LD_LIBRARY_PATH
export PATH=/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin
export LD_LIBRARY_PATH=/usr/local/lib:/lib64/usr/lib/x86_64-linux-gnu

# Install basic system software
apt-get install -y software-properties-common wget pkg-config
apt-add-repository universe
apt-get update
```
These first few lines specify the base `Ubuntu Zesty` container which will be pulled from `Docker Hub`; This is handy  as it removes any package manager dependencies on our host system although requires a few extra lines to setup a basic environment as shown.

```sh
# Install OpenMPI
apt-get install -y openmpi
```
Installing the `Ubuntu` provided `OpenMPI` package will serve as a base for building future packages within the container and will later be patched to be compatible with `Cray MPT`.

```sh
# Install the toolkit
wget -O cuda_installer.deb https://developer.nvidia.com/compute/cuda/8.0/prod/local_installers/cuda-repo-ubuntu1604-8-0-local-ga2_8.0.54-1_ppc64el-deb
dpkg -i cuda_installer.deb
apt-get update
apt-get -y install cuda-toolkit-8-0
rm -rf cuda_installer.deb

# Set CUDA env variables
export PATH=$PATH:/usr/local/cuda/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib:/usr/local/cuda/lib64

# Install GCC compatable with CUDA/8.0
apt-get install -y gcc-4.9 g++-4.9

# Patch CUDA toolkit to work with non system default GCC
ln -s /usr/bin/gcc-4.9 /usr/local/cuda/bin/gcc
ln -s /usr/bin/g++-4.9 /usr/local/cuda/bin/g++
```
The `CUDA Toolkit` can be installed using the `NVIDA` provided installation utility; The driver itself is excluded as it is maintained by the host OS. Care must be taken to ensure that the container makes available compilers which are compatible with `CUDA/8.0` and sets up appropriate environment variables.

```sh
# Install MPI4PY against mpich(python-mpi4py is built against OpenMPI)
apt-get install -y python-pip
pip install mpi4py
```
With the base container setup we are now free to install packages utilizing MPI and CUDA. Care must be taken to ensure that all packages dependent on `MPI` are built against `MPICH`. For example the prebuilt `Ubuntu` package `python-mpi4py` is built against `OpenMPI` and so must not be used; We can however install `mpi4py` with `pip` to ensure it will be built against the previously installed `MPICH` package.

```sh
# Persist PATH and LD_LIBRARY_PATH to container runtime
echo "" >> /environment
echo "export PATH=${PATH}" >> /environment
echo "export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}" >> /environment
```
Singularity sources the `/environment` file during container runtime, here we ensure that any environment variables required at runtime are persisted.

```sh
# Patch container to work on Summit
wget https://raw.githubusercontent.com/olcf/SingularityTools/master/Summit/SummitPrep.sh
sh SummitPrep.sh
rm SummitPrep.sh
```
Lastly `SummitPrep.sh` is run, setting up appropriate bindpoints.

## Building the container
```bash
$ sudo singularity create --size 8000 Ubuntu.img
$ sudo singularity bootstrap Ubuntu.img Ubuntu.def
```
Building the container does not require any Summit specific steps. The only care that must be taken is ensuring the container is large enough to handle the `CUDA Toolkit` installation. For our example application 8 gigabytes is sufficient.

## Transfering the container
Once the container has been built on a local resource it can be transferred to the OLCF using standard data transfer utilities. Currently Globus Online is the recommended way to facilitate this transfer.

## Running the container
```bash
$ module load singularity
$ singularity exec Ubuntu.img mpicc HelloMPI.c -o mpi.out
$ singularity exec Ubuntu.img nvcc HelloCuda.cu -o cuda.out
```
Within an interactive or batch job applications can be built and run utilizing the containers software stack. Full `/lustre` access is available from inside the container and the directory in which singularity is launched from will be the current working directory inside of the container. In this case the source code `HelloMPI.c`, `HelloCuda.cu`, and `HelloMPI.py` exists outside of the container on `lustre` and the applications `mpi.out` and `cuda.out` will be created in the same `lustre` directory. The singularity module sets environment variables which work in conjunction with the helper script `SummitPrep.sh` to ensure the Summit specific patches work correctly.

```bash
$ aprun -n 2 -N 1 singularity exec Ubuntu.img ./mpi.out
Hello from Ubuntu 17.04 : rank  0 of 2
Hello from Ubuntu 17.04 : rank  1 of 2

$ aprun -n 1 singularity exec Ubuntu.img ./cuda.out
hello from the GPU

$ aprun -n 2 -N 1 singularity exec Ubuntu.img python HelloMPI.py 
Hello from mpi4py ('Ubuntu', '17.04', 'zesty') : rank 1 of 2 
Hello from mpi4py ('Ubuntu', '17.04', 'zesty') : rank 0 of 2
```
Once the applications have been built they can be executed on compute nodes through `aprun`. Note that the message `WARNING: Not mounting current directory: host does not support PR_SET_NO_NEW_PRIVS` can be ignored and should be removed in the next Singularity release.
