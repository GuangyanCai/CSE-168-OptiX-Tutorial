# Assignment 1
This tutorial will cover the basics of using OptiX for the first assignment. Complete all the tasks as you are reading and you will get a GPU ray tracer, which you will expand to a path tracer in the future assignments.

## Table of Contents
- [Assignment 1](#Assignment-1)
  - [Table of Contents](#Table-of-Contents)
  - [Requirements](#Requirements)
  - [Build Instructions](#Build-Instructions)
    - [Windows](#Windows)
    - [Linux](#Linux)
  - [About OptiX](#About-OptiX)
  - [OptiX Programs](#OptiX-Programs)
    - [Basics](#Basics)
    - [Ray Generation Program](#Ray-Generation-Program)
    - [Intersection Program](#Intersection-Program)
    - [Material Programs: Closest-hit Program](#Material-Programs-Closest-hit-Program)
    - [Material Programs: Any-hit Program](#Material-Programs-Any-hit-Program)
    - [Bounding Program](#Bounding-Program)
  - [Advanced Features](#Advanced-Features)


## Requirements
* NVIDIA GPU with driver version 418+
* [CUDA Toolkit 10.0+](https://developer.nvidia.com/cuda-toolkit)
* [OptiX 6.0.0+](https://developer.nvidia.com/optix) (You will need to create a free NVIDIA developer account.)
* Starter code
  * [Windows](https://github.com/GuangyanCai/CSE-168-OptiX-Starter-Code-Windows)
  * [Linux](https://github.com/GuangyanCai/CSE-168-OptiX-Starter-Code-Linux)

To verify that you have the correct versions for the GPU driver and CUDA, you can run `nvidia-smi` to see the versions. On Windows, use PowerShell or Command Prompt to change directory to `C:\Program Files\NVIDIA Corporation\NVSMI` and run `nvidia-smi`. On Linux, simply run `nvidia-smi`.

On Windows, you will find the OptiX SDK installed at `C:\ProgramData\NVIDIA Corporation\OptiX SDK 6.0.0`. On Linux, you will get the SDK directory, `NVIDIA-OptiX-SDK-6.0.0-linux64/`, which you can install in `/usr/local/`, `/opt/`, or even just leave in your home directory. However, it's recommended that you install it in `/usr/local` so that you don't need modify the makefile. OptiX is shipped with some pre-compiled examples in the SDK directory. Try running `optixWhitted` or `optixPathTracer` to make sure you've installed everything correctly.

The starter code have two versions: `Windows` and `Linux`. Choose one of them based on your platform. Make sure you can compile the starter code (read the build instructions on how to do so). If it compiles and runs successfully, you should see a window pop out showing an image that is all red and the same image is generated in the directory.

## Build Instructions
Make sure to install all the requirements above before reading this section. 
### Windows
Install Visual Studio 2017 if you haven't already (other versions might work as well but that is not guaranteed). In the starter code, you should see: `PathTracer` and `PathTracer.sln`. `PathTracer` is a directory that contains the source codes and the test scenes. `PathTracer.sln` is a solution file. Double click to open it in Visual Studio. Click `Debug` (the one next to `x64`) to change the configuration to `Release`. 
![window_1](windows_1.png?raw=true)
![window_2](windows_2.png?raw=true)

Right-click `PathTracer` in the solution explorer and go to properties. 
![window_3](windows_3.png?raw=true)

Under `Configuration Properties`, select `Debugging` and edit `Command Arguments` to whatever scene file you want to run, for instance, `Scenes/scene1.test`. 

![window_4](windows_4.png?raw=true)

Remember to change this when you need to render other scenes. Now press `ctrl + F5` or click `Local Windows Debugger` to build and run the program. 

### Linux
In the starter codes, you should see: `PathTracer`. `PathTracer` is a directory that contains the source codes and the test scenes. It also contains a makefile to help you build the program. To run it you will need `make` and `g++`. Install them if you haven't already. On Ubuntu or other Debian-based distributions, you can install them with: 

```
sudo apt-get install build-essential
```
You will also need `GLEW` and `GLFW` for displaying the image with OpenGL. You can install them with:
```
sudo apt-get install libglew-dev libglfw3-dev
```
Now you are ready. Head to the starter code and enter the directory `PathTracer`. There should be a makefile. To build or rebuild your codes, run 
```
make
``` 
If you want a clean rebuild, run
```
make clean && make
```
If build successfully, an executable called `PathTracer` will be generated. It requires an argument specifying what scene file to render. For example, to render the first test, you can run
```
./PathTracer Scenes/scene1.test
```


## About OptiX
The OptiX API was developed by Nvidia to accelerate ray tracing applications using GPUs. It utilizes another API created by Nvidia called CUDA, which allows developers to conduct general purpose processing on GPUs. A CUDA program consists of `host` and `device` code: `host` code runs on the CPU and `device` code runs on the GPU. For this project, all `host` code belongs in `.cpp` files and all `device` code belongs in `.cu` files. OptiX provides an abstracted pipeline for ray tracing algorithms, which allows developers to define their own "programs" in `device` code to fit into the pipeline. They are analogous to shaders in OpenGL.

For reference, the [programming guide](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#guide#) and the [API documentation](https://raytracing-docs.nvidia.com/optix_6_0/api_6_0/html/index.html) will be very helpful. Do be aware that the programming guide uses the OptiX C API most of the time but for the assignment, while we will use OptiXpp, a thin C++ wrapper for the C API.

## OptiX Programs
OptiX programs are the building blocks in an OptiX application. There are several types of programs that each perform a specific job in the render pipeline. See the flowchart below for an overview of the basic OptiX programs.

![flowchart](flowchart.png?raw=true)

Once a scene is loaded, rendering can be started by "launching" OptiX. This triggers the [Ray Generation Program](#ray-generation-program) to generate primary camera rays using some user-defined camera model. OptiX will automatically traverse the scene, calling the [Intersection Program](#intersection-program) of any geometry which the rays might hit. When the nearest intersection is found, the ray's [Closest-hit Program](#material-programs-closest-hit-program) is called, which usually runs shading calculations for the intersection and can optionally fire secondary rays. If no intersection is detected, the ray's [Miss Program](#miss-program) is called, which can be used to define what the background of a scene looks like. The ray can also have an [Any-hit Program](#material-programs-any-hit-program), which is called on *any* detected intersection regardless of whether it is the closest intersection. The Any-hit Program is useful for techniques such as shadow rays which only care *if* an intersection occurs, but not necessarily *where* the closest intersection is. Additionally, if a [Bounding Program](#bounding-program) is defined for some geometry, OptiX can automatically construct a [bounding volume hierarchy](https://en.wikipedia.org/wiki/Bounding_volume_hierarchy) to accelerate geometry intersections.

### Basics
As mentioned above, OptiX programs are `device` codes and will be declared in the `.cu` files. You can use CUDA C syntax for these programs but C syntax should suffice since OptiX handles the CUDA portion for you. Each OptiX program is declared as a function:

```cpp
// Device

RT_PROGRAM void program(...) {
 ...;
}
```

All OptiX programs must have a qualifier before their function signatures and most of the time the qualifier is `RT_PROGRAM`, as shown above. Most programs will have `void` as the return type but the arguments may vary depending on their jobs. Also, there is no restriction on the function name nor the file name.

To make these programs do meaningful work, we need to supply them with data through `host` codes. To do that, we need to first declare it in the `host`:

```cpp
// Host

Program program = context->createProgramFromPTXFile(
  "PTX/foo.ptx",
  "bar");
```

The first argument of `createProgramFromPTXFile()` is the file name/location. CUDA files are compiled to PTX files hence the extension is `ptx` instead of `cu`. The starter code do it for you and store the PTX files in the `PTX/` directory. The second argument is simply the function name. `context` is an OptiX context, which you can ignore for now.

There are two ways to transfer data between `host` and `device`: through a `buffer` or a `variable`. A `buffer` is a multidimensional array, which you can think of as a vertex buffer object in OpenGL. To use it, declare the buffer in the `device` codes:
```cpp
// Device

rtBuffer<type, n> bufferName;
// Or
rtBuffer<type> bufferName; // You can ignore n if n = 1

type foo = bufferName[index]; // access an element at some index

int size = bufferName.size(); // get the number of elements in the buffer
```
`type` can be primitive types in C or user-defined structs. `n` is the dimension of the buffer. The starter code provide a function called `createBuffer()` to help you create a buffer that is filled with the data you provide:

```cpp
// Host

std::vector<type> data; // type should be the same as the one in device
... // insert your data
Buffer buffer = createBuffer(data);
```
However, it's not enough. We still need to connect it to the buffer we declared in the `device` codes:

```cpp
// Device

rtBuffer<MyType> myBuffer; // declared in some program

// Host
std::vector<MyType> myData;
...
Buffer buffer = createBuffer(myData);
program["myBuffer"]->set(buffer);
```
In the last line, we access `myBuffer` through `program["myBuffer"]` and set its content to `buffer`.

A `variable` is literally just a variable. You can think of it as a uniform variable in OpenGL.
```cpp
// Device

rtDeclareVariable(type, name, , );
```
Although we only fill in two parameters here, the type and the name of the variable, it actually needs four (thus we need to keep the extra commas). For now, we can leave the last two as blank. Similar to `buffer`, we can access a `variable` declared in `device` through `program["variableName"]`. However, we can't simply call `set()` to set the variable. Depend on the type of the variable, we need to call a different setter. For instance, if the type is `float3`, a 3D vector, we will call:
```cpp
// Host

program["myVar"]->setFloat(f1, f2, f3); // f1, f2, f3 are float
// Or
program["myVar"]->setFloat(f); // f is float3
```
You can read more [here](https://raytracing-docs.nvidia.com/optix_6_0/api_6_0/html/classoptix_1_1_variable_obj.html).

You might wonder what is `float3`. It's a type defined by OptiX to represent a 3D vector of `float`. Similarly, there are `float2`, `float1`, `int3`, etc. You can access its components by, for instance, `var.x`, `var.y` and `var.z`. It's highly recommended to read [this](https://raytracing-docs.nvidia.com/optix_6_0/api_6_0/html/optixu__math__namespace_8h.html) to learn more about the vector types and their operations. Read [this](https://raytracing-docs.nvidia.com/optix_6_0/api_6_0/html/optixu__matrix__namespace_8h.html) if you need to use the matrix types.

### Ray Generation Program
The entry point of all programs is a `ray generation program`. It is responsible for generating rays, shooting them and recording the results. In the starter code, you can find it in `PinholeCamera.cu`:

```cpp
// PinholeCamera.cu 
// Device

RT_PROGRAM void generateRays() {
 ...;
}
```

It has no arguments. Again, the function name doesn't matter. We need to inform OptiX where and which function is the `ray generation program`:
```cpp
// OptixApp.cpp
// Host

// programs is a hash table that maps std::string to Program
programs["rayGen"] = context->createProgramFromPTXFile(
  "PTX/PinholeCamera.ptx",
  "generateRays");

context->setRayGenerationProgram(0, programs["rayGen"]);
```
As previously shown, we declare the program first, then we set it as the `ray generation program` with `setRayGenerationProgram()`. `0` means we are using the first entry point. Since we only have one entry point, you don't need to care about it.

To shoot a ray, we need to create one first:
```cpp
// PinholeCamera.cu
// Device

Ray ray = make_Ray(eye, dir, BASIC_RAY, RAY_EPSILON, RT_DEFAULT_MAX);
```
`Ray` is a type defined by OptiX. We can create one using
```cpp
// Device

make_Ray(origin, direction, rayType, minT, maxT)
```
`rayType` is an `int` that tells OptiX which `Material Program` to use. In `Constants.h`, we define two ray types: `BASIC_RAY` and `SHADOW_RAY`. When you need to create a baisc ray, use `BASIC_RAY` as the ray type. Similarly, use `SHADOW_RAY` as the ray type for shadow ray. We also define a constant called `RAY_EPSILON`, which equals 0.001. It's used as `minT` in this case.

To receive the result that the ray gets, we need it to carry a `payload`, which is a user-defined `struct` that contains important variables that other programs will read or write later, such as the color of the intersection. Our definition of a minimal payload struct is located in `Payload.h`:

```cpp
// Payload.h
// Device

struct Payload {
  float3 result; // the color of intersection
  int depth; // recursion depth
};
```

Here, we simply declare one and set its depth:
```cpp
// PinholeCamera.cu
// Device

Payload payload;
payload.depth = depth.x;
```
Notice that `depth` is a variable declared above the function:
```cpp
// PinholeCamera.cu
// Device

rtDeclareVariable(int1, depth, , ); // recursion depth
```
We set the value of depth in `OptixApp::buildScene()`:
```cpp
// OptixApp.cpp
// Host

programs["rayGen"]["depth"]->setInt(depth);
```
You will need to set your own variables like this later.

Let's look at other variables and buffers we have here:
```cpp
// PinholeCamera.cu
// Device

rtBuffer<float3, 2> resultBuffer; // used to store the render result

rtDeclareVariable(rtObject, root, , ); // Optix graph

rtDeclareVariable(uint2, launchIndex, rtLaunchIndex, ); // a 2d index (x, y)
```
`resultBuffer` is a 2D buffer of `float3` and we will use it to store the rendered image. `launchIndex` is a 2D vector of `uint` and it stores the index of the pixel we are rendering.
```cpp
// PinholeCamera.cu
// Device

// Write the result
resultBuffer[launchIndex] = payload.result;
```
Notice that when we declare `launchIndex`, we fill in the third argument. This argument is for "semantic". OpitX provides some internal semantics that allows you to access some important variables it creates. For instance, `rtLaunchIndex` will give users the current launch index signifying which pixel we are working on. You can read about other internal semantics and in which programs you can access them [here](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#programs#internally-provided-semantics).

`root` is a `rtObject` and it is the root of the OptiX graph we are going to traverse. You can think of an OptiX graph as a scene graph that contains all the geometries, although it has more going on. You can read about it [here](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#host#graph-nodes) but the starter codes do it for you already.

Now we are ready to trace the ray.
```cpp
// PinholeCamera.cu
// Device

rtTrace(root, ray, payload);
```
Obviously, the current ray is not correct. To make the correct ray, we need more information.

> **Task 1**:
> 1. In `OptixAPP::buildScene()`, read the following values from the scene file and transfer them to the ray generation program: eye, center, up and fovy. The starter code contain some examples.
> 2. In `generateRays()`, use the variables you have to calculate the correct origin and direction of the ray. For now, you can test it by printing values using `rtPrintf()`, which has the same usage as  `printf()`.

<br />

 ### Miss Program
If a ray doesn't intersect with any geometries, it will invoke the `miss program`. You can find it in `PinholeCamera.cu`:

```cpp
// PinholeCamera.cu
// Device

RT_PROGRAM void miss() {
  ...
}
```

Since we haven't insert any geometries to the OptiX graph yet, you can expect this program getting called for every pixel. That's why the image we produce now is all red:
```cpp
// PinholeCamera.cu
// Device

payload.result = make_float3(1, 0, 0);
```
Here, we set the result of the current payload to be red. You can set it to another color or even use a skybox for the background.
> **Task 2**:
> 1. Before the declaration of `miss()`, we declare a variable call `backgroundColor`. Set the result of the payload to `backgroundColor`. Compile and run the application again and you should see a black image is produced instead.

<br />

### Intersection Program
As the ray traverse through the OptiX graph, it will invoke the `intersection program` of the current geometry to test whether they intersect. Take a look at `intersect()` in `Triangle.cu`:

```cpp
// Triangle.cu
// Device

RT_PROGRAM void intersect(int primIndex) {
  ...
}
```

You should notice right away that this program has a parameter: `primIndex`. It stands for "primitive index". It can be used to access the current triangle from a buffer of triangles:
```cpp
// Triangle.cu
// Device

rtBuffer<Triangle> triangles; // a buffer of all triangles

rtDeclareVariable(Ray, ray, rtCurrentRay, ); // the current ray
```
The type `Triangle` is a struct containing useful information about a triangle. You can find its declaration in `Geometries.h`:

```cpp
// Geometries.h
// Device/Host

struct Triangle {
  ...
};
```
Using the information in a `Triangle` and the current ray, you should be able to calculate `t`, the distance from the ray origin to the intersection. If there is no intersection, simply call `return` to exit. If there is an intersection, we will do:

```cpp
// Triangle.cu
// Device

if (rtPotentialIntersection(t)) {
  rtReportIntersection(0);
}
```
`rtPotentialIntersection()` will return true if `t` is in the range of `minT` and `maxT` of the current ray. If `t` is valid, then we will call `rtReportIntersection()`, which takes a material index indicating which material we are using. Since we only use one material in this assignment, we will pass a `0` as the index.

> **Task 3**:
> 1. In `OptixAPP::buildScene()`, read the following values from the scene file: tri and vertex. You should add fields to the declaration of `Triangle` so that you can create a `Triangle` to store the information read from "tri". There is a vector of `Triangle` called `triangles` in `buildScene()` and it will be used to create the triangle buffer. Remember to push the `Triangle` you create to `triangles`!
> 2. Implement the triangle intersection program in `Triangle.cu`. If done correctly, you should produce the correct image running `scene1.test`. This is also a good time to check the correctness of the ray generation program.
> 3. Repeat 1. and 2. for `Sphere`. If done correctly, you should produce an image that shows the correct geometries but in red running `scene2.test`.


<br />

### Material Programs: Closest-hit Program
Right now, the color of the geometries are all red and that's not interesting at all. The program where we do the shading is the `closest-hit program`. As its name suggests, it's invoked when a ray finds the closest intersection.

```cpp
// BlinnPhong.cu
// Device

RT_PROGRAM void closestHit() {
  ...
}
```

Currently, it simply sets the result of the payload to be red:

```cpp
// BlinnPhong.cu
// Device

float3 result = make_float3(1, 0, 0);
payload.result = result;
```

We need more information about the intersection to make better shading and we can do it with `attribute variables`.

```cpp
// Device

rtDeclareVariable(type, name, attribute semName, );
```
As usual, the first parameter is the type of the variable and the second parameter is the name of the variable. The third parameter is a semantic name. Instead of using the internal semantics we introduce earlier, we can define our own semantic with a name that follows the keyword `attribute`. For instance, we can define an attribute variable for transferring ambient color by:

```cpp
// Device

rtDeclareVariable(float3, ambient, attribute Ambient, );
```
Notice that the fourth parameter is left empty. To transfer data from the `intersection program` to the `closest-hit program`, we need to declare the attribute variables in both programs with the same semantic name:

```cpp
// Intersection Program
// Device

rtDeclareVariable(float3, ambient, attribute Ambient, );
RT_PROGRAM void intersect(int primIndex) {
  ...
}

// Closest-Hit Program
// Device

rtDeclareVariable(float3, ambient, attribute Ambient, );
RT_PROGRAM void closestHit() {
  ...
}
```
Then, we need to set the attribute variables between calls to `rtPotentialIntersection()` and `rtReportIntersection()`:
```cpp
// Intersection Program
// Device

if (rtPotentialIntersection(t)) {
  ambient = somevalue;
  rtReportIntersection(0);
}
```
This is very important: you must set them between the calls to make it work. Now, if the current intersection is the closest one, you can access this ambient value in `closestHit()`.

> **Task 4**:
> 1. In `OptixAPP::buildScene()`, read the value "ambient" from the scene file. Expand the definitions of `Triangle` and `Sphere` to include ambient. Pass this value to the closest-hit program in `BlinnPhong.cu` as shown above and use it as the final color. If done correctly, you will produce the correct image for `scene2.test`.
> 2. Read the following values for transformations: translate, scale, rotate, pushTransform and popTransform. Again, expand the definitions of `Triangle` and `Sphere` to encode their transformations. If done correctly, you will produce the correct image for `scene3.test`.
> 3. Continue to expand the definitions by reading diffuse, specular, emission and shininess. Don't forget to pass them via attribute variables as well. It's highly recommended to create a struct that contains all the variables you want to pass and use an attribute variable to transfer the struct.

<br />

### Material Programs: Any-hit Program
Using only ambient color for shading is boring as well. We would like to add lighting to the scene and that would introduce shadow. In ray tracing, we determine whether an object is in shadow by shooting a shadow ray from the intersection to the light sources and if it hits any other objects, the object is in shadow. This is what the `any-hit program` is for. Before we get into that, we need to talk about shadow ray first. Originally, we can shoot a basic ray like this:

```cpp
// PinholeCamera.cu
// Device

Ray ray = make_Ray(eye, dir, BASIC_RAY, RAY_EPSILON, RT_DEFAULT_MAX);
Payload payload;
...
rtTrace(root, ray, payload);
```
The third parameter is the ray type, We define 0 to be basic ray and 1 to be shadow ray. Therefore, you can create a shadow ray by:

```cpp
// Device

Ray shadowRay = make_Ray(origin, dir, SHADOW_RAY, RAY_EPSILON, distanceToLight);
ShadowPayload shadowPayload;
...
rtTrace(root, shadowRay, shadowPayload);
```

We use a different `payload` here that is specifically for shadow rays. These ray types are defined in the starter code. We can assign a `closest-hit program`, an `any-hit program`, or both to each ray type. When a ray of specific ray type is traced, it will call the the programs assigned to it. In `BlinnPhong.cu`, `closestHit()` is the `closest-hit program` for ray type 0 and `anyHit()` is the `any-hit program` for ray type 1. Don't worry if you don't understand this part. All you need to remember is if you create a ray of ray type 0, it will call `closestHit()` when it finds the closest intersection; if you create a ray of ray type 1, it will call `anyHit()` for every intersection.

`Any-hit program` is very simple. In our case, we only want to determine whether the object is in shadow and entering `anyHit()` means there is an intersection, so we can use a variable in the shadow payload to indicate it is in shadow. After that, we call `rtTerminateRay()` to stop tracing the shadow ray.

> **Task 5**:
> 1. Complete the definition of `ShadowPayload` in `Payload.h` and `anyHit()` in `BlinnPhong.cu`. This part should only take a few lines.
> 2. Read the following values: point, directional and attenuation. Complete the definitions of `PointLight` and `DirectionalLight` in `Lights.h` to store useful information about these lights. Create these structs while reading the values and push them into `plights` and `dlights`, respectively. `plights` and `dlights` are vectors of `PointLight` and `DirectionalLight`. They will be used to create the buffers in `BlinnPhong.cu`.
> 3. Implement direct illumination in `closestHit()` using the Blinn-Phong shading model with attributes we show above. Add variables or buffers if you need to. For example, you might want to have the intersection location as an attribute variable. Use `scene4-ambient.test`, `scene4-diffuse.test` and `scene4-emission.test` to test correctness.

We would like to have reflections as well. We don't need a new ray type for reflection ray. Just use 0:

```cpp
// Closest-Hit Program
// Device

Ray reflectionRay = make_Ray(origin, dir, BASIC_RAY, RAY_EPSILON, RT_DEFAULT_MAX);
Payload reflectionPayload;
reflectionPayload.depth = currentDepth - 1;
rtTrace(root, reflectionRay, reflectionPayload);
```

Remember that since this ray is of ray type 0, it will invoke `closestHit()`, essentially making it a recursion function. Thus, we need to set the depth variable in `Payload` to avoid stack overflow.

> **Task 6**:
> 1. Implement reflection in `closestHit()`. Use `scene4-specular.test`, `scene5.test` and `scene6.test` to test correctness.

<br />

### Bounding Program
Now, you should be able to render the correct images described by those scene files. You can try to render `scene7.test` which contains numerous triangles. It will take longer than other scene files but should only take around one minute to complete. Although the renderer is fast (compared with the CPU version in CSE 167), it can be significantly improved by adding accelerating structures. OptiX is extremely optimized for this task and makes it easy to use. To use it, define a bounding program:

```cpp
// Device

RT_PROGRAM void bound(int primIndex, float result[6]) {
  ...
}
```

The `primIndex` is the same as the one in `intersect()`. `result` is an array of `float` that you will fill it. The first three elements of `result` is the smallest vertex of the bounding box and the other three elements is the largest vertex of the bounding box. For instance, if `result = {0, 0, 0, 1, 1, 1}`, it will define a bounding box whose smallest vertex is `(0, 0, 0)`, largest vertex is `(1, 1, 1)` and the other vertices are `(1, 0, 0)`, `(0, 1, 0)`, `(0, 0, 1)`, `(1, 1, 0)`, `(1, 0, 1)` and `(0, 1, 1)`.

Once you have defined the bouding programs for triangles and spheres, enable accelerate structure in `OptixApp::buildScene`:

```cpp
// OptixApp.cpp
// Host

// Change these lines
GG->setAcceleration(context->createAcceleration("NoAccel"));
...
root->setAcceleration(context->createAcceleration("NoAccel"));

// To these lines
GG->setAcceleration(context->createAcceleration("Trbvh"));
...
root->setAcceleration(context->createAcceleration("Trbvh"));
```
We didn't have any acceleration structure and now we will be using `Trbvh`, a very fast GPU-based BVH build. You can read more [here](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#host#acceleration-structure-builders).

> **Task 7**:
> 1. Complete `bound()` in `Triangle.cu` and `Sphere.cu`. Enable acceleration structure as shown above. Test it on previous scene files to make sure the images produced are still correct, then test it on `scene7.test` and see how much the performance improves.

Awesome! Now you have a working GPU ray tracer. You can make it more powerful by adding more features from OptiX but it's not required. 

## Advanced Features 
We are only covering the basic features of OptiX in this tutorials. To get the full picture of OptiX's capabilities, read the [programming guide](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#guide#). Here are some advanced features that you might be interested in:
* [Built-in support for triangles](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#host#triangles): OptiX has built-in support for triangles, which allows faster triangle intersection tests. It doesn't require an intersection program or a bounding program so it's indeed easier than what we do in the assignment. If you happen to have a RTX card, you should definitely try this feature out because it utilizes the RT cores inside your RTX card and makes it even faster. 
* [Progressive launches](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#host#progressive-launches): When we do path tracing, it is pretty common that we need to use a lot of samples to generate a noise-free image and it often takes a long time. To better visualize the process, we can use progressive launches. It progressively adds new samples to the result buffer so that you can watch an image go from noisy to smooth.
* Interoperability with [OpenGL](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#opengl#interoperability-with-opengl) and [CUDA](https://raytracing-docs.nvidia.com/optix_6_0/guide_6_0/index.html#cuda#interoperability-with-cuda): Since OptiX, OpenGL and CUDA store data on the GPU, they can share the data within the GPU without transferring it first to the CPU, which greatly increases the performance if you are planning to use them together. 
