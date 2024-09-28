---
layout: post
title: SerializeReference attribute in Unity
description: Inspecting the useful 'SerializeReference' attribute to serialize objects as reference types instead of value types.
tags:
  - Unity
---
It is incredible that I just learned about this attribute after more than five years of development with the Unity engine.

Recently, I implemented a gameplay feature that required me to store different types of class instances in an array, with these classes sharing their base classes as common. I realized that I cannot do that with the good old '[SerializeField](https://docs.unity3d.com/ScriptReference/SerializeField.html)', and after my research, I learned about the '[SerializeReference](https://docs.unity3d.com/ScriptReference/SerializeReference.html)' attribute.

In this post, I will try to explain the difference between them.

### Core differences
**SerializeField:**
1. Serializes the data directly (by value).
2. Works well with Unity compatible types.
3. Data is usually copied upon instantiating if it is not driven from `UnityEngine.Object`.
4. Can't support polymorphism; will always try to serialize based on base classes.

**SerializeReference:**
1. Allows serialization of plan C# objects.
2. Support serialization of polymorphic types.
3. Stores and serializes objects by reference.

So the following code is totally valid, and you will be able to add different types of classes into a single list by also serializing them in the editor. (Notice how they are the children of the common base class.)
```csharp
// Base class for other effect classes to inherit from.
[System.Serializable]
public class Effect
{
    public virtual void Apply()
    {
	    throw new NotImplementedException();
    }
}  

public class Heal : Effect
{
    public override void Apply()
    {
	    // Entity.Heal(healAmount);
    }
}

public class Poison : Effect
{
    public override void Poison()
    {
	    // Entity.Heal(-healAmount);  
	    // Entity.UpdateMovementSpeed(movementSpeedDecrease);    
    }
}

public class Item : MonoBehaviour
{
    // Using 'Effect' as type but we will be able to add 'Heal' or 'Poison' from inspector.
    // Because we are serializing by reference.
    [SerializeReference] private List<Effect> effects;
}
```

### Conclusion
I am passing through a very strict period in terms of my work schedule, so for this post I wasn't able to do detailed research like the one I did for other posts, but I think I was able to express myself clearly. Thank you for sticking around!
