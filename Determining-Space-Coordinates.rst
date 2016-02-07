Deferred Shading: Determining Space Coordinates from Device Coordinates
#######################################################################
:date: 2016-02-04
:tags: graphics, openGL, thesis
:category: coding

    Hello hello, today I have for you a short article about how to build the
    coordinates of any space, given the device or clipping space coordinates.
    This is useful in shaders, when those are not directly available anymore
    yet you need the position in a certain space for a set of calculations.
    I will assume a basic knowledge of graphics, as I assume you've stumbled
    on this article, because you need to solve exactly this problem.
           
    For the sake of completeness (and because I like to add little flow chart
    like images) I will start of by discussing quickly the different spaces, and
    the terminology I use.
    Then I will go over the derivation of the calculations, which will be
    implemented in the final section. If you are only interested in the code,
    and not the background, I recommend you skip to the last section
    immediately!

Different spaces
================
    In most graphic applications a straightforward representation of positions
    in space can be represented as a vector containing for elements x, y, z, w.

From device to any other space
==============================
    We have looked at the different spaces, and how to go from the respective
    spaces to device coordinates or clipspace coordinates.
    Now we will look at the opposite direction.

    We established in the previous section the following formulas:
    .. math::
        device = clip.xyz / clip.w
        world.w = 1.0
        clip = M * coords

    We define M as the product of the different matrices discussed in the
    previous section in order to transform or specific coords to clip space
    coordinates. thus
    .. math::
        M = cameraToClip * worldToCamera * modelToWorld

    we can state the problem as the following.
    We want to calculate the set of coordinates where the device coordinates are
    given and the set of matrices, where the product is specified as M is
    constant.

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

    we define a four element vector as a three element vector plus a fourth
    element
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
    To summarise our previous section (for everyone that skipped ahead to this
    part). We need to calculate the following equation

    .. math::
        coords = \frac{M^{-1} (device, 1)}{(M^{-1} (device, 1)).w}

    where $coords$ are the coordinates of a certain space, and $M$ the matrix to
    transform $coords$ into clipspace coordinates.

    thus
    .. math::
        Camera space : M = cameraToClip 
        World space :  M = cameraToClip * worldToCamera 
        Model space :  M = cameraToClip * worldToCamera * modelToWorld 

    in this particular case I will focus on camera space, however it is trivial
    to substitute this with your own case. Furthermore, this particular
    implementation deals with deferred shading, which adds some intricacies
    regarding the device coordinates. These can be skipped over if this does not
    apply to your use case.

    Our solution will consists of two parts. We will define our matrices in
    C++ on the cpu, as these do not change per frame, and thus can be loaded
    into memory after calculated once.
    Then we will implement the formula in GLSL, to be used in the fragment shader.

Obtaining the input values
--------------------------
    Looking back out our formula, we need the following information
    * The device coordinates
    * The inverse matrix

    The inverse matrix can be easily obtained using glm or a similar framework
    to handle all matrix calculations. We have the matrices in memory from
    previous steps. Thus the only thing that needs to be calculated, is the
    inverse. This can be done by using glm::inverse or a similar framework, to
    save us from the hassle of implementing our own matrix operations.

    Thus this boils down to
    .. code::
        invCamera = glm::inverse(cameraToClip);

    Obtaining device coordinates can be done with gl_FragCoord, this will give
    back the window-relative coordinates of the current fragment.
    By default this uses a lower-left origin for window coordinates, and assumes
    pixel centers are located at half-pixel centers. Thus the lower left pixel
    is at (x, y) location (0.5, 0.5).

    Our first step is to transform these into normalized device coordinates.
    If we know the total resolution of our screen this is straightforward for the
    x and y component.
    We assume that our left most pixel will be at (-1, -1) while our upper right
    most pixel will lay at (1, 1). Thus this can be calculated by

    .. math::
        2.0 * ((gl_FragCoord.xy - (0.5, 0.5) - viewport.xy) / viewport.zw - 1.0

    where viewport is the vector (left, bottom, right, top) in pixel values.
    in case the depth is in the gl_FragCoord this can be normalized similarly.
    .. math::
      2.0 * gl_FragCoord.z -1.0

    in the deferred case we will not use the gl_FragCoord.z as the actual
    fragments are all on a plane at a set offset. However the original depth
    data was prserved by rendering a depthbuffer that we will access.
    This again will be normalized simirarly as the gl_FragCoord.z

    now we can specify our normalized device coordinates as (ndc.xy, ndc.z, 1.0)
    This leaves us with the details of the final implementation. Which will be
    just a few lines.

Implementing the formula
------------------------
    The c++ side as stated will be implemented with a single line of code.
    .. code::
       glm::mat4 invCamera = glm::inverse(cameraToClipMatrix);

    in case you would want world space coordinates this would be
    .. code::
        glm::mat4 invWorld = glm::inverse(cameraToClipMatrix worldToCameraMatrix);
    etc.

    then it is just a matter of loading it into gpu memory for use in the
    shader.
    The complete code would thus be.
    .. code::
       glm::mat4 invCamera = glm::inverse(cameraToClipMatrix)
       GLint p_invCamera = glGetUniformLocation(shader_program,
                                                shader_program_inv_matrix_name);
       glUniformMatrix4fv(p_invCamera, 1, GL_FALSE, glm::value_ptr(invCamera));

    where glGetUniformLocation gets the pointer to the memory location, and
    glUniformMatrix4fv loads the matrix into memory with openGL.

    in the shader we will define a simple function to calculate the coordinates,
    which is defined as follows:

    .. code::
        uniform mat4 invSpaceMatrix;
        uniform vec4 viewport;

        vec4 getCameraCoordinates() {
            // Define the normalized device coordinates
            // xy
            vec3 device;
            device.xy = (2.0 * (gl_FragCoord.xy - vec2(0.5f, 0.5f) - viewport.xy) /
                         viewport.zw)) - 1;
            // z
            device.z = 2.0 * getDepth() - 1.0;

            // calculate actual coordinates
            vec4 raw_coords = invSpaceMatrix * vec4(device, 1.0f);
            vec4 coords = raw_coords / raw_coords.w;
            return coords;
        }

    getDepth can be replaced with whatever method you have to get the depth of
    the fragment or vertex.
    This allows us to use getCameraCoordinates to obtain our actual location of
    the fragment within the 3d space.

    And that concludes this article
