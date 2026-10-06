---
layout: page
title: AmBe / PuBe source-term simulation
description: An analytical Geant4 primary generator for α-induced ⁹Be neutron sources.
img: assets/img/publication_preview/X3.png
importance: 1
category: work
related_publications: true
---

The centrepiece of my PhD. Instead of simulating $^{241}$Am $\alpha$ decay interacting with $^{9}$Be
through a Geant4 physics list, this simulation uses the measured differential cross sections of the
$^{9}$Be($\alpha$,n)$^{12}$C reaction to generate neutrons and $^{12}$C ions directly in the
`PrimaryGenerator` class.

- Produces Doppler-broadened 4.4 MeV and 3.2 MeV $\gamma$ rays from $^{12}$C de-excitation, so the
  excited-state ratio per neutron can be verified.
- Each stage is validated against accepted data, including the $^{241}$Am spontaneous-fission
  fragment mass distribution and the emergent neutron spectrum.
- Includes a (less extensively verified) $^{239}$PuBe implementation, and the source term plus a
  simulation template are released publicly with the paper.

Published as {% cite Falezza2025_AmBe %}.
