---
title: The cesium terrain tile format. 
date: 2017-03-28 00:08:00
---


Cesium maintains their own documentation for the terrain tile format we will be using.  It’s very well laid out and is available [here](https://cesiumjs.org/data-and-assets/terrain/formats/heightmap-1.0.html). I don’t want to recreate their documentation here so I’ll summarize the salient points.

Tiles are uniform, 65x65 grids of unsigned 16-bit elevation data, overlapping at the edges, expressing an offset from -1000m in 0.2m increments. That means elevation values can be expressed in the range of -1000m to ((65535/5)-1000) = 12,107m.

The tile decomposition is done according to the TMS standard. Two root nodes, broken recursively into quadrants, meaning each zoom level has 4x the number of tiles as the previous level. The tiles are identified by their cartesian offset from the southmost, westmost tile.  At the 3rd zoom level, for instance, there are eight tiles stretching from -180° to 180°, and four from -90° to 90°.  So all your tile addresses will be in the range 3/(0-7)/(0-3).

At least, that is how cesium’s tile generator works, and how I’m going to do it.  Remember the layers.json file?  The line:

```JSON
"tiles": ["{z}/{x}/{y}.terrain"]
```

encodes this zoom+cartesian position format into a file system structure. If you change the tile setting, you can encode whatever regular identification pattern you want.

### Tiles and tile boundaries

Some of you might be a little surprised at the specification asking for tiles with dimensions of 65x65 samples. 64x64 would be a more obvious choice.  The 65x65 requirement is because each tile overlaps with another tile on two sides. I think about this as "64 novel samples + 1 overlap sample from the next tile". Realising that the first sample is also an overlap sample (for another tile) will help us now that we come to the tile decomposition phase.

First I’ve created two functions.  One for converting a given lat/long coordinate into tile units.  Another for checking the lat/long bounds of a given tile.  Both are pretty simple, the main points to note are the x zoom is always zoom + 1 because nX = 2 when zoom = 0.
```c#
      public static TileCoordinate LatLonToTile(int zoom, double lat, double lon)
    	{
        	// Calculate the number of tiles across the map, n, using 2^zoom
        	double ny = Math.Pow(2.0, zoom);
        	double nX = Math.Pow(2.0, zoom + 1);

        	// Multiply x and y by n. Round results down to give tilex and tiley.
        	TileCoordinate coord = new TileCoordinate();
        	coord.X = (int)(nX * ((lon + 180) / 360));
        	coord.Y = (int)(ny * ((lat + 90)/180));
      	  return coord;
    	}
 ```

The second method produces a bounds structure with a south-west and a north-east component. Technically we need only the south-west coordinate and the zoom level to determine the bounding box but in practice it’s useful to calculate the whole thing now.

```c#
    	public static LatLonBounds TileBounds(int zoom, int xtile, int ytile) {
        	double nY = Math.Pow(2.0, zoom);
        	double nX = Math.Pow(2.0, zoom + 1);
 
        	LatLonCoordinate coordMin = new LatLonCoordinate();
        	coordMin.Lon = ((xtile / nX) * 360) - 180;
        	coordMin.Lat = ((ytile / nY) * 180) - 90;
        	LatLonCoordinate coordMax = new LatLonCoordinate();
        	coordMax.Lon = coordMin.Lon+((1 / nX) * 360) ;
        	coordMax.Lat = coordMin.Lat + ((1 / nY) * 180);
        	var llb = new LatLonBounds();
        	llb.Min = coordMin;
        	llb.Max = coordMax;
        	return llb;
    	}
```
In order to make our samples overlap we need to find a measurement delta that is 1/64th of the tile size, and take **65 such measurements.**

Here is a quick example of how it would be done with a smaller tile:

![Sample checkerboard]({{ site.url }}/images/image_4.png)

By doing this we ensure that two adjacent tiles will share samples without both tiles actually having to be created first*.

### Child Tiles and Watermask

The final things we need to add to our tiles are a child bit mask and a water mask.  I didn’t get around to doing the watermask here - in later code I’ll default empty samples to the sea level measurement (5000), but if cesium is not willing to interpret that as water then neither am I. To do the watermask properly I suspect you’d need water body data to hit test on, lacking that I’m going to set this byte to 0 (land). Remember that this is a terrain heightmap only, when you overlay imagery from, eg, bing maps, you will still see water wherever it appears in the satellite photography.

The child tile byte is a bitmask that tells cesium which child quads can be retrieved from the server. For our tile generation there is no way of knowing which high zoom tiles are present without reading the data.  All we can do is clip to the data bounds, and take note whether there is anything to read in a given tile using the valid flag in the iterator API ([See the Next Post]({% post_url 2017-03-28-RandomIteratorCode %})). Aside from managing our tile list, we also need something to produce child ids when given a parent:

```c#
public List<string> GetChildTiles(int tx, int ty, int zoom)
    	{
        	var bb = PointToTile.TileBounds(zoom, tx, ty);
        	var s = new List<string>();
        	var v = PointToTile.LatLonToTile(zoom + 1, bb.Min.Lat, bb.Min.Lon);
        	var x = PointToTile.LatLonToTile(zoom + 1, bb.Max.Lat, bb.Max.Lon);
        	s.Add($"{zoom + 1}/{v.X + 0}/{v.Y + 0}");
        	s.Add($"{zoom + 1}/{v.X + 1}/{v.Y + 0}");
        	s.Add($"{zoom + 1}/{v.X + 0}/{v.Y + 1}");
        	s.Add($"{zoom + 1}/{v.X + 1}/{v.Y + 1}");
 
        	return s;
    	}
```
 and some code to create the mask.

```c#
private byte GetChildQuads(int zoom, int tX, int tY, Dictionary<string, string> existingTiles)
    	{
        	byte childQuadSwitch = 0;
        	var ct = GetChildTiles(tX, tY, zoom);
        	if (existingTiles.ContainsKey(ct[0])) childQuadSwitch += 1;
        	if (existingTiles.ContainsKey(ct[1])) childQuadSwitch += 2;
        	if (existingTiles.ContainsKey(ct[2])) childQuadSwitch += 4;
        	if (existingTiles.ContainsKey(ct[3])) childQuadSwitch += 8;
        	// b = 15;
 
        	return childQuadSwitch;
    	}
```

[In the next post]({% post_url 2017-03-28-RandomIteratorCode %}) we will use the RandomIterator to produce our first pass at full tile generator.

***Note**, It isn’t clear from the doco whether Cesium actually wants a measurement from the center of one of these sample boxes rather than corner measurements as I’m providing.  The documentation states "**the first 2 bytes are the height in the northwest corner"** which I take to support my interpretation.
