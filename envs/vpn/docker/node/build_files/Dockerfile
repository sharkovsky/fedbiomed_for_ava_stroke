ARG FBM_CONTAINER_VERSION_TAG
FROM fedbiomed/vpn-basenode:${FBM_CONTAINER_VERSION_TAG:-unknown}

ARG CONTAINER_GID
ARG CONTAINER_UID
ARG CONTAINER_GROUP
ARG CONTAINER_USER

ARG PYTHON_VERSION="3.10"

# assign default values even when variable exists and is empty
# (ARG default value only applies when variable is non existent)
# + make it persistent at runtime (pass this variable to container)
ENV CONTAINER_UID=${CONTAINER_UID:-0}
ENV CONTAINER_GID=${CONTAINER_GID:-0}
# alpine does not accept users/groups with numerical names
ENV CONTAINER_USER=${CONTAINER_USER:-root}
ENV CONTAINER_GROUP=${CONTAINER_GROUP:-root}

RUN apt-get update && apt-get install -y libgmp3-dev libmpfr-dev libmpc-dev apt-utils wget git 

RUN apt-get -y install rsync

# Install miniconda  -------------------------------------------------------------
RUN wget -q --directory-prefix=$HOME https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh && \
        bash $HOME/Miniconda3-latest-Linux-x86_64.sh -b -p /miniconda && \
        rm -f $HOME/Miniconda3-latest-Linux-x86_64.sh
# easier to install in a fixed path, not in alternate account home :
# cannot use "ENV PATH=${PATH}:${HOME}/miniconda/bin" : wont work because
# ${HOME} not defined in ENV command, only in RUN/etc. (executing in subshell)
ENV PATH=${PATH}:/miniconda/bin

# Install nodejs -----------------------------------------------------------------
RUN wget -qO- https://deb.nodesource.com/setup_23.x | bash -
RUN apt-get install -y nodejs
RUN npm install --global yarn@1.22

# read config.env in interactive shells & setup for reading it in entrypoint
COPY ./envs/vpn/docker/node/build_files/bashrc_append /tmp
RUN cat /tmp/bashrc_append | tee -a /root/.bashrc /root/bashrc_entrypoint \
        /etc/skel/.bashrc /etc/skel/bashrc_entrypoint >/dev/null

RUN [ "$CONTAINER_GROUP" != 'root' ] && groupadd -g $CONTAINER_GID $CONTAINER_GROUP || :
RUN [ "$CONTAINER_USER" != 'root' ] && useradd -m -d /home/$CONTAINER_USER \
        -u $CONTAINER_UID -g $CONTAINER_GID -s /bin/bash $CONTAINER_USER || :


COPY ./envs/vpn/docker/node/build_files/entrypoint*.bash /
# Setup subset of fedbiomed code tree
RUN mkdir -p /fedbiomed/envs /fbm-node

# Copy files from fedbiomed repo - in the context of fedbiomed top directory
COPY envs/vpn /fedbiomed/envs/vpn
COPY envs/common /fedbiomed/envs/common
COPY envs/common /fedbiomed/envs/common_reference
COPY fedbiomed /fedbiomed/fedbiomed
COPY notebooks /fedbiomed/notebooks
COPY docs/tutorials /fedbiomed/docs/tutorials
COPY scripts /fedbiomed/scripts
COPY fedbiomed_gui /fedbiomed/fedbiomed_gui
COPY pyproject.toml /fedbiomed/pyproject.toml
COPY README.md /fedbiomed/README.md
COPY hatch_build.py /fedbiomed/hatch_build.py

# Alternative option : git clone & strip unused dirs
#  - ensures working with clean repo (docker build begins with a stat(), hangs for
#    a long time if you have zillion files in notebooks/data)
#  - ensures using develop branch (good or bad)


# Prepare for work in context of alternate account
RUN chown -R $CONTAINER_USER:$CONTAINER_GROUP /fbm-node # Fed-BioMed node directory
RUN chown -R $CONTAINER_USER:$CONTAINER_GROUP /fedbiomed
RUN chown -R $CONTAINER_USER:$CONTAINER_GROUP /miniconda
WORKDIR /fedbiomed

# Set container user
USER $CONTAINER_USER:$CONTAINER_GROUP

# Installs fedbiomed and necessary modules
RUN conda create -n fbm -y python=3.10
ENV PATH=/miniconda/envs/fbm/bin:$PATH
RUN pip install --no-warn-script-location -I 'open-clip-torch==2.24.0' 'ninja==1.11.1.4' 'lmdb==1.7.5' 'torch<2.8'  '.[node]'

# transmit build-time values to running container
ENV CONTAINER_BUILD_USER=${CONTAINER_USER}
ENV CONTAINER_BUILD_GROUP=${CONTAINER_GROUP}
ENV CONTAINER_BUILD_UID=${CONTAINER_UID}
ENV CONTAINER_BUILD_GID=${CONTAINER_GID}

# Changedir to main directory
WORKDIR /

# Launch as root to manage VPN
USER root:root
ENTRYPOINT ["/entrypoint.bash"]

