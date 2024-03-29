# SDK Reference

## Basic Flow
This chapter explains the basic flow of the SDK through its simplest usage. The details of each process are not explained here. A diagram of the process flow is shown below.

```mermaid
sequenceDiagram
    Application->>KickstartRT: Call CreateTaskContainer()
    activate KickstartRT
    KickstartRT->>Application: Task Container
    deactivate KickstartRT
    activate Application
    Note over Application: Register Geometries Instances<br>and schedule Render Tasks to the task container
    Application->>KickstartRT: Call BuildGPUTask() with the task container
    deactivate Application
    activate KickstartRT
    Note over KickstartRT: Traverse the task container and build GPU commands<br>on the application provided command list.
    KickstartRT->>Application: GPU Task Handle
    deactivate KickstartRT
    activate Application
    Application->>GPU Command Queue : Submit the GPU command list
    deactivate Application
    activate GPU Command Queue
    Note over Application: Keep the GPU task handle provided by the SDK<br>until the corresponded GPU commands has been finished on the GPU
    GPU Command Queue->>GPU Command Queue : Execute command list
    GPU Command Queue->>Application: Telling the finish by a synchronized object (e.g. fence).
    deactivate GPU Command Queue
    activate Application
    Application->>KickstartRT: Return the GPU Task Handle
    deactivate Application
```

After initializing the SDK, the first task in the rendering loop is to create a TaskContainer. This is a structure used to pass various input data to the SDK.
The next step is to register the geometries in the scene to this TaskContainer. The registered geometries are used in the SDK to build the Acceleration Structure (BVH).
In addition, register the task to render reflections and/or GI to the TaskContainer.To render raytraced reflections and GI in KickStartRT, it needs the location of the geometries directly visible from the viewpoint and its surface normal, roughness and various values, as well as the direct light shading results. These should be drawn to the 2D render targets in a conventional deferred render pass, so register the drawn buffers as input data.
Next, the application passes the task container and its command list to the SDK to build the rendering commands on it, which builds the acceleration structure based on the input data, performs ray tracing to draw the output to the specified render targets. 
The application builds a rendering path to composite the reflection and/or GI rendering results output by the command list built by the SDK, to render the final rendering result.
The application executes all the render passes it has built, along with the command list built by the SDK, in its own CommandQueue.
Lastly, when the application is done executing the command list on the GPU, it tells the SDK that the execution is complete.

Once the geometry has been input into the SDK, it will persistently exist in the scene, so once static geometry has been registered, it can be left as is. In subsequent frames, the application only needs to update the geometries whose position has changed, appeared, or disappeared to update the acceleration structure in the SDK..
The direct light rendering result inputs to the SDK is stored in a cache built in world space. Therefore, as multiple frames are rendered from different views, the SDK accumulates the direct light renderings outside of the current view of the GBuffer. This is then sampled by RayTracing to achieve GI and Reflection.

## NameSpace and Class structures
This section describes the overall structure of Namespace, Class, and Structure exposed by the SDK. For detailed information on individual items, please refer to the API Reference.The following diagram illustrates the namespace structure of the SDK.

``` mermaid
flowchart TB

subgraph namespace KickstartRT

subgraph root[namespace D3D12, D3D11 or VK]
ExecuteContext -- Create --> TaskContainer
TaskContainer -- BuildGPUTask --> ExecuteContext 

subgraph BT[namespace BVHTask]
BTask[Task]
BTask -- derived --> DBTask[GeometryTask<br>InstanceTask<br>BVHBuildTask]
end
BT -- Register --> TaskContainer

subgraph RT[namespace RenderTask]
RTask[Task]
RTask -- derived --> DirectLightingInjectionTask
RTask -- derived --> TraceTasks[TraceSpecularTask<br>TraceDiffuseTask<br>TraceAmbientOcclusionTask<br>TraceShadowTask<br>TraceMultiShadowTask]
RTask -- derived --> DenoiseTasks[DenoiseSpecularTask<br>DenoiseDiffuseTask<br>DenoiseSpecularAndDiffuseTask<br>DenoiseDiffuseOcclusionTask<br>DenoiseShadowTask<br>DenoiseMultiShadow]
end
RT -- Register --> TaskContainer

subgraph Log[namespace Log]
l[SetMinSeverity<br>SetCallback<br>SetDefaultMessageProc]
end

subgraph Version[namespace Version]
vv[GetLibraryVersion]
end

end

subgraph RootVersion[namespace Version]
V[Version]
end

subgraph Math[namespace Math]
vecmath[Float2,Float3,Float4<br>Float3x4,Float4x3,Float4x4]
end

end

```

