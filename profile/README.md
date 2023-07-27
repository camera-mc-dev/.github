# CAMERA mc_dev: Motion capture dev tools

[CAMERA](www.camera.ac.uk) is the Centre for the Analysis of Motion, Entertainment Research and Applications at the University of Bath.

The `camera-mc-dev` organisation on Github is where we have made available a number of repositories relating to our research on markerless motion capture.

These tools are being made available to allow researchers to fully exploit our [BioCV](??) dataset, which provides synchronised video, high quality motion capture, forceplate data and photogrammetry scans for all 15 participants.

## Markerless Motion Capture Pipeline

The [mc_base](https://github.com/camera-mc-dev/mc_base) repository provides a convenient Dockerfile for our markerless pipeline. Mostly, this consists of the repositories:

  1) [mc_core](https://github.com/camera-mc-dev/mc_core): Core library and tools for camera calibration and rendering
  2) [mc_reconstruction](https://github.com/camera-mc-dev/mc_reconstruction): pose tracking and 3D fusion tools, and visualisations.
  3) [mc_opensim](https://github.com/camera-mc-dev/mc_opensim): scripts to take motion capture data, or markerless motion capture tracks, and fit opensim models.
  
The repositories are fully documented and will detail the manual installation steps and how to use each tool.

### Installation and running

There are brief [instructions for installing and running the markeless pipeline using the BioCV dataset](mocapPipe.md), more detailed information about each part can be found in repository documentation.

## Machine Vision Capture tool

In the unlikely event you too have a Silicon Software based machine vision system and want to use our capture tool, then we have made the software available in the [mc_grabber](https://github.com/camera-mc-dev/mc_grabber) repository.

You might also be interested in the tool if you use other hardware, as it is at has been written in such a way that it should (at least theoretically), be possible to use with other framegrabber hardware, depending on just how different the APIs of that hardware are. You will however have to write the interface classes to work with your hardware.

See documentation in the repository for details.
