# DIVE environment for Mutual information and Machine Learning

[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/ccha23/mimldive/HEAD?urlpath=git-pull?repo%3Dhttps%3A%2F%2Fgithub.com%2Fccha23%2Fcscit21%26urlpath%3Dlab%2F%2Ftree%2Fcscit21)

See the [jupyterbook](https://www.cs.cityu.edu.hk/~ccha23/miml) for how to run the environment locally on your computer.

# Draft

Installation Instructions

Step 1: Environment Setup

Download and Install Conda env:

Windows x86_64:
https://repo.anaconda.com/miniconda/Miniconda3-py38_4.10.3-Windows-x86_64.exe

macOS x86_64:
https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-x86_64.sh

macOS arm64:
https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh

Installation instructons:
https://conda.io/projects/conda/en/latest/user-guide/install/index.html

Step 2: Download environment file from GitHub

Windows x86_64:
https://github.com/ccha23/mimldive/raw/main/mimlenv-cp38-windows_x86_64.yml

macOS x86_64:
https://github.com/ccha23/mimldive/raw/main/mimlenv-cp38-macos_11_x86_64.yml

macOS arm64:
https://github.com/ccha23/mimldive/raw/main/mimlenv-cp38-macos_11_arm64.yml

Step 3: Install mimldive Environment

In your terminal run this command, replacing the uppercase variables with the path to your environment.yml file and your desired name for this environment: conda env create --file=PATH_TO_ENVIRONMENT.YML --name=YOUR_ENV_NAME_HERE.

Step 4: Install Tensorflow (macOS Only)

You can choice tensorflow release from:

PRE-RELEASE Tensorflow r2.4rc0 with hardware-accelerated for macOS 11.0+:

Run this command if your Mac using Intel Chips
pip install --upgrade --force --no-dependencies https://github.com/apple/tensorflow_macos/releases/download/v0.1alpha3/tensorflow_macos-0.1a3-cp38-cp38-macosx_11_0_x86_64.whl https://github.com/apple/tensorflow_macos/releases/download/v0.1alpha3/tensorflow_addons_macos-0.1a3-cp38-cp38-macosx_11_0_x86_64.whl tensorflowjs

Run this command if your Mac using Apple Silicon
pip install --upgrade --force --no-dependencies https://github.com/apple/tensorflow_macos/releases/download/v0.1alpha3/tensorflow_macos-0.1a3-cp38-cp38-macosx_11_0_arm64.whl https://github.com/apple/tensorflow_macos/releases/download/v0.1alpha3/tensorflow_addons_macos-0.1a3-cp38-cp38-macosx_11_0_arm64.whl tensorflowjs

Stable release Tensorflow 2.6

Run this command:
pip install tensorflow tensorflow-addons tensorflowjs
-----------------------------------------------------

For gpu support, please reference to:
https://www.tensorflow.org/install/gpu

