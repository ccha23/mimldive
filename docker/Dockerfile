# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.

# Ubuntu 20.04 (focal)
# https://hub.docker.com/_/ubuntu/?tab=tags&name=focal
ARG ROOT_CONTAINER=ubuntu:focal

FROM $ROOT_CONTAINER AS base
ARG TARGETARCH

FROM base AS base-amd64
ENV TARGETARCHALIAS='x86_64'

FROM base AS base-arm64
ENV TARGETARCHALIAS='aarch64'

FROM base-$TARGETARCH

LABEL maintainer="Jupyter Project <jupyter@googlegroups.com>"
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

# Fix DL4006
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# Base-notebook
# Install all OS dependencies for notebook server that starts but lacks all
# features (e.g., download as all possible file formats)
# Install tini: init for containers
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    tini \
    wget \
    ca-certificates \
    sudo \
    locales \
    fonts-liberation \
    run-one && \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
    locale-gen

# Configure environment
ENV CONDA_DIR=/opt/conda \
    SHELL=/bin/bash \
    NB_USER="${NB_USER}" \
    NB_UID=${NB_UID} \
    NB_GID=${NB_GID} \
    LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8
ENV PATH="${CONDA_DIR}/bin:${PATH}" \
    HOME="/home/${NB_USER}"

# Copy a script that we will use to correct permissions after running certain commands
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
# hadolint ignore=SC2016
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc && \
   # Add call to conda init script see https://stackoverflow.com/a/58081608/4413446
   echo 'eval "$(command conda shell.bash hook 2> /dev/null)"' >> /etc/skel/.bashrc

# Create NB_USER with name jovyan user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -l -m -s /bin/bash -N -u "${NB_UID}" "${NB_USER}" && \
    mkdir -p "${CONDA_DIR}" && \
    chown "${NB_USER}:${NB_GID}" "${CONDA_DIR}" && \
    chmod g+w /etc/passwd && \
    fix-permissions "${HOME}" && \
    fix-permissions "${CONDA_DIR}"

USER ${NB_UID}
ARG PYTHON_VERSION=3.8.12

# Setup work directory for backward-compatibility
RUN mkdir "/home/${NB_USER}/work" && \
    fix-permissions "/home/${NB_USER}"

# Install conda as jovyan and check the sha256 sum provided on the download site
WORKDIR /tmp

# ---- Miniforge installer ----
# Check https://github.com/conda-forge/miniforge/releases
# Package Manager and Python implementation to use (https://github.com/conda-forge/miniforge)
# We're using Mambaforge installer, possible options:
# - conda only: either Miniforge3 to use Python or Miniforge-pypy3 to use PyPy
# - conda + mamba: either Mambaforge to use Python or Mambaforge-pypy3 to use PyPy
# Installation: conda, mamba, pip
RUN set -x && \
    # Miniforge installer
    miniforge_arch=$(uname -m) && \
    miniforge_installer="Mambaforge-Linux-${miniforge_arch}.sh" && \
    wget --quiet "https://github.com/conda-forge/miniforge/releases/latest/download/${miniforge_installer}" && \
    /bin/bash "${miniforge_installer}" -f -b -p "${CONDA_DIR}" && \
    rm "${miniforge_installer}" && \
    # Conda configuration see https://conda.io/projects/conda/en/latest/configuration.html
    conda config --system --set auto_update_conda false && \
    conda config --system --set show_channel_urls true && \
    if [[ "${PYTHON_VERSION}" != "default" ]]; then mamba install --quiet --yes python="${PYTHON_VERSION}"; fi && \
    mamba list python | grep '^python ' | tr -s ' ' | cut -d ' ' -f 1,2 >> "${CONDA_DIR}/conda-meta/pinned" && \
    # Using conda to update all packages: https://github.com/mamba-org/mamba/issues/1092
    conda update --all --quiet --yes && \
    conda clean --all -f -y && \
    rm -rf "/home/${NB_USER}/.cache/yarn" && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# Install Jupyter Notebook, Lab, and Hub
# Generate a notebook server config
# Cleanup temporary files
# Correct permissions
# Do all this in a single RUN command to avoid duplicating all of the
# files across image layers when the permissions change
RUN mamba install --quiet --yes \
    'notebook' \
    'jupyterhub' \
    'jupyterlab' && \
    mamba clean --all -f -y && \
    npm cache clean --force && \
    jupyter notebook --generate-config && \
    jupyter lab clean && \
    rm -rf "/home/${NB_USER}/.cache/yarn" && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

