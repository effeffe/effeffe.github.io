---
layout: page
title: Geant4 MPI update
description: with background image
img: assets/img/G4.png
importance: 1
category: work
related_publications: true
---

Simple rewrite of the G4mpi interface to allow G4mpi use with OpenMPI-3+ compliant software.<br>
The main changes involve the use of C bindings instead of C++, and rewrite of the parser function to allow the main software to parse CLI arguments.<br>
This has been published on [GitHub](https://github.com/bhamnuclear/Geant4-MPI/) and merged in Geant4 11.4 as described in the [Pull Request](https://github.com/Geant4/geant4/pull/81).
