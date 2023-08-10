# BGFX how-do-I

A quick guide on how to do common tasks with the [BGFX](https://github.com/bkaradzic/bgfx) rendering library

## How do I initialize BGFX with GLFW?

```cpp
// Change X11 to WIN32 for Windows and COCOA for MacOS
#define GLFW_EXPOSE_NATIVE_X11
#include <GLFW/glfw3native.h>

...
int width = 640;
int height = 640;

bgfx::Init bgfx_init;
// Auto chooses type.
// You can also select a specific type, for example bgfx::RendererType::Vulkan
bgfx_init.type = bgfx::RendererType::Count;
// Native handle
bgfx_init.platformData.nwh =
    (void*) glfwGetX11Window(window);
bgfx_init.platformData.ndt =
    glfwGetX11Display();
bgfx_init.resolution.width = width;
bgfx_init.resolution.height = height;
bgfx_init.resolution.reset = BGFX_RESET_VSYNC;
// To stop BGFX automatically loading renderdoc in debug build
bgfx_init.debug = false;
bgfx_init.profile = false;

if(!bgfx::init(bgfx_init)) {
    // Failed to initialize BGFX, do error handling
}

bgfx::setDebug(BGFX_DEBUG_NONE);

bgfx::setViewRect(0, 0, 0, width, height);
bgfx::setViewClear(
    0,
    BGFX_CLEAR_COLOR | BGFX_CLEAR_DEPTH,
    0x7F7F7FFF, // RGBA in bytes
    1.0f,
    0);
```

## How do I render debug text?
```cpp
...
// Straight after bgfx::init() is called
bgfx::setDebug(BGFX_DEBUG_TEXT);
...
// Assuming GLFW is in use
while(!glfwWindowShouldClose()) {
    bgfx::touch(0);

    bgfx::dbgTextClear();

    // Formatted and ANSI escape codes supported
    bgfx::dbgTextPrintf("Hello, World!");

    bgfx::frame();

    glfwPollEvents();
}
```

## How do I clean up BGFX?

```cpp
...
bgfx::shutdown(0);

// Window-specific cleanup functions go here
glfwTerminate();
```

## How do I render static geometry?

### Data
```cpp
const float vertices[] = {
    -0.5f, -0.5f, 0.0f, 0.0f, 0.0f, 1.0f, 1.0f,
    0.5f, -0.5f, 0.0f, 0.0f, 1.0f, 0.0f, 1.0f,
    0.0f, 0.5f, 0.0f, 1.0f, 0.0f, 0.0f, 1.0f,
};

const unsigned int indices[] = {
    0, 1, 2
};
```

### Vertex Layout
```cpp
bgfx::VertexLayout vertex_layout;
vertex_layout.begin()
    .add(
        bgfx::Attrib::Position, // Role
        3, // Size
        bgfx::AttribType::Float) // Type
    .add(
        bgfx::Attrib::Color0,
        4,
        bgfx::AttribType::Float)
    .end();
```

### Vertex Buffer
```cpp
bgfx::VertexBufferHandle vbh = bgfx::createVertexBuffer(
    bgfx::makeRef(vertices, sizeof(vertices)),
    vertex_layout);
```

### Index Buffer
```cpp
bgfx::IndexBufferHandle ibh = bgfx::createIndexBuffer(
    bgfx::makeRef(indices, sizeof(indices)));
```

### Vertex Shader `vs_helloworld.sc`
```
$input a_position, a_color0
$output v_color0

void main() {
    gl_Position = vec4(a_position, 1.0);
    v_color0 = a_color0;
}
```

### Fragment Shader `fs_helloworld.sc`
```
$input v_color0

#include <bgfx_shader.sh>

void main() {
    gl_FragColor = v_color0;
}
```

### `varying.def.sc` file

This file is for input/output declarations
```
vec4 v_color0 : COLOR0;

vec3 a_position : POSITION;
vec4 a_color0 : COLOR0;
```

### Shader compilation
```sh
./your/path/to/shaderc \
    --type vertex \
    -i path/to/bgfx/src/ \
    --platform linux \
    --varyingdef varying.def.sc \
    -p spirv \
    --verbose \
    -f vs_helloworld.sc \
    -o vs_helloworld.bin

./your/path/to/shaderc \
    --type fragment \
    -i path/to/bgfx/src/ \
    --platform linux \
    --varyingdef varying.def.sc \
    -p spirv \
    --verbose \
    -f fs_helloworld.sc \
    -o fs_helloworld.bin
```

### Loading compiled shaders
```cpp
std::string read_file(std::string path) {
    std::ifstream input(path);

    if(!input.good()) {
        std::cerr << "Failed to open shader at " << path << std::endl;
        std::exit(-1);
    }

    std::stringstream buf;
    buf << input.rdbuf();

    if(input.fail()) {
        std::cerr << "Failed to read shader at " << path << std::endl;
        std::exit(-1);
    }

    return input.str();
}

bgfx::ShaderHandle load_shader(std::string path) {
    auto text = read_file(path);
    auto shader = bgfx::createShader(bgfx::copy(text.c_str(), text.length()));
    bgfx::setName(shader, path.c_str());
    return shader;
}
```

```cpp
...
bgfx::ShaderHandle vertex_shader = load_shader("vs_helloworld.bin");
bgfx::ShaderHandle fragment_shader = load_shader("fs_helloworld.bin");

bgfx::ProgramHandle program = bgfx::createProgram(
    vertex_shader,
    fragment_shader,
    true);
...
```

### Rendering

```cpp
// Assuming GLFW is in use
while(!glfwWindowShouldClose(window)) {
    bgfx::touch(0);

    bgfx::setVertexBuffer(0, vbh);
    bgfx::setIndexBuffer(ibh);

    bgfx::setState(BGFX_STATE_DEFAULT);

    bgfx::submit(0, program);

    bgfx::frame();
    glfwPollEvents();
}
```

### Cleaning up
```cpp
...
bgfx::destroy(vbh);
bgfx::destroy(ibh);
bgfx::destroy(program);

// Other cleanup functions like bgfx::shutdown() go here
```

Phew, that was a lot of work!

## How do I set uniforms?

### Fragment Shader

```
$input v_color0

#include <bgfx_shader.sh>

// Only mat4 and vec4 supported due to other types not mapping to anything on hardware
uniform vec4 u_params1;
#define u_brightness u_params.x

void main() {
    gl_FragColor = v_color0 * u_brightness;
}
```

### Uniform Handles
```cpp
...
bgfx::UniformHandle u_brightness =
    bgfx::createUniform(
        "u_params1",
        bgfx::UniformType::Vec4);

// Create program
...
```

### Uploading Uniform

```cpp
...
float u_params1[] = {0.0f, 0.0f, 0.0f, 0.0f};
// Assuming GLFW is in use
float brightness = bx::abs(bx::sin((float) glfwGetTime()));
u_params1[0] = brightness;

bgfx::setUniform(u_brightness, &u_params1);

// Set vertex/index buffers
...
```

## How do I use textures?

### Vertex Shader

```
$input a_position, a_texcoord0
$ouput v_texcoord0

void main() {
    gl_Position = vec4(a_position, 1.0);
    v_texcoord0 = a_texcoord0;
}
```

### Fragment Shader

```
$input v_texcoord0

SAMPLER2D(s_tex, 0);

void main() {
    gl_FragColor = texture2D(v_texcoord0, s_tex);
}
```

### `varying.def.sc` file
```
vec2 v_texcoord0 : TEXCOORD0;

vec3 a_position : POSITION;
vec2 a_texcoord0 : TEXCOORD0;
```

### Data

```cpp
const float vertices[] = {
    -0.5f, -0.5f, 0.0f, 0.0f, 0.0f,
    0.5f, -0.5f, 0.0f, 1.0f, 0.0f,
    0.5f, 0.5f, 0.0f, 1.0f, 1.0f,
    -0.5f, 0.0f, 0.0f, 0.0f, 1.0f,
};

const unsigned int indices[] = {
    0, 1, 2,
    2, 3, 0
};
```

### Vertex Layout

```cpp
bgfx::VertexLayout vertex_layout;
vertex_layout.begin()
    .add(
        bgfx::Attrib::Position, // Role
        3, // Size
        bgfx::AttribType::Float) // Type
    .add(
        bgfx::Attrib::TexCoord0,
        2,
        bgfx::AttribType::Float)
    .end();
```

### Loading a PNG image with [STB Image](https://github.com/nothings/stb/blob/master/stb_image.h)

```cpp
#include <tuple>
#include <glm/glm.hpp>
#include <stb/stb_image.h>
...

std::tuple<glm::ivec2, bgfx::TextureHandle> load_texture(std::string path) {
    glm::ivec2 size;
    int channels;
    
    stbi_set_flip_vertically_on_load(true);

    stbi_uc *data = stbi_load(path.c_str(), &size.x, &size.y, &channels, 4);

    auto res =
	bgfx::createTexture2D(
	    size.x, size.y,
	    false, 1,
	    bgfx::TextureFormat::RGBA8,
	    BGFX_SAMPLER_U_CLAMP
	    | BGFX_SAMPLER_V_CLAMP
	    | BGFX_SAMPLER_MIN_POINT
	    | BGFX_SAMPLER_MAG_POINT,
	    bgfx::copy(data, size.x * size.y * channels));

    if(!bgfx::isValid(res)) {
	    std::cerr << "Failed to load texture " << path << std::endl;
	    std::exit(-1);
    }

    stbi_image_free(data);

    return std::tuple<glm::ivec2, bgfx::TextureHandle>(size, res);
}
```

### Creating the uniform

```cpp
...
bgfx::UniformHandle s_tex = bgfx::createUniform(
    "s_tex", bgfx::UniformType::Sampler);
...
// Create program, etc

std::tuple<glm::ivec2, bgfx::TextureHandle> tuple =
    load_texture("your_texture.png");

glm::ivec2 texture_size = std::get<0>(tuple);
bgfx::TextureHandle texture = std::get<1>(tuple);
```

### Rendering

```cpp
...
while(!glfwWindowShouldClose(window)) {
    ...
    bgfx::setTexture(0, s_tex, texture); // Sets the uniform

    // Set buffers, submit
    ...
}
```

### Cleanup

```cpp
bgfx::destroy(texture); // Loaded texture
bgfx::destroy(s_tex); // Sampler
```

## How do I use an MVP matrix?

BGFX has some built-in uniforms, and the MVP matrix (or certain parts of it) just happen to be included.

They are as follows:
- `u_model`
- `u_view`
- `u_proj`
- `u_modelView`
- `u_viewProj`
- `u_modelViewProj`

To set these values:
- `bgfx::setTransform` for the model matrix
- `bgfx::setViewTransform` for the view and projection matrices

### Vertex Shader

```
...
#include <bgfx_shader.sh>

void main() {
    gl_Position = mul(u_modelViewProj, vec4(a_position, 1.0));
    ...
}
```