EXPOSE 8888

# Configure container startup
ENTRYPOINT ["tini", "-g", "--"]
CMD ["start-notebook.sh"]

# Copy local files as late as possible to avoid cache busting
COPY start.sh start-notebook.sh start-singleuser.sh /usr/local/bin/
# Currently need to have both jupyter_notebook_config and jupyter_server_config to support classic and lab
COPY jupyter_notebook_config.py /etc/jupyter/

# Fix permissions on /etc/jupyter as root
USER root

# Prepare upgrade to JupyterLab V3.0 #1205
RUN sed -re "s/c.NotebookApp/c.ServerApp/g" \
    /etc/jupyter/jupyter_notebook_config.py > /etc/jupyter/jupyter_server_config.py && \
    fix-permissions /etc/jupyter/

# Switch back to jovyan to avoid accidental container runs as root
USER ${NB_UID}

WORKDIR "${HOME}"

# Minimal-notebook
# Install all OS dependencies for fully functional notebook server
USER root

RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    # Common useful utilities
    git \
    nano-tiny \
    tzdata \
    unzip \
    vim-tiny \
    # git-over-ssh
    openssh-client \
    # TODO: check if these are needed and describe
    inkscape \
    libsm6 \
    libxext-dev \
    libxrender1 \
    lmodern \
    netcat \
    # nbconvert dependencies
    # https://nbconvert.readthedocs.io/en/latest/install.html#installing-tex
    texlive-xetex \
    texlive-fonts-recommended \
    texlive-plain-generic && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Create alternative for nano -> nano-tiny
RUN update-alternatives --install /usr/bin/nano nano /bin/nano-tiny 10

# Scipy-notebook
# ffmpeg for matplotlib anim & dvipng+cm-super for latex labels
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends ffmpeg dvipng cm-super && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

USER ${NB_UID}

# Install Python 3 packages
RUN mamba install --quiet --yes \
    'altair' \
    'beautifulsoup4' \
    'bokeh' \
    'bottleneck' \
    'cloudpickle' \
    'conda-forge::blas=*=openblas' \
    'cython' \
    'dask' \
    'dill' \
    'h5py' \
    'ipympl'\
    'ipywidgets' \
    'matplotlib-base' \
    'numba' \
    'numexpr' \
    'pandas' \
    'patsy' \
    'protobuf' \
    'pytables' \
    'scikit-image' \
    'scikit-learn' \
    'scipy' \
    'seaborn' \
    'sqlalchemy' \
    'statsmodels' \
    'sympy' \
    'widgetsnbextension'\
    'xlrd' && \
    mamba clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# Install facets which does not have a pip or conda package at the moment
WORKDIR /tmp
RUN git clone https://github.com/PAIR-code/facets.git && \
    jupyter nbextension install facets/facets-dist/ --sys-prefix && \
    rm -rf /tmp/facets && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

WORKDIR "${HOME}"

# Extra Package
USER root

RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    build-essential \
    cmake \
    curl \
    less \
    sshfs \
    pandoc \
    pdf2svg \
    # ----
    zip \
    ffmpeg \
    dvipng \
    gcc \
    cm-super \
    libcairo2-dev \
    libffi-dev \
    libpango1.0-dev \
    pkg-config && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Texlive
