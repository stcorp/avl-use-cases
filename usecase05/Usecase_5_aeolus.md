---
jupyter:
  jupytext:
    formats: md,ipynb
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.2
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

![usecase5_banner.png](https://github.com/stcorp/avl-use-cases/raw/master/usecase5/usecase5_banner.png)
# Reading and plotting Aeolus wind profile data with updated Atmospheric Virtual Laboratory


This use case demonstrates the use of `Atmospheric Vitual Laboratory` functionalities for reading and plotting **wind profile data from Aeolus**. The Aeolus satellite was launched in 2018 and it carries a Doppler wind lidar Aladin that is able to provide profile information on winds, aerosols and clouds for the lowermost 30 km of the Earth's atmosphere. In this example **Level 2B Rayleigh wind profile product** is demonstrated using data from 29th January 2022, when a strong low pressure system was in the northern part of Europe, and high pressure in the south.


## Table of contents

1. [Obtaining Level 2B Aeolus Scientific wind product](#paragraph1)
2. [Python packages for the notebook](#paragraph2)
3. [Read wind data using HARP](#paragraph3)
4. [Plot the HLOS wind data using AVL](#paragraph4)
5. [Optional: HLOS wind data plotting without AVL](#paragraph5)
6. [References](#harp_references)



## 1. Obtaining Level 2B Aeolus Scientific wind product <a name="paragraph1"></a>

The Aeolus Scientific **L2B Rayleigh/Mie wind product** provides geo-located **HLOS (horizontal line-of-sight)** wind profile. HLOS is the the wind component along satellite's horizontal line-of-sight, which is approximately zonally oriented but not exactly the same as commonly used zonal ("U") wind  component. An illustration of the HLOS wind component definition can be found e.g. from [Krisch et al.2022](https://amt.copernicus.org/articles/15/3465/2022/):

![example_aerolus_HLOS_Krisch_2022.png](https://github.com/stcorp/avl-use-cases/raw/master/usecase5/example_aerolus_HLOS_Krisch_2022.png)

The Aeolus level 2B scientific wind data can be downloaded from several sources: 
- ESA [HTTP browser](https://aeolus-ds.eo.esa.int/oads/access/), 
- [FTP client](https://earth.esa.int/eogateway/catalog/aeolus-scientific-l2b-rayleigh-mie-wind-product) or 
- via [VirES data visualisation platform](https://aeolus.services/).

The links to access the data as well as data desriptions can be found from [here](https://earth.esa.int/eogateway/catalog/aeolus-scientific-l2b-rayleigh-mie-wind-product). **It is important to note that the files need to be downloaded in their native format as ".DBL"**, since this is the file format supported by HARP for Aeolus. If the VirES service is used for data download, the users should use the "Package original files" option to obtain the .DBL files instead of netcdf.

This notebook uses Aeolus Rayleigh wind data from 29th January 2022. The specific file is:
`AE_OPER_ALD_U_N_2B_20220129T152853_20220129T170004_0001.DBL`


## 2. Python packages for the notebook <a name="paragraph2"></a>

For this notebook `HARP` version 1.17 (or newer) is needed. In order to get the proper versions of packages it is recommended to re-create your conda environment using [this environment.yml file](https://github.com/stcorp/avl-use-cases/blob/master/environment.yml).

You can create the conda environment using `conda env create -f environment.yml`. After running this command you can activate the new environment by `conda activate avl` and open jupyter-lab.


```python
import avl
import harp
```

## 3. Read wind data using HARP <a name="paragraph3"></a>

First, the operations for `harp.import_product` are defined that are used to ingest data from the Aeolus wind product. These include the selection of the latitude and longitude limits for the area of interest, as well as the validity check for the data. The following variables that result from ingestion with HARP of the Aeolus L2B Rayleigh wind data are kept in the imported product:
- datetime_start: start datetimes of the measurements for the wind profile
- datetime_stop: stop datetimes of the measurements for the wind profile
- latitude and longitude: average latitude and longitude of the measurements used for the wind profile
- hlos wind_velocity: HLOS wind velocity
- altitude_bounds: altitude relative to geoid of layer boundaries for each accumulation 

It is noted that the default ingestion option for variable `hlos_wind_velocity` is the Rayleigh HLOS wind profile. Finally with `derive` the units of wind velocity and altitude are converted into m/s and km, respectively.

```python
file_in="AE_OPER_ALD_U_N_2B_20220129T152853_20220129T170004_0001.DBL"

operations = ";".join([
    "latitude>35;latitude<68",
    "longitude>0;longitude<20",
    "hlos_wind_velocity_validity>0", 
    "keep(datetime_start,datetime_stop,latitude,longitude,hlos_wind_velocity, altitude_bounds)",
    "derive(hlos_wind_velocity [m/s])",
    "derive(altitude_bounds [km])"
])
```

For this specific product, we can download it from the AVL example data archive:

```python
result = avl.download(file_in)
```

After defining the operations, the Aeolus file is imported as:

```python
windproduct=harp.import_product(file_in, operations)
print(windproduct)
```

If you want to visualise the location of the Aeolus data in this example, the track of the Aeolus satellite can be plotted on a map as follows:

```python
import matplotlib.pyplot as plt
import cartopy.crs as ccrs

latc=windproduct.latitude.data
lonc=windproduct.longitude.data

fig = plt.figure(figsize=(5, 10))
ax = plt.axes(projection=ccrs.PlateCarree())

img = plt.plot(lonc, latc,color='blue',marker='.',linestyle='None', transform=ccrs.PlateCarree())
ax.coastlines()

# Mark the imported product start (blue) and end (red) 
plt.plot(lonc[0],latc[0],markersize=8, marker='o', color='cyan', label='start')  # start
plt.plot(lonc[len(lonc)-1],latc[len(latc)-1],markersize=8, marker='o', color='red', label='end')  # stop

gl = ax.gridlines(draw_labels=True)
gl.top_labels = False
gl.right_labels = False

ax.legend()

plt.show()
```

## 4. Plot the HLOS wind data using AVL <a name="paragraph4"></a>


The wind profiles are plotted as so called curtain plot, where y-axis is the altitude and x-axis is the sensing time (or geolocation). The "curtain colors" indicate both HLOS wind speed as well as the direction (eastward -> positive/ westward -> negative). Since the vertical altitude bounds are changing between the sensing times, the grid cell sizes are not constant. With AVL the curtain plot can be created with just one command `avl.Curtain()`. The inputs to the function are HARP imported data (`windproduct`), parameter to be plotted (`hlos_wind_velocity`), color range and map. The colormap is from the [cmcrameri colection](https://www.fabiocrameri.ch/colourmaps-userguide/).

```python
avl.Curtain(windproduct, 'hlos_wind_velocity', colorrange=(-40,40), colormap="roma")
```

As a result, we get a plot of the Rayleigh HLOS wind profiles along the satellite track, from south towards the north. The color describes the speed and direction of the wind, positive values indicating the eastward component.


## 5. Optional: HLOS wind data plotting without AVL <a name="paragraph5"></a>


We demonstrate here the steps that are needed to plot the curtain plot without the `avl.curtain()` command.
The following Python modules are needed: `numpy`, `matplotlib`, and `netCDF4`:

```python
import numpy as np
from matplotlib.collections import PolyCollection
import matplotlib.cm as cm
import matplotlib.colors as colors
from cmcrameri import cm
from netCDF4 import num2date
```

Since the vertical altitude bounds are changing between the sensing times, the grid cell sizes are not the same troughout the dataset. Therefore each grid cell is plotted as a polygon. For this the `PolyCollection`from matplotlib was imported and `polygons` is used. The x-axis information is obtained from `datetime_start` and `datetime_stop`, which contain edge points of the datetime bin (i.e., x-bin). The edge points of the y-bin are given in the variable `altitude_bounds`.

The conversion of measuring time from `[seconds since 2000-01-01]` to UTC `[hours:minutes:seconds]` is done using the `num2date` from the netCDF4 Python module.

The x-axis bin edges are defined as follows:

```python
Xstart = windproduct.datetime_start.data  # X-axis bin start points (1D)
Xstop = windproduct.datetime_stop.data  # X-axis bin end points (1D)
```

The next step is to add corresponding number of vertical layers of each time step:

```python
numoflayers=len(windproduct.altitude_bounds.data[0])
X0 = []
X1 = []
a = np.ones(numoflayers)
for i in range(len(Xstart)):
    X0 = np.append(X0,Xstart[i]*a)
    X1 = np.append(X1,Xstop[i]*a)
```

The Y-axis using altitude bounds is defined as follows, and finally the patches can be obtained:

```python
# Altitudes
Ystart = windproduct.altitude_bounds.data[:,:,0]  # start points, 2D
Yend = windproduct.altitude_bounds.data[:,:,1]  # end points, 2D
Y0 = np.ravel(Ystart)  # 2D -> 1D
Y1 = np.ravel(Yend)    # 2D -> 1D

patches = []
for x0,x1,y0,y1 in zip(X0, X1, Y0, Y1):
    patches.append(((x0,y0),(x0,y1),(x1,y1),(x1,y0)))
```

Before actual plotting, the wind velocity data needs to be flattened from 2D to 1D to match the patches:

```python
windvelocity = windproduct.hlos_wind_velocity.data
wind = np.ravel(windvelocity)  # 2D -> 1D
```

For plotting the time is represented in a different format:

```python
# Format datetime to hours:minutes
def format_date(x, pos=None):
        dt_obj = num2date(x, units="s since 2000-01-01", only_use_cftime_datetimes=False)
        return dt_obj.strftime("%H:%M")
```

Finally the wind data can be plotted as:

```python
windunits = windproduct.hlos_wind_velocity.unit
winddescription = windproduct.hlos_wind_velocity.description
altunits = windproduct.altitude_bounds.unit

fig, ax = plt.subplots()
coll = PolyCollection(
    patches,
    array = wind,
    cmap = cm.roma,
    norm = colors.Normalize(
        vmin = -40,
        vmax = 40,
        clip = False,
    ),
)
ax.add_collection(coll)

x0min = min(X0)
x1max = max(X1)

cbar = fig.colorbar(coll,ax=ax, orientation='vertical')
cbar.set_label(f'({windunits})')
ax.set_xlim([x0min,x1max]) 
ax.set_ylim([0,25])

ax.set_title("Rayleigh " f'{winddescription}')
ax.set_ylabel("Altitude ({})".format(altunits))
ax.set_xlabel("Time (H:M)")

plt.xticks(rotation=45)
ax.xaxis.set_major_formatter(format_date)
ax.autoscale
ax.grid()

plt.show()
```

## 6. References <a name="harp_references"></a>

- [HARP AEOLUS_L2B_Rayleigh](http://stcorp.github.io/harp/doc/html/ingestions/AEOLUS_L2B_Rayleigh.html)
- [HARP operations documentation](http://stcorp.github.io/harp/doc/html/operations.html)
- [HARP ingestion definitions](https://stcorp.github.io/harp/doc/html/ingestions/index.html)
