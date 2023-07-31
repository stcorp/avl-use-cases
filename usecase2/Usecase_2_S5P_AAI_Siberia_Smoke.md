---
jupyter:
  jupytext:
    formats: md,ipynb
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.14.6
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

![VIIRS_RGB_smoke.png](https://raw.githubusercontent.com/stcorp/avl-use-cases/master/usecase2/VIIRS_RGB_smoke.png)

# Creating gridded Level 3 data with HARP from multiple TROPOMI Level 2 UVAI files

This use case demonstrates **how to create a gridded level 3 netcdf file from multiple level 2 TROPOMI UV Aerosol Index (UVAI) files**. The case is demonstrated with an extensive smoke plume from wildfires in Siberia, Russia, on 6th of August 2021. The wildfire smoke covered an extensive area in the Northern hemisphere, and therefore the best view on the plume is obtained by **gridding TROPOMI Level 2 data from all the orbits available on that day**, a total of 14 files **into one level 3 file**. This tutorial shows

-  how to read and import TROPOMI Level 2 UVAI with HARP
-  how to grid multiple orbits from one day into a common grid, and save the merged data as netcdf file

## Table of contents

1. [TROPOMI Level 2 UVAI product](#paragraph1)
2. [Python packages for the notebook](#paragraph2)
3. [Downloading TROPOMI UVAI files (optional)](#paragraph3)
4. [Viewing the content of the UVAI file (optional)](#paragraph4)
5. [Regridding level 2 data into level 3 grid](#paragraph5)
    1. [Step 1: Operations in HARP import](#subparagraph1)
    2. [Step 2: Create the merged product: reduce_operations](#subparagraph2)
    3. [Step 3: Apply HARP import command to multiple input files](#subparagraph3)
6. [Plotting gridded level 3 data on a map](#paragraph6)
7. [Save gridded data as netcdf](#paragraph7)
8. [References to HARP documentation](#harp_references)


## 1. TROPOMI Level 2 UVAI product <a name="paragraph1"></a>
UVAI (also called as Absorbing Aerosol Index AAI or Aerosol Index AI)) is a unitless parameter that is defined based on wavelength dependent changes in Rayleigh scattering in the UV spectral range. Positive values of UVAI indicate the presence of an elevated absorbing aerosol plume, like smoke from biomass burning aerosols, desert dust or volcanic ash. In this tutorial the positive UVAI values that we are interested in, are referred as **Absorbing Aerosol Index (AAI)**.

For more information on TROPOMI UVAI product can be found here:
- [UVAI Algorithm Theoretical Basis Document](https://sentinels.copernicus.eu/documents/247904/2476257/Sentinel-5P-TROPOMI-ATBD-UV-Aerosol-Index)
- [UVAI Product User Manual](https://sentinels.copernicus.eu/documents/247904/2474726/Sentinel-5P-Level-2-Product-User-Manual-Aerosol-Index-product)

## 2. Python packages for the notebook <a name="paragraph2"></a>

In addition to HARP, this notebook uses several other Python packages that needs to be installed before executing the notebook:

- **harp**: for reading and handling of TROPOMI data
- **numpy**: for working with arrays
- **matplotlib**: for visualizing data
- **cartopy**: for geospatial data processing, e.g. for plotting maps
- **cmcrameri**: for universally readable scientific colormaps

In case you want to download the TROPOMI files automatically from the Sentinel-5P Pre-Operations Data Hub, you will need also:

- **sentinelsat**: for searching, downloading and retrieving the metadata of Sentinel satellite images from the Copernicus Open Access Hub.

Note that if you have installed HARP in some specific python environment, check that you have activated the environment before running the scripts.

**The needed Python packages are imported as follows:**


```python
import harp
import numpy as np
import matplotlib.pyplot as plt
import cartopy.crs as ccrs
from cmcrameri import cm
import eofetch
```

```python tags=["remove_input", "remove_output"]
import warnings
from shapely.errors import ShapelyDeprecationWarning
warnings.filterwarnings("ignore", category=ShapelyDeprecationWarning)
```

## 3. Downloading TROPOMI UVAI files (optional) <a name="paragraph3"></a>

The TROPOMI UVAI data used in this notebook is obtained from the [Sentinel-5P Pre-Operations Data Hub](https://s5phub.copernicus.eu/dhus/#/home). For Sentinel-5P each level 2 file contains information from one orbit. There are approximately 14 orbits per day.  This notebook uses TROPOMI UVAI data from 6th August 2021 including the following files:

<i> `S5P_RPRO_L2__AER_AI_20210806T003208_20210806T021338_19758_03_020400_20221026T213121.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T021338_20210806T035507_19759_03_020400_20221026T213201.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T035507_20210806T053637_19760_03_020400_20221026T213206.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T053637_20210806T071807_19761_03_020400_20221026T213209.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T071807_20210806T085936_19762_03_020400_20221026T213211.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T085936_20210806T104106_19763_03_020400_20221026T213218.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T104106_20210806T122235_19764_03_020400_20221026T213219.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T122235_20210806T140405_19765_03_020400_20221026T213933.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T140405_20210806T154534_19766_03_020400_20221026T213938.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T154534_20210806T172704_19767_03_020400_20221026T213939.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T172704_20210806T190833_19768_03_020400_20221026T213943.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T190833_20210806T205003_19769_03_020400_20221026T213945.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T205003_20210806T223133_19770_03_020400_20221026T213953.nc` </i>
<i> `S5P_RPRO_L2__AER_AI_20210806T223133_20210807T001302_19771_03_020400_20221026T213954.nc` </i>

**Alternatively, TROPOMI data can be downloaded using the eofetch.download function**. With the eofetch.download command you can automatically download the needed files from the Copernicus Dataspace Ecosystem. In this case we download all 14 UVAI files for 6.8.2021. Note that the size of each file is about 170 Mb.


```python
filenames = [
    "S5P_RPRO_L2__AER_AI_20210806T003208_20210806T021338_19758_03_020400_20221026T213121.nc",
    "S5P_RPRO_L2__AER_AI_20210806T021338_20210806T035507_19759_03_020400_20221026T213201.nc",
    "S5P_RPRO_L2__AER_AI_20210806T035507_20210806T053637_19760_03_020400_20221026T213206.nc",
    "S5P_RPRO_L2__AER_AI_20210806T053637_20210806T071807_19761_03_020400_20221026T213209.nc",
    "S5P_RPRO_L2__AER_AI_20210806T071807_20210806T085936_19762_03_020400_20221026T213211.nc",
    "S5P_RPRO_L2__AER_AI_20210806T085936_20210806T104106_19763_03_020400_20221026T213218.nc",
    "S5P_RPRO_L2__AER_AI_20210806T104106_20210806T122235_19764_03_020400_20221026T213219.nc",
    "S5P_RPRO_L2__AER_AI_20210806T122235_20210806T140405_19765_03_020400_20221026T213933.nc",
    "S5P_RPRO_L2__AER_AI_20210806T140405_20210806T154534_19766_03_020400_20221026T213938.nc",
    "S5P_RPRO_L2__AER_AI_20210806T154534_20210806T172704_19767_03_020400_20221026T213939.nc",
    "S5P_RPRO_L2__AER_AI_20210806T172704_20210806T190833_19768_03_020400_20221026T213943.nc",
    "S5P_RPRO_L2__AER_AI_20210806T190833_20210806T205003_19769_03_020400_20221026T213945.nc",
    "S5P_RPRO_L2__AER_AI_20210806T205003_20210806T223133_19770_03_020400_20221026T213953.nc",
    "S5P_RPRO_L2__AER_AI_20210806T223133_20210807T001302_19771_03_020400_20221026T213954.nc",
]
```


```python tags=["remove_output"]
eofetch.download(filenames)
```

## 4. Viewing the content of the UVAI file (optional) <a name="paragraph4"></a>

The TROPOMI data is read with HARP using `harp.import_product()` function. First as an example, we import one TROPOMI UVAI file and view its content. Remember to check that the path to the files is correct, pointing to the location of the UVAI files in your own computer, e.g.:

"/path/to/TROPOMI/S5P_RPRO_L2__AER_AI_20210806T071807_20210806T085936_19762_03_020400_20221026T213211.nc"

(Note that because the original netcdf file is large, importing the file might take a while.)




```python
filename = "S5P_RPRO_L2__AER_AI_20210806T071807_20210806T085936_19762_03_020400_20221026T213211.nc"
uvai_data = harp.import_product(filename)
```

After a succesfull import, you can view the contents of `uvai_data`:


```python
print(uvai_data)
```

You can inspect the information of a specific `uvai_data` variable (listed above), e.g. the `absorbing_aerosol_index` that we will be using in this notebook:

```python
print(uvai_data.absorbing_aerosol_index)
```

From the listing above you see e.g. that the `absorbing_aerosol_index` has no unit, and that the index values can be either negative or positive.

## 5. Regridding level 2 data into level 3 grid <a name="paragraph5"></a>

### Step 1: Operations in HARP import <a name="subparagraph1"></a>

**Filtering and gridding of TROPOMI Level 2 data is done by including specific operations to the `harp.import_product()` function**. Please, look at [Use Case 1](https://atmospherictoolbox.org/usecases/Usecase_1_S5P_SO2_La_Soufriere/) for more detailed explanation on setting up different HARP operations. In this case the following operations will be performed by HARP while the data is imported:

**1)** `absorbing_aerosol_index_validity>80` : we only consider pixels for which the data quality is high enough. The basic **quality flag** in any TROPOMI Level 2 netcdf file is given as `qa_value`. In the [Product Readme File for UV Aerosol Index](https://sentinels.copernicus.eu/documents/247904/3541451/Sentinel-5P-Aerosol-Level-2-Product-Readme-File) you can find, that the basic recommendation for UVAI data is to use only those pixels where `qa_value > 0.8`. In HARP the `qa_value` is renamed as `absorbing_aerosol_index_validity` and the value of 80 is used instead of 0.8.

**2)** `keep(latitude_bounds,longitude_bounds,datetime_start,datetime_length,absorbing_aerosol_index)`: keep only selected variable from the original netcdf files. HARP uses weighted area average to calculate the value for each new grid cell, and therefore the corner coordinates of each satellite pixel, provided by the latitude and longitude bounds are needed.

**3)**  `derive(datetime_stop {time} [days since 2000-01-01])` and `derive(datetime_start [days since 2000-01-01])`: derive new variables from the data.

**4)** `exclude(datetime_length)`: exclude variable that is not needed.

**5)** `bin_spatial(81,50,0.5,721,-180,0.5)` : define the common Level 3 grid to combine the data from multiple orbits from one day into a single daily grid.  We define a new grid at 0.5 degrees resolution over the area covering the Northern hemisphere between 50N and 90N, and -180E to 180E. More detailed explanation of `bin_spatial()` regridding operation can be found in [Use Case 1, Step 4](https://atmospherictoolbox.org/usecases/Usecase_1_S5P_SO2_La_Soufriere/).

**6)** `derive(latitude {latitude})`and `derive(longitude {longitude})`:  derive lat and lon of the new common grid.


**To apply all the HARP operations while importing the data, all the operation strings are joined together with python `join()` command.** The `operations` variable will be given as input to the `harp.import_product()` function.


```python
operations = ";".join([
    "absorbing_aerosol_index_validity>80",
    "keep(latitude_bounds,longitude_bounds,datetime_start,datetime_length,absorbing_aerosol_index)",
    "derive(datetime_stop {time} [days since 2000-01-01])",
    "derive(datetime_start [days since 2000-01-01])",
    "exclude(datetime_length)",
    "bin_spatial(81,50,0.5,721,-180,0.5)",
    "derive(latitude {latitude})",
    "derive(longitude {longitude})",
])
```

### Step 2: Create the merged product: reduce_operations <a name="subparagraph2"></a>


This example uses aerosol index data from 14 orbits and HARP will concatenate these products together. However, to arrive at a common lat/lon grid, HARP needs to reduce these 14 orbit grids into a **single grid**. This is done by adding the `reduce_operations` parameter to the `harp.import_product()` function. The `reduce_operations` parameter can be constructed in a similar manner as the operations parameter, by separating the strings with ";". To create the merged data set, the following reduced operations will be applied:

**1)** `squash(time, (latitude, longitude, latitude_bounds, longitude_bounds))`: remove the given dimension for the variable, assuming that the content for all items in the given dimension is the same.

**2)** `bin()`: to perform the actual merging i.e. to perform an averaging in the time dimension such that all samples in the same bin get averaged


```python
reduce_operations = "squash(time, (latitude, longitude, latitude_bounds, longitude_bounds));bin()"
```

### Step 3: Apply HARP import command to multiple input files <a name="subparagraph3"></a>

Now that all the elements for `harp.import_product()` function are defined, the import command needs to be applied to all 14 TROPOMI UVAI files. When the files have all been stored in the same folder, importing the data is simple, just give the path (if needed) and the common part of the file names.


```python
filenames = "S5P_RPRO_L2__AER_AI_20210806T*.nc"
```

When gridding and merging several TROPOMI files execution of `harp.import_product()` may take a while, especially if the grid covers large area and/or the grid cell size is small. Now, the whole import command can be executed as :


```python
merged = harp.import_product(filenames, operations, reduce_operations=reduce_operations)
```

You can view the contents of the merged data variable `merged` by:


```python
print(merged)
```

As the print of the variables show, the re-gridded `absorbing_aerosol_index` variable has now two dimensions (in addition to time), latitude (80) and longitude (720).


## 6. Plotting gridded level 3 data on a map <a name="paragraph6"></a>

We use cartopy and `pcolormesh` function to visualise the merged AAI data on a map. The parameter we want to plot is the `absorbing_aerosol_index`.

The corner coordinates of each grid cell are provided by the `latitude_bounds` and `longitude_bounds` variables and these are required by the `pcolormesh` as the input for latitude and longitude. The `merged.latitude_bounds.data[:,0]` array gives the latitudes of the grid cell lower corners, whereas `merged.latitude_bounds.data[:,1]` gives the latitudes for upper corners.

To get the correct input for `pcolormesh`, we define the `gridlat` variable by appending the `merged.latitude_bounds.data[:,0]` array with the last element of the second array `merged.latitude_bounds.data[-1,1]`. The `gridlon` variable is defined similarly. More detailed description on  `pcolormesh` inputs can be found [here](https://matplotlib.org/stable/api/_as_gen/matplotlib.pyplot.pcolormesh.html).


```python
gridlat = np.append(merged.latitude_bounds.data[:,0], merged.latitude_bounds.data[-1,1])
gridlon = np.append(merged.longitude_bounds.data[:,0], merged.longitude_bounds.data[-1,1])
```

The actual parameter to be plotted is `absorbing_aerosol_index`. For colorbar label also description of the aerosol index data are read. In this plot we use colormap named `devon` from the [cmcrameri library](https://www.fabiocrameri.ch/colourmaps/). The vmin and vmax are defined for scaling of the colormap values. Positive values of AAI (>1) indicate the presence of absorbing aerosols at elevated amounts (in this case smoke), and therefore we limit the colorscale with vmin=1.


```python
AAIval = merged.absorbing_aerosol_index.data
AAIdescription = merged.absorbing_aerosol_index.description

colortable = cm.devon_r

vmin = 1
vmax = 4
```

By using matplotlib `figsize` argument the figure size can be defined. `plt.axes` set up GeoAxes, and in this case the projection is `NorthPolarStereo` from cartopy. The area of interest is set by `set_extent` where the range for latitude and longitude for the plot are given. The actual data is plotted with `plt.pcolormesh` command. Note that the dimensions of the `absorbing_aerosol_index` are time, lat and lon, and therefore the input is given as `AAIval[0,:,:]`. Finally the colorbar is added with label text, and also the location of the colorbar is set.


```python
fig = plt.figure(figsize=(20,10))
ax = plt.axes(projection=ccrs.NorthPolarStereo())
ax.set_extent([-180, 180, 50, 90], ccrs.PlateCarree())

img = plt.pcolormesh(gridlon, gridlat, AAIval[0,:,:], vmin=vmin, vmax=vmax, cmap=colortable, transform=ccrs.PlateCarree())
ax.coastlines()
ax.gridlines()

cbar = fig.colorbar(img, ax=ax,orientation='horizontal', fraction=0.04, pad=0.1)
cbar.set_label(f'{AAIdescription}')
cbar.ax.tick_params(labelsize=14)
plt.show()
```
## 7.  Save gridded data as netcdf <a name="paragraph7"></a>

Using HARP you can also save the merged product into a new netcdf file. The data is saved by using HARP export command `harp.export_product`. The inputs are the merged product variable name and the name of the new netcdf product.


```python
harp.export_product(merged, 's5p-AAI-2021Aug06_wildfiresmoke.nc')
```

## 8. References to HARP documentation <a name="harp_references"></a>

- [HARP Ingestion definitions](http://stcorp.github.io/harp/doc/html/ingestions/index.html)

- [HARP operations documentation](http://stcorp.github.io/harp/doc/html/operations.html)

The variables in the HARP product that results from an ingestion of Level 2 UVAI product data are listed here:
- [HARP_S5P_L2_UVAI](http://stcorp.github.io/harp/doc/html/ingestions/S5P_L2_AER_AI.html)


