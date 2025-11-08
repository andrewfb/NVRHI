# NVRHI for OpenGL Developers

## Introduction

If you're an OpenGL developer looking at NVRHI (NVIDIA Rendering Hardware Interface), you're about to encounter some significantly different concepts from what you're used to. NVRHI is a cross-platform abstraction layer over Direct3D 11, Direct3D 12, and Vulkan 1.2, and it's designed around the "modern" graphics API paradigm introduced by DX12 and Vulkan.

This guide will help you understand NVRHI by translating concepts you know from OpenGL into their modern equivalents.

## The Big Picture: What Changed?

### OpenGL's Philosophy
In OpenGL, you work with a **state machine**:
- There's a global context with implicit state
- You bind objects (textures, buffers, shaders) to binding points
- You set state (blending, depth testing, etc.)
- You draw, and the driver figures out what to do
- The driver handles resource management, synchronization, and optimization

### Modern API Philosophy (DX12/Vulkan/NVRHI)
Modern APIs follow an **explicit, low-overhead** philosophy:
- State is **bundled into immutable objects** (Pipeline State Objects)
- Resource binding is **explicit and pre-declared**
- You **manage synchronization** (resource states, barriers)
- Command recording is **multi-threaded** and separated from execution
- Resource lifetimes are **explicitly managed**
- Less driver magic = more control + more responsibility

NVRHI sits in the middle: it gives you modern API benefits (explicit state, PSOs, binding sets) while handling the tedious parts automatically (resource state tracking, lifetime management, upload buffers).

---

## Core Concept Mapping

### 1. The Device: From Global Context to Device Object

**OpenGL:**
```cpp
// Implicit global context created by window/surface
glGenTextures(1, &texture);
glBindTexture(GL_TEXTURE_2D, texture);
```

**NVRHI:**
```cpp
// Explicit device object - central interface for everything
nvrhi::DeviceHandle device = nvrhi::vulkan::createDevice(...);
nvrhi::TextureHandle texture = device->createTexture(textureDesc);
```

**Key Difference:**
- No global state - the `IDevice` is your explicit handle to the GPU
- The device creates ALL resources: textures, buffers, pipelines, command lists
- The device manages resource lifetime and garbage collection
- You can have multiple devices if needed (multi-GPU)

### 2. Shaders and Programs: From Linked Programs to Pipeline State Objects

**OpenGL:**
```cpp
// Compile shaders
GLuint vs = glCreateShader(GL_VERTEX_SHADER);
glShaderSource(vs, 1, &vsSource, NULL);
glCompileShader(vs);

// Link program
GLuint program = glCreateProgram();
glAttachShader(program, vs);
glAttachShader(program, fs);
glLinkProgram(program);

// Use it
glUseProgram(program);
```

**NVRHI:**
```cpp
// Create shader objects from pre-compiled bytecode
nvrhi::ShaderHandle vs = device->createShader(vsDesc, vsBlob, vsBlobSize);
nvrhi::ShaderHandle fs = device->createShader(psDesc, fsBlob, fsBlobSize);

// Create PIPELINE (includes ALL state: shaders, blend, depth, raster, etc.)
auto pipelineDesc = nvrhi::GraphicsPipelineDesc()
    .setVertexShader(vs)
    .setFragmentShader(fs)
    .setRenderState(renderState)  // blend, depth, raster state
    .setInputLayout(inputLayout)
    .setPrimTopology(nvrhi::PrimitiveType::TriangleList)
    .addBindingLayout(bindingLayout);

nvrhi::GraphicsPipelineHandle pipeline = device->createGraphicsPipeline(pipelineDesc);

// Use it via state object
GraphicsState state;
state.pipeline = pipeline;
commandList->setGraphicsState(state);
```

**Key Differences:**
1. **No shader linking** - Modern APIs use pre-compiled bytecode (DXIL, SPIR-V)
2. **Pipeline State Objects (PSOs)** - ALL rendering state bundled together:
   - Shaders (VS, HS, DS, GS, PS)
   - Input layout (vertex attributes)
   - Blend state
   - Depth/stencil state
   - Rasterizer state
   - Primitive topology
   - Binding layouts (resource declarations)
3. **PSOs are immutable** - Created once, reused many times
4. **No runtime state changes** - Want different blending? Need a different PSO
5. **Why?** Driver can pre-compile and optimize everything. No state validation at draw time.

### 3. Resource Binding: From glUniform/glBindTexture to Binding Layouts and Sets

This is probably the biggest conceptual shift.

**OpenGL:**
```cpp
// Find uniform locations
GLint texLoc = glGetUniformLocation(program, "myTexture");
GLint colorLoc = glGetUniformLocation(program, "myColor");

// Bind resources at draw time
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, texture);
glUniform1i(texLoc, 0);
glUniform4f(colorLoc, 1.0f, 0.0f, 0.0f, 1.0f);

glDrawArrays(...);
```

**NVRHI (Two-Step Process):**

**Step 1: Declare Layout (what slots your shaders expect)**
```cpp
auto layoutDesc = nvrhi::BindingLayoutDesc()
    .setVisibility(nvrhi::ShaderType::AllGraphics)
    .addItem(nvrhi::BindingLayoutItem::Texture_SRV(0))        // t0 in HLSL
    .addItem(nvrhi::BindingLayoutItem::ConstantBuffer(1));    // b1 in HLSL

nvrhi::BindingLayoutHandle layout = device->createBindingLayout(layoutDesc);
```