COPY texlive-profile.txt /tmp/
ENV PATH=/usr/local/texlive/bin/${TARGETARCHALIAS}-linux:$PATH
RUN wget -O /tmp/install-tl-unx.tar.gz http://mirror.ctan.org/systems/texlive/tlnet/install-tl-unx.tar.gz && \
    mkdir /tmp/install-tl && \
    tar -xzf /tmp/install-tl-unx.tar.gz -C /tmp/install-tl --strip-components=1 && \
    /tmp/install-tl/install-tl --profile=/tmp/texlive-profile.txt \
    && tlmgr install \
        amsmath babel-english cbfonts-fd cm-super caption ctex doublestroke dvisvgm environ eurosym everysel \
        fancyvrb float fontspec frcursive fundus-calligra gnu-freefont jknapltx latex-bin \
        mathastext microtype ms oberdiek parskip physics preview ragged2e relsize rsfs \
        setspace standalone tcolorbox tipa ucs upquote qsymbols wasy wasysym xcolor xetex xkeyval adjustbox collectbox titling booktabs enumitem \
    && fix-permissions /usr/local/texlive \
    && rm -rf /tmp/*

# Jupyter Extension
COPY fix-nbextensions /usr/local/bin/fix-nbextensions
RUN chmod a+rx /usr/local/bin/fix-nbextensions

RUN wget https://code-server.dev/install.sh && \
    sh install.sh --version 3.10.2 && \
    rm -rf ~/.cache/code-server/ && \
    rm -rf ./install.sh && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

USER ${NB_UID}
RUN conda install --quiet --yes \
    'black=21.*' \
    'ipydrawio=1.1.*' \
    'ipydrawio-export=1.1.*' \
    'jupyter-server-proxy=3.1.*' \
    'jupyter-vscode-proxy=0.1.*' \
    'jupyterlab_code_formatter=1.4.*' \
    'jupyterlab-lsp=3.8.*' \
    'jupyterlab-system-monitor=0.8.*' \
    'jupyterlab-git=0.32.*' \
    'jupytext=1.11.*' \
    'jupyter_nbextensions_configurator=0.4.*' \
    'jupyter_contrib_nbextensions=0.5.*' \
    'nbgitpuller=0.10.*' \
    'python-lsp-server=1.2.*' \
    'xeus-python=0.12.*' \
    && \
    if [ "$TARGETARCH" = "amd64" ]; then \
    conda install --quiet --yes 'rise=5.7.*'; \
    fi && \
    jupyter nbextension enable code_prettify/code_prettify --sys-prefix && \
    jupyter nbextension enable init_cell/main --sys-prefix && \
    jupyter nbextension enable scratchpad/main --sys-prefix && \
    jupyter nbextension enable spellchecker/main --sys-prefix && \
    jupyter nbextension enable codefolding/main --sys-prefix && \
    jupyter nbextension enable --section edit codefolding/edit --sys-prefix && \
    pip install --quiet --no-cache-dir \
    'aquirdturtle_collapsible_headings==3.1.*' \
    'manim==0.10.*' \
    'itikz==0.1.*' \
    'jupyter-www-proxy' \
    'mytutor' \
    'jdc' \
    && \
    conda clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# Tensorflow and PyTorch
RUN if [ "$TARGETARCH" = "arm64" ]; then \
        pip install --quiet --no-cache-dir \
        'tensorflow==2.6.*' \
        'tensorflow-datasets==4.4.*' \
        'tensorboard==2.6.*' \
        'tensorflowjs==3.8.*' \
        -f https://tf.kmtea.eu/whl/stable.html && \
        pip install --quiet --no-cache-dir --force-reinstall \
        'torch==1.9.*' \
        'torchvision==0.10.*'; \
    elif [ "$TARGETARCH" = "amd64" ]; then \
        pip install --quiet --no-cache-dir \
        'tensorflow-cpu==2.6.*' \
        'tensorflow-datasets==4.4.*' \
        'tensorflow-addons==0.13.*' \
        'tensorboard==2.6.*' \
        'tensorflowjs==3.8.*' && \
        pip install --quiet --no-cache-dir --force-reinstall \
        'torch==1.8.2+cpu' \
        'torchvision==0.9.2+cpu' \
        'torchaudio===0.8.2' \
        -f 'https://download.pytorch.org/whl/lts/1.8/torch_lts.html'; \
    fi && \
    pip install --quiet --no-cache-dir --force-reinstall \
    'numpy==1.20.*' \
    'typing-extensions' \
    && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# Import matplotlib the first time to build the font cache.
ENV XDG_CACHE_HOME="/home/${NB_USER}/.cache/"

RUN MPLBACKEND=Agg python -c "import matplotlib.pyplot" && \
    fix-permissions "/home/${NB_USER}"

USER root

COPY condarc "${CONDA_DIR}/.condarc"
RUN fix-permissions /etc/jupyter/ && fix-permissions "${CONDA_DIR}/.condarc"

# fix https://github.com/jupyter/docker-stacks/issues/815
RUN cat /etc/skel/.bashrc >> /etc/bash.bashrc 

USER ${NB_UID}

WORKDIR "${HOME}"