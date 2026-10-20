---
title: "Intermittent device lock-up"
date: 2026-02-01
slug: intermittent-device-lock-up
description: "Hunting down intermittent hang issues"
authors: ["Zoltán Papp", "Zoltan Doczi"]
categories: ["Success Story"]
tags: ["success-story"]
aliases: ["/article/intermittent-device-lock-up"]
cover:
  image: LogicANalyser_Debug.png
  alt: "Intermittent device lock-up"
  relative: true
---

This are short Z2labs success stories.

## Intermittent issues

 Intermittent issues are one of the hardest to get to the bottom of. Quite often a fresh set of eyes can move these investigations forward.  Most of the effort in such cases is put into creating a reproducer that is itself reproducible. Getting to this stage requires instrumenting up the device and running/repeating various test scenarios until the issue is caught.

## High output power self lock-up

One of these cases, a 868 MHz ISM band single chip tuner design intermittently produced a lock-up by entering into an not defined high current consumption stage with no communication to the host controller. The device had a relatively high RF output power stage. The investigation led to the observation that lock-up only occurred – albeit still intermittently – at maximum output power configurations. Theory was that some radiated power coupled back to digital lines on the PCB and got rectified on the IC input pins making the radio IC go into lock up. Sure enough, cutting off the digital lines one by one made the issue disappear. From this point on the final manufacturable solution was to revamp the PCB layout to prevent sensitive traces being directly exposed. While the PCB could not be salvaged, the next iteration made it manufacturing ready.

## Critical timing at device initialisation

Another case, again a sub GHz ISM band design produced intermittent hang-ups on some of the devices. Current consumption indicated a definite state during start-up the radio IC stack at. The key to solving the issue was instrumenting the device with DC current and clock signal monitoring in various domains and placing the device in an infinite start-up execution loop until the issue showed. The issue occurrence then could be correlated to the phase relationship between two clock signals that were meant to do a handover between start-up states. Consulting the silicon vendor it turned out that there was an extra (not published) timing parameter involved in the start-up process that was only marginally adhered to in the application. Start-up commands followed each other too slowly which from the programming guide seemed a perfectly legitimate solution. Adjusting the start-up sequence commands accordingly finally made the issue go away.
