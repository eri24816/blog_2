---
title: A Physics Based Synth for Piano
date: 2024-11-26
authors:
  - eri24816
image: https://i.imgur.com/1dGGCaN.png
draft: false
tags:
  - audio
  - Juce
  - Cpp
categories: Uncategorized
series: 
summary:
---
I really like the sounds of piano. They are a subset of all possible sound waves, with some specific mathematical characteristics which make them sound bright but gentle at the same time. I've been trying to understand what's the magic inside the sounds of piano from i was maybe 12 till now, but I guess my math is still too bad to actually understand it from a fundamental aspect.

Recently I try to make a synthesizer for piano by directly simulate the vibration of piano string and sample audio from it. Although the result sounds not actually like a real piano, the journey of implementing the synth is interesting enough to be written down here.

# The Model of a String

The simplest form of the equation describing a vibrating string is:
$$
\frac{\partial^2 u}{\partial t^2} = c^2 \frac{\partial^2 u}{\partial x^2}
$$
Where $u$ is the displacement. Intuitively, the equation says that on each point of the string, a tension is applied to push it towards its neighbors because of the string's elasticity.

Some of the strings in the real world can be well modeled by the equation above. On the other hand, piano strings have another important mechanical characteristic, stiffness, which represents how much the string tend to not bend. On the equation, stiffness introduces a $4^{th}$ order term.
$$
\begin{align}
\frac{\partial^2 u}{\partial t^2} &= c^2 \frac{\partial^2 u}{\partial x^2} - \frac{ESK^2}{\rho} \frac{\partial^4 u}{\partial x^4}
\end{align}
$$
A nice article [The wave equation for stiff strings and piano tuning](https://upcommons.upc.edu/bitstream/handle/2117/101752/GraciaSanz.piano.RSCM.2017.pdf) provides the solution to the equation of such stiff strings.
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

# Turn the Model into Code!

Here comes the most fun part when building a simulation: to turn the math into code and verify that the math (I've worked hard on) actually works.

I implemented the synthesizer with Juce, a framework to build VSTs. The actual code for Juce is complicated, so I will show simplified code in the following content while keeping the essence. The complete source code is here: https://github.com/eri24816/PhysicsBasedSynth.

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
## Simulation Class
Let's see what the line `simulation->update();` does. In the simulation, there are two objects, the string and the hammer, and one interaction, the hammer-string-interaction. For each time step, interaction->apply is called, making the hammer and the string add forces to each other. Then object->update on both objects are called to update their state.
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

## String Class

We finally reach the String class. The string class is the core for the entire simulation and contains heaviest calculation. Because a real-time synth is super performance sensitive, I put much effort and exhausted all possible ways to optimize the code in this class.

First of all, when the string is instantiated, the constructor initializes the amplitudes $a_n$ and $b_n$ and precomputes some values used heavily during the simulation.

```c++
String::String(float L, float tension, float rho, float ESK2, int nHarmonics, float damping)
	: L(L), tension(tension), rho(rho), ESK2(ESK2), nHarmonics(nHarmonics), transform(nullptr, Vector2<float>{0.0f, 0.0f}), damping(damping)
{
	// precompute constants for speed
	B = PI * PI * ESK2 / tension / L / L;
	c = std::sqrtf(tension / rho);
	f0 = c / (2 * L);
	
	for (int n = 1; n <= nHarmonics; n++)
	{
		// precompute more constants
		harmonicFreqs[n] = getHarmonicFreq(n);
		harmonicOmega[n] = TWO_PI * harmonicFreqs[n];

		// initialize the amplitudes
		a[n] = 0;
		b[n] = 0;
	}
}
```

The `sampleU` method provides a way to get the current value of $u(x)$. It is useful for sampling sound and calculating the interaction with the hammer.

```c++
float String::sampleU(float x) const
{
	float u = 0;
	float pi_x_div_L = PI * x / L;
	for (int n = 1; n <= nHarmonics; n++)
	{
		const float xComp = std::sin(n * pi_x_div_L);
		const float omegaT = harmonicOmega[n] * t;
		u += (a[n] * std::cos(omegaT) + b[n] * std::sin(omegaT)) * xComp;
	}
	return u;
}
```

The `update` method does damping. Very simple.

```c++
void String::update(float t, float dt)
{
	// A String have to store the time to have a complete state
	this->t = t;
	// TODO: use a physical damping model
	if (damping != 0) {
		for (int n = 1; n <= nHarmonics; n++)
		{
			float factor = std::exp(-damping * 0.002 * dt * harmonicFreqs[n]);
			a[n] *= factor;
			b[n] *= factor;
		}
	}
}
```

The `applyImpulse` method is called by the hammer-string-interaction. It update the values of $a_n$ and $b_n$ due to the external impulse.

```c++
void String::applyImpulse(float x, float J)
{
	for (int n = 1; n <= nHarmonics; n++)
	{
		const float xComp = 2 * J / L / rho / harmonicOmega[n] * std::sin(n * PI * x / L);
		const float omegaT = harmonicOmega[n] * t;

		a[n] += xComp * std::cos(omegaT);
		b[n] += xComp * std::sin(omegaT);
	}
}
```

# Optimization: Vectorized Operations
...