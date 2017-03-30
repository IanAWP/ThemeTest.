---
title: Summary and final comments
date: 2017-03-28 00:03:00
---

That’s the end of this series of posts. Over the course of this series we’ve set up cesium, looked at the terrain tile format, introduced ourselves to the MRR format and API, and produced a reasonably efficient terrain tile generator in C#.  The source is published to github under the MIT license so if you want to make use of it feel free to do so. 

In the end this was a two day hack project with a couple of days’ worth of writing and visualisation thrown in. It’s hard to look at it without seeing the numerous ways that it could be improved, so to that end this post will talk about what a hypothetical person might add to the existing functionality.

## Reprojection, conversion, multiple bands

Currently the only valid input to the C# tile generator is a single banded grid in the WGS84 projection. This represents only a tiny minority of real world data.  Anyone trying to expand support would run into some problems:

### Projections:

Our code currently requires that the input file be in WGS84 projection. There are plenty of projections that are very close to WGS84 for all intents and purposes (NAD83, MGA94) which would still not work with our code.  Why? The decomposition that we use requires a geographic coordinate system – the same as the terrain tileset, ie: we have to be able to convert a request for tile 2/3/12.terrain into a lat/lon based bounding box in WGS84 units, and the bounding box would then need to be converted into grid units in order to query the dataset.  

MRR doesn’t do this conversion for us – if we have a dataset that uses [projected coordinates, as opposed to geographic coordinates](http://resources.esri.com/help/9.3/arcgisengine/dotnet/89b720a5-7339-44b0-8b58-0f5bf2843393.htm), MRR will give us world units according to the source projection.  It is a really bad idea to try and account for this ourselves.  In this case my advice would be to use an API call to convert the source dataset into WGS84 automatically.

### Other Data Formats, Fields and Bands

MRR can be viewed as a container format, it may contain many grids representing different data sources. In a real world file it’s a bit hopeful to assume that there is a single field, with a single band, representing elevation. Adding a field/band selector should be fairly trivial as the API allows you to interrogate the MRR heirarchy.  The only gotcha is that you want to filter to continuous, concrete field/band combos as they are the only ones that could plausibly provide elevation data.

In terms of other Formats, MRR natively supports a fair few formats out of the box, but the exact way it supports them will depend on the utilization.  Testing block reader based performance showed that the random iterator was sometimes faster because, for smaller datasets with fewer zoom levels, calling for a block at non-native zoom triggered a full pyramiding process that generated a pyramid file next to the input. Some experimentation would be required if you wanted to optimize performance in these cases.

### Terrain Tile Features: TIN, Water masks

Finally, something that I would quite like to tackle one day is the [quantized mesh elevation format](https://cesiumjs.org/data-and-assets/terrain/formats/quantized-mesh-1.0.html), which uses a triangular irregular network to store elevation, rather than a heightmap. The quantized mesh format is significantly more complex to implement well, In particular I wonder what an effective method of triangle culling would be, given that we’d like a smooth transition from one zoom level to the next?

Best of luck to anyone who wants to extend the work here!
