# <IMAGE> corresponds to the Docker image e.g. ubuntu:latest
FROM jedaniels000/miniconda3:py39_4.12.0

# Copy the environment.yml file, and add the corresponding
# packages to the base conda environment
COPY environment.yml .
RUN CONDA_OVERRIDE_CUDA="11.2" conda env update --name base --file environment.yml --prune \
    && rm environment.yml \
    && conda clean --all