### ExecuteContext
``` mermaid
 classDiagram

class TaskContainer
class GPUTaskHandle
ExecuteContext <--> TaskContainer : [Create,BuildGPUTask]
ExecuteContext <--> GPUTaskHandle : [BuildGPUTask, MarkGPUTaskAsCompleted]

class ExecuteContext {
    +Init() static
    +Destruct() static

    +CreateTaskContainer()
    +CreateDenoisingContextHandle()
    +DestroyDenoisingContextHandle()
    +DestroyAllDenoisingContextHandles()

	+CreateGeometryHandle()
	+CreateGeometryHandles()
	+DestroyGeometryHandle()
	+DestroyGeometryHandles()
	vDestroyAllGeometryHandles()

	+CreateInstanceHandle()
	+CreateInstanceHandles()
	+DestroyInstanceHandle()
	+DestroyInstanceHandles()
	+DestroyAllInstanceHandles()

	+BuildGPUTask()
    +MarkGPUTaskAsCompleted();
	+ReleaseDeviceResourcesImmediately()
}

ExecuteContext <--> GeometryHandle:[Create,Destroy]
ExecuteContext <--> InstanceHandle:[Create,Destroy]
ExecuteContext <--> DenoisingContextHandle:[Create,Destroy]
```

ExecuteContext is an instance of the SDK itself. It is expected to be created one instance by one application. It persists until it is destroyed explicitly, and holds D3D or Vulkan resources inside.
ExecuteContext has the interface for the creation and destruction of GeometryHandle, InsntanceHandle and DenoisingContextHandle, which are necessary for building the Acceleration Structure and for managing intermediate render targets for denoising the ray tracing results. Also, it has the interface for generating Task Container, which is described below. 

### TaskContainer
``` mermaid
 classDiagram

class BVHTask
class RenderTask
class TaskContainer {
    +ScheduleRenderTask()
	+ScheduleRenderTasks()
	+ScheduleBVHTask()
	+ScheduleBVHTasks()
}
class ExecuteContext

BVHTask --> TaskContainer : [Schedule]
RenderTask --> TaskContainer : [Schedule]
TaskContainer --> ExecuteContext : [BuildGPUTask]

```

A TaskContainer is created from an ExecuteContext, and by inputting a BVHTask and a RenderTask (described below) into the TaskContainer, it can represent the processing unit of a command list generated by the SDK. A TaskContainer is destroyed by being handed over to the ExecuteContext as an input for the SDK's command list creation process. Multiple TaskContainers can be created simultaneously, and multiple threads can access different TaskContainers without blocking by mutex.

### Tasks
There are two types of Task: BVHTask::Task and RenderTask::Task. Both of them are base structures, and the actual various tasks are derived from these structures. Tasks are stored in the TaskContainer and eventually passed to the ExecuteContext to generate individual tasks as GPU drawing commands.

#### BVH tasks
``` mermaid
 classDiagram

class TaskContainer

class Task
class GeometryTask {
    +TaskOperation	taskOperation;
	+GeometryHandle	handle;
	+GeometryInput	input;
}
class InstanceTask {
 	+TaskOperation	taskOperation;
	+InstanceHandle	handle;
	+InstanceInput	input;
   
}
class BVHBuildTask {
	+uint32_t maxBlasBuildCount;
	+bool	 buildTLAS;
}
Task <|-- GeometryTask
Task <|-- InstanceTask
Task <|-- BVHBuildTask

Task --> TaskContainer : [Schedule]
```

There are three types of BVHTask. GeometryTask and InstanceTask are structures that represent the input data for the tasks to register and update the geometry information to be placed in the scene. BVHTask is to control the timing of building BLAS and TLAS of acceleration structures. 

##### GeometryTask
Task for constructing and updating the Bottom Level Acceleration Structure with the vertex buffer and index buffer as input.

##### InsntanceTask
This task is used to place the BLAS generated by the GeometryTask above in the Top Level Acceleration Structure and update its position.

##### BVHBuildTask
This task doesn't have any of geometry data, instead, it has parameters to control building BLAS and TLAS in the SDK.

