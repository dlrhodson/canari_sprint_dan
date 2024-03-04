# Computing indices with cf-python

[cf-python](https://ncas-cms.github.io/cf-python/) is a python Earth Science data analysis library that is built on a complete implementation of the CF data model. It makes reading, writing and processing of cf netcdf data very simple. This tutorial points to some basic examples for use on the CANARI SPRINT data on JASMIN

These examples can all be found in `/home/users/dlrhodso/CANARI/SPRINT_2024/examples`. All sumbit submit multiple jobs in parallel in order to process all ensemble members at once. They use the `LOTUS` script to do this - this is a wrapper script for the SLURM submission commands that make it very easy to submit jobs and not worry about log files etc.

## Computing a box average NAO index


`compute_nao.sh` loops over all MSLP files in the `/gws/nopw/j04/canari/shared/large-ensemble/priority/HIST2` directory and uses the `compute_nao.py` python script to compute the NAO index.


```
#!/usr/bin/env python

import cf
import sys
import glob
from origin import *

def get_NAO_djf(field):
    '''
    Compute a DJF NAO index by box averaging Azores (20:28W,36:40N) - Iceland (16:25W,63:70N) and averaging over Dec-Jan-Feb
    '''
    #assume field== MSLP!
    #get sub-regions for boxes
    iceland_box=field.subspace(X=cf.wi(-25,-16),Y=cf.wi(63,70))
    azores_box=field.subspace(X=cf.wi(-28,-20),Y=cf.wi(36,40))
    #compute the area means (weighted by cell area) and difference
    nao=azores_box.collapse('area: mean',weights=True,squeeze=True)-iceland_box.collapse('area: mean', weights=True,squeeze=True)
    #compute the DJF mean
    nao_mean=nao.collapse('time: mean',group=cf.djf())
    #change the name 
    nao_mean.standard_name='NAO_djf'
    #And the netcdf name
    nao_mean.nc_set_variable('NAO_djf')
    #and the long_name
    nao_mean.set_properties({'long_name':'NAO_djf'})
    #return nao index
    return(nao_mean)

#Atmosphere grid cell area
areacella='/home/users/dlrhodso/CANARI/SPRINT_2024/analysis/areacella_fx_HadGEM3-GC31-MM_piControl_r1i1p1f1_gn_fixed.nc'

#a tag definining what this index is
outfile_origin="DJF NAO index computed as the Azores (20:28W, 36:40N) box mean  minus Iceland (16:25W,63:70N) box mean "

#First argument is scratch directory
scratch=sys.argv[1]
#file pattern for the MSLP diagnostic (m01s16i222)
#Only daily mean MSLP is available
file=sys.argv[2]+'/ATM/yearly/*/*day_m01s16i222*'
#expand this pattern
files=glob.glob(file)
#create output filename
outfilename=scratch+'/nao-djf_'+files[0].split('/')[-1]

#read in data, adding in the cell area from the external file
data=cf.read(file,external=areacella)
#compute nao index
nao_djf=get_NAO_djf(data[0])

#write out NAO index with description text
o_write(nao_djf,outfilename,outfile_origin)
print("Written "+outfilename)

```

This script will write the output to `scratch_pw3/NAO/` as files for each ensemble member





## Computing a box average index for SST



`compute_SPG_T.sh`



```
#!/usr/bin/env python

import cf
import sys
import glob
from origin import *

def compute_spg(field):
    '''
    compute SPG 50:65N 0:60W
    '''
    #compute SPG over 50:65N, 0:60W and 1st ocean layer (0:1m)
    #extract subspace
    sub_field=field.subspace(latitude=cf.wi(50,65),longitude=cf.wi(-60,0)).squeeze()
    #compute mean over subspace, weighting by area
    spg=sub_field.collapse('area: mean', weights='area',squeeze=True)
    #add names
    spg.standard_name='sub_polar_gyre_index_50_65N'
    spg.nc_set_variable('SPG_50_65N')
    spg.set_properties({'long_name':'sub_polar_gyre_50_60N'})

    return(spg)


#ocean cell area file
areacello="areacello.nc"

#a tag definining what this index is
outfile_origin="SubPolar Gyre Temperaure index computed as the  box mean of  over () box mean "

#First argument is scratch directory
scratch=sys.argv[1]

#file pattern for the SST diagnostic 

#file to read in
file=sys.argv[2]
#file name to write out
outfilename=sys.argv[3]
#variable name 
variable=sys.argv[4]

#expand this pattern
year,fname=file.split('/')[-2:]

#read in data, adding in the cell area from the external file
data=cf.read(file,external=areacello)
if len(data)>1:
    print("Aggregation failed!")
    exit()
#axis labels need to be added
data[0].coord('long_name=cell index along first dimension').axis='X'
data[0].coord('long_name=cell index along second dimension').axis='Y'
#compute spg index
varname="SPG_"+variable
index=compute_spg(data[0])
index.standard_name=varname
index.nc_set_variable(varname)

#compute the monthly means
index_monthly=index.collapse('time: mean', group=cf.M())
index_monthly.standard_name=varname+"_mon"
index_monthly.nc_set_variable(varname+"_mon")

#compute the annual means
index_annual=index.collapse('time: mean', group=cf.Y())
index_annual.standard_name=varname+"_ann"
index_annual.nc_set_variable(varname+"_ann")

outlist=cf.FieldList()
outlist.append(index)
outlist.append(index_monthly)
outlist.append(index_annual)

#write out NAO index with description text
o_write(outlist,outfilename,outfile_origin)
print("Written "+outfilename)


```


This produces multiple files in `scratch_pw3` that need to be assembled into a single file for each ensemble member. `concat_SPG_T.sh` does this usings `concat_cf.py`. 