**Step 2: Create Binding Set (fill slots with actual resources)**
```cpp
auto bindingSetDesc = nvrhi::BindingSetDesc()
    .addItem(nvrhi::BindingSetItem::Texture_SRV(0, myTexture))
    .addItem(nvrhi::BindingSetItem::ConstantBuffer(1, myConstantBuffer));

nvrhi::BindingSetHandle bindingSet = device->createBindingSet(bindingSetDesc, layout);

// At draw time - just reference the set
GraphicsState state;
state.pipeline = pipeline;
state.bindings = { bindingSet };
commandList->setGraphicsState(state);
commandList->draw(...);
```

**Key Differences:**

1. **Two-phase system:**
   - **Binding Layout** = declaration of slots (like a function signature)
   - **Binding Set** = actual resources (like function arguments)

2. **Layouts used for PSO creation** - The pipeline needs to know what resources to expect

3. **Sets are pre-filled** - You fill a binding set once, use it many times
   - In OpenGL: bind resources every draw call
   - In NVRHI: create set once, reference it at draw time
   - Much more efficient for the driver

4. **Register spaces match HLSL:**
   - `t0, t1, t2...` = Textures (SRVs)
   - `s0, s1, s2...` = Samplers
   - `b0, b1, b2...` = Constant buffers (uniforms)
   - `u0, u1, u2...` = RW buffers/textures (UAVs)

5. **Multiple binding sets** - You can have multiple sets:
   ```cpp
   state.bindings = { perFrameSet, perMaterialSet, perObjectSet };
   ```
   This maps to descriptor sets in Vulkan, root signature tables in DX12

### 4. Uniforms/Constants: From glUniform to Constant Buffers

**OpenGL:**
```cpp
glUniform4f(colorLoc, r, g, b, a);
glUniformMatrix4fv(matrixLoc, 1, GL_FALSE, &matrix[0][0]);
```

**NVRHI:**
```cpp
// Define structure matching shader
struct Constants {
    float4 color;
    float4x4 matrix;
};

// Create buffer
nvrhi::BufferDesc bufferDesc;
bufferDesc.byteSize = sizeof(Constants);
bufferDesc.structStride = sizeof(Constants);
bufferDesc.isConstantBuffer = true;
bufferDesc.isVolatile = true;  // For per-frame updates
nvrhi::BufferHandle buffer = device->createBuffer(bufferDesc);

// Update data
Constants data = { ... };
commandList->writeBuffer(buffer, &data, sizeof(data));

// Bind via binding set (see previous section)
bindingSet->addItem(BindingSetItem::ConstantBuffer(0, buffer));
```

**Key Differences:**

1. **Uniform Blocks only** - Modern APIs don't support loose uniforms
   - Everything goes into constant/uniform buffers
   - Must match shader structure layout (use `layout(std140)` or similar)

