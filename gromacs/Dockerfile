# syntax=docker/dockerfile:1
ARG BASE_IMAGE=2024-06-24-aiida2.4.0-py3.11.0
FROM harbor.stfc.ac.uk/psdi-pathfinder4/aiida-testbase:$BASE_IMAGE

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

# Specify some default version, but really we want to override it in GHA.
ARG GMX_VERSION=2024.2
ENV GMX_VERSION=${GMX_VERSION}

# Change to root and build in /tmp.
USER root
WORKDIR /tmp

# Grab the specified version of gromacs and untar the source.
RUN wget ftp://ftp.gromacs.org/gromacs/gromacs-${GMX_VERSION}.tar.gz && \
    tar xvf gromacs-${GMX_VERSION}.tar.gz && \
    rm gromacs-${GMX_VERSION}.tar.gz

# Make a build dir in source dir.
WORKDIR /tmp/gromacs-${GMX_VERSION}
RUN mkdir build

# Build gromacs without mpi and without cuda.
WORKDIR /tmp/gromacs-${GMX_VERSION}/build
RUN cmake .. -DCMAKE_INSTALL_PREFIX=/opt/gromacs-${GMX_VERSION} -DGMX_BUILD_OWN_FFTW=ON -DGMX_OPENMP=ON -DGMXAPI=OFF -DCMAKE_BUILD_TYPE=Release
RUN make -j8
RUN make install
RUN rm -r /tmp/gromacs-${GMX_VERSION} && \
    fix-permissions "/opt/gromacs-${GMX_VERSION}" 

# Cleanup and fix permissions.
RUN rm -r /tmp/gromacs-${GMX_VERSION}
RUN chown ${SYSTEM_UID}:${SYSTEM_GID} -R /opt/gromacs-${GMX_VERSION} && \
    fix-permissions "/opt/gromacs-${GMX_VERSION}"

# Add the exec to the path.
ENV PATH=/opt/gromacs-${GMX_VERSION}/bin:$PATH

# Switch back to USER aiida to avoid accidental container runs as root
USER ${SYSTEM_UID}
WORKDIR "${HOME}"
