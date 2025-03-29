---
title: "How I used Manim with WSL and data points"
date: 2020-06-26
tags: 
  - Manim
---

> [!note]
> I've since switched to using MacOS and the [open source community's fork of Manim](https://www.manim.community/)!

---

I recently (last night) submitted [my video entry](https://www.youtube.com/watch?v=ixGwH2oYzzA) for [the Breakthrough Junior Challenge 2020](https://breakthroughjuniorchallenge.org/), a competition for students aged 13-18 to submit a video explaining a topic in physics, life sciences, or math in a 3 minute video. In a competition filled with entries with amazing Adobe After Effects and Element 3D editors, I sought to be unique among the 10,000 competitors by using Grant Sanderson/3Blue1Brown's math animation engine, [Manim](https://github.com/3b1b/manim). The source code for my video can be found [here](https://github.com/nathanielbd/bjc2020).

The engine is incredible, but lacks documentation and tutorials. There are a couple good ones that show up when you search "manim tutorial", like [this](https://github.com/Elteoremadebeethoven/AnimationsWithManim), [this](https://github.com/malhotra5/Manim-Tutorial), and [this](https://github.com/saturnman/manim_turorial). However, nothing relevant shows up when you search "manim WSL" or "manim data points," so I think that alone justifies making this post.

# My ramblings on WSL

Feel free to skip this if you're familiar with WSL, but it's greatly improved my software development experience, so I thought I'd share my thoughts and setup if it'd be able to help some people, even if I'm not sponsored to do that.

WSL stands for "Windows Subsystem for Linux." It allows developers to run a Linux environment on their Windows computer. You can run your favorite distro. It comes with Bash. Perhaps its best perk is giving Windows users access to all the installers and package managers that are so convenient on Linux. 

WSL requires an X server to run GUI applications. The one I use and love is [MobaXterm](https://mobaxterm.mobatek.net/). It runs smoothly, has tabs for multiple terminals, pretty colors, and even works over SSH. Personally, I'm really happy with my VSCodium/MobaXterm/Vim setup. After switching from Cygwin, PuTTY, and [my university's online Linux desktop](https://vole.cse.umn.edu/), I've never looked back.

| ![MobaXTerm](/mobaxterm.png) |
|:--:|
| MobaXTerm with X server displaying a pdf using [Zathura](https://pwmt.org/projects/zathura/) |

# How to install Manim on WSL

Manim uses [MiKTeX](https://miktex.org) to handle $\LaTeX$. I followed the intructions [here](https://miktex.org/howto/install-miktex-unx) to install.

Next, install dependencies.

```bash
# install dependencies in Ubuntu
sudo apt install pkg-config libcairo2-dev ffmpeg sox texlive

# install required latex packages
sudo apt install texlive-latex-extra    # "standalone"
sudo apt install texlive-fonts-extra    # "dsfont"
sudo apt install texlive-science        # "physics"
```

Clone Manim from GitHub, create an Anaconda environment, and install the requirements.

```bash
# install manim
git clone https://github.com/3b1b/manim.git
cd manim
# create Anaconda env
conda env create -f environment.yml
conda activate manim
python3 -m pip install -r requirements.txt
```

Test it out. This must be done in the `manim` directory.

```bash
python3 -m manim example_scenes.py SquareToCircle -pl
```

*NOTE:* the `-pl` tag means the preview the video by opening it and render it in low quality.

# How to graph data points

Use a `GraphScene` and set its configuration. These are the defaults.

```python
from manimlib.imports import *

class YourScene(GraphScene):
	CONFIG = {
	    "x_min": -1,
	    "x_max": 10,
	    "x_axis_width": 9,
	    "x_tick_frequency": 1,
	    "x_leftmost_tick": None, # Change if different from x_min
	    "x_labeled_nums": None,
	    "x_axis_label": "$x$",
	    "y_min": -1,
	    "y_max": 10,
	    "y_axis_height": 6,
	    "y_tick_frequency": 1,
	    "y_bottom_tick": None, # Change if different from y_min
	    "y_labeled_nums": None,
	    "y_axis_label": "$y$",
	    "axes_color": GREY,
	    "graph_origin": 2.5 * DOWN + 4 * LEFT,
	    "exclude_zero_label": True,
	    "num_graph_anchor_points": 25,
	    "default_graph_colors": [BLUE, GREEN, YELLOW],
	    "default_derivative_color": GREEN,
	    "default_input_color": YELLOW,
	    "default_riemann_start_color": BLUE,
	    "default_riemann_end_color": GREEN,
	    "area_opacity": 0.8,
	    "num_rects": 50
	}
```

Draw the axis.

```python
def construct(self):
	self.setup_axes(animate=True)
```

This example uses JSON like `[{"x":0, "y":1}, ..., {"x":9, "y":10}]`, but you can extract your data using any method as long as it is supported in python.

```python
import json
with open('file.json') as json_file
	coords = json.load(json_file)
data_points = VGroup(*[Dot(point=self.coords_to_point(coord['x'],coord['y'), radius=0.3) for coord in coords])
self.play(Write(dots), run_time=1)
```

| ![A screenshot from my video](/graph.png) |
|:--:|
| Nathaniel Budijono / CC-BY |

## Tangent lines to data points

Manim has [really neat functions](https://github.com/3b1b/manim/blob/a529a59abf1f6af02e664dbad1c8474f3a25a61e/manimlib/scene/graph_scene.py#L715) for [plotting tangent lines](https://github.com/3b1b/manim/blob/a529a59abf1f6af02e664dbad1c8474f3a25a61e/manimlib/scene/graph_scene.py#L988) to [visualize derivatives](https://github.com/3b1b/manim/blob/839fb4ff582103bd717b9d7937c926ef0390fb01/from_3b1b/old/eoc/chapter9.py#L1080), but they only work for graphs created from `GraphScene.get_graph`. In other words, there is no official support for tangent line plotting for our data points.

My hack around this was to enumerate the graphs of the instantaneous slopes using the slope intercept form of the [secant line for numerical differentiation](https://en.wikipedia.org/wiki/Numerical_differentiation#Finite_differences).

```python
derivatives = [self.get_graph(
	# slope intercept form is ugly as a one-liner lambda func
	lambda x: ((coords[i]['y']-coords[i-1]['y'])/(coords[i]['x']-coords[i-1]['x']))*x+(coords[i-1]['y']-coords[i-1]['x']*((coords[i]['y']-coords[i-1]['y'])/(coords[i]['x']-coords[i-1]['x']))),
	PURPLE
# start from index 1 so since computing slope requires the previous point
) for i in range(1, len(data_points)]
for derivative, next_derivative in zip(derivatives, derivatives[1::]):
	# I had hundreds of data points
	self.play(ReplacementTransform(derivative, next_derivative), run_time=0.0002)
```

| ![A gif from my video](/tangent.gif) |
|:--:|
| Nathaniel Budijono / CC-BY |

## Area under the data point curve

Similarly to the tangent lines, area under the curve [is possible](https://github.com/3b1b/manim/blob/a529a59abf1f6af02e664dbad1c8474f3a25a61e/manimlib/scene/graph_scene.py#L413), but [only for native graphs](https://github.com/3b1b/manim/blob/a529a59abf1f6af02e664dbad1c8474f3a25a61e/from_3b1b/old/eoc/chapter1.py#L1990).

My hack around this was to just draw a bunch of lines from the x-axis to the points.

```python
auc = VGroup(*[Line(self.coords_to_point(coord['x'], 0), self.coords_to_point(coord['x'], coord['y'])).set_stroke(width=8) for coord in coords])
self.play(Write(auc))
```

| ![Another gif from my video](/auc.gif) |
|:--:|
| Nathaniel Budijono / CC-BY |

# Some Documentation I wish I had

## How Points work

Manim has a class called `Point` which shows up as arguments for the constructors of geometric figures like `Dot`, `Line`, and `Arrow`. They represent points on the screen. The center of the screen has the alias `ORIGIN` and you can add unit vectors `LEFT`, `RIGHT`, `UP`, and `DOWN` to create any point on the screen. 

You can also get the center of an `Mobject` by calling `obj.get_center()`. 

Internally, these points are represented as an `[x, y, z]` coordinate array. So `ORIGIN` is just `[0.0, 0.0, 0.0]`.

## tikz Support

Manim renders `TextMobjects` in $\LaTeX$ by using `manim/manimlib/tex_template.tex`. Notably, the preamble of this document doesn't include `tikz`, so if you want to use `tikz`, you should add

```tex
\usepackage{tikz}
\usetikzlibrary{positioning}
```

to the preamble. 

To prevent automatic filling-in of rectangles and other figures in a `tikzpicture`, I found this configuration useful.

```python
class TikzMobject(TextMobject):
	CONFIG = {
		"stroke_width":	  3,
		"fill_opacity":	  0,
		"stroke_opacity": 1
	}
```

![A screenshot from my video](/tikz.png)

The above figure can be animated with the following code:

```python
fig = TikzMobject(
	r"""
            \begin{tikzpicture}[
                circlenode/.style={circle, draw},
                rectanglenode/.style={rectangle, draw, minimum width=2em},
                wheelnode/.style={circle, draw, minimum size=1.5em}
            ]
                \node[circlenode] at (-0.25,0) {};
                \draw (-0.25,0)--(0,-4);
                \node[rectanglenode] at (0,-4) {};
                \node[wheelnode] at (0.5,-4) {};
                \node[wheelnode] at (-0.5, -4) {};
            \end{tikzpicture}
	"""
)
self.play(Write(fig))
```

# Closing thoughts

I found looking through the source code and [how Grant himself used Manim](https://github.com/3b1b/manim/tree/a529a59abf1f6af02e664dbad1c8474f3a25a61e/from_3b1b) to be a decent alternative to documentation. 

I'm excited because there are [ongoing efforts to make Manim web-compatible](https://github.com/eulertour/eulerv2). The combination of this and better documentation should make for more amazing math videos and other educational visualizations.

My entry for BJC 2020 wasn't the best, but it gave me an excuse to learn Manim and video editing with [DaVinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve/)! The effort to production value ratio is really high for both of these tools.

Now I need an excuse to evaluate some presentation/slidedeck tools for this ratio. Would the winner be [Beamer](https://www.overleaf.com/learn/latex/Beamer), [reveal.js](https://revealjs.com/), or [impress.js](https://impress.js.org/)?