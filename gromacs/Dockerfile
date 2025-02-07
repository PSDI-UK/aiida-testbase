# syntax=docker/dockerfile:1
ARG BASE_IMAGE=2025-01-14-aiida2.5.2-py3.11.0
FROM harbor.stfc.ac.uk/psdi-pathfinder4/aiida-testbase:$BASE_IMAGE

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

# Specify some default version, but really we want to override it in GHA.
ARG PLUMED_VERSION=2.9.3
ENV PLUMED_VERSION=${PLUMED_VERSION}

# Specify some default version, but really we want to override it in GHA.
ARG GMX_VERSION=2024.2
ENV GMX_VERSION=${GMX_VERSION}

# Change to root and build in /tmp.
USER root
WORKDIR /tmp

# Grab the specified version of plumed and untar the source.
RUN wget https://github.com/plumed/plumed2/archive/refs/tags/v${PLUMED_VERSION}.tar.gz && \
    tar xvf v${PLUMED_VERSION}.tar.gz && \
    rm v${PLUMED_VERSION}.tar.gz

# Configure, compile and install plumed.
WORKDIR /tmp/plumed2-${PLUMED_VERSION}
RUN ./configure --prefix=/opt/plumed2-${PLUMED_VERSION}
RUN make -j8
RUN make install

# Add plumed to the various paths.
ENV PATH=/opt/plumed2-${PLUMED_VERSION}/src/lib:$PATH
ENV LIBRARY_PATH=/opt/plumed2-${PLUMED_VERSION}/src/lib:$LIBRARY_PATH
ENV LD_LIBRARY_PATH=/opt/plumed2-${PLUMED_VERSION}/src/lib/:$LD_LIBRARY_PATH
ENV PLUMED_KERNEL=/opt/plumed2-${PLUMED_VERSION}/src/lib/libplumedKernel.so
ENV PLUMED_VIMPATH=/opt/plumed2-${PLUMED_VERSION}/vim
ENV PYTHONPATH=/opt/plumed2-${PLUMED_VERSION}/python:$PYTHONPATH

# Grab the specified version of gromacs and untar the source.
RUN wget ftp://ftp.gromacs.org/gromacs/gromacs-${GMX_VERSION}.tar.gz && \
    tar xvf gromacs-${GMX_VERSION}.tar.gz && \
    rm gromacs-${GMX_VERSION}.tar.gz

# Patch gmx with plumed and make build dir
WORKDIR /tmp/gromacs-${GMX_VERSION}
RUN plumed patch -p -e gromacs-2024.3
RUN mkdir build

# Build gromacs without mpi and without cuda.
WORKDIR /tmp/gromacs-${GMX_VERSION}/build
RUN cmake .. -DCMAKE_INSTALL_PREFIX=/opt/gromacs-${GMX_VERSION} -DGMX_BUILD_OWN_FFTW=ON -DGMX_OPENMP=ON -DGMXAPI=OFF -DCMAKE_BUILD_TYPE=Release

RUN make -j8
RUN make install

# Cleanup and fix permissions.
RUN rm -r /tmp/gromacs-${GMX_VERSION}
RUN chown ${SYSTEM_UID}:${SYSTEM_GID} -R /opt/gromacs-${GMX_VERSION} && \
    fix-permissions "/opt/gromacs-${GMX_VERSION}"

# Add the exec to the path.
ENV PATH=/opt/gromacs-${GMX_VERSION}/bin:$PATH

# Switch back to USER aiida to avoid accidental container runs as root
USER ${SYSTEM_UID}
WORKDIR "${HOME}"
