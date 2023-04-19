---
title: "How to Vfx Graph"
date: 2023-04-19T13:50:48+02:00
description: "Unity VFX Graph cheatsheet."
draft: true
---

Introduction:\
Technical details and common

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
## How to rotate particle towards position
## How to spawn rotated particle mesh
## How to rotate spawn position
## How to spawn or move particles along the line

## How to sample graphics buffer.
https://forum.unity.com/threads/unable-to-sample-custom-struct-in-visual-effects-graph.1198300/#post-7661626


https://forum.unity.com/threads/new-feature-direct-link.1137253/
## How to scale object and get object transform
https://forum.unity.com/threads/access-world-scale-of-visual-effects-game-object.925025/#post-8914477
## How to use oldPosition and set it in source location
## How to use GPU events
## How to spawn spawn over distance with interpolation (oldPosition)
## How to custom spawning behavior
https://forum.unity.com/threads/spawn-over-distance-is-causing-gc-spikes.1104196/
## How to kill particle via “event”
## Update vs Output
https://forum.unity.com/threads/vfx-graph-set-color-over-life.1353839/#post-8545457
## How to use particle decals
## Mesh sampling and point cache
https://forum.unity.com/threads/uniform-distribution-with-skinned-mesh-sampling.1188571

