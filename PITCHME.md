---

### Filtering with PDAL
#### GRiD ConX 2017, 14 November 2017
Bradley J Chambers, Radiant Solutions

---

### Overview

- Getting Started
- Python Examples
- Filter Stage Roundup
- Pipeline Examples

---

### Getting Started

+++

### How to pull/start Alpine container

```console
docker pull pdal/pdal:[tag]
```

where `tag` is [master|1.6|1.5|1.4]-[alpine|ubuntu] (or just `latest`)

+++

```console
docker pull chambbj/grid-conx
docker run -it --rm chambbj/grid-conx /bin/sh
```

+++

```bash
pdal info /data/isprs/samp11-utm.laz
```

+++

```console
pdal translate /data/isprs/samp11-utm.laz /data/isprs/samp11-utm.bpf
```

+++

```json
{
  "pipeline":[
    "input.las",
    {
      "type":"filters.whatever",
      "some":"options"
    },
    "output.laz"
  ]
}
```
@[3](Inferred reader)
@[4-7](Filter with options)
@[8](Inferred writer)

+++

```bash
pdal pipeline pipeline.json
```

+++

```bash
pdal pipeline pipeline.json --readers.las.filename=input.las --writers.las.filename=output.las
```
@[1](Override options from the CLI)

+++

```json
{
  "pipeline":[
    {
      "type":"filters.whatever",
      "some":"options"
    }
  ]
}
```

+++

```bash
pdal translate input.las output.las --json pipeline.json
```

---

### Python Package

