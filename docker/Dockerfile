# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.

# ----

# Ubuntu 20.04 (focal)
# https://hub.docker.com/_/ubuntu/?tab=tags&name=focal

### BASE-NOTEBOOK ###
ARG ROOT_CONTAINER=nvidia/cuda:11.2.2-cudnn8-runtime-ubuntu20.04

FROM $ROOT_CONTAINER

LABEL maintainer="Jupyter Project <jupyter@googlegroups.com>"
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

# Fix DL4006
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

# ---- Miniforge installer ----
# Default values can be overridden at build time
# (ARGS are in lower case to distinguish them from ENV)
# Check https://github.com/conda-forge/miniforge/releases
# Conda version
ARG conda_version="4.10.3"
# Miniforge installer patch version
ARG miniforge_patch_number="3"
# Package Manager and Python implementation to use (https://github.com/conda-forge/miniforge)
# - conda only: either Miniforge3 to use Python or Miniforge-pypy3 to use PyPy
# - conda + mamba: either Mambaforge to use Python or Mambaforge-pypy3 to use PyPy
ARG miniforge_python="Mambaforge"
# Miniforge archive to install
ARG miniforge_version="${conda_version}-${miniforge_patch_number}"


# Install all OS dependencies for notebook server that starts but lacks all
# features (e.g., download as all possible file formats)
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    tini \
    wget \
    ca-certificates \
    sudo \
    locales \
    fonts-liberation \
    run-one \
    openssh-client && \
    # wget https://packages.vmware.com/bitfusion/ubuntu/20.04/bitfusion-client-ubuntu2004_3.5.0-5_amd64.deb && \
    # apt-get install --yes --no-install-recommends ./bitfusion-client-ubuntu2004_3.5.0-5_amd64.deb && \
    # rm -rf ./bitfusion-client-ubuntu2004_3.5.0-5_amd64.deb && \
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
    HOME="/home/${NB_USER}" \
    CONDA_VERSION="${conda_version}" \
    MINIFORGE_VERSION="${miniforge_version}"

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
    usermod -aG ${NB_GID} ${NB_USER} && \
    # usermod -aG bitfusion "${NB_USER}" && \
    mkdir -p "${CONDA_DIR}" && \
    chown "${NB_USER}:${NB_GID}" "${CONDA_DIR}" && \
    chmod g+w /etc/passwd && \
    fix-permissions "${HOME}" && \
    fix-permissions "${CONDA_DIR}"

USER ${NB_UID}
ARG PYTHON_VERSION=3.8.10

# Setup work directory for backward-compatibility
RUN mkdir "/home/${NB_USER}/work" && \
    fix-permissions "/home/${NB_USER}"

# Install conda as jovyan and check the sha256 sum provided on the download site
WORKDIR /tmp

# Prerequisites installation: conda, mamba, pip, tini
RUN set -x && \
    # Miniforge installer
    miniforge_arch=$(uname -m) && \
    export miniforge_arch && \
    if [ "$miniforge_arch" == "aarm64" ]; then \
      miniforge_arch="arm64"; \
    fi; \
    miniforge_installer="${miniforge_python}-${miniforge_version}-Linux-${miniforge_arch}.sh" && \
    export miniforge_installer && \
    wget --quiet "https://github.com/conda-forge/miniforge/releases/download/${miniforge_version}/${miniforge_installer}" && \
    /bin/bash "${miniforge_installer}" -f -b -p "${CONDA_DIR}" && \
    rm "${miniforge_installer}" && \
    # Conda configuration see https://conda.io/projects/conda/en/latest/configuration.html
    echo "conda ${CONDA_VERSION}" >> "${CONDA_DIR}/conda-meta/pinned" && \
    conda config --system --set auto_update_conda false && \
    conda config --system --set show_channel_urls true && \
    if [[ "${PYTHON_VERSION}" != "default" ]]; then conda install --yes python="${PYTHON_VERSION}"; fi && \
    conda list python | grep '^python ' | tr -s ' ' | cut -d ' ' -f 1,2 >> "${CONDA_DIR}/conda-meta/pinned" && \
    conda install --quiet --yes \
    "conda=${CONDA_VERSION}" \
    'pip' && \
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
RUN conda install --quiet --yes \
    'notebook' \
    'jupyterhub' \
    'jupyterlab' && \
    conda clean --all -f -y && \
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

# Fix permissions on /etc/jupyter as root
USER root

# Copy local files as late as possible to avoid cache busting
COPY start.sh start-notebook.sh start-singleuser.sh /usr/local/bin/

RUN chmod a+rx /usr/local/bin/start.sh /usr/local/bin/start-notebook.sh /usr/local/bin/start-singleuser.sh 
# Currently need to have both jupyter_notebook_config and jupyter_server_config to support classic and lab
COPY jupyter_notebook_config.py /etc/jupyter/

