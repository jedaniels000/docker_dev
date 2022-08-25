# Docker Reference
This section file is dedicated to reference material such as common usage guides and tips & tricks for Docker.
## Setup
Nothing yet.


## Usage
### Common Docker Run Flags
**Ref**: [https://docs.docker.com/engine/reference/commandline/run/](https://docs.docker.com/engine/reference/commandline/run/)  
**Syntax**: `docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`  

- -it: Open an interactive terminal for a docker container.
- -d: Run in detached mode (run the container in the background); also prints container ID
- --name \<NAME\>: Name the new instance of the docker container, according to the value of \<NAME\>
- --rm: Removes the container (not the image) once it is stopped
- -v, --mount: Mount a volume (used for persistent data storage); without volumes, each time a new docker container is created or restarted, all data stored (scripts, data files, etc.) is lost. Some examples of usage can be viewed [here](https://phoenixnap.com/kb/docker-volumes) for common cases. --mount is a more specific and verbose version of the shorthand version -v; if you don't know which to use, just use --mount. For more information on these commands, see [bind mounts](https://docs.docker.com/storage/bind-mounts/) (where a file or directory on the host machine is mounted into a container) or [volumes](https://docs.docker.com/storage/volumes/) (where a new directory is created within Docker’s storage directory on the host machine).

## Tips and Tricks
### Installing Anaconda in Docker Container
Anaconda can be very handy, and I really like it. If you want to include Anaconda (or Miniconda, technically in this example) by default in the image, the following is one method to do so. This method assumes the directory in which the Dockerfile exists contains the current installation script for Miniconda; there are other methods that, for example, use wget commands in the Dockerfile itself to pull Miniconda. However, these are more distribution-specific and, therefore, are not covered here. In the Dockerfile, include the following:
```
# <IMAGE> corresponds to the Docker image e.g. ubuntu:latest
FROM <IMAGE>
...
# (Optionally, but very handily) Add miniconda to the path.
ENV PATH="/root/miniconda3/bin:${PATH}"
ARG PATH="/root/miniconda3/bin:${PATH}"

# Copy the Miniconda installation script from host
COPY Miniconda3-latest-Linux-x86_64.sh .

# Run the installation, then delete the installation script
RUN mkdir /root/.conda \
    && bash Miniconda3-latest-Linux-x86_64.sh -b \
    && rm -f Miniconda3-latest-Linux-x86_64.sh

# Initialize conda for bash to allow all conda scripts from
# bash. Source the .bashrc file to load these new changes.
RUN conda init bash \
    && source ~/.bashrc

# (Optionally) test the Miniconda installation
RUN conda --version
```
If the Miniconda installation script is not already downloaded and available on the host machine, an easy way to retrieve it is by using the command `wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh`.

### Adding NVIDIA/GPU Support for Tensorflow
Docker is a great way to run GPU-Tensorflow because only the NVIDIA driver is required on the host machine and not the NVIDIA CUDA toolkit (well, technically NVIDIA container toolkit is also required for GPU functionality pass-through between the host machine and the Docker container). I won't go into much detail as the [Tensorflow Docker](https://www.tensorflow.org/install/docker) guide and NVIDIA Docker [Github](https://github.com/NVIDIA/nvidia-docker) and [installation guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#install-guide) provide the platform-specific integration methods. A couple quick notes, however:
- As of this writing, the Tensorflow guide suggests testing if the NVIDIA container toolkit is properly installed by running the command: `docker run --gpus all --rm nvidia/cuda nvidia-smi`. This is outdated, as the image `nvidia/cuda` now requires a tag to be specified because the 'latest' tag is deprecated.
- On Arch Linux specifically, the NVIDIA container toolkit is very easy to install from the corresponding [AUR](https://aur.archlinux.org/packages/nvidia-container-toolkit) package.

### Docker Integration into VS Code
You can edit files in a docker container directly in VSCode without having to install an editor in the container itself (to keep the container/image as small as possible). All that is needed is the extension: [Remote - Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)  
Steps:
- Open VSCode Command Palette (Default Keybinding: Ctrl+Shift+P)
- Select: Remote-Containers: Attach to Running Container...
- Select desired container

### Allow Non-Root User Docker Use
Steps:
- Add the Docker group (may already exist; either go ahead and run the command as it won't break the current group OR check current groups using `groups`): `sudo groupadd docker`
- Add the current user to the group (or replace $USER with the desired user to the group): `sudo gpasswd -a $USER docker`
- Test using: `docker run hello-world`
- If you get an error (as you probably will due to changes not being updated), the simplest way to fix is to restart the system. Recheck the test command.

### Keep Image Small by Combining Layers
When creating a Dockerfile, each command (e.g. FROM, RUN, COPY, etc.) creates a new image 'layer'. Therefore, each command that is included inherently increases the size of the image, even if it is deleting files/freeing up space from a previous layer. Below we have an example:

First, we take just the latest Ubuntu version as our image with an image size of 77.8MB.
```
FROM ubuntu:latest

# Image Size: 77.8MB
```
We then update the package sources and install wget; why we do this is obvious in the next step. This increases the size to about 118MB (~40.2MB increase)
```
FROM ubuntu:latest

RUN apt update \
    && apt install -y wget

# Image Size: 118MB
```
We then use wget to retrieve a relatively large file, the Miniconda installer, increasing the image size once again by ~76MB.
```
FROM ubuntu:latest

RUN apt update \
    && apt install -y wget \
    && wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# Image Size: 194MB
```
Now, we are 'finished' with the installer, and we want to remove it from the virtual file system to free up space in the final image. So, we use the rm command to delete the file. That should give us a smaller image, right? Nope. We removed the Miniconda installer file (~76MB), so why is the image the same size?? Well, because the previous layer is already built, and the final layer where `RUN rm Miniconda3-latest-Linux-x86_64.sh` is implemented consists of itself and all ancestor layers.
```
FROM ubuntu:latest

RUN apt update \
    && apt install -y wget \
    && wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

RUN rm Miniconda3-latest-Linux-x86_64.sh

# Image Size: 194MB
```
 However, we can fix this as follows:
```
FROM ubuntu:latest

RUN apt update \
    && apt install -y wget \
    && wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh \
    && rm Miniconda3-latest-Linux-x86_64.sh

# Image Size: 118MB
```
Hey look! It's the same size as the image with only the package updates and wget installed. This is because the final creation and removal is done in the same layer. This can be shrunk even more by clearing the package cache and uninstalling wget, but I think this enough to prove the point. 

I spent so much time trying to find out why my images were ending up so large, so if this saves you hours of problem solving, you are welcome.

## Security
### Users and Groups
When a new container is created, by default, it will be run as root. This is a security risk if, during deployment, everyone who uses the docker container is a root user. For example, if a [volume](https://docs.docker.com/storage/volumes/) links the virtual file system of the docker container to the host machine, the docker container user has root access to the host machine. To avoid this, it is a good idea to include in the deployment docker file:  
```
# <IMAGE> corresponds to the Docker image e.g. ubuntu:latest
FROM <IMAGE>  
...  
# Create a new system group (which won't have root access)
# and then add a new user and assign it to the new group
# e.g. RUN groupadd -r users && useradd -r -g users janedoe
RUN groupadd -r <USER_GROUP_NAME> && useradd -r -g <USER_GROUP_NAME> <USER_NAME>
# Set the default user to this new user
USER user  
```