FROM ubuntu:22.10

## Install Miniconda ##
# Add miniconda to the path.
ENV PATH="/root/miniconda3/bin:${PATH}"
ARG PATH="/root/miniconda3/bin:${PATH}"

# Define the Miniconda to be installed
ARG MCNAME="Miniconda3-py39_4.12.0-Linux-x86_64.sh"

# Update package sources so we can install wget
RUN apt-get update \
    # Install wget so we can retrieve miniconda
    && apt-get install -y wget \
    # Retrieve the miniconda installer
    && wget https://repo.anaconda.com/miniconda/${MCNAME} \
    # Make a directory for the conda files
    && mkdir /root/.conda \
    # Run the installation script
    && bash ${MCNAME} -b \
    # Delete the miniconda installer
    && rm ${MCNAME} \
    # Uninstall wget
    && apt-get purge -y wget \
    # Remove list of packages from apt update command
    && rm -rf /var/lib/apt/lists/* \
    # Delete package cache
    && apt-get clean \
    # Delete orphaned/unnecessary packages
    && apt-get -y autoremove \
    # Clean up any cache or other unnecessary files from conda
    && conda clean --all

# Source the conda shell script so conda functions operate
# correctly (https://github.com/conda/conda/issues/7980)
RUN conda init bash \
    && exec bash \
    && source ~/.bashrc
