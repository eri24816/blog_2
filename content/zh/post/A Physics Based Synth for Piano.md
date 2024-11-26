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
I really like the sounds of piano. They are a subset of all possible sound waves, with some specific mathematical characteristics which make them sound bright but gentle at the same time. I've been trying to understand what's the magic inside the sounds of piano from i was 12 maybe till now, but I guess my math is still too bad to actually understand it from a fundamental aspect.

Recently I try to make a synthesizer for piano by directly simulate the vibration of piano string and sample audio from it. Although the result sounds not actually like a real piano, the journey of implementing the synth is interesting enough to write down here.

# The Model of a String

Piano strings have an important characteristic: 

[Modeling Stiff String for Numeric Simulation]({{< ref "Modeling Stiff String for Numeric Simulation" >}})