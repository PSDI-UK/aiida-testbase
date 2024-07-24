# syntax=docker/dockerfile:1
ARG BASE_IMAGE=2024-07-17-aiida2.4.0-py3.11.0
FROM harbor.stfc.ac.uk/biosimulation-cloud/aiida-testbase:$BASE_IMAGE

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

# Specify some default version, but really we want to override it in GHA.
ARG AMBER_DL_PATH=null
ARG AMBER_VERSION=24
ENV AMBER_VERSION=${AMBER_VERSION}

# Change to root and build in /tmp.
USER root

RUN mkdir /opt/amber${AMBER_VERSION} && \
    chown ${SYSTEM_UID}:${SYSTEM_GID} -R /opt/amber${AMBER_VERSION} && \
    fix-permissions "/opt/amber${AMBER_VERSION}" 

USER ${SYSTEM_UID}
WORKDIR /tmp

# Download, unzip and cd into the specified AMBER source.
RUN wget $AMBER_DL_PATH/AmberTools${AMBER_VERSION}.tar.bz2 && \
    chown ${SYSTEM_UID}:${SYSTEM_GID} AmberTools${AMBER_VERSION}.tar.bz2 && \
    mkdir /tmp/amber${AMBER_VERSION}_src && \
    tar xvjf AmberTools${AMBER_VERSION}.tar.bz2 -C /tmp/amber${AMBER_VERSION}_src --strip-components 1 && \
    rm AmberTools${AMBER_VERSION}.tar.bz2
WORKDIR /tmp/amber${AMBER_VERSION}_src

# Update AMBER source.
RUN ./update_amber --update

# Make a build dir in /tmp.
RUN mkdir /tmp/build
WORKDIR /tmp/build

# Build AMBER without mpi and without cuda.
RUN pip3 install matplotlib scipy
RUN cmake /tmp/amber${AMBER_VERSION}_src -DCMAKE_INSTALL_PREFIX=/opt/amber${AMBER_VERSION} -DBUILD_PYTHON=TRUE -DDISABLE_TOOLS="pytraj" -DCMAKE_INSTALL_RPATH_USE_LINK_PATH=TRUE -DDOWNLOAD_MINICONDA=FALSE -DCOMPILER=MANUAL -DBUILD_GUI=FALSE -DCOMPILER=GNU -DOPENMP=TRUE -DCUDA=FALSE -DMPI=FALSE -DUSE_FFT=True -DBUILD_DEPRECATED=False -DBUILD_INDEV=False -DBUILD_PERL=True -DOPTIMIZE=True
RUN make -j8
RUN make install

# Cleanup.
RUN rm -r /tmp/amber${AMBER_VERSION}_src    

# Add the exec to the path.
ENV AMBERHOME=/opt/amber${AMBER_VERSION}
ENV PATH="$PATH:/opt/amber${AMBER_VERSION}/bin"
ENV LD_LIBRARY_PATH="$AMBERHOME/lib"
ENV PERL5LIB="$AMBERHOME/lib/perl"
ENV PYTHONPATH="$AMBERHOME/lib/python3.11/site-packages"

# Switch back to USER aiida to avoid accidental container runs as root
USER ${SYSTEM_UID}
WORKDIR "${HOME}"
