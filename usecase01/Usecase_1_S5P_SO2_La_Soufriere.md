---
jupyter:
  jupytext:
    formats: md,ipynb
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.15.2
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

<!-- #region -->
## ![Atmospheric Toolbox](https://atmospherictoolbox.org/media/filer_public_thumbnails/filer_public/6d/35/6d35dffd-43f1-43ec-bff6-5aa066c8aabc/toolbox-header.jpg__1080x416_q85_subsampling-2.jpg)

# Atmospheric Toolbox - Basics of HARP functionalities


In this example some of the basic functionalities of the [ESA Atmospheric Toolbox](https://atmospherictoolbox.org/) to handle TROPOMI data are demonstrated. This case focuses on the use of toolbox's HARP component in Python, implemented as a Jupyter notebook.

The ESA Copernicus TROPOMI instrument onboard Sentinel 5p satellite observes atmospheric constituents at very high  spatial resolution. In this tutorial we will demonstrate basic data reading and plotting procedures using TROPOMI SO2 observations. We use observations that were obtained during the explosive eruption of La Soufriere volcano in the Caribbean in April 2021. The eruption released large amounts of SO2 into the atmosphere, resulting extensive volcanic SO2 plumes that were transported long distances. This notebook will demonstrate how this event can be visualized using TROPOMI SO2 observations and HARP.

In the steps below this tutorial shows
-  basic data reading using HARP
-  how to plot single orbit TROPOMI data on a map, and
-  how to apply operations to the TROPOMI data when importing with HARP
<!-- #endregion -->

<!-- #region -->
## Initial preparations

To follow this notebook some preparations are needed. The TROPOMI SO2 data used in this notebook is obtained
from the [Sentinel-5P Pre-Operations Data Hub](https://s5phub.copernicus.eu/dhus/#/home).

This example uses the following TROPOMI SO2 file obtained at 12.4.2021:

`S5P_RPRO_L2__SO2____20210412T151823_20210412T165953_18121_03_020401_20230209T050738.nc`



In addition to HARP, this notebook uses several other Python packages that needs to be installed beforehand. The packages needed for running the notebook are:
- harp: for reading and handling of TROPOMI data
- numpy: for working with arrays
- matplotlib: for visualizing data
- cartopy: for geospatial data processing, e.g. for plotting maps
- cmcrameri: for universally readable scientific colormaps

The instructions on how to get started with the Atmospheric toolbox using Python and install HARP can be found here (*add link to getting started jupyter notebook*). Please note that if you have installed HARP in some specific python environment, check that you have activated the environment before running the script.
<!-- #endregion -->

## Step1: Reading TROPOMI SO2 data using HARP


First the needed Python packages are imported; harp, numpy, matplotlib, cartopy, and cmcrameri:

```python
import os
import harp
import numpy as np
import matplotlib.pyplot as plt
from matplotlib.colors import Normalize
import cartopy.crs as ccrs
from cmcrameri import cm
import eofetch
```

The second step is to import the TROPOMI Level 2 SO2 file using `harp.import_product()`. If the file does not yet exist on your local machine, we use the avl library to automatically download the file from the Copernicus Dataspace Ecosystem (CDSE). (Because the original netcdf file is large, both downloading and importing the file might take a while.)

```python
filename = "S5P_RPRO_L2__SO2____20210412T151823_20210412T165953_18121_03_020401_20230209T050738.nc"
```

We use _eofetch_ to download the S5P product. To be able to perform the download yourself you will need to retrieve and configure credentials as described [in the eofetch README](https://github.com/stcorp/eofetch#readme).
Alternatively, you can download the file manually and put it in the same directory as this notebook.

```python tags=["remove_output"]
eofetch.download(filename)
```

```python
product = harp.import_product(filename)
```

After a successful import, you have created a python variable called `product`. The variable `product` contains a record of the SO2 product variables, the data is imported as numpy arrays. You can view the contents of `product` using the Python `print()` function:

```python
print(product)
```

With print command you can also inspect the information of a specific SO2 product variable (listed above), e.g. by typing:

```python
print(product.SO2_column_number_density)
```

From the listing above you see e.g. that the unit of the SO2_column_number_density variable is mol/m^2. Type of the product and the shape (size) of the SO2_column_number_density data array can be checked with the following commands:

```python
print(type(product.SO2_column_number_density.data))
print(product.SO2_column_number_density.data.shape)
```

Here it is important to notice that `harp.import_product` command imports and converts the TROPOMI Level 2 data to a structure that is compatible with the HARP conventions. This HARP compatible structure is *different* from the netcdf file structure. This HARP conversion includes e.g. restructuring data dimensions or renaming variables. For example, from the `print` commands above it is shown that after HARP import the dimension of the SO2_column_number_density data is time (=1877400), whereas working with netcdf-files directly using e.g. a library such as netCDF4, the dimensions of the same data field would be a 2D array, having Lat x Lon dimension.

HARP has builtin converters for [a lot of atmospheric data products](http://stcorp.github.io/harp/doc/html/ingestions/index.html). For each conversion the HARP documentation contains a description of the variables it will return and how it mapped them from the original product format. The description for the TROPOMI SO2 product can be found [here](http://stcorp.github.io/harp/doc/html/ingestions/S5P_L2_SO2.html).

HARP does this conversion such that data from other satellite data products, such as OMI, or GOME-2, will end up having the same structure and naming conventions. This makes it a lot easier for users to deal with data coming from different satellite instruments.


## Step 2: Plotting a single orbit data on a map


Now that the TROPOMI SO2 data product is imported, the data will be visualized on a map. The parameter we want to plot is the "SO2_column_number_density", which gives the total atmospheric SO2 column. For this we will be using [cartopy](https://scitools.org.uk/cartopy/docs/latest/) and the `scatter` function. This plotting function is based on using only the pixel center coordinates of the satellite data, and not the actual latitude and longitude bounds. The scatter function will plot each satellite pixel as coloured single dot on a map based on their lat and lon  coordinates. Cartopy also provides other plotting options, such as pcolormesh. However, in pcolormesh the input data needs to be a 2D array. This type of plotting will be demonstrated in the another use cases.




First, the SO2, latitude and longitude center data are defined. In addition, units and description of the SO2 data are read that are needed for the colorbar label. For plotting a colormap named "batlow" is chosen from the cmcrameri library. The cmcrameri provides scientific colormaps where the colour combinations are readable both by colour-vision deficient and colour-blind people. The Crameri colormap options can be viewed [here](https://www.fabiocrameri.ch/colourmaps/). In the script the colormaps are called e.g. as `cm.batlow`. If you wish to use reversed colormap, append *_r* to the colormaps name. With vmin and vmax the scaling of the colormap values are defined.

```python
SO2val = product.SO2_column_number_density.data
SO2units = product.SO2_column_number_density.unit
SO2description = product.SO2_column_number_density.description

latc=product.latitude.data
lonc=product.longitude.data

colortable=cm.batlow
vmin=0
vmax=0.0001
```

Next the figure properties will be defined. By using matplotlib `figsize` argument the figure size can be defined, `plt.axes(projection=ccrs.PlateCarree())` sets a up GeoAxes instance, and `ax.coastlines()` adds the coastlines to the map. The actual data is plotted with `plt.scatter` command, where lat and lon coordinates are given as input, and the dots are coloured according to the pixel SO2 value (SO2val).

```python
fig=plt.figure(figsize=(20, 10))
ax = plt.axes(projection=ccrs.PlateCarree())


img = plt.scatter(lonc, latc, c=SO2val,
                vmin=vmin, vmax=vmax, cmap=colortable, s=1, transform=ccrs.PlateCarree())

ax.coastlines()

cbar = fig.colorbar(img, ax=ax, orientation='horizontal', fraction=0.04, pad=0.1)
cbar.set_label(f'{SO2description} [{SO2units}]')
cbar.ax.tick_params(labelsize=14)
plt.show()
```

## Step 3: Applying operations when importing data with HARP


In the previous blocks one orbit of TROPOMI SO2 data has been imported with HARP and plotted on a map as it is. However, there is one very important step missing that is essential to apply when working with almost any satellite data: **the quality flag(s)**. To ensure that you work with only good quality data and make correct interpretations, it is essential that the recommendations given for each TROPOMI Level 2 data are followed.

#### One of the main features of HARP is the ability to perform operations as part of the data import.

This very unique feature of HARP allows you to apply different kind of operations on the data already when importing it, and hence, no post processing is needed. These operations can include e.g. cutting the data over certain area only, converting units, and of course applying the quality flags. Information on all operations that can be applied can be found in the [HARP operations documentation](http://stcorp.github.io/harp/doc/html/operations.html).



Now, we will import the same data file as in Step 1, but now **adding four different operations as a part of the import command**:

- we only ingest data that is between -20S and 40N degrees latitude
- we only consider pixels for which the data quality is high enough. The basic quality flag in any TROPOMI Level 2 netcdf file is given as `qa_value`. In the the [Product Readme File for SO2](https://sentinels.copernicus.eu/documents/247904/3541451/Sentinel-5P-Sulphur-Dioxide-Readme.pdf) you can find, that the basic recommendation for SO2 data is to use only those pixels where `qa_value > 0.5`. When HARP imports data, the quality values are interpreted as numbers between 0 and 100 (not 0 and 1), hence our limit in this case is 50. In HARP the `qa_value` is renamed as `SO2_column_number_density_validity`. The list of variables in HARP product after ingestion of S5P TROPOMI SO2 product are found [here](http://stcorp.github.io/harp/doc/html/ingestions/S5P_L2_SO2.html).

- we limit the variables that we read to those that we need
- we convert the unit of the tropospheric SO2 column number density to Dobson Units (DU)  (instead of using mol/m2 in which the original data was stored)

All these operations will be performed by HARP while the data is being read, and before it is returned to Python.


In the following, the HARP operations that are performed when importing data are here given as "operations" variable, that includes each HARP operation (name, condition) as string. All the applied HARP operations are separated with ";" and finally joined together with `join()` command. With "keep" operation it is defined which variables from the original netcdf files are imported, while "derive" operation performs the conversion from original units to dobson units. After joining the operations together you can print the resulting string using the `print()` command. In Python defining an "operations" string parameter is a convenient way to define and keep track on different operations to be applied when importing the data. Other option would be to write the operations as an input to the HARP import command as: "operation1;operation2;operation3".

```python
operations = ";".join([
    "latitude>-20;latitude<40",
    "SO2_column_number_density_validity>50",
    "keep(datetime_start,scan_subindex,latitude,longitude,SO2_column_number_density)",
    "derive(SO2_column_number_density [DU])",
])

print(type(operations))
print(operations)
```

The import with HARP including operations is executed with the same `harp.import_product()`command as before, but in addition to filename now also the "operations" variable is given as input, separated with a comma. We will call the new imported variable as "reduced_product":

```python
reduced_product = harp.import_product(filename, operations)
```

You will see that importing the data now goes a _lot_ faster. If we print the contents of the `reduced_product`, it shows that the variable consists only those parameters we requested, and the SO2 units are as DU. Also the time dimension of the data is less than in Step 1, because only those pixels between -20S-40N have been considered:

```python
print(reduced_product)
```

Now that the new reduced data is imported, the same approach as in Step 2 can be used to plot the data on a map. Note that now the units of SO2 have changed, and therefore different scaling for the colorscheme is needed. First define the parameters for plotting:

```python
SO2val = reduced_product.SO2_column_number_density.data
SO2units = reduced_product.SO2_column_number_density.unit
SO2description = reduced_product.SO2_column_number_density.description

latc=reduced_product.latitude.data
lonc=reduced_product.longitude.data

colortable=cm.batlow
# For Dobson Units
vmin=0
vmax=8
```

And then plot the figure:

```python
fig=plt.figure(figsize=(20, 10))
ax = plt.axes(projection=ccrs.PlateCarree())


img = plt.scatter(lonc, latc, c=SO2val,
                vmin=vmin, vmax=vmax, cmap=colortable, s=1, transform=ccrs.PlateCarree())

ax.coastlines()

cbar = fig.colorbar(img, ax=ax, orientation='horizontal', fraction=0.04, pad=0.1)
cbar.set_label(f'{SO2description} [{SO2units}]')
cbar.ax.tick_params(labelsize=14)
plt.show()
```

The plot shows how the large SO2 plume originating from La Soufriere eruption extends across the orbit. There are now also white areas within the plume, where bad quality pixels have been filtered out. It is also noticeable now much faster the plotting procedure is with the reduced dataset.

## Step 4: Regridding with HARP and plotting using pcolormesh

In Steps 2 and 3 we applied the scatter function for quick plotting, however, it is not an optimal function to visualize satellite data on a map, since each pixel is plotted as a single dot. The other plot function from cartopy is pcolormesh. However, the mesh plot requires the input data (latitude, longitude, and variable to plot) as 2D matrices, and therefore the pcolormesh can not be directly applied to data imported and filtered using HARP (Step 3). This is because after these filtering operations we don't have all pixels for a scanline anymore.

A solution to this problem is to regrid the S5P data to a regular latitude/longitude grid before plotting. The regridding can be done by using a `bin_spatial()` operation when importing data with HARP. Regridding data into a lat/lon grid is also needed if we want to combine the data from multiple orbits from one day into a single daily grid. This will be demonstrated in another use case.

The `bin_spatial()` operation requires six input parameters, that defines the new grid. The input parameters are:
- the number of latitude edge points
- the latitude offset at which to start the grid
- the latitude increment (= latitude length of a grid cell)
- the number of longitude edge points
- the longitude offset at which to start the grid
- the longitude increment (= longitude length of a grid cell)

![bin_spatial grid](https://raw.githubusercontent.com/stcorp/avl-use-cases/master/usecase1/grid.png)

In this example we define a new grid at 0.05 degrees resolution over the area of the volcanic SO2 plume. The latitude and longitude offset in this case is for latitude -10S, and for longitude -70W (red point in the picture). Since the grid resolution is now 0.05 degrees and the latitudes in the new grid extend from -10S to 30N, the number of latitude edge points is 801 (=number of points from -10 to 30 at 0.05 steps). Similarly, since the the longitudes in the grid extend from -70W to -20W, the number of longitude edge points is 1001. Hence, the number edge points is one more than the number of grid cells. This is similar to the way you should provide the `X` and `Y` parameters to the `pcolormesh` function (see [matplotlib_documentation)](https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.pcolormesh.html).
For a 0.1 degree by 0.1 degree global grid we would need 1800 by 3600 grid cells which equals 1801 by 3601 grid edge points.

The input for `bin_spatial()` is given in the following order:

bin_spatial(lat_edge_length, lat_edge_offset, lat_edge_step, lon_edge_length, lon_edge_offset, lon_edge_step)

In this example, the `bin_spatial()` input is:

`bin_spatial(801, -10, 0.05, 1001, -70, 0.05)`

HARP can actually do a proper weighted area average to calculate the value for each grid cell. It will need the corner coordinates of each satellite pixel, provided by the `latitude_bounds` and `longitude_bounds`. This is why we need to add these variables to the `keep()` operation we perform below. We also add `derive()` latitude and longitude, so that the new grid center coordinates are included in the imported variable.

As a summary, in this example the operations that will be performed with HARP import are:

- considering only good quality SO2 observations: "SO2_column_number_density_validity>50"
- keeping the needed parameters: "keep(latitude_bounds,longitude_bounds,SO2_column_number_density)"
- regridding the SO2 data: "bin_spatial(801, -10, 0.05, 1001, -70, 0.05)"
- converting SO2 to Dobson Units: "derive(SO2_column_number_density [DU])"
- derive latitude and longitude coordinates of the new grid: "derive(latitude {latitude})","derive(longitude {longitude})"

```python
operations = ";".join([
    "SO2_column_number_density_validity>50",
    "keep(latitude_bounds,longitude_bounds,SO2_column_number_density)",
    "bin_spatial(801, -10, 0.05, 1001, -70, 0.05)",
    "derive(SO2_column_number_density [DU])",
    "derive(latitude {latitude})",
    "derive(longitude {longitude})",
])
```

Here the new regridded variable is named as "regridded_product". The content of the "regridded_product" can be viewed using the Python `print()` command.

```python
regridded_product = harp.import_product(filename, operations)
print(regridded_product)
```

As the printing of variables show, the re-gridded SO2 variable has now two dimensions (in addition to time), latitude (800) and longitude (1000). Hence, now it is possible to use pcolormesh function since the `SO2_column_number_density` is a 2D array.

The corner coordinates of each grid cell are provided by the `latitude_bounds` and `longitude_bounds` variables and these are used for plotting. Note that the pcolormesh function requires these corner coordinates as the input for latitude and longitude. As we see from the print above, the shape (dimensions) of `latitude_bounds` and `longitude_bounds` is 1000 x 2. The `regridded_product.latitude_bounds.data[:,0]` array gives the latitudes of the lower corners, whereas `regridded_product.latitude_bounds.data[:,1]` gives the latitudes for upper corners.

```python
print(regridded_product.latitude_bounds.data[:,0])
print(regridded_product.latitude_bounds.data[:,1])
```

As we see from the print, `regridded_product.latitude_bounds.data[:,1]` contains the j+1 coordinates of the first dimension ([:,0]) plus the upper right corner latitude of the grid. To get the correct input for pcolormesh, we define the gridlat variable by appending the `regridded_product.latitude_bounds.data[:,0]` array with the last element of the second array: `regridded_product.longitude_bounds.data[-1,1]`. The `gridlon` variable is defined similarly:

```python
gridlat = np.append(regridded_product.latitude_bounds.data[:,0], regridded_product.latitude_bounds.data[-1,1])
gridlon = np.append(regridded_product.longitude_bounds.data[:,0], regridded_product.longitude_bounds.data[-1,1])
```

```python
SO2val = regridded_product.SO2_column_number_density.data
SO2units = regridded_product.SO2_column_number_density.unit
SO2description = regridded_product.SO2_column_number_density.description

colortable=cm.batlow
# For Dobson Units
vmin=0
vmax=9
```

Next the figure properties are defined. In Steps 2 and 3 we used the scatter function, here the actual data is plotted with `plt.pcolormesh` command, having as an input gridlon, gridlat, SO2 value and the colormap definitions. Note that the dimensions of the `SO2val` are time, lat, and lon, and therefore the input is given as `SO2val[0,:,:]`. Finally the colorbar is added with label text, and also the location of the colorbar is set.

```python
fig = plt.figure(figsize=(20,10))
ax = plt.axes(projection=ccrs.PlateCarree())
img = plt.pcolormesh(gridlon, gridlat, SO2val[0,:,:], vmin=vmin, vmax=vmax,
                         cmap=colortable, transform=ccrs.PlateCarree())
ax.coastlines()
ax.gridlines()

cbar = fig.colorbar(img, ax=ax,orientation='horizontal', fraction=0.04, pad=0.1)
cbar.set_label(f'{SO2description}[{SO2units}]')
cbar.ax.tick_params(labelsize=14)
plt.show()
```
