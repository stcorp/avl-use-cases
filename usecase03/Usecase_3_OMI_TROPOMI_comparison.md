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

![madrid_banner.png](https://raw.githubusercontent.com/stcorp/avl-use-cases/master/usecase03/madrid_banner.png)

# Comparison of OMI and TROPOMI spatial resolution of NO2 observations at  city scale


This use case compares the spatial resolution of the Ozone Monitoring Instrument (OMI) and the TROPOspheric Monitoring Instrument nitrogen dioxide (NO2) observations. OMI was launched onboard NASA's Aura satellite in 2004, and since that it has been providing observations on atmospheric composition now for nearly 18 years. TROPOMI was launched in 2017 onboard ESA Copernicus Sentinel 5p satellite. TROPOMI, as the successor of the OMI instrument, is very similar in design but has e.g. much higher spatial resolution. The spatial resolution of OMI NO2 data is 13km x 24 km at nadir, whereas for TROPOMI the resolution (since August 2019) is 5.5 km × 3.5 km. Both instruments are on an afternoon orbit and their local overpassing times are very close to each other. It should be noted that OMI has been suffering from a so called [row anomaly](https://aura.gesdisc.eosdis.nasa.gov/data/Aura_OMI_Level2/OMNO2.003/doc/README.OMNO2.pdf) that affects certain pixels of the orbit, and hence data can not be obtained for those pixels affected.

This notebook illustrates how to compare the spatial resolution of these two instruments at a city scale. The new features introduced in this notebook are:

- Reading NO2 data from two different satellite instruments
- Differences in quality filtering of NO2 data for TROPOMI and OMI
- Derivation of a new parameter from the original data for pixel area 
- Plotting pixel-based data on a map



### Table of contents

1. [Importing Python modules](#paragraph1)
2. [Obtaining data files for OMI and TROPOMI](#paragraph2)
3. [HARP import for OMI and TROPOMI NO2](#paragraph3)
4. [Creating GeoDataFrames for plotting](#paragraph4)
5. [Plotting data on a map](#paragraph5)
6. [References](#paragraph6)

### 1. Importing Python modules <a name="paragraph1"></a>

In this notebook new modules "folium", "branca", and "geopandas" are used. Folium is used as a basemap for plotting, and geopandas are used to modify the output data from HARP import.



```python
import harp
import numpy as np
import matplotlib.pyplot as plt
import branca
import folium
import pyproj
import geopandas as gpd
from shapely.geometry import Polygon
import eofetch
import avl
```

### 2. Obtaining data files for OMI and TROPOMI <a name="paragraph2"></a>


The TROPOMI data file can be downloaded automatically using the eofetch library as in [use case 2](https://atmospherictoolbox.org/media//usecases/Usecase_2_S5P_AAI_Siberia_Smoke.html#paragraph3). The TROPOMI file used in this exercise is:

*S5P_OFFL_L2_\_NO2_\_\_\_20210713T113312_20210713T131441_19424_02_020200_20210715T052115.nc*

The OMI file is downloaded from a cache of example data that is used for use cases like this. The original OMI data can be found on [NASA GES DISC](https://disc.gsfc.nasa.gov/). The specific data set can be found by searching "OMNO2". To download the data one needs to have registered to NASA Earthdata. Note that currently there are two data sets for Level 2 OMI NO2 data available, make sure that the data file is named OMNO2. The OMI file that is used in this example is:

*OMI-Aura_L2-OMNO2_2021m0713t1219-o90393_v003-2021m0824t153120.he5*


```python tags=["remove_output"]
filename_tropomi = "S5P_RPRO_L2__NO2____20210713T113312_20210713T131441_19424_03_020400_20221104T113602.nc"
filename_omi = "OMI-Aura_L2-OMNO2_2021m0713t1219-o90393_v003-2021m0824t153120.he5"
eofetch.download(filename_tropomi)
result = avl.download(filename_omi)
```

### 3. HARP import for OMI and TROPOMI NO2 <a name="paragraph3"></a>

The advantage of HARP is that for the most part same operations can be used to import both TROPOMI and OMI NO2 products because the naming conventions are similar even though the original file structures are different. Hence, for both data sets operations in HARP import are very similar, **the main difference is quality filtering which is different for OMI and TROPOMI**. For both data set the parameter we are interested in is called "tropospheric_NO2_column_number_density". For TROPOMI the data quality for each pixel is indicated with a qa value varying from 0 to 100 (or 0 to 1), whereas for OMI the data quality is indicated in bitwise manner. 

To follow the recommendations of the algorithm teams, the **following quality filters should be included in operations**:

- **TROPOMI NO2**: "tropospheric_NO2_column_number_density_validity>75"
- **OMI NO2**: "validity !& 1"

For this example we are interested only data in the vicinity of Madrid, therefore the imported data is restricted to area lat 39.5N...41N, lon -5W...-2W. **In the operations we also derive a new parameter, pixel area as km^2**, that can be derived from latitude and longitude bounds:

- "derive(area {time} [km2])"; where area is the name of the parameter, time is the dimension of the imported data and km2 are the units.





```python
operations_trop = ";".join([
    "tropospheric_NO2_column_number_density_validity>75",
    "latitude>39.5",
    "latitude<41",
    "longitude<-2",
    "longitude>-5",
    "keep(latitude_bounds,longitude_bounds,tropospheric_NO2_column_number_density)",
    "derive(tropospheric_NO2_column_number_density [Pmolec/cm2])",
    "derive(area {time} [km2])",
])
```


```python
tropomi_NO2 = harp.import_product(filename_tropomi, operations=operations_trop)
```


```python
print(tropomi_NO2)
```

```python
operations_omi = ";".join([
    "validity !& 1",
    "latitude>39.5",
    "latitude<41",
    "longitude<-2",
    "longitude>-5",
    "keep(latitude_bounds,longitude_bounds,tropospheric_NO2_column_number_density)",
    "derive(tropospheric_NO2_column_number_density [Pmolec/cm2])",
    "derive(area {time} [km2])",
])
```


```python
omi_NO2 = harp.import_product(filename_omi, operations=operations_omi)
```


```python
print(omi_NO2)
```

### 4. Creating GeoDataFrames for plotting  <a name="paragraph4"></a>

This use case uses the [folium module](https://python-visualization.github.io/folium/quickstart.html) as a basemap for plotting. Folium makes it easy to visualize data that’s been manipulated in Python on an interactive Leaflet map. For plotting purposes the HARP-imported OMI and TROPOMI data are put into GeoDataFrames. 

**TROPOMI GeoDataFrame:**

First step is to create an empty GeoDataFrame for TROPMI data. For GeoDataFrame you need to define the The Coordinate Reference System (crs) that defines how the coordinates relate to places on the Earth. For that first an empty column named "geometry" is created to the data frame, and then the crs is set.



```python
crs = 'epsg:4326'
tropomi = gpd.GeoDataFrame(geometry=gpd.GeoSeries())
tropomi.crs = crs
```

The loop over imported TROPOMI data creates a polygon from each individual pixel using latitude and longitude bounds. Each pixel is represented as a polygon having a corresponding value for tropospheric NO2 and pixel area, as well as pixel id. 


```python
# index for the for loop to go through all TROPOMI pixels
idx = tropomi_NO2.latitude_bounds.data.shape[0]

# loop over pixels
for x in range(idx):
    lat_c = tropomi_NO2.latitude_bounds.data[x]
    lon_c = tropomi_NO2.longitude_bounds.data[x]
    poly = Polygon(zip(lon_c, lat_c))
    tropomi.loc[x, 'pixel_id'] = x
    tropomi.loc[x, 'geometry'] = poly
    tropomi.loc[x, 'NO2'] = tropomi_NO2.tropospheric_NO2_column_number_density.data[x]
    tropomi.loc[x, 'pixelarea'] = tropomi_NO2.area.data[x]

# print the tropomi GeoDataFrame
print(tropomi)    
```

The GeoDataFrame for OMI is created similar way:


```python
omi = gpd.GeoDataFrame(geometry=gpd.GeoSeries())
omi.crs = crs

idx2 = omi_NO2.latitude_bounds.data.shape[0]

# loop over pixels
for x in range(idx2):
    lat_c = omi_NO2.latitude_bounds.data[x]
    lon_c = omi_NO2.longitude_bounds.data[x]
    poly = Polygon(zip(lon_c, lat_c))
    omi.loc[x, 'pixel_id'] = x
    omi.loc[x, 'geometry'] = poly
    omi.loc[x, 'NO2'] = omi_NO2.tropospheric_NO2_column_number_density.data[x]
    omi.loc[x, 'pixelarea'] = omi_NO2.area.data[x]

# print the tropomi GeoDataFrame
print(omi)    
```

Next we can compare the pixel areas of TROPOMI and OMI by plotting the "pixelarea" variable 


```python
fig = plt.figure(figsize=(20, 10))
plt.subplot(121)
plt.hist(tropomi.pixelarea, bins=range(25, 550, 15))
plt.xticks(np.arange(0,550, step=50))
plt.xlabel('Pixel area [km^2]')
plt.title('TROPOMI')

plt.subplot(122)
plt.hist(omi.pixelarea, bins=range(25, 550, 15))
plt.xticks(np.arange(0,550, step=50))
plt.xlabel('Pixel area [km^2]')
plt.title('OMI')

plt.show()
```
### 5. Plotting data on a map  <a name="paragraph5"></a>

In this use case we are illustrating the satellite data resolution at a city scale, and therefore we are using folium module for plotting. For the folium map the center location is given to Madrid and zoom_start level is set to 10. In the first plot both omi and tropomi pixels are plotted as polygons.  The styles (namely color of pixel edges) for each data (omi=plum, tropomi=purple) are defined with style function. The map can be zoomed.   


```python
madrid_map = folium.Map(location=[40.416775, -3.703790], zoom_start=8)

style1 = {'fillColor': 'none', 'color': '#dd3497'}
style2 = {'fillColor': 'none', 'color': '#4a1486'}

folium.GeoJson(tropomi, style_function=lambda x:style1).add_to(madrid_map)
folium.GeoJson(omi, style_function=lambda x:style2).add_to(madrid_map)
display(madrid_map)
```


We can also plot choropleth maps, where each pixel is colored either by the value of pixel area or tropospheric NO2. In this case, since all the needed information (geometry, values) is included in the "omi" and "tropomi" GeoDataFrames, we can use those directly to create a choropleth map. The input that is needed for choropleth includes:

- geo_data: geopandas dataframe, that includes pixel geometry. In this example it is either "omi" or "tropomi"
- data: dataframe of the values we want to show. In this case an additional dataframe is not needed, we will use "omi" or "tropomi" 
- columns: column names from the input dataframe that give the keys and value information. Key in this case is variable "pixel_id", and value can be either "NO2" or "area".


First for choropleth map we choose a gray-scaled basemap, and define the colormap for NO2 values. As a colormap we use ColorBrewer palette.    


```python
map_omi = folium.Map(location=[40.416775, -3.703790], zoom_start=8, tiles='cartodbpositron')

colormap = branca.colormap.linear.YlGnBu_09.scale(0, 8)
colormap.caption = 'Tropospheric NO2 [Pmol/cm2]'  

style_function = lambda x: {"weight": 0.5, 
                            'color': ' ',
                            'fillColor': colormap(x['properties']['NO2']), 
                            'fillOpacity': 0.6}
```
```python
folium.GeoJson(
    omi,
    tooltip=folium.GeoJsonTooltip(fields=["pixel_id","NO2"]),
    style_function=style_function
).add_to(map_omi)

# Add the legend to the same canvas
colormap.add_to(map_omi)
display(map_omi)
```


### 

Similarly, the TROPOMI NO2 values can be plotted on the map, using same styling and colorscale:


```python
map_tropomi = folium.Map(location=[40.416775, -3.703790], zoom_start=8, tiles='cartodbpositron')

folium.GeoJson(
    tropomi,
    tooltip=folium.GeoJsonTooltip(fields=["pixel_id", "NO2"]),
    style_function=style_function
).add_to(map_tropomi)
colormap.add_to(map_tropomi)
display(map_tropomi)
```


### 6. References<a name="paragraph6"></a>

- [HARP ingestion of OMI NO2](https://stcorp.github.io/harp/doc/html/ingestions/OMI_L2_OMSO2.html)
- [HARP ingestion of TROPOMI NO2](https://stcorp.github.io/harp/doc/html/ingestions/index.html#s5p-l2-no2)
- [NASA GES DISC](https://disc.gsfc.nasa.gov/)
- [OMI NO2 Readme](https://aura.gesdisc.eosdis.nasa.gov/data/Aura_OMI_Level2/OMNO2.003/doc/README.OMNO2.pdf)
- [OMI Data User Guide](https://docserver.gesdisc.eosdis.nasa.gov/repository/Mission/OMI/3.3_ScienceDataProductDocumentation/3.3.2_ProductRequirements_Designs/README.OMI_DUG.pdf)
- [TROPOMI NO2 Readme](https://sentinel.esa.int/documents/247904/3541451/Sentinel-5P-Nitrogen-Dioxide-Level-2-Product-Readme-File)
- [TROPOMI Product User Manual](https://sentinel.esa.int/documents/247904/2474726/Sentinel-5P-Level-2-Product-User-Manual-Nitrogen-Dioxide)
