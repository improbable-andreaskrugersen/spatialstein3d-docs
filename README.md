# Spatialstein3D
## Overview
This repository contains the documentation for [Spatialstein 3D](https://github.com/improbable-andreaskrugersen/spatialstein3d). The project is a mixture of an example project and tutorial that shows how to get started with a SpatialOS project using the SpatialOS WorkerSDK. We start with a simple Wolfenstein3D clone without any networking or major features, then convert the project into a SpatialOS project and subsequently build features on top of that in "the Spatial way". No previous knowledge about SpatialOS is assumed but this doc will frequently point to the [official SpatialOS documentation](https://documentation.improbable.io).

Why is this a Wolfenstein3D clone? There are several reasons:
- The main concern of this project is to show SpatialOS features in the context of a game
- The game must be simple enough that it doesn't require lots of code but must have enough room for growth to add interesting, networked features
- It shouldn't require loads of external dependencies
- And just because a little bit of software rendering is still fun. I recently came upon [Lode's excellent Raycasting Tutorial](https://lodev.org/cgtutor/raycasting.html) and got inspired. Since it's completely software-rendered, it's pretty slow if you go to high resolutions, but that's not hugely important for our purposes. When in doubt, just pick a resolution that works reasonably well on your system

## Structure
This docs repository is split into several chapters. Each chapter corresponds to a matching branch in the code repository and represents a complete snapshot of the project at a specific point in time. If you just want to see all the code with the latest features, use the latest chapter. If you want to see how a SpatialOS project can be approached from the beginning, start with the first chapter. Or use any chapter in-between if you are interested in just a specific feature. Each chapter branch builds on the previous chapters but is otherwise self-contained and can be used as a basis for own experiments.

The documentation is kept in a different repository to keep it consistent across all chapters without the need to cascade it through all branches whenever the docs content changes.

### List of Chapters
- [Chapter 1 - Basic project setup](chapter1.md)
- [Chapter 2 - SpatialOS project setup](chapter2.md)
- [Chapter 3 - Connecting to a deployment](chapter3.md)

## Platforms
The project was tested on both Ubuntu 18.04 and Windows 10.

## Dependencies
External dependencies were kept to a minimum. However, these dependencies are required and should be installed before proceeding with the individual chapters:

### Bazel
We use [Bazel](https://bazel.build/) for building the project. Please follow the [official installation instructions](https://docs.bazel.build/versions/master/install.html).

### SDL
The project is completely software-rendered and uses [SDL](https://www.libsdl.org/) for all its rendering.

#### Ubuntu 18.04

`sudo apt-get install libsdl2-dev libsdl2-image-dev libsdl2-ttf-dev`

#### Windows

SDL is automatically downloaded while building with Bazel, nothing to do here.

### Other libraries
Other libraries (where needed) were included directly in the repository.
