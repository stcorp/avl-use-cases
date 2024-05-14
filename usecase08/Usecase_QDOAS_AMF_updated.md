---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.16.1
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

![HCHO_banner_20230608.jpg](attachment:094ad460-cc68-4d13-b4bb-de715a4f9b2c.jpg)
##  Demonstration of BeAMF tool to calculate Air Mass Factors

This use case demonstrates the use of `BeAMF` which is a tool to calculate **Air Mass Factor (AMF)** for retrieved trace gas slant column density produced by [**QDOAS**](https://uv-vis.aeronomie.be/software/QDOAS/). The QDOAS is a cross-platform software which performs DOAS retrievals of trace gases from spectral measurements. The `BeAMF` tool can be used to calculate AMF and to make the conversion from trace gas slant column density (SCD) to vertical column density (VCD). 

In this example  **formaldehyde (HCHO)** data retrieved using QDOAS and TROPOMI (S5P) L1B data is used. The interest is to observe enhanced levels of HCHO in Canada in June 2023, when large parts of the country were affected by series of wildfires. This use case does not include the DOAS-fit part, the starting point for this notebook is the slant column densities obtained from QDOAS. 


## Table of contents

1. [Initial preparations for the notebook](#paragraph1)
2. [Preparing input for BeAMF: converting QDOAS output](#paragraph2)
3. [Preparing input for BeAMF: adding auxiliary parameters](#paragraph3)
4. [Air Mass Factor calculation with BeAMF](#paragraph4)
5. [Plot HCHO data](#paragraph5)
6. [References](#references)


<!-- #region -->
### 1. Initial preparations <a name="paragraph1"></a>

**A. Input files**

In this tutorial the input data is based on the TROPOMI radiance spectra measured on 8 June 2023 (orbit 29289) for HCHO, **that has been already retrieved with QDOAS**. In addition, corresponding operational S5P HCHO L2 file is needed to obtain auxiliary input parameters for BeAMF. The QDOAS output file that is provided with this notebook contains typical DOAS output structure with some general information and specific information per fitting window. 

This notebook and BeAMF requires the following files:
- **QDOAS output for HCHO:** S5P_OFFL_L1B_RA_BD3_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc
   
- **Configuration file for the BeAMF tool:** harp_hcho.json
  
- **Box-AMF and Radiance Look-up-table (LUT):** LUT_AMF_I_300nm_500nm_S1_US_small_new.nc

- **Operational TROPOMI L2 HCHO file:** S5P_OFFL_L2__HCHO___20230608T193244_20230608T211414_29289_03_020401_20230610T184103.nc


**B. Python packages**

**QDOAS is  included in the most recent version atmospheric virtual laboratory (avl package)**, and hence additional installations for QDOAS are not needed. However, you should check that you have the needed qdoas components installed by running `conda list` in your environment. If your list includes `beamf`, `qdoas` and `qdoas2harp` you are good to go, otherwise avl needs to be updated.  

The following Python packages are needed for running this notebook:
- `harp` (version 1.20 or newer)
- `avl` (atmosphere-virtual-lab, version 0.5 or newer)
- `eofetch` (for downloading of operational TROPOMI HCHO L2 file) 

For more info on eofetch see e.g. [eofetch README](https://github.com/stcorp/eofetch#readme) or [eofetch example](https://atmospherictoolbox.org/media/usecases/Usecase_2_S5P_AAI_Siberia_Smoke.html).
<!-- #endregion -->

```python
import harp
import avl
import eofetch
```

### 2. Preparing input for BeAMF: converting QDOAS output <a name="paragraph2"></a>

To convert QDOAS output to HARP compliant format for BeAMF, the `qdoas2harp` command line tool is used. The command `qd2hp` performs conversion and it is used as follows:

(avl_env)%  qd2hp -outdir /path/to/outdir/ -fitwin w200_bro_rad -slcol "ch2o=HCHO" S5P_OFFL_L1B_RA_BD3_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc

where
- `-outdir`: path for the output file.
- `-fitwin`: **fitting window** appropriate for HCHO, here **w200_bro_rad** is used that is given in our QDOAS output file. Please check which fitting window to use if using other QDOAS file.
- `-slcol`: **slant column** and "ch2o=HCHO" specify which symbol indicates formaldehyde.
- S5P_OFFL_L1B_RA_BD3_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc is in this example the output from QDOAS and input to `qdoas2harp`

Note that in this example all the input and output files are kept in the same directory with this notebook. That is why  `-outdir .` is used in the command below, that refers to the current directory. Now, to run  `qd2hp` in the notebook, you need to add exclamation mark `!` at the beginning, before the actual command: 

```python
!qd2hp -outdir . -fitwin w200_bro_rad -slcol "ch2o=HCHO" S5P_OFFL_L1B_RA_BD3_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc
```

After executing the command succesfully, a file with selected information from the original file in HARP-compliant format is created, that will be the input for BeAMF:

`S5P_OFFL_QDOAS_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc`



## 3. Preparing input for BeAMF: adding auxiliary parameters <a name="paragraph3"></a> 

In AMF calculation we need information e.g. on cloud and surface properties, and hence the next step is to **add this auxiliary data** to the harp compliant QDOAS file that was created in the previous section. We take this cloud and surface information from the operational S5P L2 HCHO file that corresponds to the example DOAS fitfile we exctracted in the previous step. 

You can check which operational S5P L2 HCHO file is needed by using **aux_imp** command together with the name of the HARP compliant QDOAS file:

```python
!aux_imp S5P_OFFL_QDOAS_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc 
```

The L2 file can be downloaded e.g. using eofetch. To be able to perform `eofetch.download`, you must have environment variables, CDSE_S3_ACCESS and CDSE_S3_SECRET, set as described in in [eofetch README](https://github.com/stcorp/eofetch#readme).
You can also at this point set your environment variables within Python using the os module where "xxx" and "yyy" are your credentials.

```python
import os
os.environ["CDSE_S3_ACCESS"]="xxx"
os.environ["CDSE_S3_SECRET"]="yyy"
```

After setting the credentials, you can download the needed operational HCHO file as:

```python
eofetch.download("S5P_OFFL_L2__HCHO___20230608T193244_20230608T211414_29289_03_020401_20230610T184103.nc")
```

Now that the file is downloaded, the auxiliary parameters can be extracted. After the command is executed succesfully, the auxiliary variables will be written into the HARP compliant QDOAS file ([Sect. 2](#paragraph2)). Note that the extraction of the variables **can take a while**. As all the files are in the same working folder as the notebook, the extraction can be run as follows (--auxdir . pointing to the working folder):

```python
!aux_imp S5P_OFFL_QDOAS_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc --auxdir .
```

The resulting S5P_OFFL_QDOAS file can be read with harp to print out the contents. This data will be part of the input needed to run BeAMF.

```python
qdoas=harp.import_product('S5P_OFFL_QDOAS_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc')
print(qdoas)
```

## 4. Air Mass Factor calculation with BeAMF <a name="paragraph4"></a>

This is the actual step to calculate the AMF. The BeAMF requires several input parameters, that are defined in the configuration file **harp_hcho.json**. The input parameters include e.g. the Look-Up-Table file (for this case LUT_AMF_I_300nm_500nm_S1_US_small_new.nc) and the QDOAS file that has been prepared in the previous Sections. The BeAMF tool and AMF calculation is executed as follows (note that this can take several minutes): 


```python
!beamf -c harp_hcho.json 
```

The new parameters from BeAMF are again written into the S5P_OFFL_QDOAS - file. By printing out the file we can these new parameters that has been added, including e.g. HCHO_column_number_density:

```python
hcho=harp.import_product('S5P_OFFL_QDOAS_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc')
print(hcho)
```

## 6. Plotting HCHO data <a name="paragraph5"></a>

As the region of interest in this use case is over Canada, for plotting purposes the resulting HCHO data is imported with latitude condition. Over intense fire areas elevated HCHO is clearly visible. 

```python
filename='S5P_OFFL_QDOAS_20230608T193244_20230608T211414_29289_03_020100_20230608T230242.nc'
operations="latitude>40;latitude<62"
hcho_product = harp.import_product(filename, operations)
```

```python
avl.Geo(hcho_product,value="HCHO_column_number_density", colormap="viridis", colorrange=(0.5e16,2e16), centerlat=55, centerlon=-110, zoom=4)
```

```python
from avl import vis

varname1 = 'HCHO_slant_column_number_density'
varname2 = 'HCHO_column_number_density'
```

```python
plot = vis.Plot()
plot.add(avl.Line(hcho_product, varname2, name="HCHO vertical column density"))
plot.add(avl.Line(hcho_product, varname1, name="HCHO slant column density"))
plot
```

## 6. References <a name="references"></a>

- [BeAMF documentation](https://uvvis-bira-iasb.github.io/BeAMF/)
- [TROPOMI HCHO readme](https://sentinels.copernicus.eu/documents/247904/3541451/Sentinel-5P-Formaldehyde-Readme.pdf)
- [TROPOMI HCHO ATBD](https://sentinels.copernicus.eu/documents/247904/2476257/Sentinel-5P-ATBD-HCHO-TROPOMI.pdf/db71e36a-8507-46b5-a7cc-9d67e7c53f70?t=1658313806426)
- [HARP operations documentation](http://stcorp.github.io/harp/doc/html/operations.html)
- [HARP ingestion definitions](http://stcorp.github.io/harp/doc/html/ingestions/index.html)


```python

```
