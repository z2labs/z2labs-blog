---
title: "Failing EMI test"
date: 2024-09-23
slug: failing-emi-test
description: "What is causing an unexpected radio EMI railure?"
authors: ["Zoltán Papp"]
categories: ["Success Story"]
tags: ["success-story"]
aliases: ["/article/failing-emi-test"]
cover:
  image: GTEM-Cell.png
  alt: "Failing EMI test"
  relative: true
weight: 2
---

This is a short Z2labs success story.

This issue was an electromagnetic immunity (EMI) regulatory test failure on a 868 MHz ISM band design rather late in its product development phase. Failure rate and level of severity was somewhat intermittent dependent on the test sites the measurements were performed at. At one site it was a pass, at another a marginal fail while at yet another a clear fail. We managed to reproduce the issue straight away in a GTEM cell. A quick conclusion was drawn that the failure only occurred if the blocker signal had AM - as set out in the regulatory standard - and did not occur at all even at higher than required blocker power levels without the modulation. As a next step the RSSI provided by the receiver was monitored at a failing blocking scenario and it was observed that it jumped between a few discrete values. Suspicion rose that the modulation on the blocker might force the receiver into different gain configurations and the change between them might be responsible for the packet losses.

Following up on this idea performance was measured conducted, and the issue could be reproduced. With the conducted measurement we had a much more controllable test setup, there were no more ambiguous results measured at different sites. Measurement data showed a “black hole” in the receive performance between two distinct blocker power levels where Rx was practically non-functional. The width of the “black hole” in power was exactly the same as the peak to minimum power ratio on the blocker with AM. There was therefore one critical power level the Rx changed its gain configuration that led to missed packets and the AM content of the blocker kept on triggering this point. Interestingly enough higher than required blocker power levels may have very well stayed above the trigger point and made the test pass, which could be an explanation on the passing results at some of the measurement sites. Also an interesting aspect that although an unmodulated blocker could also hit the trigger point, it was extremely unlikely. From this point on the issue was brought up with the silicon vendor where a FW change could eliminate the issue. While it was still quite a lot of work to validate the modified FW, no HW changes had to be made on the product, which was very much desirable. It took around 3 days to get to the root cause with the vendor from the first measurements in the GTEM cell.
