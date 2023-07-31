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

The general pipeline will be much the same for other datasets, but the BioCV dataset provides a good example. BioCV already supplies calibration, but here we will go through the process of calibration. After that, we can run the mocap pipeline.

 0) Calibrate dataset
 1) Run pose detectors
 2) Run 3D fusion processes
 3) Run OpenPose solvers

This example will use the `P01_CMJM_01` trial from the BioCV dataset.

### Common config file

All the `mc_dev` tools can can make use of a common configuration file, which will identify the location of a few common datafiles, basic configuration settings, etc. The first time you run one of the tools you will probably get an error because you do not have this file, but a basic sample should get created in your home directory.

Either way, create a file `~/.mc_dev.commong.cfg` which contains (adjust the paths accordingly):

```bash
dataRoot = "/path/to/where/you/keep/your/data";
shadersRoot = "/path/to/mc_dev/mc_core/shaders;"
coreDataRoot = "/path/to/mc_dev/mc_core/data;"
scriptsRoot = "/path/to/mc_dev/mc_core/python;"
netsRoot = "/path/to/mc_dev/mc_nets/data;"
ffmpegPath = "/usr/bin/ffmpeg";
maxSingleWindowWidth = 1200;
maxSingleWindowHeight = 1000;
```

### Calibration

Full details of the [calibration procedure]() can be found in the `mc_core` documentation 

### Pose detection

The pose fusion process is agnostic to the type of pose detector you use, however the provided OpenSim tools are configured on the assumption of an OpenPose/COCO skeleton. The pose fusion processes are designed to load OpenPose style `.json` files, so keep that in mind if using other pose detectors. We have had success using `mmpose` as well as OpenPose, AlphaPose, etc.

As an example, assuming the use of `OpenPose`, we can for example do:

```bash
# navigate to your openpose install
cd /opt/software/openpose

# run openpose to output .json for each camera - here's a command for camera 00
./build/examples/openpose/openpose.bin \
                -video /data2/biocv-final/P04/P04_CMJM_01/00.mp4 \
                -write_json /data2/biocv-final/P04/P04_CMJM_01/openpose_output_00 \
                -display 0 \
                --render_pose 0 \
                -num_gpu 1
```

### Pose fusion

There are two tasks to run. The first is occupancy based pose tracking, this will "project" the detected poses into an occupancy map to identify the probable presence of people and resolve cross-camera detection associations, then track those people over time. The second task is to then reconstruct 3D pose for each person.

#### Occupancy map based Tracking

For this we use the tool `trackSparsePoses`. Get hold of a modify the appropriate configuration file, an [example of which](https://github.com/camera-mc-dev/mc_reconstruction/blob/master/configs/track-fuse.cfg) is available in the `mc_reconstruction` repository. The configuration file is heavily commented and should clearly detail the meaning of and importance of each setting.

Once you have the config file set appropriately, you can run the two tools.

```bash
cd /data2/biocv-final/P04/P04_CMJM_01
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/trackSparsePoses track-fuse.cfg
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/fuseSparsePoses track-fuse.cfg
```

You should end up with a `.c3d` tracking file as your output.


### OpenSim solvers