* The PDAL Python [package](https://pypi.python.org/pypi/PDAL) can be installed via [pip](https://pip.pypa.io/en/stable/).

  ```
  pip install pdal
  ```

* Once installed, simply

  ```python
  import pdal
  ```

---

### Python Examples

Note:
The remainder of the presentation will present examples in the context of the PDAL Python package (though CLI samples will be provided as well).

+++

### Creating a Pipeline

```python
>>> json = u'''
... {
...   "pipeline":[
...     "./data/isprs/samp11-utm.laz"
...   ]
... }'''


>>> p = pdal.Pipeline(json)
```
@[1-6](Define the pipeline JSON)
@[9](Create the pipeline)

+++

### Validating & Executing the Pipeline

```python
>>> print('Is pipeline valid? %s' % p.validate())
Is pipeline valid? True


>>> print('Pipeline processed %d points.' % p.execute())
Pipeline processed 38010 points.


>>> arr = p.arrays[0]
>>> print('Pipeline contains %d array(s).' % (len(p.arrays)))
Pipeline contains 1 array(s).
```
@[1-2](Check for a valid pipeline)
@[5-6](Execute the pipeline)
@[9-11](Check how many `ndarrays` were returned)

+++

### Use the `ndarray`

#### Print the first point record

```python
>>> print(arr[0])
```

```bash
(512743.63, 5403547.33, 308.68, 0, 1, 1, 0, 0, 2, 0.0, 0, 0)
```

+++

#### Print the first 10 X values

```python
>>> print(arr['X'][:10])
```

```bash
[ 512743.63  512743.62  512743.61  512743.6   512743.6   512741.5   512741.5
  512741.49  512741.48  512741.47]
```

+++

#### Print the mean of all Z values

```python
>>> print(arr['Z'].mean())
```

```bash
356.17143357
```

+++

### Or Pandas!

```python
>>> import pandas as pd
>>> samp11 = pd.DataFrame(arr, columns=['X','Y','Z'])
>>> samp11.head()
```

```bash
           X           Y       Z
0  512743.63  5403547.33  308.68
1  512743.62  5403547.33  308.70
2  512743.61  5403547.33  308.72
3  512743.60  5403547.34  308.68
4  512743.60  5403547.33  308.73
```

+++

```python
>>> samp11.describe()
```

```bash
                   X             Y             Z
count   38010.000000  3.801000e+04  38010.000000
mean   512767.010570  5.403708e+06    356.171434
std        38.570375  8.587360e+01     29.212680
min    512700.870000  5.403547e+06    295.250000
25%    512733.530000  5.403645e+06    329.060000
50%    512766.940000  5.403705e+06    356.865000
75%    512799.900000  5.403790e+06    385.860000
max    512834.760000  5.403850e+06    404.080000
```

+++

### Analyze

```python
>>> import seaborn as sns
>>> sns.kdeplot(samp11['Z'], cut=0, shade=True, vertical=True);
```

![Z KDE](figures/kde-z.png)

+++

### Searching Near a Point

#### Find the median point

```python
>>> med = samp11.median()
>>> print(med)
```

```bash
X     512766.940
Y    5403705.460
Z        356.865
dtype: float64
```

+++

#### Print the distance to the three nearest neighbors

```python
>>> from scipy import spatial
>>> tree = spatial.cKDTree(samp11)
>>> dists, idx = tree.query(med, k=3)
>>> print(dists)
```

```bash
[ 0.6213091   1.37645378  1.51757207]
```

+++

#### Print the point records of the three nearest neighbors

```python
>>> samp11.iloc[idx]
```

```bash
               X           Y       Z
31897  512767.16  5403706.02  357.02
31881  512767.93  5403706.29  356.39
31972  512765.75  5403706.19  356.27
```

---

### Filter Stage Roundup

+++

But first...

+++

### DimRange

* A [DimRange](https://www.pdal.io/stages/filters.range.html#ranges) is a
  * named dimension, and
  * range of values.
* Bounds can be inclusive (`[]`) or exclusive (`()`).
* Ranges can be negated (`!`).

+++

```json
{
  "pipeline":[{
    "type":"filters.assign",
    "assignment":"Classification[:]=0"
  }, {
    "type":"filters.assign",
    "assignment":"Classification[2:4]=0"
  }]
}
```
@[3-4](Reassign all classification values to 0)
@[6-7](Reassign ground (2), low (3) and medium (4) vegetation values to created, never classified (0))

+++

```json
{
  "pipeline":[{
    "type":"filters.range",
    "limits":"Z[10:)"
  }, {
    "type":"filters.range",
    "limits":"Classification[2:2]"
  }]
}
```
@[3-4](Select all points with Z greater than or equal to 10)
@[6-7](Select all points with classification of 2)

+++

### Ignoring a `DimRange`

- Available to `filters.pmf` and `filters.smrf`
- Eliminates the need to completely remove points (e.g., noise)
- Instead, points are ignored

+++

```json
{
  "pipeline":[
    {
      "type":"filters.range",
      "limits":"Classification![7:7]"
    },
    {
      "type":"filters.smrf"
    }
  ]
}
```
@[1-11](Noise points are removed!)

+++

```json
{
  "pipeline":[
    {
      "type":"filters.smrf",
      "ignore":"Classification[7:7]"
    }
  ]
}
```
@[1-8](Noise points are left intact, just ignored.)

+++

### Filter "categories"

- create/alter dimensions (other than XYZ)
- change point order
- move points
- cull points
- create points
- create new PointViews
- join PointViews
- create metadata
- create meshes
- embed other languages

+++

### Filters that create/alter dimensions (other than XYZ)

- [ApproximateCoplanar](https://www.pdal.io/stages/filters.approximatecoplanar.html)
- [Assign](https://www.pdal.io/stages/filters.assign.html)
- [Cluster](https://www.pdal.io/stages/filters.cluster.html)
- [ColorInterp](https://www.pdal.io/stages/filters.colorinterp.html)
- [Colorization](https://www.pdal.io/stages/filters.colorization.html)
- [ComputeRange](https://www.pdal.io/stages/filters.computerange.html)
- [Eigenvalues](https://www.pdal.io/stages/filters.eigenvalues.html)
- [EstimateRank](https://www.pdal.io/stages/filters.estimaterank.html)
- [Extended Local Minimum](https://www.pdal.io/stages/filters.elm.html)
- [Ferry](https://www.pdal.io/stages/filters.ferry.html)

+++

### Filters that create/alter dimensions (other than XYZ)

- [HAG](https://www.pdal.io/stages/filters.hag.html)
- [KDistance](https://www.pdal.io/stages/filters.kdistance.html)
- [LOF](https://www.pdal.io/stages/filters.lof.html)
- [Mongus](https://www.pdal.io/stages/filters.mongus.html)
- [Normal](https://www.pdal.io/stages/filters.normal.html)
- [Outlier](https://www.pdal.io/stages/filters.outlier.html)
- [Overlay](https://www.pdal.io/stages/filters.overlay.html)
- [PMF](https://www.pdal.io/stages/filters.pmf.html)
- [RadialDensity](https://www.pdal.io/stages/filters.radialdensity.html)
- [SMRF](https://www.pdal.io/stages/filters.smrf.html)

+++

### Filters that change point order

- [MortonOrder](https://www.pdal.io/stages/filters.mortonorder.html)
- [Randomize](https://www.pdal.io/stages/filters.randomize.html)
- [Sort](https://www.pdal.io/stages/filters.sort.html)

+++

### Filters that move points

- [Reprojection](https://www.pdal.io/stages/filters.reprojection.html)
- [Transformation](https://www.pdal.io/stages/filters.transformation.html)

+++

### Filters that cull points

- [Crop](https://www.pdal.io/stages/filters.crop.html)
- [Decimation](https://www.pdal.io/stages/filters.decimation.html)
- [Divider](https://www.pdal.io/stages/filters.divider.html)
- [Head](https://www.pdal.io/stages/filters.head.html)
- [IQR](https://www.pdal.io/stages/filters.iqr.html)
- [Locate](https://www.pdal.io/stages/filters.locate.html)
- [MAD](https://www.pdal.io/stages/filters.mad.html)
- [Range](https://www.pdal.io/stages/filters.range.html)
- [Sample](https://www.pdal.io/stages/filters.sample.html)
- [Tail](https://www.pdal.io/stages/filters.tail.html)
- [VoxelCenterNearestNeighbor](https://www.pdal.io/stages/filters.voxelcenternearestneighbor.html)
- [VoxelCentroidNearestNeighbor](https://www.pdal.io/stages/filters.vVoxelCentroidNearestNeighbor.html)

+++

### Filters that create new PointViews

- [Chipper](https://www.pdal.io/stages/filters.chipper.html)
- [Divider](https://www.pdal.io/stages/filters.divider.html)
- [Groupby](https://www.pdal.io/stages/filters.groupby.html)
- [Splitter](https://www.pdal.io/stages/filters.splitter.html)

+++

### Filters that join PointViews

- [Merge](https://www.pdal.io/stages/filters.merge.html)

+++

### Filters that create metadata

- [CPD](https://www.pdal.io/stages/filters.cpd.html)
- [HexBin](https://www.pdal.io/stages/filters.hexbin.html)
- [ICP](https://www.pdal.io/stages/filters.icp.html)
- [Stats](https://www.pdal.io/stages/filters.stats.html)

+++

### Filters that create meshes

- [GreedyProjection](https://www.pdal.io/stages/filters.greedyprojection.html)
- [GridProjection](https://www.pdal.io/stages/filters.gridprojection.html)
- [MovingLeastSquares](https://www.pdal.io/stages/filters.movingleastsquares.html)
- [Poisson](https://www.pdal.io/stages/filters.poisson.html)

+++

### Filters to embed other languages

- [Matlab](https://www.pdal.io/stages/filters.matlab.html)
- [Python](https://www.pdal.io/stages/filters.python.html)

---

### Example Pipelines

Credit to Chris Irwin.

+++

```json
{
  "pipeline":[{
    "type":"readers.las",
    "spatialreference":"EPSG:32610"
  }, {
    "type":"filters.assign",
    "assignment":"Classification[2:4]=0"
  }, {
    "type":"filters.smrf",
    "ignore":"Classification[7:7]"
  }, {
    "type":"writers.las",
    "compression":"true",
    "scale_x":"0.001",
    "scale_y":"0.001",
    "scale_z":"0.001",
    "offset_x":"auto",
    "offset_y":"auto",
    "offset_z":"auto"
  }]
}
```
@[3-4](Read the data, assigning a spatial reference)
@[6-7](Reset ground(2), low(3) and medium(4) vegetation to created, never classified (0))
@[9-10](Apply Simple Morphological Filter, ignoring noise (7))
@[12-19](Write compressed LAZ with mm precision and auto offsets)

+++

```json
{
  "pipeline":[{
    "type":"readers.las"
  }, {
    "type":"filters.range",
    "limits":"Classification[2:2]"
  }, {
    "type":"filters.poisson",
    "depth":"8"
  }, {
    "type":"writers.ply",
    "faces":"true",
    "storage_mode":"little endian"
  }]
}
```
@[3](Read the data)
@[5-6](Passthrough only ground returns (2))
@[8-9](Perform Poisson surface reconstruction)
@[11-13](Write PLY)

+++

```json
{
  "pipeline":[{
    "type":"readers.las",
    "spatialreference":"EPSG:32610"
  }, {
    "type":"filters.range",
    "limits":"Classification[2:4]"
  }, {
    "type":"filters.greedyprojection"
  }, {
    "type":"writers.ply",
    "faces":"true"
  }]
}
```
@[3-4](Read the data, assigning a spatial reference)
@[6-7](Passthrough only ground (2), low (3) and medium (4) vegetation)
@[9](Perform greedy projection surface reconstruction)
@[11-12](Write PLY)

+++

```json
{
  "pipeline":[{
    "type":"readers.las"
  }, {
    "type":"filters.reprojection",
    "in_srs":"EPSG:6340",
    "out_srs":"EPSG:26911"
  }, {
    "type":"filters.assign",
    "assignment":"Classification[:]=0"
  }, {
    "type":"filters.outlier"
  }, {
    "type":"filters.range",
    "limits":"Classification![7:7]"
  }, {
    "type":"filters.smrf"
  }, {
    "type":"filters.splitter",
    "length":"1000"
  }, {
    "type":"writers.las",
    "scale_x":"0.001",
    "scale_y":"0.001",
    "scale_z":"0.001",
    "offset_x":"auto",
    "offset_y":"auto",
    "offset_z":"auto"
  }, {
    "type":"filters.merge"
  }, {
    "type":"filters.range",
    "limits":"Classification[2:2]"
  }, {
    "type":"writers.gdal",
    "radius":0.7071,
    "resolution":0.5,
    "output_type":"idw",
    "nodata":-9999,
    "window_size":2,
    "gdalopts":"COMPRESS=DEFLATE,TILED=YES,PREDICTOR=3"
  }]
}
```
@[3](Read the data)
@[5-7](Reproject the data)
@[9-10](Reset all classifications to created, never classified (0))
@[12](Mark outliers (7))
@[14-15](Passthrough all points not marked as noise (7))
@[17](Apply Simple Morphological Filter)
@[19-20](Split the point cloud into 1km tiles)
@[22-28](Write uncompressed LAS with mm precision and auto offsets)
@[30](Merge all tiles)
@[32-33](Passthrough only ground returns (2))
@[35-41](Write 0.5m DEM using IDW)

+++

```json
{
  "pipeline": [
    "./data/isprs/samp11-utm.laz",
    {
      "type": "filters.smrf"
    }, {
      "type": "filters.hag"
    }, {
      "type": "filters.range",
      "limits": "HeightAboveGround[3:]"
    }, {
      "type": "filters.cluster",
      "tolerance": 3
    }, {
      "type": "filters.groupby",
      "dimension": "ClusterID"
    }, {
      "type": "filters.locate",
      "dimension": "HeightAboveGround",
      "minmax": "max"
    }, {
      "type": "filters.merge"
    }, {
      "type": "filters.range",
      "limits": "HeightAboveGround[20:]"
    }
  ]
}
```
@[3](Read the data)
@[5](Apply Simple Morphological Filter)
@[7](Compute Height Above Ground)
@[9-10](Passthrough only points with HeightAboveGround greater than 3 meters)
@[12-13](Perform Euclidean cluster extraction with tolerance of 3 meters)
@[15-16](Create separate PointView for each cluster)
@[18-20](Locate the maximum HeightAboveGround value in each cluster)
@[22](Merge maximum HeightAboveGround results from each cluster)
@[24-25](Retain only those clusters with maximum HeightAboveGround value over 20 meters)

+++

```bash
            X           Y       Z  HeightAboveGround
0   512794.22  5403576.38  317.30              21.99
1   512827.97  5403630.85  329.92              24.45
2   512786.89  5403626.56  366.60              58.15
3   512811.06  5403612.84  326.88              20.26
4   512792.11  5403626.03  368.89              59.78
5   512797.05  5403624.26  338.02              28.91
6   512796.65  5403624.90  350.39              41.28
7   512798.29  5403625.87  361.53              52.31
8   512797.34  5403623.67  347.56              38.45
9   512798.47  5403623.67  354.11              44.89
10  512798.73  5403624.20  365.42              56.20
11  512831.28  5403557.39  323.52              20.96
12  512815.86  5403621.44  370.03              63.70
13  512815.99  5403739.10  367.91              20.05
14  512730.79  5403738.80  401.93              21.80
```

+++

```json
{
  "pipeline":[
    "./data/isprs/samp11-utm.laz",
    {
      "type":"filters.smrf"
    }, {
      "type":"filters.hag"
    }, {
      "type":"filters.range",
      "limits":"HeightAboveGround[2:)"
    }, {
      "type":"filters.approximatecoplanar"
    }]
}
```
@[3](Read the data)
@[5](Apply Simple Morphological Filter)
@[7](Compute Height Above Ground)
@[9-10](Passthrough only points with HeightAboveGround greater than 2 meters)
@[12](Determine if a point is in an approximately coplanar region)

+++

```json
{
  "pipeline":[
    "./data/isprs/samp11-utm.laz",
    {
      "type":"filters.smrf"
    }, {
      "type":"filters.hag"
    }, {
      "type":"filters.range",
      "limits":"HeightAboveGround[2:)"
    }, {
      "type":"filters.estimaterank"
    }
  ]
}
```
@[3](Read the data)
@[5](Apply Simple Morphological Filter)
@[7](Compute Height Above Ground)
@[9-10](Passthrough only points with HeightAboveGround greater than 2 meters)
@[12](Estimate rank at each point)

+++

```json
{
  "pipeline":[{
    "type":"readers.las"
  }, {
    "type":"filters.smrf"
  }, {
    "type":"filters.hag"
  }, {
    "type":"filters.colorinterp",
    "dimension":"HeightAboveGround",
    "minimum":0.0,
    "maximum":20.0,
    "ramp":"blue_orange"
  }, {
    "type":"writers.prc",
    "prc_filename":"/Users/chambbj/Temp/colorinterp-prc.prc",
    "pdf_filename":"/Users/chambbj/Temp/colorinterp-prc.pdf"
  }]
}
```
@[3](Read the data)
@[5](Apply Simple Morphological Filter)
@[7](Compute Height Above Ground)
@[9-13](Colorize points by Height Above Ground)
@[15-17](Write PDF)

+++

```json
{
  "pipeline":[{
    "type":"readers.las"
  }, {
    "type":"filters.assign",
    "assignment":"Classification[:]=0"
  }, {
    "type":"filters.elm"
  }, {
    "type":"filters.smrf",
    "ignore":"Classification[7:7]"
  }, {
    "type":"filters.hag"
  }, {
    "type":"filters.python",
    "module":"anything",
    "function":"filter",
    "script":"/pipelines/viridis.py",
    "pdalargs":"{\"cmap\":\"inferno\",\"dimension\":\"HeightAboveGround\",\"norm\":\"log\"}"
  }, {
    "type":"writers.prc",
    "prc_filename":"/data/colorinterp-prc4.prc",
    "pdf_filename":"/data/colorinterp-prc4.pdf"
  }]
}
```
@[3](Read the data)
@[5-6](Reassign all classifications to 0)
@[8](Mark low outliers as noise)
@[10-11](Classify ground, ignoring noise)
@[13](Compute Height Above Ground)
@[15-19](Pass data to Python script to colorize by Height Above Ground)
@[21-23](Write PRC)

+++

```python
import matplotlib
import matplotlib.pyplot as plt
import numpy as np


def filter(ins, outs):
    ret = ins[pdalargs['dimension']]
    cmap = plt.get_cmap(pdalargs['cmap'])
    if pdalargs.get('norm') != None:
        if pdalargs['norm'] == 'log':
            norm = matplotlib.colors.LogNorm()
    else:
        norm = matplotlib.colors.Normalize()
    norm.autoscale(ret)

    rgba = cmap(norm(ret))
    Red = np.round(rgba[:, 0] * 256)
    Green = np.round(rgba[:, 1] * 256)
    Blue = np.round(rgba[:, 2] * 256)

    outs['Red'] = Red.astype('uint16')
    outs['Green'] = Green.astype('uint16')
    outs['Blue'] = Blue.astype('uint16')
    return True
```
@[7-8](Gather data and requested colormap from pdalargs)
@[9-14](Setup normalization)
@[16-19](Apply normalization and colormap, setting Red, Green, and Blue)
@[21-23](Cast and pass RGB outputs)

+++

```json
{
  "pipeline":[{
    "type":"readers.las"
  }, {
    "type":"filters.assign",
    "assignment":"Classification[:]=0"
  }, {
    "type":"filters.elm"
  }, {
    "type":"filters.smrf",
    "ignore":"Classification[7:7]"
  }, {
    "type":"filters.hag"
  }, {
    "type":"filters.range",
    "limits":"HeightAboveGround[2:)"
  }, {
    "type":"filters.approximatecoplanar"
  }, {
    "type":"filters.python",
    "module":"anything",
    "function":"filter",
    "script":"/pipelines/viridis.py",
    "pdalargs":"{\"cmap\":\"viridis\",\"dimension\":\"Coplanar\"}"
  }, {
    "type":"writers.prc",
    "prc_filename":"/data/colorinterp-prc4.prc",
    "pdf_filename":"/data/colorinterp-prc4.pdf"
  }]
}
```
@[15-16](Crop to those points 2 meters above ground and higher)
@[18](Determine if each point is possibly coplanar)

---

## Questions?
