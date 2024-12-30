---
title: Modeling Stiff String for Numeric Simulation
date: 2024-11-25
authors:
  - eri24816
image: https://i.imgur.com/ydSU5KL.png
draft: false
tags:
  - math
  - audio
  - differentialEquation
categories: Uncategorized
series: 
summary:
---
# Analytical Solution to Stiff String

According to [The wave equation for stiff strings and piano tuning](https://upcommons.upc.edu/bitstream/handle/2117/101752/GraciaSanz.piano.RSCM.2017.pdf), the wave equation for a stiff string is:

$$
\begin{align}
\frac{\partial^2 u}{\partial t^2} &= c^2 \frac{\partial^2 u}{\partial x^2} - \frac{ESK^2}{\rho} \frac{\partial^4 u}{\partial x^4}
\end{align}
$$

where $c$ is the wave speed, $E$ is the Young's modulus, $S$ is the cross-sectional area, $K$ is the radius of gyration, and $\rho$ is the density.

The equation's general solution is:

$$
\begin{align}
u(x,t) &= \sum_{n=1}^{\infty} \left( a_n \cos(2\pi f_n t) + b_n \sin(2\pi f_n t) \right) \sin\left(\frac{n\pi}{L}x\right)
\end{align}
$$

where $f_n = n f_0 \sqrt{1 + Bn^2}$, $B = \frac{\pi^2 ESK^2}{\tau L^2}$, $f_0 = \frac{c}{2L}$.

# Simulation of Stiff String

To simulate a stiff string, I tried to directly solving the displacement, $u$, using Euler's method. It turns out to be quite unstable, especially when high frequency components present.

Alternatively, we can solve $a_n$ and $b_n$, which is very stable. With $a_n$ and $b_n$ available, we can get $u$ easily using (2). With the initial condition $u(x,t=0)$ and $u'(x,t=0)$, the corresponding $a_n$ and $b_n$ are:
  

$$
\begin{align}
a_n = \frac{2}{L} &\int_0^L u(x,0) \sin\left(\frac{n\pi}{L}x\right) dx \\\
b_n = \frac{1}{\omega _n}\frac{2}{L} &\int_0^L u'(x,0) \sin\left(\frac{n\pi}{L}x\right) dx
\end{align}
$$
where $u'=\frac{du}{dt}$
Note that $a_n$ and $b_n$ remain constant over time as long as no external forces are applied to the string (explanation: $a_n$ and $b_n$ are not functions of $t$ in $(2)$). We don't need to do any computation during the simulation to handle how the wave travels. We only need to take care of how external forces affect $a_n$ and $b_n$.
## Modeling External Force as a Dirac Delta Function Shaped Impulse

  
In a real piano, the string interacts with the hammer, the damper, and the bridge, etc.. Thus, during the simulation, we would want to apply external forces to the string.

Specifically, assume the hammer hits the string at a certain position $x_0$ and time $t_0$ with an impulse $J$. Here we model the impulse as a single-point impact.

$$
\begin{equation}
u'(x, t_0^+) = u'(x, t_0^-) +
\frac{J}{\rho} \delta(x - x_0)
\end{equation}
$$

By transforming (5) into the frequency domain using (3)(4), we have:

$$
\begin{align}
b_n(t_0^+) &= b_n(t_0^-) + \frac{2J}{L\rho\omega _n} \sin\left(\frac{n\pi}{L}x_0\right),& \text{when }t_0 = 0
\end{align}
$$

The above equation is a special case at $t_0 = 0$ because (3)(4) are only for $t = 0$. For a general $t_0$, we need to go a step further to consider the phase shift:

$$
\begin{align}
a_n(t_0^+) &= a_n(t_0^-) + \frac{2J}{L\rho\omega _n} \sin\left(\frac{n\pi}{L}x_0\right) \cos(2\pi f_n t_0) \\\
b_n(t_0^+) &= b_n(t_0^-) + \frac{2J}{L\rho\omega _n} \sin\left(\frac{n\pi}{L}x_0\right) \sin(2\pi f_n t_0)
\end{align}
$$

Although a delta function shaped impulse is feasible for a simulation in the frequency domain, it introduces intense high-frequency components. I'm not sure if it's realistic enough. Anyway, it's simple (no integration in it) thus efficient when used in a simulation.

## Modeling External Force as a Gaussian Shaped Impulse

$$
\begin{align}
a_n(t_0^+) &= a_n(t_0^-) + \frac{2J}{L\rho\omega _n} \sin\left(\frac{n\pi}{L}x_0\right) e^{-\frac{1}{2}\sigma^2} \cos(2\pi f_n t_0) \\\
b_n(t_0^+) &= b_n(t_0^-) + \frac{2J}{L\rho\omega _n} \sin\left(\frac{n\pi}{L}x_0\right) \sin(2\pi f_n t_0)
\end{align}
$$