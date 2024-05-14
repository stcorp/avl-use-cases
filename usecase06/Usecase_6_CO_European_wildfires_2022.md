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

![header_case6.png](https://raw.githubusercontent.com/stcorp/avl-use-cases/master/usecase06/header_case6.png)

## Demonstrating new plotting functions in Atmospheric Virtual Lab


This notebook demonstrates the new plotting functionalities in the `atmospheric-virtual-lab` (avl) module. The `atmospheric-virtual-lab`is a toolkit developed for interactive vislualization of atmospheric satellite data.  

Here we demonstrate the MapPlot and MapPlot3D commands, and create **interactive maps** of Copernicus Sentinel 5p / **TROPOMI carbon monoxide  (CO)** observations from European wildfires on 18th July 2022. 
These  plotting commands create a Jupyter widget called ipyleaflet which a very convenient way to instantly visualize HARP products, including single orbits. 

### Table of contents
1. [Create new conda environment for avl](#paragraph1)
2. [Import S5p TROPOMI CO data with harp](#paragraph2)
3. [Plotting 2D map with avl ](#paragraph3)
4. [Plotting 3D map with avl ](#paragraph4)
5. [References](#paragraph5)

### 1. Create new conda environment for avl <a name="paragraph1"></a>

The easiest way to get started with `atmospheric-virtual-lab` is to create a new conda environment and pre-install the required packages. Here we create a new conda environment called `avl_env`, and install required packages as follows:

`$ conda create -c conda-forge -n avl_env atmosphere-virtual-lab`

Before running the jupyter lab and this notebook the environment needs to be activated:

`$ conda activate avl_env`

And after that the Jupyter lab can be launched: 

`$ jupyter-lab` 


### 2. Import S5p TROPOMI CO data with harp <a name="paragraph2"></a>

First we need to import the needed packages, and define the CO file that we want to plot. The TROPOMI CO file can be downloaded from [Sentinel-5P Pre-Operations Data Hub](https://s5phub.copernicus.eu/dhus/).
We are using the sentinelsat python package here to perform the download for us.


```python
# atmosphere-virtual-lab is imported with the name avl
import avl 
import harp
import eofetch

filename_tropomi = 'S5P_RPRO_L2__CO_____20220718T123656_20220718T141826_24674_03_020400_20230113T012004.nc'
eofetch.download(filename_tropomi)
```

Next the operations for importing the TROPOMI CO product are defined, and the data is imported with harp. In this example we will use the `carbonmonoxide_total_column_corrected` variable instead of the default  `carbonmonoxide_total_column`. From TROPOMI CO processor version 2.x.x onwards the fixed masked destriping method has been implemented to correct for the "stripes" visible in the CO observations (probably due to calibration issues), and a new parameter `carbonmonoxide_total_column_corrected` has been included. To use this new parameter, in harp import a new option of `co=corrected` needs to be added. The recommended quality assurance value for CO is qa>0.5 (with harp 50), but it is also recommended to take a look of the different qa value definitions explained in the [product Readme file](https://sentinels.copernicus.eu/documents/247904/3541451/Sentinel-5P-Carbon-Monoxide-Level-2-Product-Readme-File).


```python
operations = ";".join([
    "CO_column_number_density_validity>50",
    "derive(CO_column_volume_mixing_ratio {time} [ppbv])",
    "keep(latitude_bounds,longitude_bounds,CO_column_number_density,CO_column_volume_mixing_ratio)"
    ])
    
tropomi_co=harp.import_product(filename_tropomi, operations, options="co=corrected")
```


```python
print(tropomi_co)
```

### 3. Plotting 2D map with avl <a name="paragraph3"></a>

First the imported harp object will be plotted on a 2D interactive map. For this we use `avl.Geo` command. Within the command we can define the colormap (e.g. default matplotlib color maps), min and max values of the color range as well as the opacity of the overlayed TROPOMI data. Once the map is plotted, the colorscale can be also adjusted manually, by clicking on the palette icon at the lower right corner of the map. In this example we will plot the CO data as ppbv, i.e. use the variable `CO_column_volume_mixing_ratio`.   



```python
variable='CO_column_volume_mixing_ratio'
avl.Geo(tropomi_co, variable,colormap="rainbow",colorrange=(0, 200), opacity=0.7)
```

Now you can zoom in the interactive map to see the CO emission plumes from the wildfires in France, Spain, and Portugal. The data is automatically overlayed on top of the OpenStreetMap so that locating the plumes geographically e.g. close to cities is easy. The zoom level as well as the center latitude and longitude can be also set in the plotting command:     


```python
avl.Geo(tropomi_co, variable,colormap="rainbow",colorrange=(0, 200), opacity=0.7, centerlat=44, centerlon=-10, zoom=5)
```
### 4. Plotting 3D map with avl <a name="paragraph4"></a>

Next we will use the `avl.Geo3D` command to plot the TROPOMI CO data onto an interactive 3D globe basemap. The `avl.Geo3D` command is very similar to `avl.Geo`:


```python
avl.Geo3D(tropomi_co,variable,colorrange=(0, 200),colormap="rainbow", opacity=0.7,showcolorbar=True)
```
### 5. References <a name="paragraph5"></a>

- [Github demos of AVL plotting](https://github.com/stcorp/avl-demo-lps2022)
- [TROPOMI CO Readme File](https://sentinels.copernicus.eu/documents/247904/3541451/Sentinel-5P-Carbon-Monoxide-Level-2-Product-Readme-File)  
- [TROPOMI CO ATBD](https://sentinel.esa.int/documents/247904/2476257/Sentinel-5P-TROPOMI-ATBD-Carbon-Monoxide-Total-Column-Retrieval.pdf)

