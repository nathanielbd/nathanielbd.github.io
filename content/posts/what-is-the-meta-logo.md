---
title: "What is the Meta logo?"
date: 2021-10-28
images:
tags: 
  - parametric equations
---

Today, the Facebook company rebranded itself as Meta. PR aside, what mathematically describes its new Lemniscate-like loop logo (try saying that quickly three times)?

Answer: A [Lissajous curve](https://en.wikipedia.org/wiki/Lissajous_curve) (also called a Bowditch curve), part of a family of parametric equations where the $x$ and $y$ coordinates are sinusoids.

$$
x = A\sin(at + \delta), \quad y = B\sin(bt), \quad 0 \leq t \leq 2\pi.
$$

Here are the approximate parameters for the logo:

$$ 
(A=3, a=1, \delta=5\pi/9, B=2, b=2).
$$ 

Try tweaking the parameters and see how it changes!

<iframe src="https://www.desmos.com/calculator/brasrdtssa" width="100%" style="min-height:300px"></iframe>

After tweaking around or refreshing your high school algebra knowledge, you can find that $A$ and $B$ control the width and height of the curve. The ratio of $\frac{a}{b}$ determines the number of lobes in the figure. Lastly, $\delta$ is called the 'phase difference.' It is the difference in starting point between the $x$ and $y$ sinusoids.

| ![](/BowditchParametric.mp4) |
|:--:|
| From high school trigonometry, you might remember that the value of a sine function corresponds to the height of a triangle inscribed in a unit circle ("SOH" in "SOH-CAH-TOA"). We can use that trick to plot the Bowditch curve. $A$ and $B$ are now radii and $\delta$ is now an angle offset for their starting positions. $a$ and $b$ control the speeds of rotation of the circles. |

If we consider the Lissajous curve to be an image of a three-dimensional wireframe, then $\delta$ controls the angle at which we view it.

| ![](/Bowditch3D.mp4) |
|:--:|
| Interpretation of $\delta$ being the angle at which we view a 3D wireframe. Ignoring that closer parts of the wireframe appear larger, this is precisely our original curve in 2D. For an interactive view of the wireframe, see [here](https://christopherchudzicki.github.io/MathBox-Demos/parametric_curves_3D.html?settings=eyJmdW5jdGlvbnMiOnsiYSI6eyJ6IjoiMipzaW4oMip0KSIsInQiOjUuMDQ5OTk5OTk5OTk5OTM3fX0sImNvbnRhaW5lcklkIjoibXktbWF0aC1ib3giLCJjYW1lcmEiOnsicG9zaXRpb24iOlswLjAzLDAuMjIsMS41N119LCJub1pvb20iOnRydWUsImZvY3VzIjoxLjY5NTU4MjQ5NTc4MTMxN30=). |

The wireframe has the parameters

$$
x = A\cos(t), \quad y = A\sin(t), \quad z=B\sin\left(\frac{b}{a}t\right), \quad 0 \leq t \leq 2\pi.
$$

A good mathematical exercise would be to show that the projection of this onto a plane through the $z$-axis indeed results in the original Bowditch curve! Given that the company has the augmented reality 'metaverse' in mind, they certainly chose Lissajous curves as a figure that lives simultaneously in [two and three dimensions](https://design.facebook.com/stories/designing-our-new-company-brand-meta/).

> [!info]
> If you're interested in how I made the animations for this post, [here](https://gist.github.com/nathanielbd/25fb7b5a51535552bfa5fee62e49ebff) is the source code for generating them. I used [Manim](https://manim.community)!