### Render tasks
``` mermaid
 classDiagram

class TaskContainer

class DirectLightingInjectionTask {
	+Viewport			viewport;
	+float				averageWindow;
	+bool				enableFastAccumlation;
	+Math::Float_4x4		clipToViewMatrix;
	+Math::Float_4x4		viewToWorldMatrix;
	+bool				useInlineRT;
	+DepthInput			depth;
	+ShaderResourceTex	directLighting;
}


class Task
Task <|-- DirectLightingInjectionTask
Task <|-- TraceSpecularTask
Task <|-- TraceDiffuseTask
Task <|-- TraceAmbientOcclusionTask
Task <|-- TraceShadowTask
Task <|-- TraceMultiShadowTask

Task --> TaskContainer : [Schedule]
```
``` mermaid
 classDiagram

class TaskContainer

class DenoiseSpecularTask{
	DenoisingTaskCommon		common;
	DenoisingContextHandle	context;
	ShaderResourceTex		inSpecular;
    CombinedAccessTex		inOutSpecular;
}

class Task
Task <|-- DenoiseSpecularTask
Task <|-- DenoiseDiffuseTask
Task <|-- DenoiseSpecularAndDiffuseTask
Task <|-- DenoiseDiffuseOcclusionTask
Task <|-- DenoiseShadowTask
Task <|-- DenoiseMultiShadowTask

Task --> TaskContainer : [Schedule]
```

Render tasks can be divided into the following three types of processing.

#### Direct Lighting Injection task
DirectLightingInjection is a task that writes the result of direct lighting buffer which is a 2D input texture, into the Direct Lighting Cache built by the SDK in world space. By performing this process from various views over multiple frames, the direct lighting rendering results are accumulated in the world space. This is then referenced in the following ray tracing reflection processes.

#### TraceTasks for specular, diffuse reflections, ambient occlusion, and shadows
These are tasks that perform ray tracing in the acceleration structure built by the SDK.
Specular Reflection and Diffuse Reflection tasks use the input GBuffer's surface as the starting points of reflection rays, and perform ray trace processing to sample direct lighting cache to estimate incoming radiance to the surface.
Ambient Occlusion also uses the input GBuffer as the starting points of the AO rays, but does not use the direct lighting cache, and draws AO only by checking the intersection with the acceleration structure. Shadow draws shadow by tracing shadow ray from the GBuffer’s surface to the light source based on the light source information input with the RenderTask.

#### DenoisingTasks for specular, diffuse reflections, ambient occlusion, and shadows
Denoising taks are for denoising the various rendering results output by the ray tracing task above. KickStartRT performs denoising tasks by calling NRD library internally. These tasks need an intermediate render target to perform multi-pass rendering, and it is represented with Denoising Context Handle in the SDK (which is described later).
There are denoising tasks for Specular, Diffuse, Ambient Occlusion and Shadows, and these are used with the corresponding ray tracing tasks.

### Log
Log is a namespace that contains functions to control the output of error and other kind of messages from the SDK, which can be suppressed or retrieved by setting up a callback function and then processed by the application.

### Version
The namespace that stores the version number of the SDK. When initializing, the SDK passes the version number described in the header file as the default argument, and compares it with the version number in the library’s binary, and displays an error or message if it differs and doesn't have compatibility.

### Math
This is a namespace for some vector and matrix types. They are used as input data types for various tasks.

## Handles
There are four types of handles in the SDK, all of which are generated from an Execute Context and persistent unless explicitly destroyed or returned. The application must correctly manage these handles along with the ExecuteContext across multiple frames.

### Geometry Handle
``` mermaid
flowchart TB

ExecuteContext <-- Create,Destroy --> GeometryHandle
GeometryHandle -- Reference --> BVHTasks
BVHTasks -- Schedule --> TaskContainer
TaskContainer -- BuildGPUTask --> ExecuteContext
```
This Handle is used to identify the geometry registered by the application. Once a geometry is registered, the geometry will remain in the SDK until it is explicitly destroyed. Also, if the geometry (vertex position in local coordinates) changes, it needs to be updated. A Geometry Handle is provided as an identifier of a geometry to do these things.

### InstanceHandle
``` mermaid
flowchart TB

ExecuteContext <-- Create,Destroy --> InstanceHandle
InstanceHandle -- Reference --> BVHTasks
GeometryHandle -- Reference --> BVHTasks
BVHTasks -- Schedule --> TaskContainer
TaskContainer -- BuildGPUTask --> ExecuteContext
```
Instances reference a Geometry Handle to place a geometry in world space. Once it is placed, the instance continues to exist in world space until it is explicitly destroyed. It also needs to be updated if its location (4x3 matrix) changes. To do this, an Instance Handle is provided as an identifier for an instance.
Denoising Context Handle
Denoising Context Handle is a handle that holds an intermediate render target needed for denoising the ray tracing result. The resource continues to be held until it is explicitly destroyed. While it is preferable to reuse the intermediate render target as long as the rendering resolution does not change, but it also needs to be destroyed and regenerated if the resolution changes or  the rendering settings change.  To do this, an Denoising Context Handle is provided as an identifier for an intermediate render target and referred by various denoising render tasks.