# Prepare upgrade to JupyterLab V3.0 #1205
RUN sed -re "s/c.NotebookApp/c.ServerApp/g" \
    /etc/jupyter/jupyter_notebook_config.py > /etc/jupyter/jupyter_server_config.py && \
    fix-permissions /etc/jupyter/
### BASE-NOTEBOOK ###

### MINIMAL-NOTEBOOK ###
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    vim-tiny \
    git \
    inkscape \
    libsm6 \
    libxext-dev \
    libxrender1 \
    lmodern \
    netcat \
    # ---- nbconvert dependencies ----
    texlive-xetex \
    texlive-fonts-recommended \
    texlive-plain-generic \
    # ----
    tzdata \
    unzip \
    nano-tiny && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Create alternative for nano -> nano-tiny
RUN update-alternatives --install /usr/bin/nano nano /bin/nano-tiny 10
## MINIMAL-NOTEBOOK ###

### For installation of manim and scipy
RUN apt-get update --yes && \
    apt-get install --yes --no-install-recommends \
    build-essential \
    curl \
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

# setup a minimal texlive installation for manim
COPY texlive-profile.txt /tmp/
ENV PATH=/usr/local/texlive/bin/x86_64-linux:$PATH
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

ENV TZ=Hongkong
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Switch back to jovyan
USER ${NB_UID}

RUN conda install --quiet --yes \
    'altair=4.1.*' \
    'beautifulsoup4=4.9.*' \
    'bokeh=2.3.*' \
    'bottleneck=1.3.*' \
    'cloudpickle=1.6.*' \
    'conda-forge::blas=*=openblas' \
    'cython=0.29.*' \
    'dask=2021.6.*' \
    'dill=0.3.*' \
    'h5py=3.3.*' \
    'ipympl=0.7.*' \
    'ipywidgets=7.6.*' \
    'jupyterlab-nvdashboard=0.6.*' \
    'matplotlib-base=3.4.*' \
    'numexpr=2.7.*' \
    'pandas=1.3.*' \
    'patsy=0.5.*' \
    'protobuf=3.17.*' \
    'pytables=3.6.*' \
    'scikit-image=0.18.*' \
    'scikit-learn=0.24.*' \
    'scipy=1.6.*' \
    'seaborn=0.11.*' \
    'sqlalchemy=1.4.*' \
    'statsmodels=0.12.*' \
    'sympy=1.8.*' \
    'widgetsnbextension=3.5.*' \
    'xlrd=2.0.*' && \
    conda clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

# Install facets which does not have a pip or conda package at the moment
WORKDIR /tmp
RUN git clone https://github.com/PAIR-code/facets.git && \
    jupyter nbextension install facets/facets-dist/ --sys-prefix && \
    rm -rf /tmp/facets && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

### END MANIM & SCIPY

### JupyterLab Extension
USER root

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
    'jupyter-videochat=0.5.*' \
    'jupyterlab_code_formatter=1.4.*' \
    'jupyterlab-lsp=3.8.*' \
    'jupyterlab-system-monitor=0.8.*' \
    'jupyterlab-git=0.32.*' \
    'jupyterlab-tour=3.1.*' \
    'jupytext=1.11.*' \
    'jupyter_nbextensions_configurator=0.4.*' \
    'jupyter_contrib_nbextensions=0.5.*' \
    'nbgitpuller=0.10.*' \
    'python-lsp-server=1.2.*' \
    'rise=5.7.*' \
    'xeus-python=0.12.*' \
    && \
    jupyter nbextension enable code_prettify/code_prettify --sys-prefix && \
    jupyter nbextension enable init_cell/main --sys-prefix && \
    jupyter nbextension enable scratchpad/main --sys-prefix && \
    jupyter nbextension enable spellchecker/main --sys-prefix && \
    jupyter nbextension enable codefolding/main --sys-prefix && \
    jupyter nbextension enable --section edit codefolding/edit --sys-prefix && \
    pip install --quiet --no-cache-dir \
    'aquirdturtle_collapsible_headings==3.1.*' \
    'manim==0.7.*' \
    'itikz==0.1.*' \
    'jupyter-www-proxy' \
    'mytutor' && \
    conda clean --all -f -y && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

### Tensorflow ###
RUN pip install --quiet --no-cache-dir \
    'tensorflow-gpu==2.6.*' \
    'tensorflow-datasets==4.4.*' \
    'tensorflow-addons==0.13.*' \
    'tensorboard==2.6.*' \
    'tensorflowjs==3.8.*' && \
    fix-permissions "${CONDA_DIR}" && \
    fix-permissions "/home/${NB_USER}"

### PyTorch 1.8.2cu102 ###
RUN pip install --quiet --no-cache-dir \
    'torch==1.8.2+cu111' \
    'torchvision==0.9.2+cu111' \
    'torchaudio===0.8.2' \
    -f 'https://download.pytorch.org/whl/lts/1.8/torch_lts.html' && \
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