2. **Two types of constant buffers:**

   **Static Constant Buffers** - Immutable or rarely updated:
   ```cpp
   bufferDesc.isConstantBuffer = true;
   bufferDesc.isVolatile = false;  // Default
   nvrhi::BufferHandle staticCB = device->createBuffer(bufferDesc);

   // Update infrequently (causes GPU stall if GPU is using it)
   commandList->writeBuffer(staticCB, &data, sizeof(data));
   ```
   - Used for data that rarely changes (light properties, material constants)
   - Updating while GPU is reading will stall (synchronization point)
   - Single GPU-side allocation

   **Volatile Constant Buffers** - Updated frequently (per-frame/per-draw):
   ```cpp
   bufferDesc.isConstantBuffer = true;
   bufferDesc.isVolatile = true;  // Key flag!
   nvrhi::BufferHandle volatileCB = device->createBuffer(bufferDesc);

   // Update every frame - NO STALLS!
   for (each frame) {
       commandList->writeBuffer(volatileCB, &frameData, sizeof(frameData));
       commandList->draw(...);
   }
   ```
   - Used for per-frame/per-draw data (view/projection matrices, per-object transforms)
   - **NVRHI handles versioning automatically** - no GPU stalls!
   - Critical for performance with modern multi-buffered rendering

   **Why Volatile Constant Buffers Exist: The Frames-in-Flight Problem**

   In modern APIs, you typically have multiple frames "in flight" simultaneously:
   ```
   Frame N-2: GPU is executing commands
   Frame N-1: GPU is starting to process commands
   Frame N:   CPU is recording new commands
   ```

   **OpenGL (Simple but Inefficient):**
   ```cpp
   // OpenGL implicitly handles this
   glBufferSubData(GL_UNIFORM_BUFFER, 0, size, &newData);
   // Driver either:
   // 1. Stalls until GPU finishes reading old data (slow!)
   // 2. Creates a new backing store automatically (hidden overhead)
   ```

   **Modern APIs Problem:**
   ```cpp
   // BAD: Overwriting buffer GPU is still reading!
   commandList1->writeBuffer(cb, &frame1Data, size);
   device->executeCommandList(cmdList1);  // GPU starts reading cb

   commandList2->writeBuffer(cb, &frame2Data, size);  // OVERWRITES frame1Data!
   device->executeCommandList(cmdList2);  // GPU now sees corrupted data
   ```

   **Solution 1: Manual Ring Buffering (The Hard Way)**
   ```cpp
   // Create N versions manually
   const int framesInFlight = 3;
   std::vector<BufferHandle> constantBuffers(framesInFlight);

   for (int i = 0; i < framesInFlight; i++) {
       constantBuffers[i] = device->createBuffer(cbDesc);
   }

   int frameIndex = 0;
   while (rendering) {
       // Use different buffer each frame
       auto cb = constantBuffers[frameIndex % framesInFlight];
       commandList->writeBuffer(cb, &data, size);

       // Need to recreate binding sets!
       auto bindings = device->createBindingSet(
           BindingSetDesc().addItem(BindingSetItem::ConstantBuffer(0, cb)),
           layout);

       frameIndex++;
   }
   ```
   This works but is tedious and error-prone.

   **Solution 2: Volatile Constant Buffers (NVRHI's Solution)**
   ```cpp
   // Just mark it volatile
   bufferDesc.isVolatile = true;
   auto cb = device->createBuffer(bufferDesc);

   // NVRHI handles versioning internally!
   while (rendering) {
       commandList->writeBuffer(cb, &data, size);  // Gets new version
       commandList->draw(...);  // Uses this version
       // Previous versions kept alive until GPU finishes
   }
   ```

   **How NVRHI Implements Volatile Constant Buffers (Backend Details):**

   **DirectX 11:**
   ```cpp
   // Uses D3D11_USAGE_DYNAMIC buffers
   // D3D11 driver handles versioning automatically
   D3D11_BUFFER_DESC desc;
   desc.Usage = D3D11_USAGE_DYNAMIC;
   desc.CPUAccessFlags = D3D11_CPU_ACCESS_WRITE;
   ```

   **DirectX 12:**
   ```cpp
   // NVRHI allocates from internal upload buffer ring
   // Each writeBuffer() gets a fresh allocation
   // Bound as root CBVs (constant buffer views)

   // Conceptually:
   struct UploadBufferChunk {
       void* cpuAddress;
       uint64_t gpuAddress;
       uint64_t size;
   };

   // On writeBuffer():
   auto chunk = uploadRingBuffer.allocate(size);
   memcpy(chunk.cpuAddress, data, size);

   // At draw time, bind via root descriptor:
   commandList->SetGraphicsRootConstantBufferView(
       rootParamIndex, chunk.gpuAddress);
   ```
   - Sub-allocated from large upload heap
   - Linear allocator advances each frame
   - Old allocations freed when GPU finishes frame

   **Vulkan:**
   ```cpp
   // NVRHI creates multi-versioned VkBuffer
   // Uses dynamic offsets at bind time

   // Creation:
   VkBufferCreateInfo bufferInfo;
   bufferInfo.size = constantBufferSize * maxVersions;
   bufferInfo.usage = VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT;

   VkMemoryPropertyFlags memProps =
       VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT |
       VK_MEMORY_PROPERTY_HOST_COHERENT_BIT;

   // At draw time:
   uint32_t dynamicOffset = currentVersion * constantBufferSize;
   vkCmdBindDescriptorSets(
       commandBuffer,
       VK_PIPELINE_BIND_POINT_GRAPHICS,
       pipelineLayout,
       firstSet, descriptorSetCount, descriptorSets,
       1, &dynamicOffset);  // Different offset per frame!
   ```

   **When to Use Volatile vs Static:**

   | Use Case | Buffer Type | Example |
   |----------|-------------|---------|
   | Per-frame global data | Volatile | View matrix, projection matrix, time |
   | Per-draw object data | Volatile | Model matrix, object color, material ID |
   | Per-instance data | Volatile | Instance transforms in instanced rendering |
   | Scene constants | Static | Light positions, light colors |
   | Material properties | Static | Roughness, metallic, texture indices |
   | Config/settings | Static | Shadow map resolution, quality settings |

   **Practical Example: Complete Volatile CB Usage**
   ```cpp
   // Setup (once)
   struct PerFrameConstants {
       float4x4 viewMatrix;
       float4x4 projectionMatrix;
       float4 cameraPosition;
       float time;
   };

   nvrhi::BufferDesc cbDesc;
   cbDesc.byteSize = sizeof(PerFrameConstants);
   cbDesc.isConstantBuffer = true;
   cbDesc.isVolatile = true;  // Updated every frame
   cbDesc.debugName = "Per-Frame Constants";
   auto perFrameCB = device->createBuffer(cbDesc);

   // Create binding set ONCE (buffer handle doesn't change)
   auto bindingSet = device->createBindingSet(
       BindingSetDesc()
           .addItem(BindingSetItem::ConstantBuffer(0, perFrameCB)),
       bindingLayout);

   // Render loop
   while (rendering) {
       commandList->open();

       // Update with this frame's data
       PerFrameConstants constants;
       constants.viewMatrix = camera.getViewMatrix();
       constants.projectionMatrix = camera.getProjectionMatrix();
       constants.cameraPosition = camera.getPosition();
       constants.time = getTime();

       // Write gets new version internally - no stall!
       commandList->writeBuffer(perFrameCB, &constants, sizeof(constants));

       // Use the same binding set - NVRHI tracks the right version
       GraphicsState state;
       state.bindings = { bindingSet };
       commandList->setGraphicsState(state);
       commandList->draw(...);

       commandList->close();
       device->executeCommandList(commandList);
   }
   ```

   **Key Advantages of NVRHI's Volatile Constant Buffers:**
   1. **No manual versioning** - NVRHI handles it transparently
   2. **No GPU stalls** - Always writing to fresh memory
   3. **Binding sets don't change** - Same handle, different internal version
   4. **Backend-optimized** - Uses best strategy per API (dynamic buffers, root CBVs, dynamic offsets)
   5. **Automatic cleanup** - Old versions freed when GPU finishes
   6. **Simple API** - Just `writeBuffer()` and go!

   **Common Mistake to Avoid:**
   ```cpp
   // BAD: Trying to read back from volatile CB
   bufferDesc.isVolatile = true;
   auto cb = device->createBuffer(bufferDesc);

   // This will fail or return stale data!
   commandList->writeBuffer(cb, &data, size);
   // cb->map() // Not supported for volatile buffers

   // GOOD: Use separate staging buffer for readback
   ```

3. **Size limits:**
   - Typical CB max: 64KB (much larger than GL_MAX_UNIFORM_BLOCK_SIZE)
   - For tiny per-draw constants (< 128 bytes): use **Push Constants**

4. **Push Constants** (new concept, no OpenGL equivalent):
   ```cpp
   struct PushConstants {
       float4 color;
       uint objectID;
   };

   // No buffer needed - directly embedded in command stream
   PushConstants pc = { ... };
   commandList->setPushConstants(&pc, sizeof(pc));
   ```
   - Super fast (no indirection)
   - Limited size (128-256 bytes)
   - Great for per-draw variation

### 5. Vertex Buffers and Attributes

**OpenGL:**
```cpp
// Vertex Array Object captures all this state
glGenVertexArrays(1, &vao);
glBindVertexArray(vao);

glBindBuffer(GL_ARRAY_BUFFER, vbo);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, stride, offset);
glEnableVertexAttribArray(0);

// Draw
glBindVertexArray(vao);
glDrawArrays(...);
```

**NVRHI:**

**Step 1: Create Input Layout (part of PSO creation)**
```cpp
nvrhi::VertexAttributeDesc attributes[] = {
    nvrhi::VertexAttributeDesc()
        .setName("POSITION")
        .setFormat(nvrhi::Format::RGB32_FLOAT)
        .setOffset(0)
        .setBufferIndex(0)
        .setElementStride(sizeof(Vertex)),
    nvrhi::VertexAttributeDesc()
        .setName("TEXCOORD")
        .setFormat(nvrhi::Format::RG32_FLOAT)
        .setOffset(12)
        .setBufferIndex(0)
        .setElementStride(sizeof(Vertex))
};

nvrhi::InputLayoutHandle inputLayout = device->createInputLayout(
    attributes, sizeof(attributes)/sizeof(attributes[0]), vs);

// Input layout goes into pipeline description
pipelineDesc.setInputLayout(inputLayout);
```

**Step 2: Create Vertex Buffer**
```cpp
nvrhi::BufferDesc vbDesc;
vbDesc.byteSize = vertexCount * sizeof(Vertex);
vbDesc.isVertexBuffer = true;
vbDesc.debugName = "Vertex Buffer";
nvrhi::BufferHandle vb = device->createBuffer(vbDesc);

// Upload data
commandList->writeBuffer(vb, vertices, vertexDataSize);
```

**Step 3: Bind and Draw**
```cpp
GraphicsState state;
state.pipeline = pipeline;  // Has input layout baked in
state.vertexBuffers = { { vb, 0, 0 } };  // buffer, slot, offset
state.indexBuffer = { ib, nvrhi::Format::R32_UINT, 0 };

commandList->setGraphicsState(state);
commandList->drawIndexed(...);
```

**Key Differences:**

1. **Input Layout part of PSO** - Attribute format baked into pipeline
2. **Named attributes** - Use semantic names like HLSL ("POSITION", not location numbers)
3. **No VAOs** - State set per draw (but NVRHI caches to avoid redundant calls)
4. **Multiple vertex buffers** - Easy to use separate streams

### 6. Textures and Sampling

**OpenGL:**
```cpp
// Create texture
glGenTextures(1, &tex);
glBindTexture(GL_TEXTURE_2D, tex);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, width, height, 0,
             GL_RGBA, GL_UNSIGNED_BYTE, data);

// Sampling state on texture object
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);

// Bind for use
glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_2D, tex);
```

**NVRHI:**

**Textures and Samplers are SEPARATE:**

```cpp
// Create texture
nvrhi::TextureDesc texDesc;
texDesc.width = width;
texDesc.height = height;
texDesc.format = nvrhi::Format::RGBA8_UNORM;
texDesc.isRenderTarget = false;
texDesc.isShaderResource = true;
texDesc.initialState = nvrhi::ResourceStates::ShaderResource;
texDesc.debugName = "My Texture";
nvrhi::TextureHandle tex = device->createTexture(texDesc);

// Upload data
commandList->writeTexture(tex, 0, 0, data, rowPitch);

// Create SEPARATE sampler object
nvrhi::SamplerDesc samplerDesc;
samplerDesc.minFilter = samplerDesc.magFilter = true;  // linear
samplerDesc.addressU = samplerDesc.addressV = nvrhi::SamplerAddressMode::Repeat;
nvrhi::SamplerHandle sampler = device->createSampler(samplerDesc);

// Bind BOTH in binding set
bindingSet
    .addItem(BindingSetItem::Texture_SRV(0, tex))
    .addItem(BindingSetItem::Sampler(0, sampler));
```

**Key Differences:**

1. **Samplers separate from textures** - One sampler can be used with many textures
2. **Explicit resource usage flags:**
   - `isShaderResource` - Can be sampled/read in shaders (SRV)
   - `isRenderTarget` - Can render to it
   - `isUAV` - Can be written to in compute/pixel shaders
   - `isTypeless` - Format reinterpretation allowed

3. **Format specifications:**
   - Must specify exact format (RGBA8_UNORM, RGBA16_FLOAT, etc.)
   - "Typeless" formats allow views with different interpretations

4. **Mipmap generation:**
   ```cpp
   texDesc.mipLevels = CalculateMipLevels(width, height);
   // ... later ...
   commandList->generateMipmaps(tex);
   ```

### 7. Render Targets and Framebuffers

**OpenGL:**
```cpp
// Framebuffer object
glGenFramebuffers(1, &fbo);
glBindFramebuffer(GL_FRAMEBUFFER, fbo);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                       GL_TEXTURE_2D, colorTex, 0);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT,
                       GL_TEXTURE_2D, depthTex, 0);

// Check status
GLenum status = glCheckFramebufferStatus(GL_FRAMEBUFFER);

// Set viewport
glViewport(0, 0, width, height);

// Clear and draw
glClearColor(0, 0, 0, 1);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
glDrawArrays(...);
```

**NVRHI:**

```cpp
// Create render target textures
nvrhi::TextureDesc colorDesc;
colorDesc.width = width;
colorDesc.height = height;
colorDesc.format = nvrhi::Format::RGBA8_UNORM;
colorDesc.isRenderTarget = true;
colorDesc.useClearValue = true;
colorDesc.clearValue = nvrhi::Color(0.0f);
nvrhi::TextureHandle colorTex = device->createTexture(colorDesc);

nvrhi::TextureDesc depthDesc;
depthDesc.format = nvrhi::Format::D24S8;
depthDesc.isRenderTarget = true;  // Yes, even for depth
depthDesc.useClearValue = true;
depthDesc.clearValue = nvrhi::Color(1.0f, 0);  // depth=1, stencil=0
nvrhi::TextureHandle depthTex = device->createTexture(depthDesc);

// Create framebuffer
nvrhi::FramebufferDesc fbDesc;
fbDesc.addColorAttachment(colorTex);
fbDesc.setDepthAttachment(depthTex);
nvrhi::FramebufferHandle fb = device->createFramebuffer(fbDesc);

// Set framebuffer in GraphicsState
GraphicsState state;
state.framebuffer = fb;
state.viewport.addViewportAndScissorRect(
    nvrhi::Viewport(width, height));

commandList->setGraphicsState(state);

// Clear
commandList->clearTextureFloat(colorTex, nvrhi::AllSubresources,
                               nvrhi::Color(0.0f));
commandList->clearDepthStencilTexture(depthTex, nvrhi::AllSubresources,
                                      true, 1.0f, true, 0);

// Draw
commandList->draw(...);
```

**Key Differences:**

1. **Framebuffers are lightweight** - Just references to textures
2. **Viewport part of pipeline state** - Set with framebuffer
3. **Clear values** - Can be specified at texture creation time (optimization)
4. **No bind/unbind** - Set framebuffer in state, draw, done
5. **Multiple render targets** - Easy to add more color attachments

### 8. Command Buffers: The Biggest Change

**OpenGL:**
```cpp
// Commands execute immediately (in the context of the implicit state)
glClear(...);
glUseProgram(program);
glBindVertexArray(vao);
glBindTexture(...);
glDrawArrays(...);
// GPU is already working on it (or driver is batching)
```

**NVRHI:**

```cpp
// Create command list
nvrhi::CommandListHandle cmdList = device->createCommandList();

// RECORD commands (doesn't execute yet)
cmdList->open();

cmdList->clearTextureFloat(colorTex, ...);
cmdList->setGraphicsState(state);
cmdList->draw(...);
// ... more commands ...

cmdList->close();

// NOW execute (submits to GPU)
device->executeCommandList(cmdList);
```

**Key Differences:**

1. **Explicit recording** - Commands go into a buffer, not directly to GPU
2. **open() / close()** - Delineate recording phase
3. **Reusable** - Can record once, execute many times
4. **Multi-threaded** - Multiple threads can record separate command lists
5. **Multiple queues** - Graphics, compute, copy queues (not covered here)

**Benefits:**
- Better multi-threading (record in parallel)
- CPU/GPU parallelism (GPU executes previous frame while CPU records next)
- Explicit control over submission

### 9. Resource States and Barriers (New Concept!)

This is probably the most "foreign" concept coming from OpenGL.

**OpenGL:**
```cpp
// OpenGL handles all synchronization automatically
glBindImageTexture(0, tex, 0, GL_FALSE, 0, GL_READ_WRITE, GL_RGBA8);
glDispatchCompute(...);
glMemoryBarrier(GL_SHADER_IMAGE_ACCESS_BARRIER_BIT);  // Sometimes needed

glBindTexture(GL_TEXTURE_2D, tex);
glDrawArrays(...);
// Driver inserts barriers as needed
```

**NVRHI (with automatic tracking):**

```cpp
// Tell NVRHI to track this texture's state
cmdList->beginTrackingTextureState(tex,
    nvrhi::ResourceStates::ShaderResource);

// Use as compute UAV (NVRHI inserts barrier)
cmdList->setTextureState(tex, nvrhi::ResourceStates::UnorderedAccess);
cmdList->setComputeState(...);
cmdList->dispatch(...);

// Use as pixel shader SRV (NVRHI inserts barrier)
cmdList->setTextureState(tex, nvrhi::ResourceStates::ShaderResource);
cmdList->setGraphicsState(...);
cmdList->draw(...);

// Or: let NVRHI return to initial state automatically
cmdList->commitBarriers();  // Happens automatically at close()
```

**What are Resource States?**

In modern APIs, resources exist in specific "states" that determine how they can be accessed:
- `ShaderResource` - Read-only in shaders (OpenGL: sampled texture)
- `UnorderedAccess` - Read-write in shaders (OpenGL: image)
- `RenderTarget` - Being rendered to
- `CopySource` / `CopyDest` - Being copied
- `VertexBuffer`, `IndexBuffer`, `ConstantBuffer` - Used as such
- etc.

**Barriers** are commands that transition resources between states and ensure:
1. Previous operations finish
2. Caches are flushed
3. Resource layout is transitioned (internal format changes)

**Why does this exist?**
- OpenGL hides this complexity (driver figures it out)
- Modern APIs expose it for performance (manual control)
- NVRHI provides **optional automatic tracking** - best of both worlds

**Alternative: "KeepInitialState" mode**
```cpp
// For textures that go back-and-forth often
texDesc.keepInitialState = true;  // Auto return to initial state
texDesc.initialState = ResourceStates::ShaderResource;

// NVRHI automatically transitions before/after each use
cmdList->setTextureState(tex, ResourceStates::UnorderedAccess);
cmdList->dispatch(...);
// Auto-transitions back to ShaderResource
```

### 10. Compute Shaders

**OpenGL:**
```cpp
glUseProgram(computeProgram);
glBindImageTexture(0, outputTex, 0, GL_FALSE, 0, GL_WRITE_ONLY, GL_RGBA8);
glDispatchCompute(width/16, height/16, 1);
glMemoryBarrier(GL_SHADER_IMAGE_ACCESS_BARRIER_BIT);
```

**NVRHI:**

```cpp
// Create compute pipeline
auto computePipelineDesc = nvrhi::ComputePipelineDesc()
    .setComputeShader(computeShader)
    .addBindingLayout(bindingLayout);
nvrhi::ComputePipelineHandle pipeline =
    device->createComputePipeline(computePipelineDesc);

// Setup binding set with UAV
auto bindingSet = device->createBindingSet(
    BindingSetDesc()
        .addItem(BindingSetItem::Texture_UAV(0, outputTex)),
    bindingLayout);

// Dispatch
ComputeState state;
state.pipeline = pipeline;
state.bindings = { bindingSet };
cmdList->setComputeState(state);
cmdList->dispatch(width/16, height/16, 1);
```

**Key Differences:**
1. **Compute pipelines** - Separate from graphics pipelines
2. **Same binding model** - Layouts and sets work identically
3. **UAV = Unordered Access View** - Read-write resources (images/buffers)
4. **Resource states** - Must transition to UnorderedAccess state

---

## Advanced Features Not in OpenGL

### 1. Ray Tracing

NVRHI supports DXR (DirectX Raytracing) and Vulkan ray tracing:

```cpp
// Build acceleration structure
nvrhi::rt::AccelStructDesc blasDesc;
blasDesc.geometries = { geometryDesc };
nvrhi::rt::AccelStructHandle blas =
    device->createAccelStruct(blasDesc);

cmdList->buildBottomLevelAccelStruct(blas);

// Create ray tracing pipeline
nvrhi::rt::PipelineDesc rtPipelineDesc;
rtPipelineDesc.shaders = { rayGenShader, missShader, hitShader };
// ... more configuration ...

// Dispatch rays
cmdList->setRayTracingState(rtState);
cmdList->dispatchRays(rtDispatchDesc);
```

No OpenGL equivalent - this is entirely new hardware functionality.

### 2. Mesh Shaders

New programmable geometry pipeline:

```cpp
// Amplification Shader + Mesh Shader + Pixel Shader
auto meshletPipeline = nvrhi::MeshletPipelineDesc()
    .setAmplificationShader(asShader)
    .setMeshShader(msShader)
    .setPixelShader(psShader)
    // ... render state ...
```

Replaces vertex/geometry shader stages with more flexible model.

### 3. Variable Rate Shading (VRS)

Control shading rate per region:

```cpp
state.shadingRateState.enabled = true;
state.shadingRateState.shadingRate =
    nvrhi::ShadingRate::Shading2x2;  // 1 shading per 2x2 pixels
```

### 4. Multi-Queue Rendering

Separate queues for graphics, compute, and copy operations that run concurrently.

---

## Memory Management Differences

### OpenGL
```cpp
glGenTextures(1, &tex);
// ... use texture ...
glDeleteTextures(1, &tex);  // Immediate (or driver-deferred)
```

### NVRHI
```cpp
// Reference counting (like COM)
nvrhi::TextureHandle tex = device->createTexture(desc);  // RefCount = 1

// NVRHI internally holds references while in use
cmdList->clearTexture(tex, ...);
device->executeCommandList(cmdList);  // Internal ref kept

// You can release immediately - it won't actually be destroyed
tex = nullptr;  // RefCount decremented, but internal ref keeps it alive

// Must run garbage collection to free finished resources
device->runGarbageCollection();  // Checks GPU, releases finished resources
```

**Key Differences:**
1. **Reference counting** - Like shared_ptr
2. **Deferred destruction** - Resources freed after GPU finishes
3. **Must call runGarbageCollection()** - Usually once per frame
4. **"Fire and forget"** - Can release immediately, NVRHI keeps it alive as needed

---

## Typical Rendering Loop Comparison

### OpenGL
```cpp
while (!quit) {
    // Clear
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    // Update uniforms
    glUniformMatrix4fv(mvpLoc, 1, GL_FALSE, &mvp[0][0]);

    // Draw
    glUseProgram(program);
    glBindVertexArray(vao);
    glBindTexture(GL_TEXTURE_2D, tex);
    glDrawElements(...);

    // Present
    swapBuffers();
}
```

### NVRHI
```cpp
// One-time setup
nvrhi::DeviceHandle device = ...;
nvrhi::CommandListHandle cmdList = device->createCommandList();
nvrhi::GraphicsPipelineHandle pipeline = ...;
nvrhi::BindingLayoutHandle layout = ...;
nvrhi::BufferHandle vertexBuffer = ...;
nvrhi::TextureHandle texture = ...;
nvrhi::SamplerHandle sampler = ...;
nvrhi::FramebufferHandle framebuffer = ...;

// Per-frame resources
nvrhi::BufferHandle constantBuffer = device->createBuffer(cbDesc);
nvrhi::BindingSetHandle bindingSet = device->createBindingSet(
    BindingSetDesc()
        .addItem(BindingSetItem::Texture_SRV(0, texture))
        .addItem(BindingSetItem::Sampler(0, sampler))
        .addItem(BindingSetItem::ConstantBuffer(1, constantBuffer)),
    layout);

while (!quit) {
    // Open command list
    cmdList->open();

    // Update constants
    Constants constants = { .mvp = mvp };
    cmdList->writeBuffer(constantBuffer, &constants, sizeof(constants));

    // Clear
    cmdList->clearTextureFloat(colorTexture, ...);
    cmdList->clearDepthStencilTexture(depthTexture, ...);

    // Setup state
    GraphicsState state;
    state.pipeline = pipeline;
    state.framebuffer = framebuffer;
    state.vertexBuffers = { { vertexBuffer, 0, 0 } };
    state.indexBuffer = { indexBuffer, Format::R32_UINT, 0 };
    state.bindings = { bindingSet };
    state.viewport.addViewportAndScissorRect(viewport);

    cmdList->setGraphicsState(state);

    // Draw
    cmdList->drawIndexed(DrawArguments()
        .setVertexCount(indexCount));

    // Close and execute
    cmdList->close();
    device->executeCommandList(cmdList);

    // Present
    swapChain->present();

    // Cleanup
    device->runGarbageCollection();
}
```

**It's more code, but you get:**
- Better performance (pre-compiled state, efficient descriptors)
- Multi-threading potential (record multiple cmdLists in parallel)
- Explicit control over GPU work
- Cross-platform (same code works on DX12, Vulkan, DX11)

---

## Common Patterns and Idioms

### Pattern 1: Updating Dynamic Data

**OpenGL:**
```cpp
glBindBuffer(GL_UNIFORM_BUFFER, ubo);
glBufferSubData(GL_UNIFORM_BUFFER, 0, sizeof(data), &data);
```

**NVRHI:**
```cpp
// Use volatile constant buffers for per-frame data
bufferDesc.isVolatile = true;  // Important for frequent updates

// Update is just:
cmdList->writeBuffer(buffer, &data, sizeof(data));

// NVRHI handles:
// - Ring buffer allocation (doesn't stall)
// - Versioning (different versions for frames in flight)
// - Upload buffer management (no manual staging)
```

### Pattern 2: Texture Upload

**OpenGL:**
```cpp
glBindTexture(GL_TEXTURE_2D, tex);
glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, width, height,
                GL_RGBA, GL_UNSIGNED_BYTE, pixels);
```

**NVRHI:**
```cpp
// Simple upload
cmdList->writeTexture(texture,
    0,           // array slice
    0,           // mip level
    pixels,      // data
    rowPitch);   // bytes per row

// NVRHI handles:
// - Upload buffer allocation
// - Memory alignment
// - State transitions
```

### Pattern 3: Render to Texture

**OpenGL:**
```cpp
// Create texture
glGenTextures(1, &tex);
glBindTexture(GL_TEXTURE_2D, tex);
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA8, width, height, 0,
             GL_RGBA, GL_UNSIGNED_BYTE, NULL);

// Attach to FBO
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0,
                       GL_TEXTURE_2D, tex, 0);

// Render
glDrawArrays(...);

// Use as texture later
glBindTexture(GL_TEXTURE_2D, tex);
glDrawArrays(...);
```

**NVRHI:**
```cpp
// Create texture with both usages
nvrhi::TextureDesc texDesc;
texDesc.isRenderTarget = true;      // Can render to it
texDesc.isShaderResource = true;    // Can sample from it
texDesc.initialState = ResourceStates::RenderTarget;
nvrhi::TextureHandle tex = device->createTexture(texDesc);

// Create framebuffer
nvrhi::FramebufferHandle fb = device->createFramebuffer(
    FramebufferDesc().addColorAttachment(tex));

// Render to it
cmdList->beginTrackingTextureState(tex, ResourceStates::RenderTarget);
GraphicsState state;
state.framebuffer = fb;
cmdList->setGraphicsState(state);
cmdList->draw(...);

// Transition to shader resource
cmdList->setTextureState(tex, ResourceStates::ShaderResource);

// Use as texture
state.framebuffer = mainFramebuffer;
state.bindings = { bindingSetWithTex };
cmdList->setGraphicsState(state);
cmdList->draw(...);
```

### Pattern 4: Instanced Rendering

**OpenGL:**
```cpp
glDrawElementsInstanced(GL_TRIANGLES, indexCount, GL_UNSIGNED_INT,
                        0, instanceCount);
```

**NVRHI:**
```cpp
cmdList->drawIndexed(DrawArguments()
    .setVertexCount(indexCount)
    .setInstanceCount(instanceCount));
```

Same concept, slightly different API.

---

## Debugging Tips for OpenGL Developers

### 1. Validation Layer
```cpp
nvrhi::DeviceHandle device = ...;
nvrhi::ValidationLayerHandle validation =
    device->createValidationLayer();

// Catches:
// - Incorrect resource states
// - Mismatched binding layouts
// - Out-of-bounds accesses
// - Resource lifetime issues
```

Similar to OpenGL's debug output, but more comprehensive.

### 2. Debug Names
```cpp
textureDesc.debugName = "Shadow Map";
bufferDesc.debugName = "Vertex Buffer: Cube";
pipelineDesc.debugName = "Pipeline: Forward Opaque";
```

Shows up in GPU debuggers (RenderDoc, Nsight Graphics, PIX).

### 3. Common Mistakes

**Mistake 1: Forgetting to call runGarbageCollection()**
```cpp
// BAD: Memory leak, resources never freed
while (!quit) {
    // ... render ...
}

// GOOD: Cleanup per frame
while (!quit) {
    // ... render ...
    device->runGarbageCollection();
}
```

**Mistake 2: Wrong resource states**
```cpp
// BAD: Using texture without proper state
cmdList->setTextureState(tex, ResourceStates::UnorderedAccess);
cmdList->dispatch(...);
// Forgot to transition back!
state.bindings = { setWithTex };  // Expects ShaderResource state
cmdList->draw(...);  // Validation error or wrong rendering

// GOOD: Proper transitions
cmdList->setTextureState(tex, ResourceStates::UnorderedAccess);
cmdList->dispatch(...);
cmdList->setTextureState(tex, ResourceStates::ShaderResource);
cmdList->draw(...);
```

**Mistake 3: Binding layout mismatch**
```cpp
// Layout says: texture at slot 0, buffer at slot 1
auto layout = BindingLayoutDesc()
    .addItem(BindingLayoutItem::Texture_SRV(0))
    .addItem(BindingLayoutItem::ConstantBuffer(1));

// BAD: Wrong slots in binding set
auto set = BindingSetDesc()
    .addItem(BindingSetItem::ConstantBuffer(0, buffer))  // Wrong!
    .addItem(BindingSetItem::Texture_SRV(1, texture));   // Wrong!

// GOOD: Matching slots
auto set = BindingSetDesc()
    .addItem(BindingSetItem::Texture_SRV(0, texture))
    .addItem(BindingSetItem::ConstantBuffer(1, buffer));
```

---

## Performance Considerations

### OpenGL Habits to Unlearn

1. **Don't recreate pipelines** - They're expensive!
   ```cpp
   // BAD: Creating pipeline every frame
   while (!quit) {
       auto pipeline = device->createGraphicsPipeline(desc);
       // ...
   }

   // GOOD: Create once, reuse
   auto pipeline = device->createGraphicsPipeline(desc);
   while (!quit) {
       state.pipeline = pipeline;
       // ...
   }
   ```

2. **Don't recreate binding sets unnecessarily**
   ```cpp
   // BAD: Every frame
   while (!quit) {
       auto set = device->createBindingSet(...);
   }

   // GOOD: Reuse or use ring buffer
   std::vector<BindingSetHandle> sets(framesInFlight);
   // Create once, cycle through them
   ```

3. **Batch state changes**
   ```cpp
   // BAD: Setting state multiple times
   cmdList->setGraphicsState(state1);
   cmdList->draw(...);
   cmdList->setGraphicsState(state2);
   cmdList->draw(...);

   // GOOD: Same state, multiple draws
   cmdList->setGraphicsState(state);
   cmdList->draw(args1);
   cmdList->draw(args2);
   cmdList->draw(args3);
   ```

### New Optimization Opportunities

1. **Multi-threaded command recording**
   ```cpp
   #pragma omp parallel for
   for (int i = 0; i < numThreads; i++) {
       auto cmdList = commandLists[i];
       cmdList->open();
       // Record chunk of work
       cmdList->close();
   }

   // Execute all
   for (auto& cmdList : commandLists) {
       device->executeCommandList(cmdList);
   }
   ```

2. **Async compute**
   ```cpp
   auto computeCmdList = device->createCommandList(
       CommandListParameters().setQueueType(CommandQueue::Compute));

   // Runs in parallel with graphics work
   ```

---

## Summary: Quick Reference

| OpenGL Concept | NVRHI Equivalent | Key Difference |
|----------------|------------------|----------------|
| glUseProgram | GraphicsPipeline | Bundles all state (shaders, blend, depth, etc.) |
| glBindTexture | BindingSet | Pre-filled descriptor sets |
| glUniform* | Constant Buffer | Must use uniform blocks |
| glVertexAttribPointer | InputLayout (in PSO) | Part of pipeline creation |
| Immediate execution | CommandList | Explicit record/execute |
| glGenFramebuffers | Framebuffer | Lightweight, just texture references |
| glMemoryBarrier | Resource States | Automatic tracking available |
| GL_TEXTURE_2D | Texture + Sampler | Separate objects |
| glMapBuffer | writeBuffer | No manual mapping |
| Context | Device | Explicit device object |

## Final Thoughts

NVRHI sits at a sweet spot:
- **More explicit than OpenGL** - You see and control more
- **Less tedious than raw DX12/Vulkan** - Automatic state tracking, lifetime management
- **Modern features** - Ray tracing, mesh shaders, multi-queue
- **Cross-platform** - One codebase for DX11, DX12, Vulkan

The learning curve is real, but the concepts translate once you understand:
1. **PSOs** = Bundle all state together
2. **Binding model** = Declare (layout) then fill (set)
3. **Command lists** = Explicit recording
4. **Resource states** = What can be done with a resource
5. **Explicit lifetime** = You control creation/destruction timing

For more details, see:
- `/home/user/NVRHI/doc/ProgrammingGuide.md` - Comprehensive guide
- `/home/user/NVRHI/doc/Tutorial.md` - Step-by-step tutorial
- `/home/user/NVRHI/include/nvrhi/nvrhi.h` - Full API reference

Good luck with your modern graphics API journey!