### GPUTaskHandle
``` mermaid
flowchart TB

TaskContainer -- BuildGPUTask --> ExecuteContext
ExecuteContext -- BuildGPUTask --> GPUTaskHandle
GPUTaskHandle -- MarkGPUTaskAsComplete --> ExecuteContext

```
This handle is passed to the application by the SDK when a command list is built. The SDK does not interact with the GPU's execution queue at all, so the application needs to notify the SDK when the corresponding GPU processing has been completed by returning the handle. The SDK will then know when the GPU processing is complete and can safely release or reuse the resources that have been used by the corresponding command List.
Input Resources
There are a variety of input resources from the application to the SDK. In order for the SDK to process the input resources correctly, the following rules must be observed.
Supported Resource types and formats
Supported resource types and formats are depending on the individual Task. Please refer to the API Reference for details.

## Resource states
All input resources to the SDK has expected resource states. It is shown in the table below.
The only complicated case is Reflection, AO, Shadow drawn by SDK and then denoising them in the same task container. In this case, the same resource is appeared for both input and output of render tasks and resource transitions are performed inside the SDK. The application needs to set the resource state to the expected one for the first use in the scheduled render tasks.
Also, if a resource state transition is made inside the SDK, it will transition to the expected resource state at the end of the command list SDK built. Notice that SDK doesn't know the actual original resource state, so it will be set to the expected resource state. This should be fine as long as the application keeps the SDK's expected resource state.
As for D3D12, the SDK will set AssertResourceState if debug command list interface is available in the debug build, to verify all input resource states are set correctly. In other cases, the SDK does not check the input resource states.
As for Vulkan, besides meeting the expectation of image layout for textures, all resources must satisfy  pipeline barrier to safely consume resources at compute shader stage. 

| Inpput Resources | Expected Resource State in D3D12 | Expected image layout in VK |
| ---- | ---- | ---- |
| BVHTask::VertexBufferInput | NON_PIXEL_SHADER_RESOURCE | N/A |
| BVHTask::IndexBufferInput | NON_PIXEL_SHADER_RESOURCE | N/A |
| RenderTask::ShaderResourceTex | NON_PIXEL_SHADER_RESOURCE | SHADER_READ_ONLY_OPTIMAL |
| RenderTask::UnorderedAccessTex | UNORDERED_ACCESS | GENERAL |
| RenderTask::CombinedAccessTex | UNORDERED_ACCESS | GENERAL |

## Resource life time
All input resources must not be destroyed until the application returns a GPU Task Handle for the command list built with a scheduled task using the resource. Therefore, discarding a resource while using it in the SDK or command list built by the SDK will result in unpredictable behavior.
Internal Resources
This chapter describes the resources that the SDK creates and uses internally. If you simply want to use the SDK, you don't need to read through this chapter and can skip it.

### Descriptor Heap and Constant Buffer
When the SDK builds a command list from a Task Container, it uses a set of a descriptor heap and a constant buffer that the SDK manages internally. These are managed by a class named TaskWorkingSet. When performing ray tracing, the SDK generates a descriptor table based on the number of registered instances. This descriptor table is approximately twice as large as the number of registered instances. Therefore, if a large number of Instances are registered, the descriptor heap may not be able to allocate memory and an error may occur.
The SDK does not dynamically change the size of the descriptor heap. The size of the descriptor heap can be set in the ExecuteContext_InitSettings structure when initializing the SDK. If this value is too large, it will be a waste of memory, and if it is too small, it will cause an error if too many instances are registered. Choose an appropriate value according to the contents.

### Buffers for Index and Vertex Buffers
The index and vertex buffers input from the application are copied to a buffer once in the SDK, along with transformations, index reordering, and other processing. These buffers are allocated on a shared buffer, meaning that one large buffer accommodates multiple geometries. This reduces the frequent VRAM allocation and deallocation.
For static geometries, the buffer is released when its BLAS has been built, but for dynamic geometries (a geometry that performs an update to local vertex position change) is not released even after the BLAS is built. Therefore, static and dynamic geometry are managed in separate shared buffers.
By generating a copy of the vertex buffer, the SDK can tweak the number of BLASs created per BVHBuildTask in order to adjust the BLAS creation load.

