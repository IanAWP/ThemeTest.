---
title: Using resolution level
date: 2017-03-28 00:05:00
---


The MRR format has another trick up its sleeve that ought to make our terrain sampling a little easier.  MRR is pyramided data format, meaning that data is stored at multiple resolutions with progressive zoom levels. It’s also possible to generate underviews of the data by asking for a block at a zoom level higher than the base layer. These are two really useful features for creating cesium tiles because it helps solve two of the problems we’ve had up until now:

* Aliasing when using the nearest neighbour interp method for terrain samples at close to the source cell size.

* Allocation limits when retrieving blocks at base zoom level.

Aliasing is always going to be present to some degree when using nearest neighbour.  Cesium tiles have 65x65 samples, and at the base level, over the same area, we may only have 50x50 cells in our source data.  

![Alias]({{ site.url }}/images/image_8.png)

Aliasing occurs when two of our cesium samples resolve to the same grid cell. In practice that means we have to be conservative when picking our max zoom level lest we get a Minecraft like stair-step profile at our closest zoom.  We could solve this by moving to bilinear or bicubic interpolation, but it will be almost as good for our purposes to use the built in interpolation in the MRR API to get an underview with a smaller virtual cell size instead. (Note that this approach only makes sense if the underview is generated with something other than nearest neighbour!)

![Stairstep]({{ site.url }}/images/image_9.png)

As for the allocation problems, using overviews instead of the base resolution data will mean we can use the block iterator at all zoom levels without excessive memory overhead.

### Working with overviews

This is another one of the less than obvious parts of the API which I had to experiment with. Long story short, the resolution is a property of the iterator, while the cell size, bounding box etc are all part of the dataset.  This means that when you open an iterator at a different zoom level from the base data, you’ll have to remember to calculate everything in zoom units.

What does that mean? See what happens when I use the same calculated extents, and column-and-row-spans at different resolutions:
```c#
//-1= underview
for (int zoomLevel = -1; zoom < 2; zoom++) {
    GetGridBlock(westmost, southmost, width, width, zoomLevel, dataset, out actualData, out actualValid);
}
```
![view Levels]({{ site.url }}/images/image_10.png)

The block iterator returns the same number of values but the extents are different.  Essentially the cell size is now adjusted by a factor of two.

 We don’t have to make many modifications to incorporate underview/overview scaling in our code.  The main requirements are:

* Identify the appropriate resolution level to query

* Calculate the virtual cell size

* Scale the block query appropriately

The rule of thumb is that if the ratio between sample size and cell size is < 1.0 we will have nasty aliasing, so we should go to a higher resolution.  When the ratio is >2.0 we can halve the resolution without having nasty aliasing.
```c#
//get the distance between samples
 var dx = (boundingBox.Max.lon - boundingBox.Min.lon) / 64;
 var dy = (boundingBox.Max.lat - boundingBox.Min.lat) / 64;
 var cellSize = dataset.Info.FieldInfo[0].CellSizeX.Decimal;
 var rat = dx / cellSize;
 //0 corresponds to the base level dataset
 int resolution = 0;
 //when sample size is less than cell size use an underview
 while (rat < 1.0) {
     resolution--;
     cellSize = cellSize / 2.0;
     rat = rat * 2.0;
 }

 //when sample size is greater than cell size use an overview
 while (rat > 2.0) {
     resolution++;
     cellSize = cellSize * 2.0;
     rat = rat / 2.0;
 }

//We've added an optional resolution parameter to our cell position calculation
 int westmost, southmost;
 GetCellPosition(boundingBox.Min.lon, boundingBox.Min.lat, dataset, out westmost, out southmost, resolution);
```
Inside the cell position method scale the calculation by the resolution level:

```c#
double multiplyer = Math.Pow(2, resolution);
 dCellSizeX = dCellSizeX * multiplyer;
 dCellSizeY = dCellSizeY * multiplyer;
```
The results are now pretty good! No jagged edges, and a much smoother, more realistic looking tile.

![Smoothing comparison]({{ site.url }}/images/image_11.png)

Unfortunately we’ve lost a little speed here as a result of utilizing the underview zoom level.  [In the next post we will use the task parallel library to try and get that time back.]({% post_url 2017-03-28-ParallelProcessing %})
