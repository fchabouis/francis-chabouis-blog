---
title: "Draw an image with a sound: 50 lines of Python"
date: 2024-10-11T11:00:00+02:00
draft: false
tags: ["python"]
---

Recently a musician friend of mine was showing me [Spectroid](https://play.google.com/store/apps/details?id=org.intoorbit.spectrum), a nice and simple application that shows you in real time the frequencies associated with a sound.

We were playing a little bit with it, but I am not a music guy at all, I am more a visual person than an acoustic one. So pretty fast, I was trying to draw things on the application by withling. Some letters, like I or L are pretty easy to draw. And then we tried to draw an "O", but obviously you shoud be two to do that, as I can't produce two different frequencies at the same time. So we were withling at the same time, me trying to draw half of the "O", a "C" and my friend doing the opposite.

We couldn't do it. But I kept wondering, what would a sound that draws an "O" sound like? Would it be possible to draw anything with the proper sound?

I started playing a little bit with Python and was surprised to see how easy it was to find out!

## Principle overview

My idea was to take a simple, black and white image. The black could be the frequencies contained in the sound at each time, whereas the white would be the frequencies absent from the sound. At each moment in time, the frequencies present would change therfore creating a drawing when the application listens to the sound.

First step is to create a simple image. [Pixilart](https://www.pixilart.com/draw) is a perfect website for that.

![draw a circle using Pixilart](/sound-Pixilart.png)

Then I need to open the file with Python and read the pixel data. I used [Pillow](https://python-pillow.org/), a python package for image manipulation.

Next step is to play a sound with Python. One way to do it is using the Ipython `diplay` [module](https://ipython.org/ipython-doc/3/api/generated/IPython.display.html). It accepts as an argument a numpy array containing the sound wave and outputs a sound.

For example, playing a 400Hz sound during 1 second is done like this:

```python
import numpy as np
import IPython.display as ipd

sample_rate = 44100  # samples per second
duration = 1

t = np.linspace(0, duration, int(sample_rate * duration))

# Generate a sine wave
output = np.sin(2 * np.pi * frequency * t)

ipd.Audio(output, rate=sample_rate)
```

Now I would like to play multiple frequencies at the same time. So I need to add different sin waves with different frequencies to the numpy array. That can be done with a simple loop:

```python
import numpy as np
import IPython.display as ipd

sample_rate = 44100  # samples per second

def makesine(frequencies, duration):
    t = numpy.linspace(0, dur, math.ceil(sample_rate * duration))
    x = 0
    for f in frequencies:
        x = x + numpy.sin(2 * numpy.pi * f * t)
    return x

# create a 1 sec sound containing 400HZ and 800HZ frequencies 
output = makesine([400, 800], 1)
ipd.Audio(output, rate=sample_rate)
```

## Putting it all together
I know how to create the pixel image, how to read the pixel values and how to produce sound with the desired frequencies, let's plug everything together!

```python
import IPython.display as ipd
import matplotlib.pyplot as plt
import math
import numpy as np
from PIL import Image

sample_rate = 22050

# the function used to produce the numpy array containing the sin waves 
# at the proper frequencies.
def makesine(frequencies, duration):
    t = np.linspace(0, duration, math.ceil(sample_rate * duration))
    x = 0
    for f in frequencies:
        x = x + np.sin(2 * np.pi * f * t)
    return x

# Open image with Pillow
# https://www.pixilart.com/draw
image = Image.open('/Users/francis/Downloads/image.png')

# Convert Pillow image to NumPy array
img_array = np.array(image, dtype=np.uint8)

# initialise the output that will contain the sound
output = np.array(())

# create the list of frequencies that can be activated or not
start_frequency = 500
frequency_step = 10
# the number of available frequency corresponds to the width of the image
n_step = img_array.shape[1]

frequencies = []
frequency = start_frequency

for i in range(n_step):
    frequencies.append(frequency)
    frequency = frequency + frequency_step

# loop on the image pixels
for row in reversed(img_array):
    pixels = []
    for col in row:
        # detect if that pixel contains something.
        # the fourth numbers corresponds to the opacity: something is drawn when the opacity is >0.
        if col[3] > 0:
            pixels.append(1)
        else:
            pixels.append(0)
    
    row_freq = []
    for i, freq in enumerate(frequencies):
        if pixels[i] == 1:
            # we want to draw something at that frequency
            row_freq.append(freq)
    
    if len(row_freq) > 0:
        # we create a 0.15sec sound containing the selected frequencies
        y = makesine(row_freq, 0.15)
    else:
        # empty line: no sound
        y = makesine([1], 0.15)
    output = np.concatenate((output, y))
    
ipd.Audio(output, rate=sample_rate)
```

<figure>
  <figcaption>Listen to the resulting sound:</figcaption>
  <audio controls src="/static/circle.wav"></audio>
  <a href="/static/circle.wav"> Download audio </a>
</figure>