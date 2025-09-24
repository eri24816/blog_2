---
title: Implementing reverb plugin with JUCE
date: 2022-03-16T17:33:37+08:00
draft: false
image: https://i.imgur.com/8qGh9bk.png
categories: music
summary: My first attempt at making an audio plugin and the math concepts I learned from it.
tags:
  - highlights
  - cpp
  - audio
  - VST
---
## Background

When something I’ve always wanted to do ends up being taught in a class, it feels really good.

In the course _Analysis of Digital Music Signal_ in NCKU, each group of students used **JUCE** (a C++ framework for making VSTs) to build a VST effect. Our group made a **reverb**. The purpose of reverb is to add echoes to music, creating the feeling of being in a cathedral or concert hall.

This was the most fun course I’ve taken this year. I learned many interesting concepts, like the **z-transform**, how to read **zero-pole plots**, and how to do **OOP in C++** (while stepping into all kinds of pointer traps XD).


## Overall Architecture

We built the reverb filter based on this diagram from https://ccrma.stanford.edu/~jos/pasp/Zita_Rev1.html:

![Image](https://i.imgur.com/gLgWwXH.jpg#center)

First, the 2 input channels are split into 8 channels by a **2×8 matrix**, then sent into a loop. Inside the loop, all 8 channels are processed in parallel. The signal passes sequentially through four basic filters: **all pass**, **feedback matrix**, **lowpass**, and **delay line**, and then repeats. After the feedback matrix, there’s an output path that exits the loop, going through an **8×2 matrix** before being sent back to 2 output channels.

Although the system looks straightforward (we “just” need to implement each basic module), the real difficulty lies in ensuring two aspects at the same time:

1. Producing good sound
2. Maintaining IIR stability

This is because we used an **IIR** structure, which contains a feedback loop and becomes unstable if the parameters are not tuned carefully. With an **FIR**, stability wouldn’t be an issue, but it requires more computational cost.

So, after coding it, we spent a long time experimenting with parameters to achieve the best sound possible while keeping the IIR stable.

## Implementing the basic modules

Our reverb filter implementation is built from several modules (delay line, low-pass filter, etc.), each of which is a multi-channel causal filter. A module can itself contain smaller sub-modules, with the reverb filter being the top-level module.

Each module is a class which implements

```c++
float* update(float* input)
```
In each time step of calculation, this method receives a float array as input, do calculations, and returns another float array as output. Most modules maintain internal state, making the `update` function not pure.

### Delay line
A delay line simply outputs the input after a delay of several samples. In the frequency domain, it acts as $z^{-n}$ . It can be easily implemented with `std::queue`.
```c++
float* update(float* input) override{
    for (int i = 0; i < inputDim; i++) {
        queues[i].push(input[i]);
        outputBuffer[i] = queues[i].front();
        queues[i].pop();
    }
    return outputBuffer;
}
```
Later, to reduce computation time, I wrote another version without using `std::queue`.
```c++
float* update(float* input) override {
    for (int i = 0; i < inputDim; i++) {
        outputBuffer[i] = arr[i][pos[i]];
        arr[i][pos[i]] = input[i];
        pos[i]++;
        if (pos[i] == len[i])pos[i] = 0;
    }
    return outputBuffer;
}
```
### Low pass filter
This module is a first-order low pass filter. It acts as$(1-a)+az^{-1}$.

We implemented it by adding the previous output to the current input and using that as the current output, which applies exponential smoothing on the waveform.
```c++
float* update(float* input) override {
    mult(inputDim, input, 1 - a);
    mult(inputDim, feedback, a);
    add(inputDim, input, feedback);
    copy(inputDim, feedback, input);
    return mult(inputDim, input,1);
}
```

Parameter `a` is used to control the cutoff frequency. Their relationship is:
$$a=e^{-2\pi \frac{ \mathit{Cutoff}} {\mathit{SampleRate}}}$$

### All-pass filter

A second-order all-pass filter. This was the hardest filter to implement in the project—we had to derive it from the properties the filter must satisfy.

The pole-zero plot of a second-order all-pass filter has two poles and two zeros, and its properties are constrained by the following:

1. According to the **complex conjugate root theorem**, the two poles (and the two zeros) must be complex conjugates.
    
2. To ensure the **amplitude response is 1 at all frequencies**, each pole must have a corresponding zero that is the inverse with respect to the unit circle, and vice versa.
    

Combining these constraints means that a single set of parameters $(r, \theta)$ is enough to determine the position of one pole, which in turn determines the positions of all poles and zeros—fully defining this second-order filter.

![Image](https://i.imgur.com/4HVI7Xu.png#centers)

Next, we use $(r, \theta)$ to derive the filter's difference equation.

Let the filter’s response be $H(z) = \frac{P(z)}{Q(z)}$, where the two roots of $P$ are the zeros and the two roots of $Q$ are the poles. $P$ and $Q$ can be written as:

$$\begin{aligned}
P(z) &= r^2(z-r^{-1}e^{iθ}) (z-r^{-1}e^{-iθ})  \\\\
&=r^2z^2-r(e^{iθ}+e^{-iθ})z+1\\\\
&=r^2z^2-2r\cos(θ)z+1
\end{aligned}$$

$$\begin{aligned}
Q(z)&=(z-re^{iθ})(z-re^{-iθ})\\\\
&=z^2-r(e^{iθ}+e^{-iθ})z+r^2\\\\
&=z^2-2r\cos(θ)z+r^2
\end{aligned}$$

Let $X(z)$ the input，$Y(z)$ the output:
$$\begin{aligned}
Y(z)&=H(z)X(z)\\\\
&=\frac{P(z)}{Q(z)}X(z)\\\\
\end{aligned}$$

$$\begin{aligned}
Q(z)Y(z)&=P(z)X(z)\\\\
(z^2-2r\cos(θ)z+r^2)Y(z)&=(r^2z^2-2r\cos(θ)z+1)X(z)\\\\
(1-2r\cos(θ)z^{-1}+r^2z^{-2})Y(z)&=(r^2-2r\cos(θ)z^{-1}+z^{-2})X(z)\\\\
\end{aligned}$$

Rearrange and get the following difference equation:
$$y[n] = r^2 * x[n]-2r\cos(θ) * x[n-1]+x[n-2]+2r \cos(θ) * y[n-1]-r^2* y[n-2]$$

Here, $x[n]$ is the current input sample, and $y[n]$ is the current output sample. Because calculating $y[n]$ requires four previous values, this filter needs four memory slots. The implementation is as follows:

```c++
float* update(float* input)override {
    for (int i =0;i<inputDim;i++) {
        output[i] = R2[i] * input[i] - twoRCosTheta[i] * x1[i] + x2[i] + twoRCosTheta[i] * y1[i] - R2[i]*y2[i];

        x2[i] = x1[i];
        x1[i] = input[i];

        y2[i] = y1[i];
        y1[i] = output[i];
    }
    return output;
}
```

### Comb filter
The comb filter is simple because it is an FIR, no feedback.
```c++
float* update(float* input)override {
    add(inputDim, input, delay.update(input));
    return input;
}
```

![Image](https://i.imgur.com/u0USmfH.png#centers)

(Actually we didn't use it in the reverb eventually. We used the all pass filter instead.)

### Reverb

Reverb, the ultimate module, is made by combining of all basic modules.

```c++
float* update(float* input) override{
    
    float dry[2];
    copy(inputDim, dry, input);

    input = distrib * inDelay.update(input);

    delayFilters.update(feedBack);

    add(NCH,input, fbDelayLine.update(feedBack));

    input = allpass.update(input);

    input = mult(inputDim, input, _decay);

    float* output = feedbackmatrix * input;

    copy(NCH,feedBack, output);

    return dcBlocker.update(add(inputDim,mult(inputDim,outDistrib*output,wetAmount), mult(inputDim, dry,dryAmount)));
}
```


## Tweaking parameters

After assembling all the filters, the first problem we encountered was that the values could easily blow up after running for a while. This happens because, in the main loop, if the amplitude response at any frequency exceeds 1, that frequency’s intensity will grow exponentially over time.

To address that, we then reduced the values in the loop’s feedback matrix to lower the amplitude response. This introduced another problem: the reverb time became too short. When the loop’s amplitude response is too far below 1, the signal decays quickly and disappears after a few iterations.

That tells us the amplitude response at each frequency must be less than 1, but only slightly.

The approach we eventually adopted was:

1. Make the low-pass filter’s amplitude response ≤ 1
2. The sum of absolute values in each row of the feedback matrix is slightly less than 1

This ensures the system won’t blow up. Here’s why:

In the strictest case, assume both the low-pass and all-pass filters have an amplitude response of 1. Then, as the signal travels from the feedback matrix’s output back to its input, the amplitude gain is 1, so the signal strength remains unchanged. The feedback matrix mixes the 8 channels together, and the constraint that the sum of absolute values in each row is slightly less than 1 ensures that the mixed signal does not increase in amplitude due to addition.

However, because the signal accumulates complex phase changes as it goes through the loop, even if the row sums are very close to 1, destructive interference can still cause significant decay. Moreover, the VST sample rate is very high (e.g., 44,100 Hz), so the sound can almost disappear in under a second.

Finally, we find a solution, to add 50 ms of convolution before the IIR. This brute-force approach worked well in practice.