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

This example will use the `P03_CMJM_01` trial from the BioCV dataset.

### Common config file

All the `mc_dev` tools can can make use of a common configuration file, which will identify the location of a few common datafiles, basic configuration settings, etc. The first time you run one of the tools you will probably get an error because you do not have this file, but a basic sample should get created in your home directory.

Either way, create a file `~/.mc_dev.commong.cfg` which contains (adjust the paths accordingly):

```bash
dataRoot = "/path/to/where/you/keep/your/data";
shadersRoot = "/path/to/mc_dev/mc_core/shaders";
coreDataRoot = "/path/to/mc_dev/mc_core/data";
scriptsRoot = "/path/to/mc_dev/mc_core/python";
netsRoot = "/path/to/mc_dev/mc_nets/data";
ffmpegPath = "/usr/bin/ffmpeg";
maxSingleWindowWidth = 1200;
maxSingleWindowHeight = 1000;
```

### Calibration

Full details of the [calibration procedure]() can be found in the `mc_core` documentation, but here is a reduced step-by-step. In practice, when using the BioCV dataset, you will use the already existing calibration files.

The calibration videos are in the `P03/calib_00` trial. Copy the base camera calibration configuration file from the `mc_dev/data/baseConfigs`. So assuming you have the data under `~/data/biocv-final`:

```bash
cd ~/data/biocv-final/P03
cp /path/to/mc_dev/mc_core/data/baseConfigs/calib.cfg .
```
Open the file for editing and set the `dataRoot`, `testRoot` and `imgDirs` (imgDirs can be directories under `/dataRoot/testRoot/` or can be video files). The default settings for the grid finder should be correct for the BioCV dataset. Disable `useExistingGrids`, disable `useSBA`, comment out `matchesFile`. 

Run the calibration tool - this first run is primarily just to detect the grids, it will probably crash or do something silly after detection completes (lingering unfixed bug)

```bash
cd ~/data/biocv-final/P03
/path/to/mc_dev/mc_core/build/optimised/bin/circleGridCamNetwork calib.cfg
```
After that, edit the config file to enable `useSBA` and `useExistingGrids`.

Run the tool again. You now have a calibration, but the world coordinate system is not aligned to the floor plane. Edit the config file and uncomment the `matchesFile=matches`. `mc_dev` contains a tool for annotating points in the scene. At a minimum, you will need to annotate 3 points on the ground plane, the desired scene origin, a point on the world x-axis, and a point on the world y-axis.

```bash
cd ~/data/biocv-final/P03
/path/to/mc_dev/mc_core/build/optimised/bin/pointMatcher calib.cfg
```

Once you have the matches, you can run the tool to align the world to the desired origin. 

```bash
cd ~/data/biocv-final/P03
/path/to/mc_dev/mc_core/build/optimised/bin/manualAlignNetwork calib.cfg
```

For troubleshooting and tools to test/visualise the calibration, see the full documentation of `mc_core`.

NOTE: If you're using the BioCV dataset you will probably want to use the calibration files that come with it, as those have been painstakingly aligned to the best of our ability with the BioCV motion capture data. These instructions are mostly for relevance on other data.

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

#### Visualising pose detections

`mc_reconstruction` provides a tool to visualise the sparse pose detections - you probably already have that from your pose detector anyway, but it is useful for seeing that `mc_dev` is loading it the way you expect it to be loaded. The tool is called `renderSparsePoses`. You can either run it normal, or headless - if you run it headless you must specify the name of the output directory or video file you want to write output to. You will need to have a suitable skeleton configuration file that tells `mc_dev` the layout of keypoints from your detector. We provide examples for OpenPose.

normal (renders to a window):

```bash
cd ~/data/biocv-final/P03_CMJM_01
cp /path/to/mc_dev/mc_reconstruction/configs/open.skel.cfg ../
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/renderSparsePose 00.mp4 ../open.skel.cfg jsonDir openpose_output_00/
```

headless (render to video file):

```bash
cd ~/data/biocv-final/P03_CMJM_01
cp /path/to/mc_dev/mc_reconstruction/configs/open.skel.cfg ../
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/renderSparsePose 00.mp4 ../open.skel.cfg jsonDir openpose_output_00/ op-render-00.mp4 
```

### Pose fusion

There are two tasks to run. The first is occupancy based pose tracking, this will "project" the detected poses into an occupancy map to identify the probable presence of people and resolve cross-camera detection associations, then track those people over time. The second task is to then reconstruct 3D pose for each person.

#### Occupancy map based Tracking

For this we use the tool `trackSparsePoses`. Get hold of a modify the appropriate configuration file, an [example of which](https://github.com/camera-mc-dev/mc_reconstruction/blob/master/configs/track-fuse.cfg) is available in the `mc_reconstruction` repository. The configuration file is heavily commented and should clearly detail the meaning of and importance of each setting. You will also need a configuration file describing the "skeleton" provided by the pose detector. There are examples for OpenPose and AlphaPose - we will use the OpenPose one here.

```bash
cd ~/data/biocv-final/P03_CMJM_01
cp /path/to/mc_dev/mc_reconstruction/configs/track-fuse.cfg .
cp /path/to/mc_dev/mc_reconstruction/configs/open.skel.cfg ../
```
Once you have the config files and have suitably edited them, you can run the tools:

```
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/trackSparsePoses track-fuse.cfg
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/fuseSparsePoses track-fuse.cfg
```

You should end up with a `.c3d` tracking file as your output at the location you specified in the config file.


### OpenSim solvers
