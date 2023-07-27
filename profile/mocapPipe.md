# Markerless motion capture pipeline

## Installation

### Getting the software

First, clone the `mc_base` repository, and then the other needed repositories.

```bash
cd /where/you/want/to/put/the/code
git clone git@github.com:camera-mc-dev/mc_base mc_dev
cd mc_dev
bash cloneMocapRepos.sh
```

### Installing Dependencies

For more details, see the documentation for [mc_core](https://github.com/camera-mc-dev/mc_core) and [mc_reconstruction](https://github.com/camera-mc-dev/mc_reconstruction), [mc_opensim](https://github.com/camera-mc-dev/mc_opensim)

The following script _might_ do most of the work for you, assuming an Apt/Ubuntu based system:

```bash
python3 getMocapDeps.py
```



### Building

Assuming everything is properly installed to standard locations:

```bash
scons mc_reconstruction/build/optimised/ -j8
```

## Running on BioCV

