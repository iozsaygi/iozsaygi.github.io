---
layout: post
title: Leveraging JSON patching to ship seamlessly
description: Taking advantage of JSON patching to reduce frictions in the release pipeline.
tags:
  - C#
---
If you maintain game configurations for each version of a game, you will eventually run into a familiar problem: configuration maintenance becomes increasingly difficult over time.

At first, it feels manageable. Each new version simply introduces a few changes to an existing configuration. But as the game evolves and configurations grow larger, the effort required to maintain them increases rapidly.

The situation becomes even more complicated when multiple feature branches introduce their own configuration changes. Merging these updates across different game versions can quickly turn the workflow into a frustrating and error-prone process.

After running into this bottleneck myself, I started looking for a way to apply changes across multiple configurations without manually editing each one. Ideally, I wanted a solution that would let me define configuration changes once and apply them wherever needed.

During this search, I came across the concept of JSON patch.

## What is JSON patch?
JSON patch is a standardized format used to describe a sequence of operations applied to a JSON document.

Instead of sending or storing an entire JSON file, JSON patch allows you to represent only the changes that should be applied to an existing document.

This concept is commonly used in networking scenarios, where transmitting only the differences between documents can significantly reduce payload size.

A JSON patch itself is also represented as a JSON document, which contains a list of operations that will be applied to a target JSON object.

Here is a simple example:
```csharp
[
  { "op": "replace", "path": "/baz", "value": "boo" },
  { "op": "add", "path": "/hello", "value": ["world"] },
  { "op": "remove", "path": "/foo" }
]
```

Each entry in the patch document defines:
- `op` which is the operation to perform such as _add_, _remove_, or _replace_
- `path` which is the location in the JSON document where the change should occur
- `value` which is the new value to apply _(if required)_

Because these operations target specific paths within a document, a patch can represent complex modifications with very little overhead.

This makes JSON Patch particularly useful when you want to apply consistent changes across multiple configuration files.

## Detailing the implementation in .NET
The implementation involves two separate steps. For each step, you will likely want to rely on a third-party library to handle most of the heavy lifting.

The first step is generating the _differences between two JSON documents_. For this purpose, I've been using [JSON Diff Patch](https://github.com/wbish/jsondiffpatch.net), which makes the process very straightforward.

Below is a simple example:
```csharp
var originalToken = JToken.Parse(existingJson);
var patchedToken = JToken.Parse(json);
var diff = new JsonDiffPatch().Diff(originalToken, patchedToken);

// Check if any changes occurred.
if (diff == null)
{
    // No differences found between the files.
    return;
}

// You can now build your patch operations based on the 'diff' object generated.
```

In this example, both JSON documents are parsed into `JToken` objects. The `JsonDiffPatch` library then compares them and produces a `diff object` that represents the changes between the two documents.

If the result is `null`, it means the two JSON files are identical and no changes need to be applied.

Once we’ve generated the differences between two JSON documents, the next step is to apply those changes to another JSON document. This is where Microsoft’s [JSON Patch](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.jsonpatch?view=aspnetcore-10.0) library comes in handy.

JSON Patch allows us to define a sequence of operations and apply them directly to a target object. Here’s how it works in practice:
```csharp
var patchDoc = new JsonPatchDocument();

// Convert 'diff' into patch document.
foreach (var property in diff.Children<JProperty>())
{
    var path = "/" + property.Name;

    if (property.Value is JArray array && array.Count >= 2)
    {
        // The second element in the array is the new value.
        patchDoc.Replace(path, array[1]);
    }
    else
    {
        // If it's not an array, treat it as an add.
        patchDoc.Add(path, property.Value);
    }
}

// Deserialize the target JSON file. 
var targetConfig = JsonConvert.DeserializeObject<YourTypeToDeserialize>(File.ReadAllText($"{json-name}.json"));

// Apply the patch.
patchDoc.ApplyTo(targetConfig);
```

So far, we’ve focused mostly on the required code, but in a real application, much of this would be handled through the UI. For example, you could build a tree view that lets users select or deselect specific parts of the patch, making the implementation more interactive and detailed.

Also, note that we didn’t include the `remove` operation in our examples. In real-world scenarios, however, you’ll likely want to support removals as well.

Current mental model of the workflow:

```Original Config → Generate Diff → Create Patch → Apply Patch → Updated Config```

## How JSON patching improved my workflow
For a long time, I was performing these operations manually, which became a major bottleneck during release periods.

By implementing JSON patching, I was able to:
* Handle merges in just a few minutes, freeing up time for critical release tasks. 
* Avoid errors that often occur during manual merging and patching.
* Allow other team members to create their own versions of configurations based on their needs, without relying on me.
* More easily demonstrate the differences between versions and releases.

## Conclusion
One fun idea is that you can also leverage patching to make a game moddable. So far, this has been the most exciting feature I’ve explored this year, and it was definitely worth writing about.

Some resources I found useful for both development and writing this post:
* [jsonpatch.com](https://jsonpatch.com/) 
* [Modding: JSON Patching](https://wiki.vintagestory.at/Modding:JSON_Patching) 

This post comes after a long period of inactivity. Hopefully, I can get back up to speed and publish more posts throughout the year.

See you soon!