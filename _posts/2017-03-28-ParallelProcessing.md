---
title: Parallel processing
date: 2017-03-28 00:04:00
---


In the last post we started using data underviews on demand to make the best use of our source material when the input cell size is close to or slightly smaller than the sample interval of our cesium terrain tile. Underviews are pretty fast but they disproportionately effect tiles at the maximum zoom level which, as we have discussed, are about two thirds of our tiles.  In this post we will use the Task parallel Library to bring our performance back up.

Once again not much has to change.  Our tile byte array can’t be shared by multiple threads, so it either needs to be declared inside the loop, or needs to be made thread local.
```c#
ThreadLocal<byte[]> terrainTileLocal = new ThreadLocal<byte[]>(() => new byte[(bytes + 2)]);
```

We need to restrict the parallel processing to within the same zoom level. Lower zooms depend on higher zooms both because of the tile saving optimisation, and because of the child tile byte.  I’ve chosen simply to use:
```c#
Parallel.For(0, dx + 1, (easterlyOffset) => {
```
for each dimension. Finally, the progress reporting needs to change
```c#
System.Progress<int> progress = new Progress<int>((i) => viewModel.AddToProgress(1));
  
  ...
  
  
internal void AddToProgress(int dy)
        {
            System.Threading.Interlocked.Add(ref progress, dy);
            RaisePropertyChanged("Progress");
            SetProgressText();
        }
```
…And bob’s your uncle!

<table>
  <tr>
    <td>Method</td>
    <td>Average Time</td>
  </tr>
  <tr>
    <td>naive</td>
    <td>00:31.9</td>
  </tr>
  <tr>
    <td>write saving</td>
    <td>00:23.9</td>
  </tr>
  <tr>
    <td>tile saving</td>
    <td>00:20.9</td>
  </tr>
  <tr>
    <td>Block @ Max Zoom</td>
    <td>00:12.1</td>
  </tr>
  <tr>
    <td>Parallel</td>
    <td>00:06.1</td>
  </tr>
</table>

So that's that for now.  [In the final post in this series]({% post_url 2017-03-28-SummaryAndFinalComments %}) I'll give a quick summary of what's left undone in this implementation, and introduce the quantized mesh terrain format, which I'm interested in supporting with this project in future.
