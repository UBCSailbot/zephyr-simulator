# Ubuntu 26.04.1 (resolute) + ROS 2 Lyrical Luth + Gazebo Jetty
# Gazebo Jetty is installed through ros-lyrical-ros-gz (ROS vendor packages).

# CA certificates (needed for HTTPS apt/curl)
FROM ubuntu:resolute-20261105.1 AS ca-certificates

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
ENV DEBIAN_FRONTEND=

# locale, timezone, ROS 2 apt source, ros-lyrical-ros-base
FROM ca-certificates AS ros

ENV DEBIAN_FRONTEND=noninteractive

# Install language
RUN apt-get update && apt-get install -y --no-install-recommends \
    locales \
    && locale-gen en_US.UTF-8 \
    && update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8 \
    && rm -rf /var/lib/apt/lists/*
ENV LANG=en_US.UTF-8

# Install timezone
RUN ln -fs /usr/share/zoneinfo/UTC /etc/localtime \
    && apt-get update \
    && apt-get install -y --no-install-recommends tzdata \
    && dpkg-reconfigure --frontend noninteractive tzdata \
    && rm -rf /var/lib/apt/lists/*

# Install ROS 2
ARG ROS_APT_SOURCE_VERSION=1.3.0
RUN apt-get update && apt-get install -y --no-install-recommends \
        curl \
        sudo \
    && curl -fsSL --retry 3 -o /tmp/ros2-apt-source.deb \
        "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.resolute_all.deb" \
    && dpkg -i /tmp/ros2-apt-source.deb \
    && apt-get update && apt-get install -y --no-install-recommends \
        ros-lyrical-ros-base \
        python3-argcomplete \
    && rm -rf /var/lib/apt/lists/* /tmp/*

ENV ROS_DISTRO=lyrical
ENV ROS_PYTHON_VERSION=3
ENV ROS_VERSION=2
ENV DEBIAN_FRONTEND=

# Gazebo Jetty + ros_gz bridge, build tools, GL/X11 support
FROM ros AS gazebo

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        cmake \
        git \
        libgl1-mesa-dri \
        mesa-utils \
        ninja-build \
        python3-rosdep \
        ros-dev-tools \
        ros-lyrical-ros-gz \
        x11-apps \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && (rosdep init || echo "rosdep already initialized")
ENV DEBIAN_FRONTEND=

# debugging/editor tools and the non-root user
FROM gazebo AS dev-user

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    bash-completion \
    gdb \
    vim \
    && rm -rf /var/lib/apt/lists/*

ARG USERNAME=ros
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# Create a non-root user
# -o allows a non-unique ID: Ubuntu 24.04+ base images already use 1000
RUN groupadd -o --gid $USER_GID $USERNAME \
    && useradd -o -s /bin/bash --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && usermod -aG video $USERNAME \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME \
    && echo "source /usr/share/bash-completion/completions/git" >> /home/$USERNAME/.bashrc
ENV DEBIAN_FRONTEND=

# shell setup, history/log persistence, config files, final user
FROM dev-user AS devcontainer

ARG USERNAME=ros
ARG HOME=/home/$USERNAME

ENV ROS_WORKSPACE=/workspaces/boat_simulator

# persist ROS logs
RUN mkdir -p ${HOME}/.ros/log \
    && chown -R ${USERNAME} ${HOME}/.ros

# persist bash history
RUN SNIPPET="export PROMPT_COMMAND='history -a' && export HISTFILE=${HOME}/commandhistory/.bash_history" \
    && mkdir ${HOME}/commandhistory \
    && touch ${HOME}/commandhistory/.bash_history \
    && chown -R ${USERNAME} ${HOME}/commandhistory \
    && echo $SNIPPET >> "${HOME}/.bashrc"

# ros bash configuration: underlay, then the workspace overlay if it has been built
RUN echo "source /opt/ros/${ROS_DISTRO}/setup.bash" >> ${HOME}/.bashrc \
    && echo "[ -f ${ROS_WORKSPACE}/install/setup.bash ] && source ${ROS_WORKSPACE}/install/setup.bash" >> ${HOME}/.bashrc

# install other helpful apt packages
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        less \
        openssh-client \
        tmux \
        unzip \
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
ENV DEBIAN_FRONTEND=

# Copy configuration files (e.g., .vimrc) from config/ to the container's home directory
COPY --chown=${USERNAME}:${USERNAME} config ${HOME}

# # Uncomment this section to install additional apt packages
# ENV DEBIAN_FRONTEND=noninteractive
# RUN apt-get update \
#     && apt-get install -y --no-install-recommends \
#         # Your package list here
#     # Clean up
#     && apt-get autoremove -y \
#     && apt-get clean -y \
#     && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# set user and working directory
USER ${USERNAME}
WORKDIR ${ROS_WORKSPACE}