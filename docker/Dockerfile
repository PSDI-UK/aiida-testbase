# syntax=docker/dockerfile:1
ARG BASE_IMAGE=ubuntu:latest
FROM $BASE_IMAGE

LABEL maintainer="James Gebbie-Rayet <james.gebbie@stfc.ac.uk>"

ARG SYSTEM_USER="aiida"
ARG SYSTEM_UID="1001"
ARG SYSTEM_GID="100"
ARG PYTHON_VERSION="3.11.0"
ARG AIIDA_VERSION="2.5.2"
ARG PGSQL_VERSION="15.1"
ARG RMQ_VERSION="3.10.14"

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

ENV SYSTEM_USER="${SYSTEM_USER}"

# Install all OS dependencies
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update --yes && \
    apt-get upgrade --yes && \
    apt-get install --yes --no-install-recommends \
    build-essential \
    bzip2 \
    ca-certificates \
    cmake \
    curl \
    erlang \
    git \
    locales \
    openssh-client \
    sudo \
    vim \
    wget \
    xz-utils && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Configure environment
ENV CONDA_DIR=/opt/conda \
    SHELL=/bin/bash \
    SYSTEM_USER="${SYSTEM_USER}" \
    SYSTEM_UID=${SYSTEM_UID} \
    SYSTEM_GID=${SYSTEM_GID} \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV PATH="${CONDA_DIR}/bin:${PATH}" \
    HOME="/home/${SYSTEM_USER}"

# Copy a script that we will use to correct permissions after running certain commands
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

# Enable prompt color in the skeleton .bashrc before creating the default SYSTEM_USER
# hadolint ignore=SC2016
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc && \
   # Add call to conda init script see https://stackoverflow.com/a/58081608/4413446
   echo 'eval "$(command conda shell.bash hook 2> /dev/null)"' >> /etc/skel/.bashrc

# Create SYSTEM_USER with name aiida  with UID=1001 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -l -m -s /bin/bash -N -u "${SYSTEM_UID}" "${SYSTEM_USER}" && \
    mkdir -p "${CONDA_DIR}" && \
    chown "${SYSTEM_USER}:${SYSTEM_GID}" "${CONDA_DIR}" && \
    chmod g+w /etc/passwd && \
    fix-permissions "${HOME}" && \
    fix-permissions "${CONDA_DIR}"

# Install RabbitMQ
RUN wget -c --no-check-certificate https://github.com/rabbitmq/rabbitmq-server/releases/download/v${RMQ_VERSION}/rabbitmq-server-generic-unix-${RMQ_VERSION}.tar.xz && \
     tar -xf rabbitmq-server-generic-unix-${RMQ_VERSION}.tar.xz && \
     rm rabbitmq-server-generic-unix-${RMQ_VERSION}.tar.xz && \
     ln -sf /opt/rabbitmq_server-${RMQ_VERSION}/sbin/* /usr/local/bin/ && \
     fix-permissions /opt/rabbitmq_server-${RMQ_VERSION}

# make a github actions mount point.
RUN mkdir /github && \
    chown "${SYSTEM_USER}:${SYSTEM_GID}" /github

USER ${SYSTEM_UID}

# Download and install Micromamba, and initialize Conda prefix.
COPY --chown="${SYSTEM_UID}:${SYSTEM_GID}" initial-condarc "${CONDA_DIR}/.condarc"
WORKDIR /tmp
RUN set -x && \
    arch=$(uname -m) && \
    if [ "${arch}" = "x86_64" ]; then \
        # Should be simpler, see <https://github.com/mamba-org/mamba/issues/1437>
        arch="64"; \
    fi && \
    wget --progress=dot:giga -O /tmp/micromamba.tar.bz2 \
        "https://micromamba.snakepit.net/api/micromamba/linux-${arch}/latest" && \
    tar -xvjf /tmp/micromamba.tar.bz2 --strip-components=1 bin/micromamba && \
    rm /tmp/micromamba.tar.bz2 && \
    PYTHON_SPECIFIER="python=${PYTHON_VERSION}" && \
    if [[ "${PYTHON_VERSION}" == "default" ]]; then PYTHON_SPECIFIER="python"; fi && \
    # Install the packages
    ./micromamba install \
        --root-prefix="${CONDA_DIR}" \
        --prefix="${CONDA_DIR}" \
        --yes \
        "${PYTHON_SPECIFIER}" \
        'mamba' && \
    rm micromamba && \
    # Pin major.minor version of python
    mamba list python | grep '^python ' | tr -s ' ' | cut -d ' ' -f 1,2 >> "${CONDA_DIR}/conda-meta/pinned" && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${SYSTEM_USER}"

# Add ~/.local/bin to PATH where the dependencies get installed via pip
# This require the package installed with `--user` flag in pip
ENV PATH=${PATH}:/home/${NB_USER}/.local/bin

# Install and pin aiida-core and postgresql.
RUN mamba install -c conda-forge -y aiida-core=${AIIDA_VERSION} postgresql=${PGSQL_VERSION}
RUN conda config --env --add pinned_packages aiida-core=${AIIDA_VERSION}
RUN conda config --env --add pinned_packages postgresql=${PGSQL_VERSION}

# Copy start and stop scripts to simplify things.
COPY --chown=${SYSTEM_USER}:${SYSTEM_GID} aiida-start /home/${SYSTEM_USER}
COPY --chown=${SYSTEM_USER}:${SYSTEM_GID} aiida-stop /home/${SYSTEM_USER}

# Switch back to USER aiida to avoid accidental container runs as root
USER ${SYSTEM_UID}

WORKDIR "${HOME}"
