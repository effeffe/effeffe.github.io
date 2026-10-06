---
layout: page
title: MC40 digital acquisition system
description: Designing and commissioning the DAQ for the Birmingham MC40 cyclotron.
img: assets/img/publication_preview/ISS_Magnet.jpg
importance: 3
category: work
related_publications: false
---

A digital data-acquisition system for experiments at the Birmingham MC40 cyclotron, built to
replace ageing analogue/VME chains.

- **Front end:** a VME crate with four CAEN V2745, one V2740 and one V1730S digitisers, read out
  through a VME bridge.
- **Back end:** two Dell R360 servers with dual 10 GbE NICs in a round-robin bond over a 10 GbE
  switch for redundancy, each running a ZFS RAIDZ2 pool on SSD.
- Commissioned end-to-end and used for LaBr$_3$ + DSSD telescope measurements.