### Buffers for Direct Lighting Cache
The Direct Lighting Cache is a buffer for storing the direct lighting result over the surfaces of the instances input to the SDK. Along with the vertex buffer copy process described above, the SDK internally calculates the size of this buffer for each geometry based on the surface area of the polygon and its edge length, and the SDK reads back the calculated size to the CPU side to reserve a buffer for actual direct lighting storage. For these processes, buffers for read back and atomic counter are required. Since it is not memory efficient to allocate many small buffers, these buffers are shared within the SDK.

### Buffers for BLAS, TLAS
Buffers for BLAS construction also use shared buffers, but they are divided into several types depending on their use. Scratch buffer is required for BLAS construction. The buffer for BLAS and scratch buffer are managed in separate shared buffers. In addition, the SDK internally performs BLAS compactions on static geometries. Therefore, there are four types of shared buffers for BLAS.
TLAS, on the other hand, uses dedicated buffers that are not shared at all.
Intermediate render targets for denoising
The intermediate render target for denoising is rather explicitly managed by the application with the Denoising Context Handle, but the actual memory management is done by the SDK.  One point to note is that after calling Register and Destroy in ExecuteContext, the actual creation and destruction of resources are done in the next BuildGPUTask call.

## Threading  
The SDKs for D3D12 and Vulkan do not have worker threads internally. This is to keep the SDK itself simple, and because the implementation of multi-threading is expected by the application. This chapter explains how to use the SDK on multithreading.

### GPUTasks and its completion.
When a command list is generated by BuildGPUTask, a GPU Task Handle is returned to the application at the same time. The application needs to hold this until the generated command list’s execution has been completed on the GPU. Also, when the GPU processing is complete, the application should call MarkGPUTaskAsCompleted to return it to the SDK as soon as possible.
When initializing the SDK, the supportedWorkingsets in the ExecuteContext_InitSettings structure correspond to the maximum number of command lists that can be “in-flight” at the same time. If it is set to a large number, many command lists can be put into the GPU’s execution queue at the same time, but if it is unnecessarily large, it will be wasted. Please set it appropriately.

### Multi threaded implementation model with multiple task containers
Execute Context creates a Task Container and can schedule various tasks in it. Since multiple task containers can be created and each of them has no exclusions, they can be handled efficiently on multi-threads.
However, a BuildGPUTask call of ExecuteContext is blocked by a mutex even if it is called from multiple threads. This is because there is a dependency between the acceleration structure update by the BVH Task and the GBuffers used by the Render Task, so their execution schedule  needs to be clear. (Also, the SDK does not set up mutexes for individual internal resources.)
Also, for the above reasons, the command list generated by the SDK needs to be executed on the GPU in the order in which it was generated.

### Lifetime of handles and BuildGPUTask
Each handle is referenced by a task and it is stored in a task container and is eventually used in a command list. Tasks are stored in a task container while handles are created and destroyed via ExecuteContext. After destroying the handle referenced by the task, if you call BuildGPUTask with the task container which schedules it, an invalid handle will be detected while in the BuildGPUTask.
On the other hand, even if the handle is destroyed after the BuildGPUTask is called, the actual destruction of the resources inside the SDK will be at the beginning of the first BuildGPUTask call after the corresponding GPUTaskHandle is returned to the SDK.
While this is convenient for creating and destroying handles in the ordinal rendering loop, it also causes a problem in cases where large scale release and allocation of handles occur, such as level loading, where the required VRAM is doubled for a short time. In such a case, VRAM can be released immediately by calling ReleaseDeviceResourceImmediately. However, for this process, all SDK built command lists must be completed and all GPUTaskHandle must be returned.

## Mutex and Blocking call conditions in SDK.
The Mutex set by the SDK API is divided as shown in the figure below.
[diagram here]
API Mutex
The following functions completely exclude re-entrants and other calls.
ExecuteContext
Init
Destruct
BuildGPUTask
ReleaseDeviceResourcesImmediately
Context Handle Mutex
This is a mutex that avoids conflicts between handle creation and destruction processes. BuildGPUTask also locks the mutex only at the beginning of the process, but releases it as soon as after the processing list is copied.
ExecuteContext
Create***Handle(s)
Destroy***Handle(s)
DestroyAll***Handle(s)
BuildGPUTask
ReleaseDeviceResourcesImmediately
Task Container Mutex
This is a mutex that is held by each task container to exclude other schedule***task calls on the same task container. Once it has passed as an argument to BuildGPUTask, the mutex will be locked and the task container will eventually be destroyed by the SDK.
TaskContainer
ScheduleBVHTask
ScheduleRenderTask
ExecuteContext
BuildGPUTask

