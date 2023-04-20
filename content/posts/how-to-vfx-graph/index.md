---
title: "How to Vfx Graph"
date: 2023-04-19T13:50:48+02:00
description: "Unity VFX Graph cheatsheet."
draft: true
---

Introduction:\
Technical details and common
https://docs.unity3d.com/Packages/com.unity.visualeffectgraph@14.0/manual/index.html

## How to enable experimental blocks
Some nodes and features of vfx graph are hidden by default. If you can't find some nodes that should be available in your Unity editor you need to enable them in preferences.\
Navigate to `Edit > Preferences > Visual Effects` and turn on "*Experimental Operators/Blocks*".
{{% figure src="experimental-blocks.png" %}}

## Context and block inspector
You can access all available properties of context and operator/block nodes by selecteing them in the graph window. You will find them in standard Inspector window.
{{% figure src="inspector.png" %}}

## How to pass particle data to shader
You may want want to customize rendering of particles with custom shader that renders particles based on their data. 
To do that, expose the needed shader properties - they will appear in the output context. Then get the particle data with the `Get Attribute` block (location must be set to *current*) and connect to exposed property.
{{% figure src="pass-data-to-shader.png" caption="Particle color and size pased to the shader." %}}

## How to use custom attributes
Using nodes `Get CustomAttribute` and `Set CustomAttribute` allows to use custom particle attributes.\
You can set name and type of attribute in the [inspector](#context-and-block-inspector).\
It is important to set matching types of attribute in both getter and setter.
{{% figure src="custom-attribute.png" caption="Custom attribute `secondaryColor` of type Vector3." %}}

## How to send event with attributes
You can initialize particles with data sent via event using method `SendEvent(eventName, eventAttribute)`.
To pass the data you need to create instance of `VFXEventAttribute`, fill it with data you want and pass with the event. Following script sends `size`, `velocity`, and custom attrribute `secondaryColor` to visual effect instance.

```csharp
using UnityEngine;
using UnityEngine.VFX;

public class SendEventExample : MonoBehaviour
{
    public VisualEffect vfx;
    public Color secondaryColor;
    public Vector3 velocity;
    private VFXEventAttribute eventAttribute;

    private void Start()
    {
        // Create vfx event attribute object
        // There is no need to recreate this object every frame
        eventAttribute = vfx.CreateVFXEventAttribute(); 
    }

    // Call this method to the send event
    private void PlayVFX()
    {
        // Set event data
        eventAttribute.SetFloat("size", Random.Range(0f, 1f));
        eventAttribute.SetVector3("velocity", velocity);
        // Custom attribute: secondaryColor
        eventAttribute.SetVector3("secondaryColor", new Vector3(secondaryColor.r, secondaryColor.g, secondaryColor.b));
        
        // Data is copied from eventAttribute, so this object can be used again
        vfx.SendEvent("OnPlay", eventAttribute);
    }
}
```
You need to capture the data inside the graph using **Inherit** attribute block. You can inherit event data only inside Initialize context.
In case of custom attributes there is no dedicated Inherit block, instead use `Get CustomAttribute` node with location set to "source".
Data inheritance is essentially getting the attribute from source location.

{{% figure src="event-data.png" %}}

## How to use Direct Link feature
Calling `SendEvent()` on systems with spawn context is not cumulative - in case the same event is triggered multiple times per frame (for single instance) it overrides previous one and only the last one counts.
Direct link gives better control over spawn events and allows to bypass event limitation per frame.
To use this feature you need to connect Event context directly to Initialize particle context. By default it will spawn single particle, but you can specify number of generated particles in the [script](#how-to-send-event-with-attributes) via `VFXEventAttribute` by setting `spawnCount` attribute.

{{% figure src="direct-link.png" %}}

```csharp
// Important: Use SetFloat to set spawnCount
eventAttribute.SetFloat("spawnCount", 2f);
vfx.SendEvent("OnPlay", eventAttribute);
```

> *Additional resources: https://forum.unity.com/threads/new-feature-direct-link.1137253/*

## How to spawn or move particles along the line/shape
Below you can find two simple examples of spawning particles on line and moving them along the line.

{{% figure src="spawn-on-line.png" caption="Particles evenly distributed between two points." %}}
{{% figure src="move-along-line.png" caption="Particles moving along the line." %}}
{{% figure src="particles-line.gif" %}}
There are other various useful nodes to spawn particles:
- Line (block)
- Circle (block)
- Sphere (block)
- Torus (block)
- Cone (block)
- Sample Bezier (operator)
- Sequential 3D (block, operator)

## Update context vs Output context
These two contexts may look similar, but adding blocks to them can produce completely different results.\
Update context is executed as compute shader and operates directly on particle data, meanwhile Output context is part of vertex shader and any changes to particle data are discarded.
It means that Output context changes last only until the end of frame, but because modified data is not written back to memory this operation might be cheaper in case of simple particles, but more costful when rendering mesh particles with higher vertex count. Another difference is that Update context can have more than one Output context connected and it might be worth to calculate something once instead of repeating the same operation twice or more. Changing position attribute in the Output Context may mess up particle sorting and particle strip Orientation.[^1]
[^1]: [Unity forum - Update vs Output](https://forum.unity.com/threads/vfx-graph-set-color-over-life.1353839/#post-8545457)

## How to sample graphics buffer
Graphics buffer is one of the most useful tools, it allows to initialize particles with concrete data or update them in specific way. You must allocate memory before graphics buffer can be used, and after that its not possible to change its size. If you had to change the size you would need to dispose the old buffer and allocate new one, however allocation is not free operation, so you should avoid this when possible.
Following example script takes the list of points and sends them through graphics buffer to visual effect:

```csharp
using UnityEngine;
using UnityEngine.VFX;
using UnityEngine.VFX.Utility;
using System.Collections.Generic;

public class SetGraphicBuffer : MonoBehaviour
{
    private const int BufferStride = 12; // 12 Bytes for a Vector3 (4,4,4)

    [SerializeField] private int bufferInitialCapacity = 8;
    [SerializeField] private VisualEffect visualEffect;
    // Special helper class
    [SerializeField] private ExposedProperty bufferProperty = "SpawnPoints";
    [SerializeField] private ExposedProperty bufferCountProperty = "SpawnPointsCount";
    public List<Vector3> spawnPoints = new List<Vector3>();
    private GraphicsBuffer graphicsBuffer;

    void Awake()
    {
        // Initial allocation of graphics buffer
        EnsureBufferCapacity(ref graphicsBuffer, bufferInitialCapacity, BufferStride, visualEffect, bufferProperty);
    }

    void LateUpdate()
    {
        // Set Buffer data, but before that ensure there is enough capacity
        EnsureBufferCapacity(ref graphicsBuffer, spawnPoints.Count, BufferStride, visualEffect, bufferProperty);
        graphicsBuffer.SetData(spawnPoints);
        // Update current number of elements
        visualEffect.SetInt(bufferCountProperty, spawnPoints.Count);
    }

    void OnDestroy()
    {
        ReleaseBuffer(ref graphicsBuffer);
    }

    private void EnsureBufferCapacity(ref GraphicsBuffer buffer, int capacity, int stride, VisualEffect vfx, int vfxBufferProperty)
    {
        // Reallocate new buffer only when buffer is null or capacity is not sufficient
        if (buffer == null || buffer.count < capacity)
        {
            // Buffer memory must be released
            buffer?.Release();
            // Vfx Graph uses structured buffer
            buffer = new GraphicsBuffer(GraphicsBuffer.Target.Structured, capacity, stride);
            // Update buffer referenece
            vfx.SetGraphicsBuffer(vfxBufferProperty, buffer);
        }
    }

    private void ReleaseBuffer(ref GraphicsBuffer buffer)
    {
        // Buffer memory must be released
        buffer?.Release();
        buffer = null;
    }
}
```
The important thing is Buffer stride - it's size (in bytes) of one element in the buffer. In this example we pass buffer of Vector3, so we need 12 bytes, 4 for each component.
Inside the graph you need to select matching type of data used by the buffer. You can do it by clicking the small cog icon in the right top corner of the `SampleBuffer` node.

{{% figure src="buffer-type.png" %}}

Following graph samples the buffer using `particle id` and spawns particles between points provided by buffer. Because we do not shrink the buffer length, we must wrap index by number of current elements *(index % count)* in case we removed any spawn point.

{{% figure src="gbuffer.png" %}}

If you want to use custom struct as buffer elements you can mark it with attribute `[VFXType(VFXTypeAttribute.Usage.GraphicsBuffer)]` and get stride size with `System.Runtime.InteropServices.Marshal.SizeOf(typeof(YourStruct));` - more information [here](https://forum.unity.com/threads/unable-to-sample-custom-struct-in-visual-effects-graph.1198300/#post-7661626).

{{% figure src="buffer-points.gif" caption="Adding and removing spawn points" %}}

## How to rotate particle towards position or direction
Generally `Orient` block is enough to rotate particles towards camera, direction, position or along velocity, however particles will be 'biased' towards camera and sometimes this is not desired behaviour - especially for mesh particles. To fix this you need to specify new axes for rendering, you need two directions: the forward vector and "up" (or left) direction. The third vector can be calculated with cross product. The following graph rotates particles along velocity, or towards target direction (when result of substraction is connected to the safe normalize instead).

{{% figure src="rotate-particle.png" %}}

{{% figure src="particles-along-velocity.gif" caption="Along velocity" %}}
{{% figure src="particles-towards-direction.gif" caption="Towards target position" %}}

https://forum.unity.com/threads/rotate-particles-towards-direction-from-shape.1352003/

## How to spawn rotated particle mesh



## How to scale object and get object transform
https://forum.unity.com/threads/access-world-scale-of-visual-effects-game-object.925025/#post-8914477
## How to custom spawning behavior
## How to use oldPosition and set it in source location
## How to spawn spawn over distance with interpolation (oldPosition)
## How to use GPU events
https://forum.unity.com/threads/spawn-over-distance-is-causing-gc-spikes.1104196/
## How to kill particle via “event”
## How to use particle decals
## Mesh sampling and point cache
https://forum.unity.com/threads/uniform-distribution-with-skinned-mesh-sampling.1188571

