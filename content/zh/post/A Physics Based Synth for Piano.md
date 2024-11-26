---
title: A Physics Based Synth for Piano
date: 2024-11-26
authors:
  - eri24816
image: https://i.imgur.com/1dGGCaN.png
draft: true
tags:
  - audio
  - Juce
  - Cpp
categories: Uncategorized
series: 
summary:
---
I really like the sounds of piano. It's a specific subset of all possible sound waves, with some specific mathematical characteristics, which make it sounds bright but gentle at the same time. I've been trying to understand what's the magic inside the sounds of piano from maybe 12 till now, but I guess my math is still too bad to actually understand it from a fundamental aspect.

Recently I try to make a synthesizer for piano by directly simulate the vibration of piano string and sample audio from it. Although the result sounds not quite like a real piano, I still feel the journey of implementing the synth is interesting enough to write down here.

[Modeling Stiff String for Numeric Simulation]({{< ref "Modeling Stiff String for Numeric Simulation" >}})