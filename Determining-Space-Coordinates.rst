Deferred Shading: Determining Space Coordinates from Device Coordinates
#######################################################################
:date: 2016-02-04
:tags: graphics, openGL, thesis
:category: coding

Hello hello, today I have for you a short article about how to build the
coordinates of any space, given the device or clipping space coordinates. This
is useful in shaders, when those are not directly available anymore yet you
need the position in a certain space for a set of calculations. I will assume
a basic knowledge of graphics, as I assume you've stumbled on this article,
because you need to solve exactly this problem.
For the sake of completeness (and because I like to add little flow chart like
images) I will start of by discussing quickly the different spaces, and the
terminology I use.
Then I will go over the derivation of the calculations, which will be
implemented in the final section. If you are only interested in the code, and
not the background, I recommend you skip to the last section immediately!

Different spaces
================
In most graphic applications a straightforward representation of positions in
space can be represented as a vector containing for elements x, y, z, w.

From device to any other space
==============================
We have looked at the different spaces, and how to go from the respective spaces
to device coordinates or clipspace coordinates.
Now we will look at the opposite direction.
We established in the previous section the following formulas:

.. math::
device = clip.xyz / clip.w
world.w = 1.0
clip = M * coords

We define M as the product of the different matrices discussed in the previous
section in order to transform or specific coords to clip space coordinates.
thus

.. math::
M = cameraToClip * worldToCamera * modelToWorld

we can state the problem as the following.
We want to calculate the set of coordinates where the device coordinates are
given and the set of matrices, where the product is specified as M is constant.

Formula derivation
------------------
.. math::
clip = M * coords
M^{-1} clip = M^{-1} * M * coords
M^{-1} clip = I * coords
M^{-1} clip = coords

.. math::
device = clip.xyz / clip.w
clip.w * device = clip.xyz

we define a four element vector as a three element vector plus a fourth element
.. math::
clip = (clip.xyz, clip.w)
clip = (clip.w * device, clip.w)

now we can insert this in the previous equation)
.. math::
M^{-1} clip = coords
M^{-1} (clip.w * device, clip.w) = coords
M^{-1} * clip.w (device, 1) = coords
clip.w * M^{-1} * (device, 1) = coords

finally we define $clip.w$
.. math::
world.w = 1
clip.w * (M^{-1} (device, 1)).w = 1
clip.w = \frac{1}{(M^{-1} (device, 1))}

now we can define $coords$ as
.. math::
coords = \frac{M^{-1} (device, 1)}{(M^{-1} (device, 1)).w}

or

.. math::
coords = (M^{-1}(device, 1)).xyz/w

Which leaves us with the end result, an elegant formula to retrieve our
coordinates in the space we need.
In the following section we will look at how to implement this in code.


Putting formula in code
=======================
To summarise our previous section (for everyone that skipped ahead to this part)
. We need to calculate the following equation

.. math::
coords = \frac{M^{-1} (device, 1)}{(M^{-1} (device, 1)).w}

where $coords$ are the coordinates of a certain space, and $M$ the matrix to
transform $coords$ into clipspace coordinates.

thus

* Camera space : $$ M = cameraToClip $$
* World space : $$ M = cameraToClip * worldToCamera $$
* Model space : $$ M = cameraToClip * worldToCamera * modelToWorld $$

in this particular case I will focus on world space, however it is trivial to
substitute this with your own case.

Our solution will consists of two parts. We will define our matrices in
C++ on the cpu, as these do not change per frame, and thus can be loaded into
memory after calculated once.
Then we will implement the formula in GLSL, to be used in the fragment shader.

Loading the matrices
--------------------

Implementing the formula
------------------------
