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
Where $u$ is the displacement. Intuitively, the equation says that on each point of the string, a tension is applied to push it towards its neighbors because of the string's elasticity.

Some of the strings in the real world can be well modeled by the equation above. On the other hand, piano strings have another important mechanical characteristic, stiffness, which means the string tend to not bend. On the equation, stiffness introduces a $4^{th}$ order term.
$$
\begin{align}
\frac{\partial^2 u}{\partial t^2} &= c^2 \frac{\partial^2 u}{\partial x^2} - \frac{ESK^2}{\rho} \frac{\partial^4 u}{\partial x^4}
\end{align}
$$
A nice article [The wave equation for stiff strings and piano tuning](https://upcommons.upc.edu/bitstream/handle/2117/101752/GraciaSanz.piano.RSCM.2017.pdf) provides the solution to the equation of stiff strings.
$$
\begin{align}
u(x,t) &= \sum_{n=1}^{\infty} \left( a_n \cos(2\pi f_n t) + b_n \sin(2\pi f_n t) \right) \sin\left(\frac{n\pi}{L}x\right)
\end{align}
$$
where $f_n = n f_0 \sqrt{1 + Bn^2}$, $B = \frac{\pi^2 ESK^2}{\tau L^2}$, $f_0 = \frac{c}{2L}$.

The $\sqrt{1 + Bn^2}$ illustrates how stiffness causes inharmonicity to the sound of the string.

The amplitudes of harmonics $a_n$ and $b_n$ are the variables to be tracked and calculated in each time step in the simulation. In a real piano, the string interacts with the hammer, the damper, and the bridge, etc.. When external forces are applied to the string at a certain position $x0$ and with $J$ , $a_n$ and $b_n$ changes accordingly (see [Modeling Stiff String for Numeric Simulation]({{< ref "Modeling Stiff String for Numeric Simulation" >}}) for detail):
$$
\begin{align}
a_n(t_0^+) &= a_n(t_0^-) + \frac{2J}{L\rho\omega _n} \sin\left(\frac{n\pi}{L}x_0\right) \cos(2\pi f_n t_0) \\\
b_n(t_0^+) &= b_n(t_0^-) + \frac{2J}{L\rho\omega _n} \sin\left(\frac{n\pi}{L}x_0\right) \sin(2\pi f_n t_0)
\end{align}
$$

# Turning the Model into Code!

Here comes the most fun part when building a simulation: to turn the math into code and verify that the math (I've worked hard on) actually works.

I implemented the synthesizer with Juce, a framework to build VSTs. The actual code for Juce is complicated. I will show simplified code in the following content, but keep the essence.

## Main Loop
This is the main loop for rendering audio. For each time step, the loop update the simulation then sample the displacement of the string at x=0.01meter. The displacement is then output as the audio sample.
```c++
// synthVoice.h
// class SynthVoice : public juce::SynthesiserVoice

void renderNextBlock (AudioBuffer <float> &outputBuffer, int startSample, int numSamples) override
{
	for (int sample = 0; sample < numSamples; ++sample)
	{
		simulation->update();
		
		const float currentSample = string->sampleU(0.01);
		
		outputBuffer.addSample(startSample + sample, currentSample);
	}
}
```
## Simulation
Let's see what the line `simulation->update();` does. In the simulation, there are two objects, the string and the hammer, and one interaction, the hammer-string-interaction. For each time step, interaction->apply is called, making the hammer and the string add forces to each other. Then object->update on both objects are called to update their step
```c++
void Simulation::update()
{
	for (auto& interaction : interactions)
	{
		interaction->apply(t, dt);
	}
	for (auto& object : objects)
	{
		object->update(t, dt);
	}
	t += dt;
}
```