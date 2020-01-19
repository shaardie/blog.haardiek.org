# Plotting Sentinel 5P Data

__Sven Haardiek, 2020-01-19__

[Sentinel 5P](http://www.esa.int/Applications/Observing_the_Earth/Copernicus/Sentinel-5P) is one of [ESA](https://www.esa.int/)'s earth observation satellites developed as part of the [Copernicus](https://www.esa.int/Applications/Observing_the_Earth/Copernicus) program.

It is dedicated to monitor air pollution and use a spectrometer to measure ozone, methane, nitrogen dioxide and other gases in the atmosphere.

I currently work with a small group on a project called [Emissions API](https://emissions-api.org) which is trying to create a web API to make it easier to access the data of Sentinel 5P and its successor [Sentinel 5](https://earth.esa.int/web/guest/missions/esa-future-missions/sentinel-5).
While working we created some small libraries for communicating with the servers of the ESA and for processing the data from the satellite.
To verify that those libraries are working fine and to understand the data, it is often useful to visualize the results.
So I would like to share some of those plots with you.

At the end we should see something like this

![Scatter Plot](images/plotting-sentinel-5p-data/scatter-plot.png)

## Preparation

We are using the [Sentinel-5P Downloader](https://pypi.org/project/sentinel5dl/) to download the data from the ESA and [Sentinel-5 Algorithms](https://pypi.org/project/s5a/) to read and pre-process those data.

For plotting we are using [GeoPandas](http://geopandas.org/) which is internally using [Matplotlib](https://matplotlib.org/) and [Descartes](https://pypi.org/project/descartes/).

```bash
$ pip install geopandas s5a sentinel5dl matplotlib descartes
```

Now we download the data using the `sentinel5dl` binary.

```bash
$ mkdir -p data/S5P_OFFL_L2__NO2____
$ sentinel5dl --begin-ts '2019-12-31' --end-ts '2020-01-02' \
    --mode Offline --product 'L2__NO2___' data/S5P_OFFL_L2__NO2____
```

I chose to use nitrogen dioxide and New Year's Eve, since I hoped to find a lot of pollution from the fireworks.

The result should look like this

```bash
$ ls data/S5P_OFFL_L2__NO2____
S5P_OFFL_L2__NO2____20191231T011147_20191231T025317_11473_01_010302_20200101T180133.nc
S5P_OFFL_L2__NO2____20191231T011147_20191231T025317_11473_01_010302_20200101T180133.nc.md5sum
S5P_OFFL_L2__NO2____20191231T025317_20191231T043447_11474_01_010302_20200101T192226.nc
S5P_OFFL_L2__NO2____20191231T025317_20191231T043447_11474_01_010302_20200101T192226.nc.md5sum
...
S5P_OFFL_L2__NO2____20200101T192916_20200101T211046_11498_01_010302_20200103T121312.nc
S5P_OFFL_L2__NO2____20200101T192916_20200101T211046_11498_01_010302_20200103T121312.nc.md5sum
S5P_OFFL_L2__NO2____20200101T211046_20200101T225216_11499_01_010302_20200103T140339.nc
S5P_OFFL_L2__NO2____20200101T211046_20200101T225216_11499_01_010302_20200103T140339.nc.md5sum
```

We are interested in the `*.nc` files.
These are in the [NetCDF](https://de.wikipedia.org/wiki/NetCDF) format and containing a lot of data gathered from the satellite.
If you are interested, you can explore those files for example with a tool like [Panoply](https://www.giss.nasa.gov/tools/panoply/).
To make it easier, we are using `s5a` to load the data, since we only need a very small subset.


```python
import s5a

data = s5a.load_ncfile(
    'data/S5P_OFFL_L2__NO2____/S5P_OFFL_L2__NO2____20191231T025317_20191231T043447_11474_01_010302_20200101T192226.nc'
)
```

`data` should now contain something like this

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>timestamp</th>
      <th>quality</th>
      <th>value</th>
      <th>longitude</th>
      <th>latitude</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2019-12-31 03:17:57.205000+00:00</td>
      <td>0.07</td>
      <td>-0.000011</td>
      <td>-15.975651</td>
      <td>-65.399246</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2019-12-31 03:17:57.205000+00:00</td>
      <td>0.07</td>
      <td>-0.000019</td>
      <td>-16.178890</td>
      <td>-65.418892</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2019-12-31 03:17:58.045000+00:00</td>
      <td>0.07</td>
      <td>-0.000006</td>
      <td>-15.965775</td>
      <td>-65.446426</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2019-12-31 03:17:58.045000+00:00</td>
      <td>0.07</td>
      <td>-0.000012</td>
      <td>-16.169365</td>
      <td>-65.466110</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2019-12-31 03:17:58.885000+00:00</td>
      <td>0.07</td>
      <td>-0.000010</td>
      <td>-15.955670</td>
      <td>-65.493546</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>1578390</th>
      <td>2019-12-31 04:10:38.894000+00:00</td>
      <td>0.03</td>
      <td>-0.000070</td>
      <td>91.619942</td>
      <td>62.235840</td>
    </tr>
    <tr>
      <th>1578391</th>
      <td>2019-12-31 04:10:38.894000+00:00</td>
      <td>0.03</td>
      <td>-0.000045</td>
      <td>91.716927</td>
      <td>62.305752</td>
    </tr>
    <tr>
      <th>1578392</th>
      <td>2019-12-31 04:10:39.734000+00:00</td>
      <td>0.03</td>
      <td>-0.000052</td>
      <td>91.440414</td>
      <td>62.197773</td>
    </tr>
    <tr>
      <th>1578393</th>
      <td>2019-12-31 04:10:39.734000+00:00</td>
      <td>0.03</td>
      <td>-0.000070</td>
      <td>91.538177</td>
      <td>62.268784</td>
    </tr>
    <tr>
      <th>1578394</th>
      <td>2019-12-31 04:10:40.574000+00:00</td>
      <td>0.03</td>
      <td>-0.000075</td>
      <td>91.358582</td>
      <td>62.230633</td>
    </tr>
  </tbody>
</table>
<p>1578395 rows × 5 columns</p>
</div>

One and a half million points per data set is a lot.
Luckily `s5a` does have some functionality to reduce this.
First note that every data point does have a `quality`.
With `s5a` we can drop points with poor quality quite easily.

```python
data = s5a.filter_by_quality(data)
```

Now `data` does contain `1416967` which is still to much.

To reduce those points even more,
`s5a` is using [Uber’s Hexagonal Hierarchical Spatial Index H3](https://eng.uber.com/h3/),
which is a grid system partitioning the earth into hexagons.
We calculate the H3 indices for every point,
aggregate points with the same index by calculating the mean `value` and recalculate the `longitude` and `latitude` as the center of the hexagons.
The `resolution` defines the size of the hexagons with a resolution of 5 partitioning the world into approximately 2 million unique hexagons.
For more detailed information take a look at the [Table of Cell Areas for H3 Resolutions](https://uber.github.io/h3/#/documentation/core-library/resolution-table).


```python
data = s5a.point_to_h3(data, resolution=5)
data = s5a.aggregate_h3(data)
data = s5a.h3_to_point(data)
```

So let's compare the number of points in the different sets.

![Number of Points](images/plotting-sentinel-5p-data/number-of-points.svg)

With those techniques we have reduced the number of points in this data and also have limit the total amount of points we have to plot for the whole world to approximately 2 millions.

Our last preparation will be converting the `pandas.core.frame.DataFrame` into a `geopandas.geodataframe.GeoDataFrame` to be able to use the spatial operations and plotting functionality.

```python
import geopandas

geometry = geopandas.points_from_xy(data.longitude, data.latitude)
data = geopandas.GeoDataFrame(data, geometry=geometry, crs={'init' :'epsg:4326'})
```

## Plotting the File

We want to plot the values from the satellite on a map of the earth.
Luckily GeoPandas does have one included.

```python
world = geopandas.read_file(geopandas.datasets.get_path('naturalearth_lowres'))
world.plot(figsize=(10, 5))
```

![world](images/plotting-sentinel-5p-data/world.png)

Our data and the world map does have the same projection, so we can plot them easily together.
But since projection is widely spread on the pols, we are switching to the [Robinson projection](https://en.wikipedia.org/wiki/Robinson_projection).

```python
robinson_projection = '+a=6378137.0 +proj=robin +lon_0=0 +no_defs'
world = world.to_crs(robinson_projection)
data = data.to_crs(robinson_projection)
```

So now let's plot the data.
The explanation is embedded in the code.
For more information take a look at [GeoPandas Mapping](http://geopandas.org/mapping.html).

```python
import matplotlib.pyplot as plt

# Define base of the plot.
fig, ax = plt.subplots(1, 1, figsize=(40, 40), dpi=100)
# Disable the axes
ax.set_axis_off()
# Plot the data
data.plot(
    column='value',  # Column defining the color
    cmap='rainbow',  # Colormap
    marker='H',  # marker layout. Here a Hexagon.
    markersize=1,
    ax=ax,  # Base
    vmax=0.0005,  # Used as max for normalize luminance data
)

# Plot the boundary of the countries on top
world.geometry.boundary.plot(color=None, edgecolor='black', ax=ax)
```

![single plot](images/plotting-sentinel-5p-data/single-plot.png)

We can already see a lot on this plot.
For one we can see the movement of the satellite and we can see that we do not have any data points in the north.
This is due to the fact that the satellite's instruments need light to work and are using the sunlight for that.
Since it is winter, there is not enough sunlight in the north.

Also we can see a high peak of nitrogen dioxide on the south east of Australia which are probably the bush fires.

## Plotting multiple Files

Now we use all data we downloaded before in a single plot. So we get a nice view over the nitrogen dioxide values over the whole world.

First we load all files and combine them using [Pandas `concat`](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.concat.html).
After that we are using the same techniques as before to reduce the data and create a `geopandas.geodataframe.GeoDataFrame`.

```python
import glob

import pandas


# Read in all files
data = []
for filename in glob.glob('data/S5P_OFFL_L2__NO2____/*.nc'):
    data.append(s5a.load_ncfile(filename))

# Combine points
data = pandas.concat(data, ignore_index=True)

# Reduce points
data = s5a.filter_by_quality(data)
data = s5a.point_to_h3(data, resolution=5)
data = s5a.aggregate_h3(data)
data = s5a.h3_to_point(data)

# Create geopandas dataframe
geometry = geopandas.points_from_xy(data.longitude, data.latitude)
data = geopandas.GeoDataFrame(data, geometry=geometry, crs={'init' :'epsg:4326'})

# Projection change
data = data.to_crs(robinson_projection)
```

Now we plot this data the same we way than before and the following result

![Scatter Plot](images/plotting-sentinel-5p-data/scatter-plot.png)

That's all!

If you want to try this yourself,
you can also take a look at this [Jupyter Notebook](https://nbviewer.jupyter.org/github/shaardie/sentinel5p-plots/blob/master/Scatter%20Plots.ipynb).
