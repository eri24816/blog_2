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

The simplest form of the equation describing a vibrating string is:
$$
\frac{\partial^2 u}{\partial t^2} = c^2 \frac{\partial^2 u}{\partial x^2}
$$
Intuitively, the equation says that on each point of the string, a tension is applied to push it towards its neighbors because of the string's elasticity.

Other than elasticity, piano strings also have stiffness. Stiffness means the string tend to not bend. Stiffness introduces a $4^{th}$ order term to the equation.
$$
\begin{align}
\frac{\partial^2 u}{\partial t^2} &= c^2 \frac{\partial^2 u}{\partial x^2} - \frac{ESK^2}{\rho} \frac{\partial^4 u}{\partial x^4}
\end{align}
$$


[Modeling Stiff String for Numeric Simulation]({{< ref "Modeling Stiff String for Numeric Simulation" >}})