+++
title = "Putting GLTF into DynamoDB"
date = "2023-07-17"
transparent = true

[taxonomies]
categories = ["DynamoDB", "GLTF"]
tags = ["dynamodb","typescript","gltf","3d"]
+++

<img src="header.png" style="display: none"/>

Since finding out GLTF was just glorified JSON, I've been curious about what storing it in DynamoDB would be like and if it was possible.

## GLTF Transform!
So, GLTF is just a JSON file full of all of our 3D data so we could go through it and read it all ourselves, but [GLTF Transform](https://www.npmjs.com/package/@gltf-transform/core) is a lovely package that makes dealing with it much nicer (Developed by the legendary [Don McCurdy](https://www.donmccurdy.com/))

I've not much more to say about it really, other than it made this project much easier, especially when it came to separating data from the buffers.

## DynamoDB and Toolbox!
A super handy tool that I'm slowly growing to love, and one that I've used in this project, is [DynamoDB Toolbox](https://www.npmjs.com/package/dynamodb-toolbox).
It helps with single table designs (which is what I'm using here 😉), allowing you to define a table schema, giving you beautifully typed interfaces to interact with Dynamo with.

It's best to show you with an example.
Here is the entity definition for a GLTF node in the final code.

```typescript
new Entity({
    name: 'Node',
    attributes: {
        PK: { partitionKey: true, hidden: true, delimiter: '#' },
        SK: { sortKey: true, hidden: true, type: 'string', default: (entity: any) => entity.path.join('#') },

        path: { required: true, type: 'list' },

        type: ['PK', 0, { type: 'string', required: true, hidden: true, default: () => 'Node' }],
        model: ['PK', 1, { type: 'string', required: true }],
        sceneRef: ['PK', 2, { type: 'number', required: true }],

        ref: { type: 'number', required: true },

        name: { type: 'string', required: false },
        children: { type: 'list', required: false },
        meshRef: { type: 'number', required: false },
        rootNodeRefs: { type: 'list', required: true },

        data: { type: 'map', required: true }
    },
    table
} as const)
```

Here, each key maps to a key in our final Dynamo item, but each value determines what we can put in the value for that key. There's obviously `type`, which maps directly to base typescript types, although less helpful when working with maps and lists. We have `required`, which gives our resulting property a little `?` if it's false.

We then get to the `hidden` property. This allows is to hide the property in the resulting type but keep it in the record. This only works if we have a way of toolbox letting toolbox figure out what the value should be, however. That is why for `type`, `model` and `sceneRef` we have, we are defining them with the `['PK', 0, ...` format. It allows us to build a composite key from each of those elements, in the defined positions and put it into the PK value without having to manually join them 🥳.

The `hidden` property is handy in the sort key here too. It allows us to take other attributes in the entity, `path` in this case, to determine its value. The usage here lets us turn our `path` array into another composite key, this time a representation of the scene's node tree structure. Using this path in the sort key like this allows us to query the nodes using `begins_with` to only grab a subset of the model, which is some awesome functionality which is practically free!

## Smooshing them together
We've got the node entity down now. So, let's tackle the others. We have `Mesh`, `Model`, `Scene` and `Material` which are all quite basic, so I won't cover them. They just contain the data they would normally represent but all of their references are altered to map to correct items in the table.

The only other interesting part is the `Accessor` entity. It's in here that we store all of our mesh `vertices`, `normals` and `indices`. For each `accessor` in the model, I'm using GLTFTransform's `getArray` method to get the slice of the buffer, then encoding it in `Base64` and sticking it in the item as a string.

You may realise the problem with this already, and it's quite annoying. The maximum size for a single item in DynamoDB is 400KB, which means if any of our primitives holds an value greater than that, we can't store it. The problem is certainly not helped by the 33% bloat that using `Base64` brings to the table. Regardless, there is a solution, and I didn't implement it.

Meshes are made up of primitives, so if any of your primitives are getting too big, there should be no issue in splitting them up (right !?!). My lack of understanding here is why I've left it hanging. I don't know the implications of splitting the primitives up more. Thankfully though, the scene I was developing with didn't have any massive meshes 🥳.

## Popping a model into the table
With all of our entities being defined in DynamoDB Toolbox, and all of the GLTF elements easily accessible and iterable, writing to the table was as easy as batch writing them:

```typescript
private async commitMeshes (dbEntities: Entities): Promise<void> {
    const batchPuts = this.meshes.map((mesh) => dbEntities.MeshEntity.putBatch(mesh.toItem()))

    await chunkRunner(batchPuts, 25, async (batch) => {
        await dbEntities.Table.batchWrite(batch)
    })
}
```

## Getting it back out
Getting the back out is a little bit more complicated, we do the following.

1. Select a model by name.
2. Select a scene by ref that's part of that model.
3. Do a query on the nodes of that scene (This is where we can select only a subset, explained in a second).
4. Do batch gets on each of the meshes referenced by the nodes.
5. Do some more batch gets for each of the materials and primitives.
6. Change all of the references in the gltf data to match what they are in the new file as node order will change in dynamo (especially when we're only getting a subset).
7. For each of the accessors, decode them from `Base64`, concatenate the buffers, generate `accessor` and `bufferView` objects with correct sizes and re-encode the concatenated buffer back into `Base64`.
8. Take all of the parts and jam them into a gltf object (With typing help from GLTF Transform 🙏).


As mentioned earlier, in the sort key for the `Node` entity, we're storing the tree structure of the scene by using the path like so:

{{image(src="tree.png", alt="A 3D file with visible tree structure", caption="")}}

Now, if in my initial query for the nodes in the table does `begins_with` for `0#1`, only the `torus`, `cylinder` and `suzanne` will be retrieved!

## Implementation limitations

It's important to note that my implementation of this is missing some key aspects of the GLTF spec.

- Animations are a big one, and I skipped them due to lack of interest. I see no reason why they couldn't be added though.
As far as I know, animations are stored in buffers in the same way our `position` and `index` data are already stored for our primitives.

- Textures are another chunk I've missed out, and I'm not sure how to handle them. They're stored in buffers too, but images are usually pretty big.
Bigger than the 400KB item limit imposed by Dynamo.
So maybe S3 is to come to the rescue for this.
Obviously not as quick to interact with as Dynamo, but it can handle whatever you throw at it, and it's much more convenient for allowing image modification.

- Draco is a no go. I don't know how Draco works, and getting this working without Draco was confusing enough. I have doubts over whether Draco could even be possible but would like to have a look into it.

## Future 

Besides trying to add what I just mentioned above, I'd like to run through some stuff that would be fun to explore with this.

- Keeping a scene stored in the database like this means if we wanted to, we could access all of the node and material data, without having to download the meshes and their primitives.
This would allow us to arrange a scene, via its nodes, and set the materials on the meshes (still, without downloading them, all we need is to update the mesh ref!).
And if we were to get the bounding boxes for each of the meshes (Which might be available on the min/max of each primitive), we could represent each of the nodes as cubes generated on the client.

- Realtime database synchronisation should be possible without much effort. If I was to rotate an object for example, that would be a few bytes of data being set in a Dynamo update.
Combine that with some websockets syncing, (which would technically be possible without this database system) and you're onto a winner.

- Huge 3D files could be fun to have a go of.

## Bad ERD

I made an ERD to try and help explain how this ties together. Idk if it helps 😅.

{{image(src="GltfERD.png", alt="An ERD representing the structure of the table", caption="I wanted an ERD but I hate making them, so this is all you're getting")}}

---

Only recently started listening to some more Poppy, she's hella weird. Here's a couple of her songs.
{{spotify(src="https://open.spotify.com/embed/track/6KwSYJxD2zqROxBB9mUQ20?utm_source=generator")}}
{{spotify(src="https://open.spotify.com/embed/track/7Bc59U2nhCp608JlIEMEGl?utm_source=generator")}}
