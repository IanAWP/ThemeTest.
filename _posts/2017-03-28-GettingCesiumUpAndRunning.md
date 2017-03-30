---
title: Getting cesium up and running
date: 2017-03-28 00:09:00
---


The cesium terrain builder/terrain server combo I mentioned in my first post didn’t work perfectly for me on a windows environment. The terrain builder docker image was easy enough to get running, but right now it seems impossible to mount folders from my local (win7) machine into the image (you might have better luck with the win10 version of docker, which is the one they actually want you to use). 

I followed the [clever but torturous workaround described here](http://blog.mastermaps.com/2014/10/3d-terrains-with-cesium.html) to at least get something to compare which allowed me to get an initial version of my Seattle MRR file into cesium. Following the the guidelines for the terrain builder, I reprojected my source raster in to WGS84, and then converted the file into one of the [supported GDAL raster formats](http://www.gdal.org/formats_list.html) 

For the Seattle elevation dataset the tile generator produced 11 zoom levels, and at level 11 there were 28 ‘X’ segments and 20 ‘Y’ segments. The 1.4:1 ratio we see in the output dataset gels nicely with what we know about our input file which has a raster size of 2048/1536 or about 1.3:1

Like pretty much everyone who’s tried this I eventually found that you needed a layer.json file that exposed the tileset – the thing you might have to change here is the `{z}/{x}/{y}.terrain` path template – the one below matches the output from the docker image.
```JSON
{

  "tilejson": "2.1.0",

  "format": "heightmap-1.0",

  "version": "1.0.0",

  "scheme": "tms",

  "tiles": ["{z}/{x}/{y}.terrain"]

}
```
And because my data doesn’t cross the prime meridian I also had to put a blank tile in position 0,1,0.  The reason for that may not be immediately obvious, so it’s a good time for a quick digression into the tile format.

From the [documentation](https://cesiumjs.org/data-and-assets/terrain/formats/heightmap-1.0.html):

![Root Tiles]({{ site.url }}/images/image_1.png)

And

![Bitmasks]({{ site.url }}/images/image_2.png)

So there are two top level tiles, and cesium determines which tiles are present using a bitmask at the end of the heightmap. Given that there is no mechanism for telling cesium which root tile holds the data it follows that:

* Cesium will always request both root level tiles even if the dataset only contains information for one of them

* All other tiles must have a parent zoom or they will never be loaded

Now that we have everything we need we can run cesium with our exported tileset:

![Borked initial terrain]({{ site.url }}/images/image_3.png)

The resulting image clearly has some bad artifacting at the data boundary, but that is OK because we are only using it for comparison.   [In the next post]({% post_url 2017-03-28-TerrainTileFormat %}) I’m going to go into some more detail about the terrain tile format, and how we will be attempting to align our MRR dataset with the tile boundaries required by cesium.
