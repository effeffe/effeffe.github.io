---
layout: page
title: Geant4-MPI for OpenMPI 3+
description: Rewriting the Geant4 MPI interface — merged upstream into Geant4 11.4.
img: assets/img/G4.png
importance: 2
category: work
github: https://github.com/bhamnuclear/Geant4-MPI
related_publications: false
---

The `G4mpi` example interface shipped with Geant4 did not build against OpenMPI 3+ because it
relied on the deprecated C++ MPI bindings. This project rewrites the interface using the C
bindings and reworks the argument parser so the host application can still receive its own
command-line arguments.

- Repository: [bhamnuclear/Geant4-MPI](https://github.com/bhamnuclear/Geant4-MPI)
- Merged upstream in [Geant4 PR #81](https://github.com/Geant4/geant4/pull/81), released in **Geant4 11.4**.

This is what lets the AmBe simulation scale across the Birmingham compute nodes.
