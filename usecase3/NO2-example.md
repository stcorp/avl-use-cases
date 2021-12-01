---
jupyter:
  jupytext:
    formats: md,ipynb
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.13.1
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

# TROPOMI and OMI NO2 gridding and regridding

## Table of Contents
1. **Obtaining the data files**
2. **Using HARP to read and grid the data**
3. **Plotting the gridded data**
4. **Regridding and replotting the data**

## Obtaining the data files

In this example, we'll use one orbit of TROPOMI and three orbits of OMI. The TROPOMI data file can be downloaded automatically using the ```sentinelsat``` library, but the OMI files need to be downloaded manually from https://aura.gesdisc.eosdis.nasa.gov/data/Aura_OMI_Level2/OMNO2.003/2021/182/. You need Earthdata Login to download the files.


```python
import os
import sentinelsat
import harp

filename_tropomi = "S5P_OFFL_L2__NO2____20210701T115854_20210701T134024_19254_01_010400_20210703T054933.nc"
filenames_omiaura = ["OMI-Aura_L2-OMNO2_2021m0701t1334-o90219_v003-2021m0824t151422.he5",
                     "OMI-Aura_L2-OMNO2_2021m0701t1155-o90218_v003-2021m0824t151349.he5",
                     "OMI-Aura_L2-OMNO2_2021m0701t1016-o90217_v003-2021m0824t151358.he5"]

if not os.path.exists(filename_tropomi):
    api = sentinelsat.SentinelAPI('s5pguest', 's5pguest', 'https://s5phub.copernicus.eu/dhus', show_progressbars=False)
    api.download_all(api.query(filename=filename_tropomi))
```

## Using HARP to read and grid the data

Let's now read the data we downloaded. HARP is very convenient when reading data from multiple different sources, because we can use the same functions regardless of the data source.

First, let's define a global 2 degrees by 2 degrees grid into which we read our data. The are the ```lat_grid``` and the ```lon_grid``` variables. 

Secondly, we have to define the HARP operations we use on our data. The ```keep``` operation denotes the variables we want as the data file will contain a wide range of those. Here we just want the ```latitude```, ```longitude``` and ```NO2_column_number_density``` variables.

On the next row, we do the gridding. Since our spatial grid contains quite a many points, we don't want to type it all out as a HARP command, but instead we convert the NumPy array into a tuple and that furthermore into a string. Feel free to print out the operations to see how it works out. This gridding modified the data file so that the ```latitude``` and ```longitude``` variables are changed into ```latitude_bounds``` and ```longitude_bounds```

Finally we convert the data into molecules per cm^2. This is important step, because even if the variables are the same, then the units might differ between data sources. Natively these OMI files contain the NO2 column number density in molecules per cm^2 but the TROPOMI's unit is moles per m^2. 

Then, because the OMI data is composed of multiple orbits, we reduce them into a single array. This is done using the reduction operation ```squash``` which removes the time dimension. The reduce operations are done after each individual file is read using the normal operations as defined previously.

Finally, the data itself, its description and its unit are extracted from the HARP product.


```python
import numpy as np
lat_grid = np.linspace(-90, 90, 90)
lon_grid = np.linspace(-180,180,180)

operations = ";".join([
    "keep(latitude,longitude,NO2_column_number_density)",
    "bin_spatial(%s,%s)" % (str(tuple(lat_grid)),str(tuple(lon_grid))),
    "derive(NO2_column_number_density [molec/cm^2])",
])

reduce_ops = "squash(time, (latitude_bounds, longitude_bounds));bin()"

product_tropomi = harp.import_product(filename_tropomi,operations)
product_omiaura = harp.import_product(filenames_omiaura,operations,reduce_operations=reduce_ops)

no2_omiaura = product_omiaura.NO2_column_number_density.data
no2_tropomi = product_tropomi.NO2_column_number_density.data

no2_description = product_tropomi.NO2_column_number_density.description
no2_unit = product_tropomi.NO2_column_number_density.unit
```

## Plotting the gridded data

Now we plot the gridded data. We use the Matplotlib's ```imshow``` function to visualize the gridded data and cartopy to draw the coastlines of the planet. 

The ```imshow``` function is very convenient for displaying data in a uniform grid. The ```extent``` argument defines the boundary coordinates for the whole image, as opposed to ```pcolormesh``` where each pixel's extent can be defined separately.


```python
import matplotlib.pyplot as plt
import cartopy.crs as ccrs

def plot_no2_map(no2_map, title_string, lims):
    fig = plt.figure(figsize=(20, 10))
    
    ax = plt.axes(projection=ccrs.PlateCarree())
    
    img = plt.imshow(no2_map,origin='lower',extent=(-180,180,-90,90),vmin=lims[0],vmax=lims[1])

    ax.coastlines()

    cbar = fig.colorbar(img, ax=ax, orientation='horizontal', fraction=0.04, pad=0.1)
    plt.title(title_string)
    cbar.ax.tick_params(labelsize=14)
    plt.show()
    
lims = [0.0, 1e16]
plot_no2_map(no2_tropomi[0,:,:], f'2x2 degree TROPOMI {no2_description} [{no2_unit}]', lims)
plot_no2_map(no2_omiaura[0,:,:], f'2x2 degree OMI {no2_description} [{no2_unit}]', lims)
```
## Regridding and replotting the data

We see that same kinds of features can be observed over Europe and Africa, but how do they compare? We can plot the difference of these products using the same function as before. Only the pixels where both of the instruments have data are plotted.

If we wish to do away with the empty streaks in the OMI data, we can do a more coarser gridding. It is possible to further process the data product with the HARP's ```execute_operations``` function. We'll define a 6 degrees by 6 degrees grid and regrid the data using the ```rebin``` HARP operation.

We can then plot the regridded data and the difference using the same function we defined before.


```python
difference_lims = [-0.3e16,0.3e16]
plot_no2_map(no2_tropomi[0,:,:] - no2_omiaura[0,:,:],'2 degrees by 2 degrees TROPOMI - OMI difference',difference_lims)

lat_grid_coarse = np.linspace(-90, 90, 30)
lon_grid_coarse = np.linspace(-180,180,60)

regrid_operations = ";".join([
    "rebin(latitude, latitude_bounds [degree_north], %s)" % str(tuple(lat_grid_coarse)),
    "rebin(longitude, longitude_bounds [degree_east], %s)" % str(tuple(lon_grid_coarse))
])

no2_tropomi_coarse = harp.execute_operations(product_tropomi,regrid_operations).NO2_column_number_density.data
no2_omiaura_coarse = harp.execute_operations(product_omiaura,regrid_operations).NO2_column_number_density.data

plot_no2_map(no2_tropomi_coarse[0,:,:], f'6x6 degree TROPOMI {no2_description} [{no2_unit}]', lims)
plot_no2_map(no2_omiaura_coarse[0,:,:], f'6x6 degree OMI {no2_description} [{no2_unit}]', lims)
plot_no2_map(no2_tropomi_coarse[0,:,:] - no2_omiaura_coarse[0,:,:],'2 degrees by 2 degrees TROPOMI - OMI difference',difference_lims)

```
