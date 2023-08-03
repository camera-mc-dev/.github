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

#### Visualising .c3d tracks

The `.c3d` pose reconstructions (or indeed, `.c3d` mocap files from other sources) can be rendered onto the images using the tools `projectMocap` or `compareMocap`.

As with `renderSparsePoses` both normal and headless rendering are possible.  A short config file is used to specify what to render, rather than have a long list of arugments. You can get a fully documented example config file from the `mc_reconstruction/configs` directory.

```bash
cd ~/data/biocv-final/P03_CMJM_01
cp /path/to/mc_dev/mc_reconstruction/configs/proj.cfg .
```
Edit the config file then run the needed render tool:

```bash
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/projectMocap proj.cfg
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/compareMocap proj.cfg
```

`projectMocap` and `compareMocap` are only mildly different. `projectMocap` will just project all the points in the input `.c3d` files without caring about such things as skeletons or whether the points are from different files. `compareMocap` will require a skeleton file for each input `.c3d` file and will render lines as per that skeleton - skeletons of different input files will be drawn with different colours.




### OpenSim solvers
The OpenSim tools are mostly little python script wrappers.
If you've not already done so, build and install OpenSim, create a virtual environment, activate it, and install the python requirements:

```bash
cd /path/to/mc_dev/mc_opensim
bash opensim-core-linux-build-script.sh -p /opt/opensim
python3 -m venv venv --system-site-packages
source venv/bin/activate
python3 -m pip install update pip
python3 -m pip install -r requirements.txt
```

The OpenSim process consists of gap filling, smoothing, model scaling, and model fitting.

Gap filling and smoothing _can_ be run for one or more files using:

```bash
python3 FillSmoothC3D.py  False <trans noise> <obs noise> < file00.c3d > [ file01.c3d] ... [ file##.c3d ] 
```

For example:

```bash
FillSmoothC3D.py  False -1 -1 ~/data/biocv-final/P03/P03_CMJM_01/op-recon/body-00.c3d
```

The noise parameters can be set to something specific, or set to negative values to automatically compute something appropriate.

Mostly however the `mc_opensim` tools are used as a batch process. Edit the `config.py` file and the set the PATH to your dataset or a specific trial - the scripts will search for .c3d files beneath that path and process them. While editing the `config.py` file you can also prepare the configuration for the scaling and fitting processes.

To scale the opensim model to the openpose data (or to any suitable .c3d point set) you will need to have a static trial - i.e. a trial where your subject stands still. In the event that you don't have such a thing, pick one of your trials where the person has a moment of standing still, and adjust the `SCALE_TIME_RANGE` to identify that moment of standing still. For our `biocv` dataset exampel, we can use the `P03_CMJM_01` data and just the first second before the subject jumps, so we set `STATIC_NAME = "CMJM_01"` and `SCALE_TIME_RANGE = [0, 1]`

We can now run:
```bash
python3 FillSmoothC3D.py  True
python3 TrcGenerator.py
python3 BatchIK.py
```

To do filling and smoothing, then convert `.c3d` to `.trc` and then run OpenSim model fitting based on the settings in the config file.

#### Visualising OpenSim solves

The OpenSim solves can be rendered back over the images using the `renderOpensim` tool from `mc_reconstruction`. Get hold of the config file, update it with the relevant settings, and then simply run the tool. The tool can render into a window, or headless into a video file.

```bash
cd ~/data/biocv-final/P03_CMJM_01
cp /path/to/mc_dev/mc_reconstruction/configs/renderOsim.cfg .
/path/to/mc_dev/mc_reconstruction/build/optimised/bin/renderOpensim renderOsim.cfg
```
Note: You will need to obtain the `.vtp` geometry files from OpenSim, and then convert those files to `.obj`. We did this conversion much as [VTK-OBJ](https://github.com/lodeguns/VTK-OBJ).

