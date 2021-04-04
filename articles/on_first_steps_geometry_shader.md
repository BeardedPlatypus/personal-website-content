title: Learning Unity: Rendering a simple 2D with the geometry shader.
date: 2021-04-03

Within this post, we will take a look at geometry shaders within Unity, in
particular how to utilise it to render simple 2D grids in a 3D scene.

Over the past few weeks I have slowly been getting my feet wet again with 
Unity. I submitted an entry into the [Lego Ideas x Unity](https://ideas.lego.com/challenges/6811cf30-f944-4dfa-8714-9b38be6fbb52?query=&sort=top) 
competition, which forced me to learn basics. And over the past few weeks I
have been looking at generating 3D worlds from geographical data. 
One of the challenges I set for myself is to render a simple 2D grid in a 3D 
scene. I figured this would be a simple introduction into writing my own 
shaders and explore geometry shaders for a first time. This blog post is
short write-up of the results, and how they were achieved.

As I am a firm believer in showing the results first and then expanding upon
how it was achieved, this is the final result:

<p align='center'><img align='center' src='https://github.com/BeardedPlatypus/media-storage/blob/main/blog/geometry-shader/final.png?raw=true' width='100%'></p>

As you can see we have a simple 2D grid consisting of lines and points. The
size of these lines and points can be adjusted through the shader. The 
geometry is generated on the fly with a simple C# script. Building this
scene consists of three steps:

* Creating a new scene with a simple rotating camera.
* Rendering the lines of the grid from line geometry.
* Rendering the points of the grid from point geometry.

The following sections will drill down further on these topics. Do note that
I am a complete beginner when it comes to Unity, and as such, anything written
here might not be the ideal solution to the problem, always be critical!

*Most of the information of this post has been obtained from [this tutorial](https://jayjingyuliu.wordpress.com/2018/01/24/unity3d-intro-to-geometry-shader/), do check it out!*

## Set up an empty scene with a simple rotating camera.

In order to build anything within Unity we will first need to create a new
project with a simple scene. In my case, I created a simple Universal Render
Pipeline (URP) project and added a new empty scene to this project.

Because the main focus of this write-up is how to utilise the geometry shader,
I opted to create simple unlit shaders as a basis. This means we can remove
the directional light within the scene, as it is not utilised in the materials.

Next I added a simple script to my camera to take care of the rotation 
behaviour. It might well be possible to achieve the same effect with built in
tooling, however because I am a software engineer by trade, this was the path
of least resistance to me.

The script has the following content:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/1d6e286997a09c9d59e367bd8791cb42.js"></script>
</div>

As you can see it is rather trivial. Within the update script we update the 
rotation of the transform of the object is attached to and rotate it around
the z-axis located at the world center. We can adjust the speed of the rotation
with the `timePerRotation` field.

We can slap this on our, and it wil automatically rotate around the world
center (which can be observed by looking at the transformation component)
in the inspector while the game is running.

Lastly, we make some minor changes to the camera, to ensure you will have 
a scene identical to mine.

* The position of the camera is set to: (0, 10, -18)
* The rotation of the camera is set to: (30, 0, 0)
* The Background Type i set to 'Solid Color'
* The background is set to the following hexadecimal value: 292D33

This should now give you an empty scene with a dark blue-ish background.
In the following sections we will get to the juicy bits, and actually 
start implementing our simple grid.

## Rendering thick lines with the help of a geometry shader

The first step in generating our simple 2D Grid is to visualise the lines of 
the grid. This step will consist out of two steps, generating a simple plane
consisting of vertices connected with lines and then writing our shader to
give the lines of the plane some width. The shader should work for any mesh
consisting of lines, however for the sake of simplicity we will just generate
a simple grid to visualise.

### Generating a simple plane as geometry

The easiest way to represent a grid is as a collection of lines connecting 
vertices. This is exactly how will represent our geometry. First we create a 
new script that will contain our mesh generation code. Once this is created
let's move to your favourite IDE and get coding.

First we provide some fields which can be customised in the editor:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/4da98624980b704239db00a7e19e070a.js"></script>
</div>

* Shader: The shader to render the geometry with
* Total Width: the total width of our plane
* Total height: the total height of our plane
* Subdivision X: The number of subdivisions in the local x-axis
* Subdivision Y: The number of subdivisions in the local y-axis


Next we will create the necessary components on our game object when starting
the player:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/408e3dda00b7a51d5f3eb7bd1102686e.js"></script>
</div>

* A mesh renderer linking to our selected shader
* A Mesh filter containing our mesh

All of this will be generated upon starting, thus adjusting the values while in
play mode will not influence the created geometry at all. This is an acceptable
limitation in my opinion, for the sake of simplicity.

Next we have a method that generates the actual mesh. A mesh consists of a set 
of vertices and indices defining the topology. These are respectively generated 
in the aptly named `GenerateVertices` and `GenerateIndices` methods. The 
vertices are basically spaced out over the plane according to the user 
specified values. The indices define the lines between the vertices. These are 
set with the `SetIndices` method, and specified as being `MeshTopology.Lines`.
This will ensure that in our shader the lines are interpreted as lines, and not
triangles.

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/663fd5bb130979ed04568102c301d504.js"></script>
</div>

When rendered with a default unlit shader, this has the following result:

<p align='center'><img align='center' src='https://github.com/BeardedPlatypus/media-storage/blob/main/blog/geometry-shader/lines.png?raw=true' width='100%'></p>

The full script can be found [here](https://gist.github.com/BeardedPlatypus/14ef94e2cf5a7adb2dc7ddb9c76e42e8)

### Giving lines depth with the geometry shader

Now that we have our lines, as shown above we can take a look at how to give 
these lines a width. Before we start doing anything, first create an unlit 
shader if you had not done so before, and assign it to the `GenerateLines` 
lines script. 

Next we will enable the geometry shader. For this we need to make the following
changes to the default unlit shader:

* Define a `_Width` float property in the properties
* Add the `#pragma geometry geom` line
* Rename `v2f` to `v2g` to indicate that the vertex dat is passed to the geom shader
* Add a new struct `g2f` which will hold the data being passed from the geom shader to the frag shader
* Add a new `void geom` method to the shader, under the `v2g vert` method
* Change the `v2g vert` shader to hand over the world coordinate vertex data instead of clip space

This should lead to code similar to the following:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/69fd0926c005ba95297780f9cfc3f0fa.js"></script>
</div>

Before discuss how to generate lines with width, let's first look at the 
declaration of the geometry shader:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/69a9399f4c29d17459dbec37d5e9532f.js"></script>
</div>

The `geom` is defined by the pragma we added at the beginning of the shader. 
Next we indicate that the geometry shader will receive line primitives 
consisting of two vertex shader structs, this is our input data. Lastly we 
define a `TriangleStream<g2f>` will be used to output our triangles consisting 
of three vertex each defined by a `g2f` struct. This is necessary because the
geometry shader itself is a void, and thus does not directly output elements.
Instead it uses a stream to do so. Lastly, we define the attribute 
`maxvertexcount` on the geometry shader. This specifies the maxmimum of new 
primitives being generated by this shader. As we will see in the next 
paragraph, this will be 6 in our case.

If we want to give our lines depth, we will need to represent each line as a
quad, or two triangles. These quad will be the same length as the provided 
line, and will have a width of `_Width`. In order to generate the vertices
of the quad we can adopt the following strategy given the following points:

* We will only generate a two dimensional grid, thus we do not need to worry about the y-axis location of the end points
* The new vertices will lay perpendicular to the existing two points at +/- half the defined width

This is illustrated in the following figure:

<p align='center'><img align='center' src='https://github.com/BeardedPlatypus/media-storage/blob/main/blog/geometry-shader/lines_illustration.png?raw=true' width='100%'></p>

Rotating a vector (x, y) by 90 degrees, corresponds with the vector (y, - x). 
We can define the direction of the line and the corresponding offset as

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/337ed23961bf7cae5473500c94f83776.js"></script>
</div>

Next we can define the four new vertices as follows:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/63903e04278ed419d1d552d52a38ed79.js"></script>
</div>

With the vertices of a line defined, we can generate the two triangles, as shown in the figure:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/ec39ef3001d562286a1e38ec9e97a176.js"></script>
</div>

You see that we generate two triangles, each closed with a `RestartStrip` call. 
Further note that the order of the vertices is important. If the wrong order is
specified it might flip the normal in the opposite direction than what you are
expecting. If you do not see anything, it might be the case that you need to 
look at the grid from underneath instead of on top. In order to fix this you 
want to flip the order of vertices (switch the first and third vertices).

When finished you should see the following when starting the game:

<p align='center'><img align='center' src='https://github.com/BeardedPlatypus/media-storage/blob/main/blog/geometry-shader/lines_with_thickness.png?raw=true' width='100%'></p>

The full code for the shader can be found [here](https://gist.github.com/BeardedPlatypus/303b43347b271f44cd88526cf568cc9d)


## Rendering points as circles with the help of a geometry shader

When the lines are thin enough and the lines are basically a uniform grid, then
current geometry and shader set up might be sufficient. However, when grid 
is less uniform, and if the lines are thicker, you might start seeing small 
artifacts at the points where the lines do not correctly flow into each other.
We can remedy this by rendering the points explicitly. Furthermore, we can 
actually emphasise the vertices of our plane, by rendering our points with a
larger diameter than our line width, ensuring they show up separately.

In order to render the points, we will again first create a script that creates
the geometry at runtime, and then the shader that transforms the geometry into
the actual circles.

### Generating the vertices of a plane

Generating the geometry for the point shader is even simpler than the lines. 
It mostly looks similar to the generation of the geometry of the lines. The
vertices are generated completely the same as in the line geometry script, and
could in theory be shared between the two, however for this particular example
I thought that would be overkill. The two major differences are in the indices.
First, the generation of the indices is simpler, basically each vertex will
now be a point primitive, thus we only need a range equal to the size of the
vertices and we need to specify that the mesh topology consists of points this
time.

In order to create this script we will do the same as with the lines:

* Add a new empty game object
* Create a new script and assign it to the empty game object
* Add the script code defined [here](https://gist.github.com/BeardedPlatypus/615db9f5bf0cd5bda1632559749be3f3)

As you can see, the code is basically a simplified version of the line creation
code. Do note the `SetIndices` line though.

If we temporarily disable the lines object, and create a new unlit shader, we
should see the following:

<p align='center'><img align='center' src='https://github.com/BeardedPlatypus/media-storage/blob/main/blog/geometry-shader/points.png?raw=true' width='100%'></p>

### Turning the points into circles with a given radius

With the geometry set up, we can again set up a simple shader to turn our points
into circles. In order to do this, it is easiest to start with a copy of your 
lines shader and remove the content of the geometry function. Next we adjust
the function declaration to the following:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/0d09f5282614ea8150beb20b5d12e6e2.js"></script>
</div>

As you can see, this time we use point primitives, which consist of only a single
primitive.

Next we can take a look how to generate a circle. In our case, we want to create a
simple approximation of a circle consisting of a number of triangles. In order to 
generate such an approximation we will generate 'n' number of vertices, where n is
any number greater than 3, for example 12. In order to fill the vertices, we will
create 'n-2' triangles, thus the `maxvertexcount` is set to '(n-2)*3', or in the
case of 12, to 30.

With the initial definition out of the way let's define how we generate our circles

* The vertex provided by the vertex shader is going to be the centre of our circle.
* The width is going to be equal to our diameter of the circle
* The circles will be generated in two dimensions, thus we will again take over the y-axis value of the original vertex
* The vertices generated by our geometry shader are going to be evenly spread out on a circle of half width

With that knowledge out of the way, we can define n vertices offsetted from our 
centre vertex. The offset will be equal to the vector (0.5 * width, 0) rotated
by the index of the vertex times '360 degrees / n' around. Lastly, we can fill
our circle by creating triangles from a single vertex, and walking over the 
other vertices, as illustrated here:

<p align='center'><img align='center' src='https://github.com/BeardedPlatypus/media-storage/blob/main/blog/geometry-shader/points_illustration.png?raw=true' width='100%'></p>

If we put this into code we will get the following:

<div style="margin: 5px 0px 5px 0px;">
<script src="https://gist.github.com/BeardedPlatypus/51444f0528f9b65ad4a00979d5497700.js"></script>
</div>

In tihs code snippet we first define our initial vertex, which will act as our 
anchor for the triangle strip. Next we generate the other vertices, and append
them within the for-loop to generate our triangles. The full shader code can be
found [here](https://gist.github.com/BeardedPlatypus/47910ac6559ef126e9ade5fb9113f9ac). 
In order to ensure the points render on top, I have moved them 0.0001 above the
y axis position of the grid lines.

When set up correctly it should look like this:

<p align='center'><img align='center' src='https://github.com/BeardedPlatypus/media-storage/blob/main/blog/geometry-shader/circles.png?raw=true' width='100%'></p>

When we enable the lines again, we get the result as shown at the beginning, 
which completes our set up.

## Final thoughts and next steps

With the grid completely set up, there are some avenues we could pursue further. 
The next step I will most likely take is combining the scripts into a single 
component, that will generate a complete grid and corresponding geometry. Once
this is done, it should be easier to create a script that can generate a grid 
from provided data.

Another interesting next step would be to investigate how to visualise data
associated with the grid on the grid itself. While I have not tested it, I
believe it should be possible to associate colours with the primitives and
use vertex colours to generate the appropriate styling. If we want to render
data on top of the faces, we would need to extend the geometry, and render 
the faces as well.

In either case, we have a solid foundation to further extend our grid from.
Thank you for reading, and I am looking forward to seeing you again in the 
future.