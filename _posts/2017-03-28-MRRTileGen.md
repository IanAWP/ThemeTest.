---
title: MRR and cesium terrain
date: 2017-03-28 00:10:00
---

# ![main banner]({{ site.url }}/images/image_0.png)


I have some time on my hands at the moment so I felt like it was time to write up one of my old hacks: Using the MIRaster C# API to generate terrain tiles for cesium.js.

Cesium is a web-based GIS written in javascript and webGL.  It does a lot of what the google earth plugin used to do, but the main difference from my perspective is that it’s plugin free, under active development, and released under the apache 2.0 license.  I’ve been following cesium for a while now and am always impressed by the number of features and improvements that make it into the monthly releases.

MapInfo Raster is a relatively new Raster format and API designed to support large, sparse, multiscale rasters. There is currently no GDAL support for MRR, meaning that the existing cesium tile generator cannot generate terrain tiles from input data in this format.

### Cesium Terrain Tiles
One of the neat features that cesium has had for a while is support for terrain tilesets. In fact, AGI host their own terrain server that you can [try out inside the cesium sandbox](https://cesiumjs.org/Cesium/Apps/Sandcastle/gallery/Terrain.html).  If you want to create your own tiles and host them then you can, with a little bit of messing around, utilize the [Cesium Terrain Builder](https://github.com/geo-data/cesium-terrain-builder) to generate the tiles, and the [Cesium Terrain Server](https://github.com/geo-data/cesium-terrain-server) to host them. Your data needs to be able to be read by GDAL and depending on your computing environment you may have to get your hands a little dirty.

There are still a couple of reasons why I wanted to look at creating my own terrain generator.  The main one is that I wanted to be able to generate tiles from native MRR, rather than having to export to a GDAL format.  I am also interested in how cesium could be used as part of a desktop GIS. Terrain tiles for such an application would require the ability to generate tiles on demand, and MRR would be a good fit.

In the following series of posts I’ll go through the process of getting set up in cesium and creating a nice little tile generator app in C#.

[Click here to go straight to the source code](https://github.com/IanAWP/MRRCesiumTiler) or Skip to the post about:

[Getting set up with cesium and producing a comparison dataset]({% post_url 2017-03-28-GettingCesiumUpAndRunning %})

[The cesium terrain tile format. Converting between tile and world coordinates.]({% post_url 2017-03-28-TerrainTileFormat %})

[Creating the initial tile generation code]({% post_url 2017-03-28-RandomIteratorCode %})

[Using the block reader]({% post_url 2017-03-28-BlockIterator %})

[Using overviews and underviews]({% post_url 2017-03-28-OverUnderview %})

[Parallel Processing]({% post_url 2017-03-28-ParallelProcessing %})

[Summary and final comments]({% post_url 2017-03-28-SummaryAndFinalComments %})

