---
title: "dB voltage or power?"
date: 2026-04-14
slug: db-voltage-or-power
description: "Are power and voltage ratios the same in dB?"
authors: ["Zoltán Papp"]
categories: ["Trivia"]
tags: ["trivia"]
aliases: ["/article/db-voltage-or-power"]
cover:
  image: db_vna.png
  alt: "dB voltage or power?"
  relative: true
weight: 2
---

Ratios expressed in decibel (dB) may be confusing in terms of what quantities are involved. In electronics engineering, these are typically power or voltage.

**A common question is: if a quantity is 3 dB, does it refer to a voltage or power ratio? Intuition may suggest that this distinction matters — and in general, it does. However, if the quantities are measured over the same impedance, the distinction becomes irrelevant.**

Below are the calculated ratios for 3 dB in power and in voltage:

P_2/P_1 =10^(dB/10)=10^(3/10)≈1.995≈2
V_2/V_1 =10^(dB/20)=10^(3/20)≈1.412≈√2

So, 3 dB corresponds to a factor of 2 in power and  a factor of √2 in voltage.

What does this mean? **If the voltages are measured across the same impedance, then a √2x increase in voltage results in a 2x increase in power. Therefore, 3 dB represents the same physical change whether expressed as a voltage or power ratio.**

If the impedances are different, this equivalence no longer holds. In that case, the voltage ratio alone is insufficient, and the impedance ratio must be known to determine the corresponding power ratio.

Armed with all of the above, what would our answer be to another common question: **does a VNA display power or voltage ratios?** Since the VNA ports have equal impedance (typically 50 Ω), the voltage and power interpretations are equivalent so the VNA displays both at the same time.

VNAs measure incident and reflected traveling waves via voltage sampling and form their ratios. These ratios are displayed in dB using the 20·log() expression (since the measured quantities are voltage derived). Because the system impedance is the same at all ports, these amplitude ratios directly correspond to power ratios.

While travelling wave amplitudes (often denoted by a and b) are not used often to describe circuits (only their ratios that are the S parameters themselves), it is useful to know that these amplitudes are normalised tarvelling wave voltages. Normalisation is done in a way that amplitude square gives power directly.

a=V+/√(Z0 ) [√W]

b=V-/√(Z0 )  [√W]

Above normalisation ensures that with different impedances on the ports S parameters would still return true power ratios. While this is not something anyone would care about when measuring with a VNA, it comes handy when you simulate circuits with different source and load impedances (like an attenuator for example that works between different impedances).
