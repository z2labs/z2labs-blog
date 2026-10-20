---
title: "Poor GPS Rx performance"
date: 2025-12-30
slug: poor-gps-rx-performance
description: "What is blinding GPS reception?"
authors: ["Zoltan Doczi"]
categories: ["Success Story"]
tags: ["success-story"]
aliases: ["/article/poor-gps-rx-performance"]
cover:
  image: noisy_spectrum.png
  alt: "Poor GPS Rx performance"
  relative: true
---

This is a short Z2labs success story.

This issue was rather poor GPS sensitivity with intermittent satellite acquisitions. With a few initial measurements it was concluded that the GPS receiver antenna port noise floor was unacceptably high. The source of the noise was soon traced back to the nearby operating software radio (SDR) that generated wideband noise from its USB3 connection to the PC.

Since the noise could not be eliminated by any configuration change on the USB communication, the task from that point on was to confine the radiated noise power close to the source with various shielding techniques and minimize leakage through the boundaries of the shield. With a few iterations we zeroed in on an acceptable solution that recovered GPS sensitivity.
