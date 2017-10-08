---
layout: notebook
title: ""
---
# The Boston Light Swim temperature analysis with Python

In the past we demonstrated how to perform a CSW catalog search with [`OWSLib`](https://ioos.github.io/notebooks_demos//notebooks/2016-12-19-exploring_csw),
and how to obtain near real-time data with [`pyoos`](https://ioos.github.io/notebooks_demos//notebooks/2016-10-12-fetching_data).
In this notebook we will use both to find all observations and model data around the Boston Harbor to access the sea water temperature.


This workflow is part of an example to advise swimmers of the annual [Boston lighthouse swim](http://bostonlightswim.org/) of the Boston Harbor water temperature conditions prior to the race. For more information regarding the workflow presented here see [Signell, Richard P.; Fernandes, Filipe; Wilcox, Kyle.   2016. "Dynamic Reusable Workflows for Ocean Science." *J. Mar. Sci. Eng.* 4, no. 4: 68](http://dx.doi.org/10.3390/jmse4040068).

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
import warnings

# Suppresing warnings for a "pretty output."
warnings.simplefilter('ignore')
```

This notebook is quite big and complex,
so to help us keep things organized we'll define a cell with the most important options and switches.

Below we can define the date,
bounding box, phenomena `SOS` and `CF` names and units,
and the catalogs we will search.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
%%writefile config.yaml

# Specify a YYYY-MM-DD hh:mm:ss date or integer day offset.
# If both start and stop are offsets they will be computed relative to datetime.today() at midnight.
# Use the dates commented below to reproduce the last Boston Light Swim event forecast.
date:
    start: -5 # 2016-8-16 00:00:00
    stop: +4 # 2016-8-29 00:00:00

run_name: 'latest'

# Boston harbor.
region:
    bbox: [-71.3, 42.03, -70.57, 42.63]
    # Try the bounding box below to see how the notebook will behave for a different region.
    #bbox: [-74.5, 40, -72., 41.5]
    crs: 'urn:ogc:def:crs:OGC:1.3:CRS84'

sos_name: 'sea_water_temperature'

cf_names:
    - sea_water_temperature
    - sea_surface_temperature
    - sea_water_potential_temperature
    - equivalent_potential_temperature
    - sea_water_conservative_temperature
    - pseudo_equivalent_potential_temperature

units: 'celsius'

catalogs:
    - https://data.ioos.us/csw
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Overwriting config.yaml

</pre>
</div>
We'll print some of the search configuration options along the way to keep track of them.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
import os
import shutil
from datetime import datetime
from ioos_tools.ioos import parse_config

config = parse_config('config.yaml')

# Saves downloaded data into a temporary directory.
save_dir = os.path.abspath(config['run_name'])
if os.path.exists(save_dir):
    shutil.rmtree(save_dir)
os.makedirs(save_dir)

fmt = '{:*^64}'.format
print(fmt('Saving data inside directory {}'.format(save_dir)))
print(fmt(' Run information '))
print('Run date: {:%Y-%m-%d %H:%M:%S}'.format(datetime.utcnow()))
print('Start: {:%Y-%m-%d %H:%M:%S}'.format(config['date']['start']))
print('Stop: {:%Y-%m-%d %H:%M:%S}'.format(config['date']['stop']))
print('Bounding box: {0:3.2f}, {1:3.2f},'
      '{2:3.2f}, {3:3.2f}'.format(*config['region']['bbox']))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Saving data inside directory /home/filipe/IOOS/notebooks_demos/notebooks/latest
    *********************** Run information ************************
    Run date: 2017-09-13 23:07:52
    Start: 2017-09-08 00:00:00
    Stop: 2017-09-17 00:00:00
    Bounding box: -71.30, 42.03,-70.57, 42.63

</pre>
</div>
We already created an `OWSLib.fes` filter [before](https://ioos.github.io/notebooks_demos//notebooks/2016-12-19-exploring_csw).
The main difference here is that we do not want the atmosphere model data,
so we are filtering out all the `GRIB-2` data format.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
def make_filter(config):
    from owslib import fes
    from ioos_tools.ioos import fes_date_filter
    kw = dict(wildCard='*', escapeChar='\\',
              singleChar='?', propertyname='apiso:AnyText')

    or_filt = fes.Or([fes.PropertyIsLike(literal=('*%s*' % val), **kw)
                      for val in config['cf_names']])

    not_filt = fes.Not([fes.PropertyIsLike(literal='GRIB-2', **kw)])

    begin, end = fes_date_filter(config['date']['start'],
                                 config['date']['stop'])
    bbox_crs = fes.BBox(config['region']['bbox'],
                        crs=config['region']['crs'])
    filter_list = [fes.And([bbox_crs, begin, end, or_filt, not_filt])]
    return filter_list


filter_list = make_filter(config)
```

In the cell below we ask the catalog for all the returns that match the filter and have an OPeNDAP endpoint.

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
from ioos_tools.ioos import service_urls, get_csw_records
from owslib.csw import CatalogueServiceWeb


dap_urls = []
print(fmt(' Catalog information '))
for endpoint in config['catalogs']:
    print('URL: {}'.format(endpoint))
    try:
        csw = CatalogueServiceWeb(endpoint, timeout=120)
    except Exception as e:
        print('{}'.format(e))
        continue
    csw = get_csw_records(csw, filter_list, esn='full')
    OPeNDAP = service_urls(csw.records, identifier='OPeNDAP:OPeNDAP')
    odp = service_urls(csw.records, identifier='urn:x-esri:specification:ServiceType:odp:url')
    dap = OPeNDAP + odp
    dap_urls.extend(dap)

    print('Number of datasets available: {}'.format(len(csw.records.keys())))

    for rec, item in csw.records.items():
        print('{}'.format(item.title))
    if dap:
        print(fmt(' DAP '))
        for url in dap:
            print('{}.html'.format(url))
    print('\n')

# Get only unique endpoints.
dap_urls = list(set(dap_urls))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Catalog information **********************
    URL: https://data.ioos.us/csw
    Number of datasets available: 19
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near SCRIPPS NEARSHORE, CA from 2015/01/07 23:00:00 to 2017/09/13 18:00:18.
    G1SST, 1km blended SST
    HYbrid Coordinate Ocean Model (HYCOM): Global
    NECOFS (FVCOM) - Scituate - Latest Forecast
    NECOFS GOM3 (FVCOM) - Northeast US - Latest Forecast
    NECOFS Massachusetts (FVCOM) - Boston - Latest Forecast
    NECOFS Massachusetts (FVCOM) - Massachusetts Coastal - Latest Forecast
    NERACOOS Gulf of Maine Ocean Array: Realtime Buoy Observations: A01 Massachusetts Bay: A01 ACCELEROMETER Massachusetts Bay
    NOAA Coral Reef Watch Operational Daily Near-Real-Time Global 5-km Satellite Coral Bleaching Monitoring Products
    A01 Accelerometer - Waves
    A01 Directional Waves (waves.mstrain Experimental)
    A01 Met - Meteorology
    A01 Optode - Oxygen
    A01 Sbe37 - CTD
    COAWST Modeling System: USEast: ROMS-WRF-SWAN coupled model (aka CNAPS)
    Coupled Northwest Atlantic Prediction System (CNAPS)
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near LAKESIDE, OR from 2017/03/31 23:00:00 to 2017/09/13 17:42:44.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near LOWER COOK INLET, AK from 2016/12/16 00:00:00 to 2017/09/13 18:11:01.
    Directional wave and sea surface temperature measurements collected in situ by Datawell Mark 3 directional buoy located near OCEAN STATION PAPA from 2015/01/01 01:00:00 to 2017/09/13 17:41:06.
    ***************************** DAP ******************************
    http://oos.soest.hawaii.edu/thredds/dodsC/hioos/satellite/dhw_5km.html
    http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/166p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/201p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/204p1_rt.nc.html
    http://thredds.cdip.ucsd.edu/thredds/dodsC/cdip/realtime/231p1_rt.nc.html
    http://thredds.secoora.org/thredds/dodsC/G1_SST_GLOBAL.nc.html
    http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc.html
    http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/Accelerometer/HistoricRealtime/Agg.ncml.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_GOM3_FORECAST.nc.html
    
    

</pre>
</div>
We found some models, and observations from NERACOOS there.
However, we do know that there are some buoys from NDBC and CO-OPS available too.
Also, those NERACOOS observations seem to be from a [CTD](http://www.neracoos.org/thredds/dodsC/UMO/DSG/SOS/A01/CTD1m/HistoricRealtime/Agg.ncml.html) mounted at 65 meters below the sea surface. Rendering them useless from our purpose.

So let's use the catalog only for the models by filtering the observations with `is_station` below.
And we'll rely `CO-OPS` and `NDBC` services for the observations.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
from ioos_tools.ioos import is_station

# Filter out some station endpoints.
non_stations = []
for url in dap_urls:
    try:
        if not is_station(url):
            non_stations.append(url)
    except (RuntimeError, OSError, IOError) as e:
        print('Could not access URL {}. {!r}'.format(url, e))

dap_urls = non_stations

print(fmt(' Filtered DAP '))
for url in dap_urls:
    print('{}.html'.format(url))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ************************* Filtered DAP *************************
    http://oos.soest.hawaii.edu/thredds/dodsC/hioos/satellite/dhw_5km.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_GOM3_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc.html
    http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global.html
    http://thredds.secoora.org/thredds/dodsC/G1_SST_GLOBAL.nc.html
    http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc.html

</pre>
</div>
Now we can use `pyoos` collectors for `NdbcSos`,

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
from pyoos.collectors.ndbc.ndbc_sos import NdbcSos

collector_ndbc = NdbcSos()

collector_ndbc.set_bbox(config['region']['bbox'])
collector_ndbc.end_time = config['date']['stop']
collector_ndbc.start_time = config['date']['start']
collector_ndbc.variables = [config['sos_name']]

ofrs = collector_ndbc.server.offerings
title = collector_ndbc.server.identification.title
print(fmt(' NDBC Collector offerings '))
print('{}: {} offerings'.format(title, len(ofrs)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ******************* NDBC Collector offerings *******************
    National Data Buoy Center SOS: 993 offerings

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
import pandas as pd
from ioos_tools.ioos import collector2table

ndbc = collector2table(collector=collector_ndbc,
                       config=config,
                       col='sea_water_temperature (C)')

if ndbc:
    data = dict(
        station_name=[s._metadata.get('station_name') for s in ndbc],
        station_code=[s._metadata.get('station_code') for s in ndbc],
        sensor=[s._metadata.get('sensor') for s in ndbc],
        lon=[s._metadata.get('lon') for s in ndbc],
        lat=[s._metadata.get('lat') for s in ndbc],
        depth=[s._metadata.get('depth') for s in ndbc],
    )

table = pd.DataFrame(data).set_index('station_code')
table
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>depth</th>
      <th>lat</th>
      <th>lon</th>
      <th>sensor</th>
      <th>station_name</th>
    </tr>
    <tr>
      <th>station_code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>44013</th>
      <td>0.6</td>
      <td>42.346</td>
      <td>-70.651</td>
      <td>urn:ioos:sensor:wmo:44013::watertemp1</td>
      <td>BOSTON 16 NM East of Boston, MA</td>
    </tr>
  </tbody>
</table>
</div>



and `CoopsSos`.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
from pyoos.collectors.coops.coops_sos import CoopsSos

collector_coops = CoopsSos()

collector_coops.set_bbox(config['region']['bbox'])
collector_coops.end_time = config['date']['stop']
collector_coops.start_time = config['date']['start']
collector_coops.variables = [config['sos_name']]

ofrs = collector_coops.server.offerings
title = collector_coops.server.identification.title
print(fmt(' Collector offerings '))
print('{}: {} offerings'.format(title, len(ofrs)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Collector offerings **********************
    NOAA.NOS.CO-OPS SOS: 1188 offerings

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
coops = collector2table(collector=collector_coops,
                        config=config,
                        col='sea_water_temperature (C)')

if coops:
    data = dict(
        station_name=[s._metadata.get('station_name') for s in coops],
        station_code=[s._metadata.get('station_code') for s in coops],
        sensor=[s._metadata.get('sensor') for s in coops],
        lon=[s._metadata.get('lon') for s in coops],
        lat=[s._metadata.get('lat') for s in coops],
        depth=[s._metadata.get('depth') for s in coops],
    )

table = pd.DataFrame(data).set_index('station_code')
table
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>depth</th>
      <th>lat</th>
      <th>lon</th>
      <th>sensor</th>
      <th>station_name</th>
    </tr>
    <tr>
      <th>station_code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>44013</th>
      <td>0.6</td>
      <td>42.346</td>
      <td>-70.651</td>
      <td>urn:ioos:sensor:wmo:44013::watertemp1</td>
      <td>BOSTON 16 NM East of Boston, MA</td>
    </tr>
  </tbody>
</table>
</div>



We will join all the observations into an uniform series, interpolated to 1-hour interval, for the model-data comparison.

This step is necessary because the observations can be 7 or 10 minutes resolution,
while the models can be 30 to 60 minutes.

<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
data = ndbc + coops

index = pd.date_range(start=config['date']['start'].replace(tzinfo=None),
                      end=config['date']['stop'].replace(tzinfo=None),
                      freq='1H')

# Preserve metadata with `reindex`.
observations = []
for series in data:
    _metadata = series._metadata
    obs = series.reindex(index=index, limit=1, method='nearest')
    obs._metadata = _metadata
    observations.append(obs)
```

In this next cell we will save the data for quicker access later.

<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
import iris
from ioos_tools.tardis import series2cube

attr = dict(
    featureType='timeSeries',
    Conventions='CF-1.6',
    standard_name_vocabulary='CF-1.6',
    cdm_data_type='Station',
    comment='Data from http://opendap.co-ops.nos.noaa.gov'
)


cubes = iris.cube.CubeList(
    [series2cube(obs, attr=attr) for obs in observations]
)

outfile = os.path.join(save_dir, 'OBS_DATA.nc')
iris.save(cubes, outfile)
```

Taking a quick look at the observations:

<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
%matplotlib inline

ax = pd.concat(data).plot(figsize=(11, 2.25))
```


![png](data:image/png;base64,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
QmCC
)


Now it is time to loop the models we found above,

<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
from iris.exceptions import (CoordinateNotFoundError, ConstraintMismatchError,
                             MergeError)
from ioos_tools.ioos import get_model_name
from ioos_tools.tardis import quick_load_cubes, proc_cube, is_model, get_surface

print(fmt(' Models '))
cubes = dict()
for k, url in enumerate(dap_urls):
    print('\n[Reading url {}/{}]: {}'.format(k+1, len(dap_urls), url))
    try:
        cube = quick_load_cubes(url, config['cf_names'],
                                callback=None, strict=True)
        if is_model(cube):
            cube = proc_cube(cube,
                             bbox=config['region']['bbox'],
                             time=(config['date']['start'],
                                   config['date']['stop']),
                             units=config['units'])
        else:
            print('[Not model data]: {}'.format(url))
            continue
        cube = get_surface(cube)
        mod_name = get_model_name(url)
        cubes.update({mod_name: cube})
    except (RuntimeError, ValueError,
            ConstraintMismatchError, CoordinateNotFoundError,
            IndexError) as e:
        print('Cannot get cube for: {}\n{}'.format(url, e))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    **************************** Models ****************************
    
    [Reading url 1/8]: http://oos.soest.hawaii.edu/thredds/dodsC/hioos/satellite/dhw_5km
    
    [Reading url 2/8]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc
    
    [Reading url 3/8]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_GOM3_FORECAST.nc
    
    [Reading url 4/8]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc
    
    [Reading url 5/8]: http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global
    
    [Reading url 6/8]: http://thredds.secoora.org/thredds/dodsC/G1_SST_GLOBAL.nc
    
    [Reading url 7/8]: http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc
    
    [Reading url 8/8]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc

</pre>
</div>
Next, we will match them with the nearest observed time-series. The `max_dist=0.08` is in degrees, that is roughly 8 kilometers.

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
import iris
from iris.pandas import as_series
from ioos_tools.tardis import (make_tree, get_nearest_water,
                               add_station, ensure_timeseries, remove_ssh)

for mod_name, cube in cubes.items():
    fname = '{}.nc'.format(mod_name)
    fname = os.path.join(save_dir, fname)
    print(fmt(' Downloading to file {} '.format(fname)))
    try:
        tree, lon, lat = make_tree(cube)
    except CoordinateNotFoundError as e:
        print('Cannot make KDTree for: {}'.format(mod_name))
        continue
    # Get model series at observed locations.
    raw_series = dict()
    for obs in observations:
        obs = obs._metadata
        station = obs['station_code']
        try:
            kw = dict(k=10, max_dist=0.08, min_var=0.01)
            args = cube, tree, obs['lon'], obs['lat']
            try:
                series, dist, idx = get_nearest_water(*args, **kw)
            except RuntimeError as e:
                print('Cannot download {!r}.\n{}'.format(cube, e))
                series = None
        except ValueError as e:
            status = 'No Data'
            print('[{}] {}'.format(status, obs['station_name']))
            continue
        if not series:
            status = 'Land   '
        else:
            raw_series.update({station: series})
            series = as_series(series)
            status = 'Water  '
        print('[{}] {}'.format(status, obs['station_name']))
    if raw_series:  # Save cube.
        for station, cube in raw_series.items():
            cube = add_station(cube, station)
            cube = remove_ssh(cube)
        try:
            cube = iris.cube.CubeList(raw_series.values()).merge_cube()
        except MergeError as e:
            print(e)
        ensure_timeseries(cube)
        try:
            iris.save(cube, fname)
        except AttributeError:
            # FIXME: we should patch the bad attribute instead of removing everything.
            cube.attributes = {}
            iris.save(cube, fname)
        del cube
    print('Finished processing [{}]'.format(mod_name))
```
<div class="output_area"><div class="prompt"></div>
<pre>
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/hioos_satellite-dhw_5km.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [hioos_satellite-dhw_5km]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/FVCOM_Forecasts-NECOFS_GOM3_FORECAST.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [FVCOM_Forecasts-NECOFS_GOM3_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc 
    [No Data] BOSTON 16 NM East of Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_BOSTON_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/pacioos_hycom-global.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [pacioos_hycom-global]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/G1_SST_GLOBAL.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [G1_SST_GLOBAL]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/SECOORA_NCSU_CNAPS.nc 
    [Water  ] BOSTON 16 NM East of Boston, MA
    Finished processing [SECOORA_NCSU_CNAPS]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc 
    [No Data] BOSTON 16 NM East of Boston, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST]

</pre>
</div>
Now it is possible to compute some simple comparison metrics. First we'll calculate the model mean bias:

$$ \text{MB} = \mathbf{\overline{m}} - \mathbf{\overline{o}}$$

<div class="prompt input_prompt">
In&nbsp;[16]:
</div>

```python
from ioos_tools.ioos import stations_keys


def rename_cols(df, config):
    cols = stations_keys(config, key='station_name')
    return df.rename(columns=cols)
```

<div class="prompt input_prompt">
In&nbsp;[17]:
</div>

```python
from ioos_tools.ioos import load_ncs
from ioos_tools.skill_score import mean_bias, apply_skill

dfs = load_ncs(config)

df = apply_skill(dfs, mean_bias, remove_mean=False, filter_tides=False)
skill_score = dict(mean_bias=df.to_dict())

# Filter out stations with no valid comparison.
df.dropna(how='all', axis=1, inplace=True)
df = df.applymap('{:.2f}'.format).replace('nan', '--')
```

And the root mean squared rrror of the deviations from the mean:
$$ \text{CRMS} = \sqrt{\left(\mathbf{m'} - \mathbf{o'}\right)^2}$$

where: $\mathbf{m'} = \mathbf{m} - \mathbf{\overline{m}}$ and $\mathbf{o'} = \mathbf{o} - \mathbf{\overline{o}}$

<div class="prompt input_prompt">
In&nbsp;[18]:
</div>

```python
from ioos_tools.skill_score import rmse

dfs = load_ncs(config)

df = apply_skill(dfs, rmse, remove_mean=True, filter_tides=False)
skill_score['rmse'] = df.to_dict()

# Filter out stations with no valid comparison.
df.dropna(how='all', axis=1, inplace=True)
df = df.applymap('{:.2f}'.format).replace('nan', '--')
```

The next 2 cells make the scores "pretty" for plotting.

<div class="prompt input_prompt">
In&nbsp;[19]:
</div>

```python
import pandas as pd

# Stringfy keys.
for key in skill_score.keys():
    skill_score[key] = {str(k): v for k, v in skill_score[key].items()}

mean_bias = pd.DataFrame.from_dict(skill_score['mean_bias'])
mean_bias = mean_bias.applymap('{:.2f}'.format).replace('nan', '--')

skill_score = pd.DataFrame.from_dict(skill_score['rmse'])
skill_score = skill_score.applymap('{:.2f}'.format).replace('nan', '--')
```

<div class="prompt input_prompt">
In&nbsp;[20]:
</div>

```python
import folium
from ioos_tools.ioos import get_coordinates


def make_map(bbox, **kw):
    line = kw.pop('line', True)
    layers = kw.pop('layers', True)
    zoom_start = kw.pop('zoom_start', 5)

    lon = (bbox[0] + bbox[2]) / 2
    lat = (bbox[1] + bbox[3]) / 2
    m = folium.Map(width='100%', height='100%',
                   location=[lat, lon], zoom_start=zoom_start)

    if layers:
        url = 'http://oos.soest.hawaii.edu/thredds/wms/hioos/satellite/dhw_5km'
        w = folium.WmsTileLayer(
            url,
            name='Sea Surface Temperature',
            fmt='image/png',
            layers='CRW_SST',
            attr='PacIOOS TDS',
            overlay=True,
            transparent=True)
        w.add_to(m)

    if line:
        p = folium.PolyLine(get_coordinates(bbox),
                            color='#FF0000',
                            weight=2,
                            opacity=0.9,
                            latlon=True)
        p.add_to(m)
    return m
```

<div class="prompt input_prompt">
In&nbsp;[21]:
</div>

```python
bbox = config['region']['bbox']

m = make_map(
    bbox,
    zoom_start=11,
    line=True,
    layers=True
)
```

The cells from `[20]` to `[25]` create a [`folium`](https://github.com/python-visualization/folium) map with [`bokeh`](http://bokeh.pydata.org/en/latest/) for the time-series at the observed points.

Note that we did mark the nearest model cell location used in the comparison.

<div class="prompt input_prompt">
In&nbsp;[22]:
</div>

```python
all_obs = stations_keys(config)

from glob import glob
from operator import itemgetter

import iris
from folium.plugins import MarkerCluster

iris.FUTURE.netcdf_promote = True

big_list = []
for fname in glob(os.path.join(save_dir, '*.nc')):
    if 'OBS_DATA' in fname:
        continue
    cube = iris.load_cube(fname)
    model = os.path.split(fname)[1].split('-')[-1].split('.')[0]
    lons = cube.coord(axis='X').points
    lats = cube.coord(axis='Y').points
    stations = cube.coord('station_code').points
    models = [model]*lons.size
    lista = zip(models, lons.tolist(), lats.tolist(), stations.tolist())
    big_list.extend(lista)

big_list.sort(key=itemgetter(3))
df = pd.DataFrame(big_list, columns=['name', 'lon', 'lat', 'station'])
df.set_index('station', drop=True, inplace=True)
groups = df.groupby(df.index)


locations, popups = [], []
for station, info in groups:
    sta_name = all_obs[station]
    for lat, lon, name in zip(info.lat, info.lon, info.name):
        locations.append([lat, lon])
        popups.append('[{}]: {}'.format(name, sta_name))

MarkerCluster(locations=locations, popups=popups, name='Cluster').add_to(m)
```




    <folium.plugins.marker_cluster.MarkerCluster at 0x7fcec9ca8dd8>



Here we use a dictionary with some models we expect to find so we can create a better legend for the plots. If any new models are found, we will use its filename in the legend as a default until we can go back and add a short name to our library.

<div class="prompt input_prompt">
In&nbsp;[23]:
</div>

```python
titles = {
    'coawst_4_use_best': 'COAWST_4',
    'global': 'HYCOM',
    'NECOFS_GOM3_FORECAST': 'NECOFS_GOM3',
    'NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST': 'NECOFS_MassBay',
    'OBS_DATA': 'Observations'
}
```

<div class="prompt input_prompt">
In&nbsp;[24]:
</div>

```python
from bokeh.resources import CDN
from bokeh.plotting import figure
from bokeh.embed import file_html
from bokeh.models import HoverTool
from itertools import cycle
from bokeh.palettes import Category20

from folium import IFrame

# Plot defaults.
colors = Category20[20]
colorcycler = cycle(colors)
tools = 'pan,box_zoom,reset'
width, height = 750, 250


def make_plot(df, station):
    p = figure(
        toolbar_location='above',
        x_axis_type='datetime',
        width=width,
        height=height,
        tools=tools,
        title=str(station)
    )
    for column, series in df.iteritems():
        series.dropna(inplace=True)
        if not series.empty:
            if 'OBS_DATA' not in column:
                bias = mean_bias[str(station)][column]
                skill = skill_score[str(station)][column]
                line_color = next(colorcycler)
                kw = dict(alpha=0.65, line_color=line_color)
            else:
                skill = bias = 'NA'
                kw = dict(alpha=1, color='crimson')
            line = p.line(
                x=series.index,
                y=series.values,
                legend='{}'.format(titles.get(column, column)),
                line_width=5,
                line_cap='round',
                line_join='round',
                **kw
            )
            p.add_tools(HoverTool(tooltips=[('Name', '{}'.format(titles.get(column, column))),
                                            ('Bias', bias),
                                            ('Skill', skill)],
                                  renderers=[line]))
    return p


def make_marker(p, station):
    lons = stations_keys(config, key='lon')
    lats = stations_keys(config, key='lat')

    lon, lat = lons[station], lats[station]
    html = file_html(p, CDN, station)
    iframe = IFrame(html, width=width+40, height=height+80)

    popup = folium.Popup(iframe, max_width=2650)
    icon = folium.Icon(color='green', icon='stats')
    marker = folium.Marker(location=[lat, lon],
                           popup=popup,
                           icon=icon)
    return marker
```

<div class="prompt input_prompt">
In&nbsp;[25]:
</div>

```python
dfs = load_ncs(config)

for station in dfs:
    sta_name = all_obs[station]
    df = dfs[station]
    if df.empty:
        continue
    p = make_plot(df, station)
    marker = make_marker(p, station)
    marker.add_to(m)

folium.LayerControl().add_to(m)

m
```




<div style="width:100%;"><div style="position:relative;width:100%;height:0;padding-bottom:60%;"><iframe src="data:text/html;charset=utf-8;base64,PCFET0NUWVBFIGh0bWw+CjxoZWFkPiAgICAKICAgIDxtZXRhIGh0dHAtZXF1aXY9ImNvbnRlbnQtdHlwZSIgY29udGVudD0idGV4dC9odG1sOyBjaGFyc2V0PVVURi04IiAvPgogICAgPHNjcmlwdD5MX1BSRUZFUl9DQU5WQVMgPSBmYWxzZTsgTF9OT19UT1VDSCA9IGZhbHNlOyBMX0RJU0FCTEVfM0QgPSBmYWxzZTs8L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmpzIj48L3NjcmlwdD4KICAgIDxzY3JpcHQgc3JjPSJodHRwczovL2FqYXguZ29vZ2xlYXBpcy5jb20vYWpheC9saWJzL2pxdWVyeS8xLjExLjEvanF1ZXJ5Lm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvanMvYm9vdHN0cmFwLm1pbi5qcyI+PC9zY3JpcHQ+CiAgICA8c2NyaXB0IHNyYz0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL2xlYWZsZXRAMS4yLjAvZGlzdC9sZWFmbGV0LmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9tYXhjZG4uYm9vdHN0cmFwY2RuLmNvbS9ib290c3RyYXAvMy4yLjAvY3NzL2Jvb3RzdHJhcC5taW4uY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL21heGNkbi5ib290c3RyYXBjZG4uY29tL2Jvb3RzdHJhcC8zLjIuMC9jc3MvYm9vdHN0cmFwLXRoZW1lLm1pbi5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vbWF4Y2RuLmJvb3RzdHJhcGNkbi5jb20vZm9udC1hd2Vzb21lLzQuNi4zL2Nzcy9mb250LWF3ZXNvbWUubWluLmNzcyIgLz4KICAgIDxsaW5rIHJlbD0ic3R5bGVzaGVldCIgaHJlZj0iaHR0cHM6Ly9jZG5qcy5jbG91ZGZsYXJlLmNvbS9hamF4L2xpYnMvTGVhZmxldC5hd2Vzb21lLW1hcmtlcnMvMi4wLjIvbGVhZmxldC5hd2Vzb21lLW1hcmtlcnMuY3NzIiAvPgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL3Jhd2dpdC5jb20vcHl0aG9uLXZpc3VhbGl6YXRpb24vZm9saXVtL21hc3Rlci9mb2xpdW0vdGVtcGxhdGVzL2xlYWZsZXQuYXdlc29tZS5yb3RhdGUuY3NzIiAvPgogICAgPHN0eWxlPmh0bWwsIGJvZHkge3dpZHRoOiAxMDAlO2hlaWdodDogMTAwJTttYXJnaW46IDA7cGFkZGluZzogMDt9PC9zdHlsZT4KICAgIDxzdHlsZT4jbWFwIHtwb3NpdGlvbjphYnNvbHV0ZTt0b3A6MDtib3R0b206MDtyaWdodDowO2xlZnQ6MDt9PC9zdHlsZT4KICAgIAogICAgICAgICAgICA8c3R5bGU+ICNtYXBfMTVmYWMwODZmYzg5NDcxNGFkZWVlM2E3ZTA1MjQxNTkgewogICAgICAgICAgICAgICAgcG9zaXRpb24gOiByZWxhdGl2ZTsKICAgICAgICAgICAgICAgIHdpZHRoIDogMTAwLjAlOwogICAgICAgICAgICAgICAgaGVpZ2h0OiAxMDAuMCU7CiAgICAgICAgICAgICAgICBsZWZ0OiAwLjAlOwogICAgICAgICAgICAgICAgdG9wOiAwLjAlOwogICAgICAgICAgICAgICAgfQogICAgICAgICAgICA8L3N0eWxlPgogICAgICAgIAogICAgPHNjcmlwdCBzcmM9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8xLjEuMC9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIuanMiPjwvc2NyaXB0PgogICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2NkbmpzLmNsb3VkZmxhcmUuY29tL2FqYXgvbGlicy9sZWFmbGV0Lm1hcmtlcmNsdXN0ZXIvMS4xLjAvTWFya2VyQ2x1c3Rlci5jc3MiIC8+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9Imh0dHBzOi8vY2RuanMuY2xvdWRmbGFyZS5jb20vYWpheC9saWJzL2xlYWZsZXQubWFya2VyY2x1c3Rlci8xLjEuMC9NYXJrZXJDbHVzdGVyLkRlZmF1bHQuY3NzIiAvPgo8L2hlYWQ+Cjxib2R5PiAgICAKICAgIAogICAgICAgICAgICA8ZGl2IGNsYXNzPSJmb2xpdW0tbWFwIiBpZD0ibWFwXzE1ZmFjMDg2ZmM4OTQ3MTRhZGVlZTNhN2UwNTI0MTU5IiA+PC9kaXY+CiAgICAgICAgCjwvYm9keT4KPHNjcmlwdD4gICAgCiAgICAKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGJvdW5kcyA9IG51bGw7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgdmFyIG1hcF8xNWZhYzA4NmZjODk0NzE0YWRlZWUzYTdlMDUyNDE1OSA9IEwubWFwKAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgJ21hcF8xNWZhYzA4NmZjODk0NzE0YWRlZWUzYTdlMDUyNDE1OScsCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB7Y2VudGVyOiBbNDIuMzMsLTcwLjkzNV0sCiAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB6b29tOiAxMSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIG1heEJvdW5kczogYm91bmRzLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgbGF5ZXJzOiBbXSwKICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgIHdvcmxkQ29weUp1bXA6IGZhbHNlLAogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgY3JzOiBMLkNSUy5FUFNHMzg1NwogICAgICAgICAgICAgICAgICAgICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHRpbGVfbGF5ZXJfNGYxZjdmZjUxYTdhNGIwOTlkZWExMWQ5ODllMWEwOTIgPSBMLnRpbGVMYXllcigKICAgICAgICAgICAgICAgICdodHRwczovL3tzfS50aWxlLm9wZW5zdHJlZXRtYXAub3JnL3t6fS97eH0ve3l9LnBuZycsCiAgICAgICAgICAgICAgICB7CiAgImF0dHJpYnV0aW9uIjogbnVsbCwKICAiZGV0ZWN0UmV0aW5hIjogZmFsc2UsCiAgIm1heFpvb20iOiAxOCwKICAibWluWm9vbSI6IDEsCiAgIm5vV3JhcCI6IGZhbHNlLAogICJzdWJkb21haW5zIjogImFiYyIKfQogICAgICAgICAgICAgICAgKS5hZGRUbyhtYXBfMTVmYWMwODZmYzg5NDcxNGFkZWVlM2E3ZTA1MjQxNTkpOwogICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBtYWNyb19lbGVtZW50XzAyNTE1Y2RkMWRmYzRjYTk4MGM0Y2Y0MDQxN2Q5ZGI2ID0gTC50aWxlTGF5ZXIud21zKAogICAgICAgICAgICAgICAgJ2h0dHA6Ly9vb3Muc29lc3QuaGF3YWlpLmVkdS90aHJlZGRzL3dtcy9oaW9vcy9zYXRlbGxpdGUvZGh3XzVrbScsCiAgICAgICAgICAgICAgICB7CiAgImF0dHJpYnV0aW9uIjogIlBhY0lPT1MgVERTIiwKICAiY3JzIjogbnVsbCwKICAiZm9ybWF0IjogImltYWdlL3BuZyIsCiAgImxheWVycyI6ICJDUldfU1NUIiwKICAic3R5bGVzIjogIiIsCiAgInRyYW5zcGFyZW50IjogdHJ1ZSwKICAidXBwZXJjYXNlIjogZmFsc2UsCiAgInZlcnNpb24iOiAiMS4xLjEiCn0KICAgICAgICAgICAgICAgICkuYWRkVG8obWFwXzE1ZmFjMDg2ZmM4OTQ3MTRhZGVlZTNhN2UwNTI0MTU5KTsKCiAgICAgICAgCiAgICAKICAgICAgICAgICAgICAgIHZhciBwb2x5X2xpbmVfZTY4NmJhNzExZTQxNDJmN2JlNTIxNDg0ZDE0YjdiZjggPSBMLnBvbHlsaW5lKAogICAgICAgICAgICAgICAgICAgIFtbNDIuMDMwMDAwMDAwMDAwMDAxLCAtNzEuMjk5OTk5OTk5OTk5OTk3XSwgWzQyLjAzMDAwMDAwMDAwMDAwMSwgLTcwLjU2OTk5OTk5OTk5OTk5M10sIFs0Mi42MzAwMDAwMDAwMDAwMDMsIC03MC41Njk5OTk5OTk5OTk5OTNdLCBbNDIuNjMwMDAwMDAwMDAwMDAzLCAtNzEuMjk5OTk5OTk5OTk5OTk3XSwgWzQyLjAzMDAwMDAwMDAwMDAwMSwgLTcxLjI5OTk5OTk5OTk5OTk5N11dLAogICAgICAgICAgICAgICAgICAgIHsKICAiYnViYmxpbmdNb3VzZUV2ZW50cyI6IHRydWUsCiAgImNvbG9yIjogIiNGRjAwMDAiLAogICJkYXNoQXJyYXkiOiBudWxsLAogICJkYXNoT2Zmc2V0IjogbnVsbCwKICAiZmlsbCI6IGZhbHNlLAogICJmaWxsQ29sb3IiOiAiI0ZGMDAwMCIsCiAgImZpbGxPcGFjaXR5IjogMC4yLAogICJmaWxsUnVsZSI6ICJldmVub2RkIiwKICAibGluZUNhcCI6ICJyb3VuZCIsCiAgImxpbmVKb2luIjogInJvdW5kIiwKICAibm9DbGlwIjogZmFsc2UsCiAgIm9wYWNpdHkiOiAwLjksCiAgInNtb290aEZhY3RvciI6IDEuMCwKICAic3Ryb2tlIjogdHJ1ZSwKICAid2VpZ2h0IjogMgp9KS5hZGRUbyhtYXBfMTVmYWMwODZmYzg5NDcxNGFkZWVlM2E3ZTA1MjQxNTkpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgbWFya2VyX2NsdXN0ZXJfMWNhMDc4NTNlNjY5NGY4ZWFhZWZjYzRlMTI5OThmZTYgPSBMLm1hcmtlckNsdXN0ZXJHcm91cCh7CiAgICAgICAgICAgICAgICAKICAgICAgICAgICAgfSk7CiAgICAgICAgICAgIG1hcF8xNWZhYzA4NmZjODk0NzE0YWRlZWUzYTdlMDUyNDE1OS5hZGRMYXllcihtYXJrZXJfY2x1c3Rlcl8xY2EwNzg1M2U2Njk0ZjhlYWFlZmNjNGUxMjk5OGZlNik7CiAgICAgICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2RiNjk0ZmY3NTdhMDRiOTk4ZWEwNjkzMzVmYjY5ZjM5ID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNDIuMzI1MDA0NTc3NiwtNzAuNjc0OTk1NDIyNF0sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFya2VyX2NsdXN0ZXJfMWNhMDc4NTNlNjY5NGY4ZWFhZWZjYzRlMTI5OThmZTYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfYmQzYTRhMDVlYTI2NDZjYWE3ZTI5OTg3MDVjZTA5ODkgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfYWE0NTJiNjNmNTBiNGNiODg4ZWEzYTAyZjc5Mjg1ZjcgPSAkKCc8ZGl2IGlkPSJodG1sX2FhNDUyYjYzZjUwYjRjYjg4OGVhM2EwMmY3OTI4NWY3IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5bZGh3XzVrbV06IEJPU1RPTiAxNiBOTSBFYXN0IG9mIEJvc3RvbiwgTUE8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2JkM2E0YTA1ZWEyNjQ2Y2FhN2UyOTk4NzA1Y2UwOTg5LnNldENvbnRlbnQoaHRtbF9hYTQ1MmI2M2Y1MGI0Y2I4ODhlYTNhMDJmNzkyODVmNyk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2RiNjk0ZmY3NTdhMDRiOTk4ZWEwNjkzMzVmYjY5ZjM5LmJpbmRQb3B1cChwb3B1cF9iZDNhNGEwNWVhMjY0NmNhYTdlMjk5ODcwNWNlMDk4OSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl85ZDI3YTgyNGU4Y2U0MDIxYWE2MTllMjZkMDhkZDE1ZCA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzQyLjM0NTAwMTIyMDcsLTcwLjY1NDk5ODc3OTNdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcmtlcl9jbHVzdGVyXzFjYTA3ODUzZTY2OTRmOGVhYWVmY2M0ZTEyOTk4ZmU2KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzljYjVmNmYxYzRiYjQxYjE5ZGNiOWI0Y2YzY2RjZDk3ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sXzMwOGY2ZTI5N2YzNzRmYWE5M2VmMTVhNjZjZjFlZGViID0gJCgnPGRpdiBpZD0iaHRtbF8zMDhmNmUyOTdmMzc0ZmFhOTNlZjE1YTY2Y2YxZWRlYiIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+W0cxX1NTVF9HTE9CQUxdOiBCT1NUT04gMTYgTk0gRWFzdCBvZiBCb3N0b24sIE1BPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF85Y2I1ZjZmMWM0YmI0MWIxOWRjYjliNGNmM2NkY2Q5Ny5zZXRDb250ZW50KGh0bWxfMzA4ZjZlMjk3ZjM3NGZhYTkzZWYxNWE2NmNmMWVkZWIpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl85ZDI3YTgyNGU4Y2U0MDIxYWE2MTllMjZkMDhkZDE1ZC5iaW5kUG9wdXAocG9wdXBfOWNiNWY2ZjFjNGJiNDFiMTlkY2I5YjRjZjNjZGNkOTcpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfNWI0ODdiN2Y0YTg2NDE5MjgyYjk4ZmE2NDZhYTg0OTUgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zMjQ4LC03MC42Mzk5N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFya2VyX2NsdXN0ZXJfMWNhMDc4NTNlNjY5NGY4ZWFhZWZjYzRlMTI5OThmZTYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZWI5YmMzYWFkNGEyNDg4ZmJkYmQ0YjNkNTM1ODk4YjAgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfOGE5N2UxM2VmMDc3NDc2OTk2ZDY2ODRjYzc4NTI0ZjQgPSAkKCc8ZGl2IGlkPSJodG1sXzhhOTdlMTNlZjA3NzQ3Njk5NmQ2Njg0Y2M3ODUyNGY0IiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5bZ2xvYmFsXTogQk9TVE9OIDE2IE5NIEVhc3Qgb2YgQm9zdG9uLCBNQTwvZGl2PicpWzBdOwogICAgICAgICAgICAgICAgcG9wdXBfZWI5YmMzYWFkNGEyNDg4ZmJkYmQ0YjNkNTM1ODk4YjAuc2V0Q29udGVudChodG1sXzhhOTdlMTNlZjA3NzQ3Njk5NmQ2Njg0Y2M3ODUyNGY0KTsKICAgICAgICAgICAgCgogICAgICAgICAgICBtYXJrZXJfNWI0ODdiN2Y0YTg2NDE5MjgyYjk4ZmE2NDZhYTg0OTUuYmluZFBvcHVwKHBvcHVwX2ViOWJjM2FhZDRhMjQ4OGZiZGJkNGIzZDUzNTg5OGIwKTsKCiAgICAgICAgICAgIAogICAgICAgIAogICAgCgogICAgICAgICAgICB2YXIgbWFya2VyX2E2NTgyNGE3ZjExZDRlOTc4ZGU0NTVjNzZjZGY5NDEzID0gTC5tYXJrZXIoCiAgICAgICAgICAgICAgICBbNDIuMzQxMzI3NjY3MiwtNzAuNjQ4MzE1NDI5N10sCiAgICAgICAgICAgICAgICB7CiAgICAgICAgICAgICAgICAgICAgaWNvbjogbmV3IEwuSWNvbi5EZWZhdWx0KCkKICAgICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICApCiAgICAgICAgICAgICAgICAuYWRkVG8obWFya2VyX2NsdXN0ZXJfMWNhMDc4NTNlNjY5NGY4ZWFhZWZjYzRlMTI5OThmZTYpOwogICAgICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgcG9wdXBfZDUwYTBmNWJmNjQyNDAyMzgwYjJkMTYzYTkyZjE3NTYgPSBMLnBvcHVwKHttYXhXaWR0aDogJzMwMCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGh0bWxfNmMwY2RjMjE0NTIzNDYzYTlmMTY4NDQyMDRlMjQ0N2EgPSAkKCc8ZGl2IGlkPSJodG1sXzZjMGNkYzIxNDUyMzQ2M2E5ZjE2ODQ0MjA0ZTI0NDdhIiBzdHlsZT0id2lkdGg6IDEwMC4wJTsgaGVpZ2h0OiAxMDAuMCU7Ij5bTkVDT0ZTX0ZWQ09NX09DRUFOX01BU1NCQVlfRk9SRUNBU1RdOiBCT1NUT04gMTYgTk0gRWFzdCBvZiBCb3N0b24sIE1BPC9kaXY+JylbMF07CiAgICAgICAgICAgICAgICBwb3B1cF9kNTBhMGY1YmY2NDI0MDIzODBiMmQxNjNhOTJmMTc1Ni5zZXRDb250ZW50KGh0bWxfNmMwY2RjMjE0NTIzNDYzYTlmMTY4NDQyMDRlMjQ0N2EpOwogICAgICAgICAgICAKCiAgICAgICAgICAgIG1hcmtlcl9hNjU4MjRhN2YxMWQ0ZTk3OGRlNDU1Yzc2Y2RmOTQxMy5iaW5kUG9wdXAocG9wdXBfZDUwYTBmNWJmNjQyNDAyMzgwYjJkMTYzYTkyZjE3NTYpOwoKICAgICAgICAgICAgCiAgICAgICAgCiAgICAKCiAgICAgICAgICAgIHZhciBtYXJrZXJfZGIwNGY4ZGQxMzk1NGMyYmJlZDkyNGYwM2JjMGZjYmIgPSBMLm1hcmtlcigKICAgICAgICAgICAgICAgIFs0Mi4zNDM3ODQzMzIzLC03MC42NTI0NTgxOTA5XSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXJrZXJfY2x1c3Rlcl8xY2EwNzg1M2U2Njk0ZjhlYWFlZmNjNGUxMjk5OGZlNik7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9mMDVkOTI1NWU0NGM0N2JlOWM1YmM1NDA5NjBlYmI0NSA9IEwucG9wdXAoe21heFdpZHRoOiAnMzAwJ30pOwoKICAgICAgICAgICAgCiAgICAgICAgICAgICAgICB2YXIgaHRtbF8yOWI2MzExNzQxODM0MGI1YTgwZmI1ZDg0NGQ3MGMzMiA9ICQoJzxkaXYgaWQ9Imh0bWxfMjliNjMxMTc0MTgzNDBiNWE4MGZiNWQ4NDRkNzBjMzIiIHN0eWxlPSJ3aWR0aDogMTAwLjAlOyBoZWlnaHQ6IDEwMC4wJTsiPltORUNPRlNfR09NM19GT1JFQ0FTVF06IEJPU1RPTiAxNiBOTSBFYXN0IG9mIEJvc3RvbiwgTUE8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2YwNWQ5MjU1ZTQ0YzQ3YmU5YzViYzU0MDk2MGViYjQ1LnNldENvbnRlbnQoaHRtbF8yOWI2MzExNzQxODM0MGI1YTgwZmI1ZDg0NGQ3MGMzMik7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2RiMDRmOGRkMTM5NTRjMmJiZWQ5MjRmMDNiYzBmY2JiLmJpbmRQb3B1cChwb3B1cF9mMDVkOTI1NWU0NGM0N2JlOWM1YmM1NDA5NjBlYmI0NSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl9jNDA5NDNkZTU0MmU0Y2MwYWMwNTJhNWM2N2ZiMWZiNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzQyLjM1MzcwMjgzMDQsLTcwLjY0MTQwMjcxNDhdLAogICAgICAgICAgICAgICAgewogICAgICAgICAgICAgICAgICAgIGljb246IG5ldyBMLkljb24uRGVmYXVsdCgpCiAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgKQogICAgICAgICAgICAgICAgLmFkZFRvKG1hcmtlcl9jbHVzdGVyXzFjYTA3ODUzZTY2OTRmOGVhYWVmY2M0ZTEyOTk4ZmU2KTsKICAgICAgICAgICAgCiAgICAKICAgICAgICAgICAgdmFyIHBvcHVwXzdlOWFhMzEzNDE5YjRjYWU4ZWE1NzVlYzVlNTY2YTI5ID0gTC5wb3B1cCh7bWF4V2lkdGg6ICczMDAnfSk7CgogICAgICAgICAgICAKICAgICAgICAgICAgICAgIHZhciBodG1sX2Q2NTFkM2M5ZDU1NzRjNDY5NmI5ZDdiODg2MDVkNDUwID0gJCgnPGRpdiBpZD0iaHRtbF9kNjUxZDNjOWQ1NTc0YzQ2OTZiOWQ3Yjg4NjA1ZDQ1MCIgc3R5bGU9IndpZHRoOiAxMDAuMCU7IGhlaWdodDogMTAwLjAlOyI+W1NFQ09PUkFfTkNTVV9DTkFQU106IEJPU1RPTiAxNiBOTSBFYXN0IG9mIEJvc3RvbiwgTUE8L2Rpdj4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwXzdlOWFhMzEzNDE5YjRjYWU4ZWE1NzVlYzVlNTY2YTI5LnNldENvbnRlbnQoaHRtbF9kNjUxZDNjOWQ1NTc0YzQ2OTZiOWQ3Yjg4NjA1ZDQ1MCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyX2M0MDk0M2RlNTQyZTRjYzBhYzA1MmE1YzY3ZmIxZmI2LmJpbmRQb3B1cChwb3B1cF83ZTlhYTMxMzQxOWI0Y2FlOGVhNTc1ZWM1ZTU2NmEyOSk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAoKICAgICAgICAgICAgdmFyIG1hcmtlcl81ZTE4ZmI0NzAxOTc0MTEyYTQwNzUwMTdkODU5ZTBkNiA9IEwubWFya2VyKAogICAgICAgICAgICAgICAgWzQyLjM0NiwtNzAuNjUxXSwKICAgICAgICAgICAgICAgIHsKICAgICAgICAgICAgICAgICAgICBpY29uOiBuZXcgTC5JY29uLkRlZmF1bHQoKQogICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICkKICAgICAgICAgICAgICAgIC5hZGRUbyhtYXBfMTVmYWMwODZmYzg5NDcxNGFkZWVlM2E3ZTA1MjQxNTkpOwogICAgICAgICAgICAKICAgIAoKICAgICAgICAgICAgICAgIHZhciBpY29uXzdlNTM3ODk5YTQxNzRkM2E5NmRiMWNhOWQ4NDQ2NTU5ID0gTC5Bd2Vzb21lTWFya2Vycy5pY29uKHsKICAgICAgICAgICAgICAgICAgICBpY29uOiAnc3RhdHMnLAogICAgICAgICAgICAgICAgICAgIGljb25Db2xvcjogJ3doaXRlJywKICAgICAgICAgICAgICAgICAgICBtYXJrZXJDb2xvcjogJ2dyZWVuJywKICAgICAgICAgICAgICAgICAgICBwcmVmaXg6ICdnbHlwaGljb24nLAogICAgICAgICAgICAgICAgICAgIGV4dHJhQ2xhc3NlczogJ2ZhLXJvdGF0ZS0wJwogICAgICAgICAgICAgICAgICAgIH0pOwogICAgICAgICAgICAgICAgbWFya2VyXzVlMThmYjQ3MDE5NzQxMTJhNDA3NTAxN2Q4NTllMGQ2LnNldEljb24oaWNvbl83ZTUzNzg5OWE0MTc0ZDNhOTZkYjFjYTlkODQ0NjU1OSk7CiAgICAgICAgICAgIAogICAgCiAgICAgICAgICAgIHZhciBwb3B1cF9jYzMzNjk0YWI3NDk0Yjk1Yjk5NWFlNmFiYTY2NThlOCA9IEwucG9wdXAoe21heFdpZHRoOiAnMjY1MCd9KTsKCiAgICAgICAgICAgIAogICAgICAgICAgICAgICAgdmFyIGlfZnJhbWVfYTMwNDM4MmViNDI2NDljMDk2OGI0ODYwMTYzMTNiZjAgPSAkKCc8aWZyYW1lIHNyYz0iZGF0YTp0ZXh0L2h0bWw7Y2hhcnNldD11dGYtODtiYXNlNjQsQ2lBZ0lDQUtQQ0ZFVDBOVVdWQkZJR2gwYld3K0NqeG9kRzFzSUd4aGJtYzlJbVZ1SWo0S0lDQWdJRHhvWldGa1Bnb2dJQ0FnSUNBZ0lEeHRaWFJoSUdOb1lYSnpaWFE5SW5WMFppMDRJajRLSUNBZ0lDQWdJQ0E4ZEdsMGJHVStORFF3TVRNOEwzUnBkR3hsUGdvZ0lDQWdJQ0FnSUFvOGJHbHVheUJ5Wld3OUluTjBlV3hsYzJobFpYUWlJR2h5WldZOUltaDBkSEJ6T2k4dlkyUnVMbkI1WkdGMFlTNXZjbWN2WW05clpXZ3ZjbVZzWldGelpTOWliMnRsYUMwd0xqRXlMall1YldsdUxtTnpjeUlnZEhsd1pUMGlkR1Y0ZEM5amMzTWlJQzgrQ2lBZ0lDQWdJQ0FnQ2p4elkzSnBjSFFnZEhsd1pUMGlkR1Y0ZEM5cVlYWmhjMk55YVhCMElpQnpjbU05SW1oMGRIQnpPaTh2WTJSdUxuQjVaR0YwWVM1dmNtY3ZZbTlyWldndmNtVnNaV0Z6WlM5aWIydGxhQzB3TGpFeUxqWXViV2x1TG1weklqNDhMM05qY21sd2RENEtQSE5qY21sd2RDQjBlWEJsUFNKMFpYaDBMMnBoZG1GelkzSnBjSFFpUGdvZ0lDQWdRbTlyWldndWMyVjBYMnh2WjE5c1pYWmxiQ2dpYVc1bWJ5SXBPd284TDNOamNtbHdkRDRLSUNBZ0lDQWdJQ0E4YzNSNWJHVStDaUFnSUNBZ0lDQWdJQ0JvZEcxc0lIc0tJQ0FnSUNBZ0lDQWdJQ0FnZDJsa2RHZzZJREV3TUNVN0NpQWdJQ0FnSUNBZ0lDQWdJR2hsYVdkb2REb2dNVEF3SlRzS0lDQWdJQ0FnSUNBZ0lIMEtJQ0FnSUNBZ0lDQWdJR0p2WkhrZ2V3b2dJQ0FnSUNBZ0lDQWdJQ0IzYVdSMGFEb2dPVEFsT3dvZ0lDQWdJQ0FnSUNBZ0lDQm9aV2xuYUhRNklERXdNQ1U3Q2lBZ0lDQWdJQ0FnSUNBZ0lHMWhjbWRwYmpvZ1lYVjBienNLSUNBZ0lDQWdJQ0FnSUgwS0lDQWdJQ0FnSUNBOEwzTjBlV3hsUGdvZ0lDQWdQQzlvWldGa1Bnb2dJQ0FnUEdKdlpIaytDaUFnSUNBZ0lDQWdDaUFnSUNBZ0lDQWdQR1JwZGlCamJHRnpjejBpWW1zdGNtOXZkQ0krQ2lBZ0lDQWdJQ0FnSUNBZ0lEeGthWFlnWTJ4aGMzTTlJbUpyTFhCc2IzUmthWFlpSUdsa1BTSXlNMlJsWmpsbFpTMW1NelZsTFRRMFkyTXRPV1kyTlMwME9HRTRNMlk1TWpSaU5UWWlQand2WkdsMlBnb2dJQ0FnSUNBZ0lEd3ZaR2wyUGdvZ0lDQWdJQ0FnSUFvZ0lDQWdJQ0FnSUR4elkzSnBjSFFnZEhsd1pUMGlkR1Y0ZEM5cVlYWmhjMk55YVhCMElqNEtJQ0FnSUNBZ0lDQWdJQ0FnS0daMWJtTjBhVzl1S0NrZ2V3b2dJQ0FnSUNBZ0lDQWdkbUZ5SUdadUlEMGdablZ1WTNScGIyNG9LU0I3Q2lBZ0lDQWdJQ0FnSUNBZ0lFSnZhMlZvTG5OaFptVnNlU2htZFc1amRHbHZiaWdwSUhzS0lDQWdJQ0FnSUNBZ0lDQWdJQ0IyWVhJZ1pHOWpjMTlxYzI5dUlEMGdleUk1TlRJMVl6WmxZeTB6TkdJeExUUXpaRGd0WVRSbVlTMHlZakptTmpjNVpEZG1ZV01pT25zaWNtOXZkSE1pT25zaWNtVm1aWEpsYm1ObGN5STZXM3NpWVhSMGNtbGlkWFJsY3lJNmV5SmpZV3hzWW1GamF5STZiblZzYkgwc0ltbGtJam9pT0dZeE1USTNaamN0TnpneU9DMDBNemMzTFRrNU9UWXRPR05oTm1RMFlUSmhPV1ExSWl3aWRIbHdaU0k2SWtSaGRHRlNZVzVuWlRGa0luMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteHBibVZmWVd4d2FHRWlPbnNpZG1Gc2RXVWlPakF1TmpWOUxDSnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2ZXlKMllXeDFaU0k2SWlNeVkyRXdNbU1pZlN3aWJHbHVaVjlxYjJsdUlqb2ljbTkxYm1RaUxDSnNhVzVsWDNkcFpIUm9JanA3SW5aaGJIVmxJam8xZlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pWXpVeE9ESmhOMll0WVRNek1pMDBOR0prTFRreU9HTXRaR1JtWVdZd09URmpaRE0zSWl3aWRIbHdaU0k2SWt4cGJtVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2lZMkZzYkdKaFkyc2lPbTUxYkd3c0luQnNiM1FpT25zaWFXUWlPaUkwTWpBNVpURXdOaTFpTldaa0xUUmtaRE10T0RneU1TMHlNMlZoWm1VNE1EWTROamdpTENKemRXSjBlWEJsSWpvaVJtbG5kWEpsSWl3aWRIbHdaU0k2SWxCc2IzUWlmU3dpY21WdVpHVnlaWEp6SWpwYmV5SnBaQ0k2SWpGaU1UWTJORGxqTFdZMFpXUXROREUxTlMwNU5EZzJMV001TmpBNVlXRTNOelk1TmlJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjFkTENKMGIyOXNkR2x3Y3lJNlcxc2lUbUZ0WlNJc0lrOWljMlZ5ZG1GMGFXOXVjeUpkTEZzaVFtbGhjeUlzSWs1QklsMHNXeUpUYTJsc2JDSXNJazVCSWwxZGZTd2lhV1FpT2lJeU5UVmhZemxsTnkwNE5UQTJMVFF3TWpNdFlqRTVPQzB3TkdZMlpUYzRaamd5TXpFaUxDSjBlWEJsSWpvaVNHOTJaWEpVYjI5c0luMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltUmhlWE1pT2xzeExERTFYWDBzSW1sa0lqb2lPVGczTURJMk9ETXRNV0k0WXkwMFltWXhMVGd4TWpJdFlqWTBNbVkwWWpVMlpUTmpJaXdpZEhsd1pTSTZJa1JoZVhOVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnQ5TENKcFpDSTZJbU5rTURobU1ESXhMVGs0WXpRdE5EUTRaQzFpTVdRMkxXVmhPVE0wT1RVNE1qa3lOaUlzSW5SNWNHVWlPaUpNYVc1bFlYSlRZMkZzWlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKbWIzSnRZWFIwWlhJaU9uc2lhV1FpT2lJNU5UZ3paVGRpWWkwM1lXWmxMVFF5WWpBdE9ESmxNUzAyWWpjelpqVmlObVZsTldZaUxDSjBlWEJsSWpvaVFtRnphV05VYVdOclJtOXliV0YwZEdWeUluMHNJbkJzYjNRaU9uc2lhV1FpT2lJME1qQTVaVEV3TmkxaU5XWmtMVFJrWkRNdE9EZ3lNUzB5TTJWaFptVTRNRFk0TmpnaUxDSnpkV0owZVhCbElqb2lSbWxuZFhKbElpd2lkSGx3WlNJNklsQnNiM1FpZlN3aWRHbGphMlZ5SWpwN0ltbGtJam9pTVdRMlpXVXpOMk10WXpkbU5DMDBZVGhqTFRneU9ESXRNVGcyWVRBMU5EVTVNVEE1SWl3aWRIbHdaU0k2SWtKaGMybGpWR2xqYTJWeUluMTlMQ0pwWkNJNkltUXpaVEU1WWpFMkxUa3labVl0TkRVNFpTMDVOMkUzTFRGaVltSTVZVGt4TXpoa055SXNJblI1Y0dVaU9pSk1hVzVsWVhKQmVHbHpJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdmU3dpYVdRaU9pSXhaRFpsWlRNM1l5MWpOMlkwTFRSaE9HTXRPREk0TWkweE9EWmhNRFUwTlRreE1Ea2lMQ0owZVhCbElqb2lRbUZ6YVdOVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnQ5TENKcFpDSTZJams0TXpZMk5qWm1MV1UzWVRZdE5EUmxZaTFpTnpkaExXWXlaRFZoWmpsbFlqQTNOaUlzSW5SNWNHVWlPaUpNYVc1bFlYSlRZMkZzWlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKallXeHNZbUZqYXlJNmJuVnNiQ3dpWTI5c2RXMXVYMjVoYldWeklqcGJJbmdpTENKNUlsMHNJbVJoZEdFaU9uc2llQ0k2ZXlKZlgyNWtZWEp5WVhsZlh5STZJa0ZCUkVGckszcHNaRlZKUVVGTFowTTRUMVl4VVdkQlFXdElTSG8xV0ZaRFFVRkNORFJRWW14a1ZVbEJRVWRDVUN0MVZqRlJaMEZCVTB3M09UVllWa05CUVVGM1RGRkliV1JWU1VGQlFtbGpRazlhTVZGblFVRkJRWE5KTlc1V1EwRkJSRzlsVVhadFpGVkpRVUZPUkc5RWRWb3hVV2RCUVhWR1kxTTFibFpEUVVGRFozaG9XRzFrVlVsQlFVbG5NVWRsV2pGUlowRkJZMHRSWXpWdVZrTkJRVUpaUlhsRWJXUlZTVUZCUlVORFNTdGFNVkZuUVVGTFVFVnROVzVXUTBGQlFWRlpRM0p0WkZWSlFVRlFhazlNWlZveFVXZEJRVFJFTUhnMWJsWkRRVUZFU1hKRVZHMWtWVWxCUVV4QllrOVBXakZSWjBGQmJVbHZOelZ1VmtOQlFVTkJLMVEzYldSVlNVRkJSMmh2VVhWYU1WRm5RVUZWVG1SR05XNVdRMEZCUVRSU2EyNXRaRlZKUVVGRFF6RlVUMW94VVdkQlFVTkRVbEUxYmxaRFFVRkVkMnRzVUcxa1ZVbEJRVTVuUWxZcldqRlJaMEZCZDBoQ1lUVnVWa05CUVVOdk16RXpiV1JWU1VGQlNrSlBXV1ZhTVZGblFVRmxUREZyTlc1V1EwRkJRbWRNUjJwdFpGVkpRVUZGYVdKaEsxb3hVV2RCUVUxQmNIWTFibFpEUVVGQldXVllURzFrVlVsQlFVRkViMlJsV2pGUlowRkJOa1phTlRWdVZrTkJRVVJSZUZoNmJXUlZTVUZCVEdjd1owOWFNVkZuUVVGdlMwOUVOVzVXUTBGQlEwbEZiMlp0WkZWSlFVRklRMEpwZFZveFVXZEJRVmRRUTA0MWJsWkRRVUZDUVZnMVNHMWtWVWxCUVVOcVQyeFBXakZSWjBGQlJVUXlXVFZ1VmtOQlFVUTBjVFYyYldSVlNVRkJUMEZoYml0YU1WRm5RVUY1U1cxcE5XNVdRMEZCUTNjclMxaHRaRlZKUVVGS2FHNXhaVm94VVdkQlFXZE9ZWE0xYmxaRFFVRkNiMUppUkcxa1ZVbEJRVVpETUhNcldqRlJaMEZCVDBOUE16VnVWa05CUVVGbmEzSnliV1JWU1VGQlFXZENkblZhTVZGblFVRTRSeTlDTlc1V1EwRkJSRmt6YzFSdFpGVkpRVUZOUWs1NVQxb3hVV2RCUVhGTWVrdzFibFpEUVVGRFVVczRMMjFrVlVsQlFVaHBZVEIxV2pGUlowRkJXVUZ1VnpWdVZrTkJRVUpKWlU1dWJXUlZTVUZCUkVSdU0wOWFNVkZuUVVGSFJtSm5OVzVXUTBGQlFVRjRaVkJ0WkZWSlFVRlBaM28xSzFveFVXZEJRVEJMVEhFMWJsWkRRVUZETkVWbE4yMWtWVWxCUVV0RFFUaGxXakZSWjBGQmFVOHZNRFZ1VmtOQlFVSjNXSFpxYldSVlNVRkJSbXBPS3l0YU1WRm5RVUZSUkhvdk5XNVdRMEZCUVc5eGQweHVaRlZKUVVGQ1FXRkNkV1F4VVdkQlFTdEpaMG8xTTFaRFFVRkVaemwzZW01a1ZVbEJRVTFvYlVWUFpERlJaMEZCYzA1VlZEVXpWa05CUVVOWlVrSm1ibVJWU1VGQlNVTjZSM1ZrTVZGblFVRmhRMGxsTlROV1EwRkJRbEZyVTBodVpGVkpRVUZFWjBGS1pXUXhVV2RCUVVsSE9HODFNMVpEUVVGQlNUTnBkbTVrVlVsQlFWQkNUVXdyWkRGUlowRkJNa3h6ZVRVelZrTkJRVVJCUzJwaWJtUlZTVUZCUzJsYVQyVmtNVkZuUVVGclFXYzVOVE5XUTBGQlFqUmtNRVJ1WkZWSlFVRkhSRzFSSzJReFVXZEJRVk5HVmtnMU0xWkRRVUZCZDNoRmNtNWtWVWxCUVVKbmVsUjFaREZSWjBGQlFVdEtValV6VmtOQlFVUnZSVVpZYm1SVlNVRkJUa0l2VjA5a01WRm5RVUYxVHpWaU5UTldRMEZCUTJkWVZpOXVaRlZKUVVGSmFrMVpkV1F4VVdkQlFXTkVkRzAxTTFaRFFVRkNXWEZ0Ym01a1ZVbEJRVVZCV21KbFpERlJaMEZCUzBsb2R6VXpWa05CUVVGUk9UTlFibVJWU1VGQlVHaHNaQ3RrTVZGblFVRTBUbEkyTlROV1EwRkJSRWxSTXpkdVpGVkpRVUZNUTNsblpXUXhVV2RCUVcxRFIwWTFNMVpEUVVGRFFXdEphbTVrVlVsQlFVZHFMMmtyWkRGUlowRkJWVWMyVURVelZrTkJRVUUwTTFwTWJtUlZTVUZCUTBKTmJIVmtNVkZuUVVGRFRIVmFOVE5XUTBGQlJIZExXak51WkZWSlFVRk9hVmx2VDJReFVXZEJRWGRCWldzMU0xWkRRVUZEYjJSeFptNWtWVWxCUVVwRWJIRjFaREZSWjBGQlpVWlRkVFV6VmtOQlFVSm5kemRJYm1SVlNVRkJSV2Q1ZEdWa01WRm5RVUZOUzBjME5UTldRMEZCUVZsRlRIcHVaRlZKUVVGQlFpOTJLMlF4VVdkQlFUWlBNME0xTTFaRFFVRkVVVmhOWW01a1ZVbEJRVXhxVEhsbFpERlJaMEZCYjBSeVRqVXpWa05CUVVOSmNXUkVibVJWU1VGQlNFRlpNVTlrTVZGblFVRlhTV1pZTlROV1F5SXNJbVIwZVhCbElqb2labXh2WVhRMk5DSXNJbk5vWVhCbElqcGJNVFEwWFgwc0lua2lPbnNpWDE5dVpHRnljbUY1WDE4aU9pSkJRVUZCZDAxaFFVMXJRVUZCUVVGQk1VWlplVkZCUVVGQlJVUm9URVJLUVVGQlFVRm5UelJEVFd0QlFVRkJRV2RSWlRoNFVVRkJRVUZMUTFReWVrWkJRVUZCUVZGUFlraE5WVUZCUVVGRFoyTmlTWGhSUVVGQlFVRkVPVzVFUmtGQlFVRkJXVWxwU0UxVlFVRkJRVVJuVVZoWmVGRkJRVUZCU1VRM1drUkdRVUZCUVVGQlRGWlVUVlZCUVVGQlJHYzFWalI0VVVGQlFVRk5RVmRoYWtaQlFVRkJRVzlGWkRGTlZVRkJRVUZFUVdsaWIzaFJRVUZCUVUxRVRDOTZSa0ZCUVVGQk5FRXhSazFyUVVGQlFVTkJXRmRWZVZGQlFVRkJRME4wYUZSS1FVRkJRVUYzVUhsc1RXdEJRVUZCUWtGNFdVRjVVVUZCUVVGUFEwNVhla3BCUVVGQlFWbEdXVEpOYTBGQlFVRkNaMmRTTUhsUlFVRkJRVWxEYzBKRVNrRkJRVUZCWjA1bWNrMVZRVUZCUVVGQlJVNDRlRkZCUVVGQlNVSkpNR3BHUVVGQlFVRkJTVWhHVFZWQlFVRkJRMEZvY21kNFVVRkJRVUZQUTB4eGVrWkJRVUZCUVZsS1IyVk5WVUZCUVVGQloybzFXWGhSUVVGQlFVOURUV3BxUmtGQlFVRkJiMGx4UjAxVlFVRkJRVUpCV25CbmVGRkJRVUZCVDBKQ2NXcEdRVUZCUVVGblFqSTRUVlZCUVVGQlFXZFZaRGg0VVVGQlFVRk5RMFZCYWtwQlFVRkJRVmxNWjJ4TmEwRkJRVUZEWnpORVZYbFJRVUZCUVVGQlFsSnFTa0ZCUVVGQlVVTldWMDFyUVVGQlFVUm5aMFEwZVZGQlFVRkJTVVJqU21wS1FVRkJRVUZKUkdkUVRXdEJRVUZCUW1jNGQwMTVVVUZCUVVGSlEzVXJSRVpCUVVGQlFYZEhiblJOVlVGQlFVRkVRV2xQUVhoUlFVRkJRVTlEYmpCNlJrRkJRVUZCTkUxaVIwMVZRVUZCUVVGQk5XSlplRkZCUVVGQlJVRkVjSHBHUVVGQlFVRlpRMGRZVFZWQlFVRkJSR2RLU1RSNFVVRkJRVUZIUVc5b1ZFWkJRVUZCUVRSRGREaE5WVUZCUVVGQlowNDBaM2hSUVVGQlFVbENRMnhFUmtGQlFVRkJkMFV5WjAxVlFVRkJRVU5CYldKTmVGRkJRVUZCUjBSc2VHcEdRVUZCUVVGSlJFaGhUVlZCUVVGQlEyZEhUMFY0VVVGQlFVRkJRVUUyUkVaQlFVRkJRV2RQWm5WTlZVRkJRVUZEWjBwbFNYaFJRVUZCUVU5Q2FqRlVSa0ZCUVVGQlFVdE1TVTFWUVVGQlFVRkJOM0V3ZUZGQlFVRkJRMEUyYTNwR1FVRkJRVUZKU1ZvMFRWVkJRVUZCUVdkTlYzTjRVVUZCUVVGRFJHTllWRVpCUVVGQlFVbEpaRkZOVlVGQlFVRkVRVzVWVlhoUlFVRkJRVWRETUU5cVJrRkJRVUZCUVUxemRrMVZRVUZCUVVSQmJubGplRkZCUVVGQlNVSXdTSHBHUVVGQlFVRlJSV3RZVFZWQlFVRkJRMmRoZWtGNFVVRkJRVUZCUTA5VFZFWkJRVUZCUVZsTVFtbE5WVUZCUVVGQlFYRmhXWGhSUVVGQlFVbERhRFpxUmtGQlFVRkJTVXB2ZFUxclFVRkJRVUZuVW13MGVWRkJRVUZCUVVSNWFsUktRVUZCUVVGQlNqWTVUV3RCUVVGQlEyZDNObWQ1VVVGQlFVRkhSSEJyZWtwQlFVRkJRVUZCT1M5TmEwRkJRVUZEWjBvd1dYbFJRVUZCUVVOQ1FVUlVTa0ZCUVVGQmQwWnFWVTFWUVVGQlFVRkJWemMwZUZGQlFVRkJRMEprY1VSR1FVRkJRVUZaUml0VFRWVkJRVUZCUVdkaWIxRjRVVUZCUVVGUFFqaGtha1pCUVVGQlFXOUpkRzlOVlVGQlFVRkVRVkF5UVhoUlFVRkJRVTFFZWxaNlJrRkJRVUZCTkV0a1VFMVZRVUZCUVVOQlVraFJlRkZCUVVGQlEwUm9iVVJHUVVGQlFVRjNTREk1VFZWQlFVRkJSR2RGVVhkNVVVRkJRVUZQUTJ4WGFrcEJRVUZCUVVGRWNYQk5hMEZCUVVGRVFUZE9hM2xSUVVGQlFVZERaa05xVGtGQlFVRkJTVVpKTjAwd1FVRkJRVUZuU1hRNGVWRkJRVUZCUTBSNVoycEtRVUZCUVVGSlRVbHRUV3RCUVVGQlEwRTRkMEY1VVVGQlFVRkJRV3d5ZWtaQlFVRkJRVmxHWVRGTlZVRkJRVUZDWjJseFdYaFJRVUZCUVVWREsyeDZSa0ZCUVVGQlVWQkxTVTFWUVVGQlFVTkJNVmc0ZUZGQlFVRkJUVU0wWkdwR1FVRkJRVUZCU25oMFRWVkJRVUZCUW1kSmJXOTRVVUZCUVVGTlEyOWFha1pCUVVGQlFVbERPV3BOVlVGQlFVRkRRV1p3UlhoUlFVRkJRVUZFVDNaNlJrRkJRVUZCV1VJemRVMVZRVUZCUVVGQldtc3dlVkZCUVVGQlNVTjFja1JLUVVGQlFVRkpVR05NVFRCQlFVRkJRMEZrVWtGNlVVRkJRVUZOUkhwR1JFNUJRVUZCUVVsSVNWcE5NRUZCUVVGQloyTm9hM3BSUVVGQlFVTkNlVWRVVGtFaUxDSmtkSGx3WlNJNkltWnNiMkYwTmpRaUxDSnphR0Z3WlNJNld6RTBORjE5Zlgwc0ltbGtJam9pTVRnd01qZGhOREl0Wm1Zd05pMDBPRFUxTFdFMVptUXRNVE14Wm1GaFlqUTFaREpoSWl3aWRIbHdaU0k2SWtOdmJIVnRia1JoZEdGVGIzVnlZMlVpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdGaVpXd2lPbnNpZG1Gc2RXVWlPaUprYUhkZk5XdHRJbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lJd05qY3lZVFE0TVMwMll6ZzBMVFEzTW1VdFlUZzFOUzFqTm1VellUaGhOV1JqWVRNaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFgwc0ltbGtJam9pWm1Nellqa3dNVEV0TlRsaU1DMDBOek0zTFRnM05ETXRZbUl6Tm1JMk0yRXlZMlJoSWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW1OdmJIVnRibDl1WVcxbGN5STZXeUo0SWl3aWVTSmRMQ0prWVhSaElqcDdJbmdpT25zaVgxOXVaR0Z5Y21GNVgxOGlPaUpCUVVKQldEVkliV1JWU1VGQlEycFBiRTlhTVZGblFVRkZSREpaTlc1V1EwRkJSRFJ4TlhadFpGVkpRVUZQUVdGdUsxb3hVV2RCUVhsSmJXazFibFpEUVVGRGR5dExXRzFrVlVsQlFVcG9ibkZsV2pGUlowRkJaMDVoY3pWdVZrTkJRVUp2VW1KRWJXUlZTVUZCUmtNd2N5dGFNVkZuUVVGUFEwOHpOVzVXUTBGQlFXZHJjbkp0WkZWSlFVRkJaMEoyZFZveFVXZEJRVGhITDBJMWJsWkRRVUZFV1ROelZHMWtWVWxCUVUxQ1RubFBXakZSWjBGQmNVeDZURFZ1VmtOQlFVTlJTemd2YldSVlNVRkJTR2xoTUhWYU1WRm5RVUZaUVc1WE5XNVdRMEZCUWtsbFRtNXRaRlZKUVVGRVJHNHpUMW94VVdkQlFVZEdZbWMxYmxaRFFVRkJRWGhsVUcxa1ZVbEJRVTluZWpVcldqRlJaMEZCTUV0TWNUVnVWa05CUVVNMFJXVTNiV1JWU1VGQlMwTkJPR1ZhTVZGblFVRnBUeTh3Tlc1V1EwRkJRbmRZZG1wdFpGVkpRVUZHYWs0cksxb3hVV2RCUVZGRWVpODFibFpEUVVGQmIzRjNURzVrVlVsQlFVSkJZVUoxWkRGUlowRkJLMGxuU2pVelZrTkJRVVJuT1hkNmJtUlZTVUZCVFdodFJVOWtNVkZuUVVGelRsVlVOVE5XUTBGQlExbFNRbVp1WkZWSlFVRkpRM3BIZFdReFVXZEJRV0ZEU1dVMU0xWkRRVUZDVVd0VFNHNWtWVWxCUVVSblFVcGxaREZSWjBGQlNVYzRielV6VmtOQlFVRkpNMmwyYm1SVlNVRkJVRUpOVEN0a01WRm5RVUV5VEhONU5UTldRMEZCUkVGTGFtSnVaRlZKUVVGTGFWcFBaV1F4VVdkQlFXdEJaemsxTTFaRFFVRkNOR1F3Ukc1a1ZVbEJRVWRFYlZFclpERlJaMEZCVTBaV1NEVXpWa05CUVVGM2VFVnlibVJWU1VGQlFtZDZWSFZrTVZGblFVRkJTMHBTTlROV1EwRkJSRzlGUmxodVpGVkpRVUZPUWk5WFQyUXhVV2RCUVhWUE5XSTFNMVpEUVVGRFoxaFdMMjVrVlVsQlFVbHFUVmwxWkRGUlowRkJZMFIwYlRVelZrTkJRVUpaY1cxdWJtUlZTVUZCUlVGYVltVmtNVkZuUVVGTFNXaDNOVE5XUTBGQlFWRTVNMUJ1WkZWSlFVRlFhR3hrSzJReFVXZEJRVFJPVWpZMU0xWkRRVUZFU1ZFek4yNWtWVWxCUVV4RGVXZGxaREZSWjBGQmJVTkhSalV6VmtOQlFVTkJhMGxxYm1SVlNVRkJSMm92YVN0a01WRm5RVUZWUnpaUU5UTldRMEZCUVRReldreHVaRlZKUVVGRFFrMXNkV1F4VVdkQlFVTk1kVm8xTTFaRFFVRkVkMHRhTTI1a1ZVbEJRVTVwV1c5UFpERlJaMEZCZDBGbGF6VXpWa05CUVVOdlpIRm1ibVJWU1VGQlNrUnNjWFZrTVZGblFVRmxSbE4xTlROV1EwRkJRbWQzTjBodVpGVkpRVUZGWjNsMFpXUXhVV2RCUVUxTFJ6UTFNMVpEUVVGQldVVk1lbTVrVlVsQlFVRkNMM1lyWkRGUlowRkJOazh6UXpVelZrTkJRVVJSV0UxaWJtUlZTVUZCVEdwTWVXVmtNVkZuUVVGdlJISk9OVE5XUTBGQlEwbHhaRVJ1WkZWSlFVRklRVmt4VDJReFVXZEJRVmRKWmxnMU0xWkRRVUZDUVRsMGNtNWtWVWxCUVVOb2JETjFaREZSWjBGQlJVNVVhRFV6VmtOQlFVUTBVWFZZYm1SVlNVRkJUME40Tms5a01WRm5RVUY1UTBSek5UTldRMEZCUTNkcUt5OXVaRlZKUVVGS2FpczRkV1F4VVdkQlFXZEhNekkxTTFaRFFVRkNiek5RYm01a1ZVbEJRVVpDVEM5bFpERlJaMEZCVDB4dlFUWklWa05CUVVGblMxRlViMlJWU1VGQlFXbFpRaXRvTVZGblFVRTRRVmxNTmtoV1EwRkJSRmxrVVRkdlpGVkpRVUZOUkd0RlpXZ3hVV2RCUVhGR1RWWTJTRlpEUVVGRFVYZG9hbTlrVlVsQlFVaG5lRWhQYURGUlowRkJXVXRCWmpaSVZrTkJRVUpKUkhsUWIyUlZTVUZCUkVJclNuVm9NVkZuUVVGSFR6QndOa2hXUTBGQlFVRllRek52WkZWSlFVRlBha3ROVDJneFVXZEJRVEJFYXpBMlNGWkRRVUZETkhGRVptOWtWVWxCUVV0QldFOHJhREZSWjBGQmFVbFpLelpJVmtOQlFVSjNPVlZJYjJSVlNVRkJSbWhyVW1Wb01WRm5RVUZSVGs1Sk5raFdRMEZCUVc5UmEzcHZaRlZKUVVGQ1EzaFVLMmd4VVdkQlFTdENPVlEyU0ZaRFFVRkVaMnBzWW05a1ZVbEJRVTFxT1ZkbGFERlJaMEZCYzBkNFpEWklWa05CUVVOWk1qSkViMlJWU1VGQlNVSkxXazlvTVZGblFVRmhUR3h1TmtoV1EwRkJRbEZMUjNadlpGVkpRVUZFYVZoaWRXZ3hVV2RCUVVsQlduazJTRlpEUVVGQlNXUllXRzlrVlVsQlFWQkVhbVZQYURGUlowRkJNa1pLT0RaSVZrTWlMQ0prZEhsd1pTSTZJbVpzYjJGME5qUWlMQ0p6YUdGd1pTSTZXekUwTkYxOUxDSjVJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lRVUZCUVZsT00yWk5SVUZCUVVGRFFXTmxjM2RSUVVGQlFVdEJSamw2UWtGQlFVRkJkMHByUTAxVlFVRkJRVUpuV0M5amQxRkJRVUZCUTBGc04wUkNRVUZCUVVGM1QzSm5UVVZCUVVGQlFrRkhaR2QzVVVGQlFVRkxRa2g2ZWtKQlFVRkJRVWxJWWtkTlJVRkJRVUZEWjFnNFNYZFJRVUZCUVVGQ1NuWnFRa0ZCUVVGQlowUkxOazFGUVVGQlFVTkJSVGhaZDFGQlFVRkJTVVF3TUZSQ1FVRkJRVUZuVGxoa1RVVkJRVUZCUTJkMEt6UjNVVUZCUVVGUFExb3Zla0pCUVVGQlFVRklkMUZOVlVGQlFVRkJRVWRSTkhoUlFVRkJRVUZETWtONlJrRkJRVUZCUVVaTlNrMVZRVUZCUVVGQlFrRk5lRkZCUVVGQlFVTXhMMFJDUVVGQlFVRkJSMkl5VFVWQlFVRkJSRUZEZDAxNFVVRkJRVUZKUTNoRWVrWkJRVUZCUVZGR1kyTk5WVUZCUVVGRVFVNUJPSGhSUVVGQlFVTkJVMEZxUmtGQlFVRkJiMDh2TUUxRlFVRkJRVVJCZUhWdmQxRkJRVUZCUVVObE5FUkNRVUZCUVVGSlNGaFhUVVZCUVVGQlJFRk1PRFIzVVVGQlFVRkpSSEY0VkVKQlFVRkJRVWxMVnpsTlJVRkJRVUZCWjNOamIzZFJRVUZCUVVWRE9URjZRa0ZCUVVGQlVVMXVhMDFGUVVGQlFVUm5SMUJ6ZDFGQlFVRkJTVUp2UlZSR1FVRkJRVUZKVEdkdVRWVkJRVUZCUTBGU2VUaDRVVUZCUVVGTlJGZE9ha1pCUVVGQlFVbEhXU3ROVlVGQlFVRkVRVGQ2UlhoUlFVRkJRVVZDTlVwVVJrRkJRVUZCTkVGSldrMVZRVUZCUVVSQmMyZFZlRkZCUVVGQlMwSnBPR3BDUVVGQlFVRm5Ra3htVFVWQlFVRkJRbWRMZEVWM1VVRkJRVUZEUWtOM2VrSkJRVUZCUVVGR2NURk5SVUZCUVVGQ1p6RkxhM2RSUVVGQlFVMUNUMjVxUWtGQlFVRkJTVTF0VTAxRlFVRkJRVUpuVFRSemQxRkJRVUZCU1VOa1ozcENRVUZCUVVGM1FXUTRUVVZCUVVGQlFXZGtTV04zVVVGQlFVRkhSR2RyYWtKQlFVRkJRWGRGZVdWTlJVRkJRVUZEWjBoeVJYZFJRVUZCUVV0RWQzZDZRa0ZCUVVGQlowMU1WMDFGUVVGQlFVRm5kWFIzZDFGQlFVRkJUME40TkdwQ1FVRkJRVUZuUzI1dlRVVkJRVUZCUkdkNVpEUjNVVUZCUVVGSFJIRXhSRUpCUVVGQlFYZEJja3hOUlVGQlFVRkJRVEZNTkhkUlFVRkJRVU5EWkhOcVFrRkJRVUZCV1VkaGJVMUZRVUZCUVVOQmIxcDNkMUZCUVVGQlRVUmphMnBDUVVGQlFVRTBRbVZLVFVWQlFVRkJSRUZ3TXpSM1VVRkJRVUZOUVROa1JFSkJRVUZCUVc5TlpIQk5SVUZCUVVGRFp6TkhVWGRSUVVGQlFVbEVlRmg2UWtGQlFVRkJaMEZhWWsxRlFVRkJRVU5CTUcxbmQxRkJRVUZCUjBObFpHcENRVUZCUVVGWlIzRkZUVVZCUVVGQlEyZDRjRlYzVVVGQlFVRlBRV2x3ZWtKQlFVRkJRVWxJS3pSTlJVRkJRVUZEWjFRM2EzZFJRVUZCUVVOQlozVnFRa0ZCUVVGQmIxQkROazFGUVVGQlFVRkJTamRWZDFGQlFVRkJTVUprY25wQ1FVRkJRVUUwU2s5d1RVVkJRVUZCUTBGSWNVMTNVVUZCUVVGQlEzQnVSRUpCUVVGQlFXOUVUMWROUlVGQlFVRkJRVWsxVFhkUlFVRkJRVVZCVTJ0RVFrRkJRVUZCYjBGSFRrMUZRVUZCUVVSbmNEUlZkMUZCUVVGQlJVSlBabXBDUVVGQlFVRm5VRkl5VFVWQlFVRkJRbWQ0YmtsM1VVRkJRVUZEUTFsaWFrSkJRVUZCUVVGSGNIRk5SVUZCUVVGRFoyZHVZM2RSUVVGQlFVVkRZbWhFUWtGQlFVRkJORXhQVWsxRlFVRkJRVVJCTXpVd2QxRkJRVUZCUzBGTWNXcENRVUZCUVVGblJHVXlUVVZCUVVGQlFtZFdURFIzVVVGQlFVRkRRbmg0YWtKQlFVRkJRVUZKTjA5TlJVRkJRVUZEUVdaTlRYZFJRVUZCUVVGQ2NuVkVRa0ZCUVVGQlowWnRkRTFGUVVGQlFVUm5lbkZuZDFGQlFVRkJRMEpGY0VSQ1FVRkJRVUZuVEcxbVRVVkJRVUZCUTJkWU5YZDNVVUZCUVVGTFFVWnRWRUpCUVVGQlFYZExkVlpOUlVGQlFVRkRRV1JaTUhkUlFVRkJRVWRCTDJoVVFrRkJRVUZCU1VGc09VMUZRVUZCUVVGbk5XNVpkMUZCUVVGQlEwUkVZMFJDUVVGQlFVRkpTMEp4VFVWQlFVRkJRVUZpYmtWM1VVRkJRVUZOUVRkbFJFSkJRVUZCUVc5QmJDOU5SVUZCUVVGRVoyOXZZM2RSUVVGQlFVRkJPR3RFUWtGQlFVRkJVVTVYV1UxRlFVRkJRVU5uUkdGUmQxRkJRVUZCUVVKSGNucENRVUZCUVVGWlNEWTJUVVZCUVVGQlFtZG1jbTkzVVVGQlFVRkhRaXQxYWtKQklpd2laSFI1Y0dVaU9pSm1iRzloZERZMElpd2ljMmhoY0dVaU9sc3hORFJkZlgxOUxDSnBaQ0k2SWpjeE0yUTJaVGMzTFRreU56QXROREU0WWkxaU5EZ3dMVFl5Wmpnd05UVXpOVEUzTXlJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1admNtMWhkSFJsY2lJNmV5SnBaQ0k2SW1SbU0yVmtZbVJpTFdFNE1HWXROR1poTWkwNE9UQXhMV1F5TXpWbU1UWTVaalpoTXlJc0luUjVjR1VpT2lKRVlYUmxkR2x0WlZScFkydEdiM0p0WVhSMFpYSWlmU3dpY0d4dmRDSTZleUpwWkNJNklqUXlNRGxsTVRBMkxXSTFabVF0TkdSa015MDRPREl4TFRJelpXRm1aVGd3TmpnMk9DSXNJbk4xWW5SNWNHVWlPaUpHYVdkMWNtVWlMQ0owZVhCbElqb2lVR3h2ZENKOUxDSjBhV05yWlhJaU9uc2lhV1FpT2lJMk1HVTVaV1ZsWlMxaE5qUTBMVFEyTTJZdFlqQmxOUzAwWW1ReU0yWXdPV1l4TWpRaUxDSjBlWEJsSWpvaVJHRjBaWFJwYldWVWFXTnJaWElpZlgwc0ltbGtJam9pWW1ZeU1UUmxabVl0WmpSbU9TMDBZekV4TFdGbE9HUXRNalV4WmpNNE1XWXlPVEV3SWl3aWRIbHdaU0k2SWtSaGRHVjBhVzFsUVhocGN5SjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnVkVzFmYldsdWIzSmZkR2xqYTNNaU9qVjlMQ0pwWkNJNklqWXdaVGxsWldWbExXRTJORFF0TkRZelppMWlNR1UxTFRSaVpESXpaakE1WmpFeU5DSXNJblI1Y0dVaU9pSkVZWFJsZEdsdFpWUnBZMnRsY2lKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKd2JHOTBJanA3SW1sa0lqb2lOREl3T1dVeE1EWXRZalZtWkMwMFpHUXpMVGc0TWpFdE1qTmxZV1psT0RBMk9EWTRJaXdpYzNWaWRIbHdaU0k2SWtacFozVnlaU0lzSW5SNWNHVWlPaUpRYkc5MEluMHNJblJwWTJ0bGNpSTZleUpwWkNJNklqWXdaVGxsWldWbExXRTJORFF0TkRZelppMWlNR1UxTFRSaVpESXpaakE1WmpFeU5DSXNJblI1Y0dVaU9pSkVZWFJsZEdsdFpWUnBZMnRsY2lKOWZTd2lhV1FpT2lKa1pXRmlZMlEzWlMxbE5XTmxMVFEwTWpBdFltVTBZeTFoTmpsbVpqYzRaV1JoT0RZaUxDSjBlWEJsSWpvaVIzSnBaQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzYVc1bFgyRnNjR2hoSWpwN0luWmhiSFZsSWpvd0xqRjlMQ0pzYVc1bFgyTmhjQ0k2SW5KdmRXNWtJaXdpYkdsdVpWOWpiMnh2Y2lJNmV5SjJZV3gxWlNJNklpTXhaamMzWWpRaWZTd2liR2x1WlY5cWIybHVJam9pY205MWJtUWlMQ0pzYVc1bFgzZHBaSFJvSWpwN0luWmhiSFZsSWpvMWZTd2llQ0k2ZXlKbWFXVnNaQ0k2SW5naWZTd2llU0k2ZXlKbWFXVnNaQ0k2SW5raWZYMHNJbWxrSWpvaU9XSmpNV0l5WkdFdE9XUXpOQzAwWlRGa0xUa3hOak10TkRkbE9HUmtNV1ppWkRNeElpd2lkSGx3WlNJNklreHBibVVpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWTJGc2JHSmhZMnNpT201MWJHd3NJbkJzYjNRaU9uc2lhV1FpT2lJME1qQTVaVEV3TmkxaU5XWmtMVFJrWkRNdE9EZ3lNUzB5TTJWaFptVTRNRFk0TmpnaUxDSnpkV0owZVhCbElqb2lSbWxuZFhKbElpd2lkSGx3WlNJNklsQnNiM1FpZlN3aWNtVnVaR1Z5WlhKeklqcGJleUpwWkNJNkltRTNPR0U0WXpkakxXRXdNamN0TkRFMFpTMDRNamxpTFdZNVlXUmpaVFJqTWpZeE1pSXNJblI1Y0dVaU9pSkhiSGx3YUZKbGJtUmxjbVZ5SW4xZExDSjBiMjlzZEdsd2N5STZXMXNpVG1GdFpTSXNJa2N4WDFOVFZGOUhURTlDUVV3aVhTeGJJa0pwWVhNaUxDSXRNQzQyTkNKZExGc2lVMnRwYkd3aUxDSXdMak14SWwxZGZTd2lhV1FpT2lJNE9UUmlOalE1TXkwelpXVTBMVFF5T0dJdFlXVmhNeTB3TkRGa05EZG1NakZqTWpjaUxDSjBlWEJsSWpvaVNHOTJaWEpVYjI5c0luMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltMWhlRjlwYm5SbGNuWmhiQ0k2TlRBd0xqQXNJbTUxYlY5dGFXNXZjbDkwYVdOcmN5STZNSDBzSW1sa0lqb2lNVGxsWWpNMlpURXRZalEzWVMwME4yTTBMV0ptTldJdFpqQTJZalpoTTJFNU0yRTFJaXdpZEhsd1pTSTZJa0ZrWVhCMGFYWmxWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteHBibVZmWVd4d2FHRWlPbnNpZG1Gc2RXVWlPakF1TmpWOUxDSnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2ZXlKMllXeDFaU0k2SWlOaFpXTTNaVGdpZlN3aWJHbHVaVjlxYjJsdUlqb2ljbTkxYm1RaUxDSnNhVzVsWDNkcFpIUm9JanA3SW5aaGJIVmxJam8xZlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pWmpSbE9XTXhNREl0T0RJM1lTMDBORGMzTFRrelpEVXRNbU5oWXpRek1qZGpZalpqSWl3aWRIbHdaU0k2SWt4cGJtVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2ljR3h2ZENJNmJuVnNiQ3dpZEdWNGRDSTZJalEwTURFekluMHNJbWxrSWpvaVpUZzFORFUyWXpjdFptWXpNQzAwT0RjeUxUZzNZV0V0T0dFNE5qUTNObUUwWW1Oaklpd2lkSGx3WlNJNklsUnBkR3hsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14cGJtVmZZV3h3YUdFaU9uc2lkbUZzZFdVaU9qQXVNWDBzSW14cGJtVmZZMkZ3SWpvaWNtOTFibVFpTENKc2FXNWxYMk52Ykc5eUlqcDdJblpoYkhWbElqb2lJekZtTnpkaU5DSjlMQ0pzYVc1bFgycHZhVzRpT2lKeWIzVnVaQ0lzSW14cGJtVmZkMmxrZEdnaU9uc2lkbUZzZFdVaU9qVjlMQ0o0SWpwN0ltWnBaV3hrSWpvaWVDSjlMQ0o1SWpwN0ltWnBaV3hrSWpvaWVTSjlmU3dpYVdRaU9pSmlZemRsTUdFM1l5MWlZbVE1TFRRME1XUXRPRE0yWVMxbU5EWmxOREUxT1dZNVpUSWlMQ0owZVhCbElqb2lUR2x1WlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKa1lYUmhYM052ZFhKalpTSTZleUpwWkNJNklqY3hNMlEyWlRjM0xUa3lOekF0TkRFNFlpMWlORGd3TFRZeVpqZ3dOVFV6TlRFM015SXNJblI1Y0dVaU9pSkRiMngxYlc1RVlYUmhVMjkxY21ObEluMHNJbWRzZVhCb0lqcDdJbWxrSWpvaVpqUmxPV014TURJdE9ESTNZUzAwTkRjM0xUa3paRFV0TW1OaFl6UXpNamRqWWpaaklpd2lkSGx3WlNJNklreHBibVVpZlN3aWFHOTJaWEpmWjJ4NWNHZ2lPbTUxYkd3c0ltMTFkR1ZrWDJkc2VYQm9JanB1ZFd4c0xDSnViMjV6Wld4bFkzUnBiMjVmWjJ4NWNHZ2lPbnNpYVdRaU9pSmlZemRsTUdFM1l5MWlZbVE1TFRRME1XUXRPRE0yWVMxbU5EWmxOREUxT1dZNVpUSWlMQ0owZVhCbElqb2lUR2x1WlNKOUxDSnpaV3hsWTNScGIyNWZaMng1Y0dnaU9tNTFiR3g5TENKcFpDSTZJbVkzTXpKaE1tSXpMV1kwWXpRdE5ESTBZaTFoTkdFd0xUZGlaamt5Tm1VM1l6UTROaUlzSW5SNWNHVWlPaUpIYkhsd2FGSmxibVJsY21WeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteGhZbVZzSWpwN0luWmhiSFZsSWpvaVNGbERUMDBpZlN3aWNtVnVaR1Z5WlhKeklqcGJleUpwWkNJNklqVmlNbVl3TkdSaExUSTRNamt0TkRrMVpTMWlNR05pTFRkbU9XUTVaV1E1TVRoaFl5SXNJblI1Y0dVaU9pSkhiSGx3YUZKbGJtUmxjbVZ5SW4xZGZTd2lhV1FpT2lKaFltSXpZamt3TUMxa1kyTm1MVFExTlRNdFlXTmxOeTAzTWpJek1UQmlNamcyTnpFaUxDSjBlWEJsSWpvaVRHVm5aVzVrU1hSbGJTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2ZXlKMllXeDFaU0k2SW1OeWFXMXpiMjRpZlN3aWJHbHVaVjlxYjJsdUlqb2ljbTkxYm1RaUxDSnNhVzVsWDNkcFpIUm9JanA3SW5aaGJIVmxJam8xZlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pWXpabU5ETTBPV010WW1NMVpTMDBNRFpqTFdJd1lXTXRORGRpTkRrMFpqRTRZVGRoSWl3aWRIbHdaU0k2SWt4cGJtVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2laR0YwWVY5emIzVnlZMlVpT25zaWFXUWlPaUkzWlRjNE1tSTNOeTFsWXpjekxUUTJNVFF0T1RRNU15MWtNbUV5TlRZMVptVmlZekVpTENKMGVYQmxJam9pUTI5c2RXMXVSR0YwWVZOdmRYSmpaU0o5TENKbmJIbHdhQ0k2ZXlKcFpDSTZJbU0yWmpRek5EbGpMV0pqTldVdE5EQTJZeTFpTUdGakxUUTNZalE1TkdZeE9HRTNZU0lzSW5SNWNHVWlPaUpNYVc1bEluMHNJbWh2ZG1WeVgyZHNlWEJvSWpwdWRXeHNMQ0p0ZFhSbFpGOW5iSGx3YUNJNmJuVnNiQ3dpYm05dWMyVnNaV04wYVc5dVgyZHNlWEJvSWpwN0ltbGtJam9pT1dKak1XSXlaR0V0T1dRek5DMDBaVEZrTFRreE5qTXRORGRsT0dSa01XWmlaRE14SWl3aWRIbHdaU0k2SWt4cGJtVWlmU3dpYzJWc1pXTjBhVzl1WDJkc2VYQm9JanB1ZFd4c2ZTd2lhV1FpT2lJeFlqRTJOalE1WXkxbU5HVmtMVFF4TlRVdE9UUTROaTFqT1RZd09XRmhOemMyT1RZaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUprWVhSaFgzTnZkWEpqWlNJNmV5SnBaQ0k2SWpGbE0ySTROamMxTFROaFl6QXRORFF3WkMwNVpURmtMVFUwT0dabU9XRmxOalkyTXlJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc0ltZHNlWEJvSWpwN0ltbGtJam9pWXpVeE9ESmhOMll0WVRNek1pMDBOR0prTFRreU9HTXRaR1JtWVdZd09URmpaRE0zSWl3aWRIbHdaU0k2SWt4cGJtVWlmU3dpYUc5MlpYSmZaMng1Y0dnaU9tNTFiR3dzSW0xMWRHVmtYMmRzZVhCb0lqcHVkV3hzTENKdWIyNXpaV3hsWTNScGIyNWZaMng1Y0dnaU9uc2lhV1FpT2lKa09UZ3pZMlpoT1MwelpqVTFMVFJtTjJZdE9ERmlaQzFrWkRBMlpUVTRPRGc0WlRVaUxDSjBlWEJsSWpvaVRHbHVaU0o5TENKelpXeGxZM1JwYjI1ZloyeDVjR2dpT201MWJHeDlMQ0pwWkNJNklqQTJOekpoTkRneExUWmpPRFF0TkRjeVpTMWhPRFUxTFdNMlpUTmhPR0UxWkdOaE15SXNJblI1Y0dVaU9pSkhiSGx3YUZKbGJtUmxjbVZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14aFltVnNJanA3SW5aaGJIVmxJam9pVGtWRFQwWlRYMDFoYzNOQ1lYa2lmU3dpY21WdVpHVnlaWEp6SWpwYmV5SnBaQ0k2SW1ZM016SmhNbUl6TFdZMFl6UXROREkwWWkxaE5HRXdMVGRpWmpreU5tVTNZelE0TmlJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjFkZlN3aWFXUWlPaUpoWXpGbU1HSmxPUzAyWW1JNUxUUmxaRFF0T0RNeE1DMWpaRGt6WkRrNVlXRmlaV0VpTENKMGVYQmxJam9pVEdWblpXNWtTWFJsYlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKallXeHNZbUZqYXlJNmJuVnNiQ3dpY0d4dmRDSTZleUpwWkNJNklqUXlNRGxsTVRBMkxXSTFabVF0TkdSa015MDRPREl4TFRJelpXRm1aVGd3TmpnMk9DSXNJbk4xWW5SNWNHVWlPaUpHYVdkMWNtVWlMQ0owZVhCbElqb2lVR3h2ZENKOUxDSnlaVzVrWlhKbGNuTWlPbHQ3SW1sa0lqb2lNRFkzTW1FME9ERXRObU00TkMwME56SmxMV0U0TlRVdFl6WmxNMkU0WVRWa1kyRXpJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZWMHNJblJ2YjJ4MGFYQnpJanBiV3lKT1lXMWxJaXdpWkdoM1h6VnJiU0pkTEZzaVFtbGhjeUlzSWkwd0xqRTNJbDBzV3lKVGEybHNiQ0lzSWpBdU1USWlYVjE5TENKcFpDSTZJbVZsWmpObE1tSTJMV014WlRFdE5ERTVOeTA1T1RFeExUTTVNbVpoWkRka05tRTFOaUlzSW5SNWNHVWlPaUpJYjNabGNsUnZiMndpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWW1GelpTSTZOakFzSW0xaGJuUnBjM05oY3lJNld6RXNNaXcxTERFd0xERTFMREl3TERNd1hTd2liV0Y0WDJsdWRHVnlkbUZzSWpveE9EQXdNREF3TGpBc0ltMXBibDlwYm5SbGNuWmhiQ0k2TVRBd01DNHdMQ0p1ZFcxZmJXbHViM0pmZEdsamEzTWlPakI5TENKcFpDSTZJak5qWldOall6Qm1MVEZrTUdJdE5ESXhNQzFoTURJNUxUTmxNakJrTm1FM05HTXpPQ0lzSW5SNWNHVWlPaUpCWkdGd2RHbDJaVlJwWTJ0bGNpSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmtZWFJoWDNOdmRYSmpaU0k2ZXlKcFpDSTZJamM0T1RKaVlqVTBMV000TkdFdE5HWTVZaTFpTXpJNExXVXpaV1l5WmpnME9HWmtOeUlzSW5SNWNHVWlPaUpEYjJ4MWJXNUVZWFJoVTI5MWNtTmxJbjBzSW1kc2VYQm9JanA3SW1sa0lqb2lZMlEzWVRZME56QXROakl4TUMwME1XSTRMVGt6TldFdE1XVmtPR1JqWVdRd056SmpJaXdpZEhsd1pTSTZJa3hwYm1VaWZTd2lhRzkyWlhKZloyeDVjR2dpT201MWJHd3NJbTExZEdWa1gyZHNlWEJvSWpwdWRXeHNMQ0p1YjI1elpXeGxZM1JwYjI1ZloyeDVjR2dpT25zaWFXUWlPaUppTW1aaE1XSXpPUzB5TWpBNUxUUXhPR010WVRVeE1DMHpOekUyWmpJelpUSmtOVEFpTENKMGVYQmxJam9pVEdsdVpTSjlMQ0p6Wld4bFkzUnBiMjVmWjJ4NWNHZ2lPbTUxYkd4OUxDSnBaQ0k2SWpWaU1tWXdOR1JoTFRJNE1qa3RORGsxWlMxaU1HTmlMVGRtT1dRNVpXUTVNVGhoWXlJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbUpsYkc5M0lqcGJleUpwWkNJNkltSm1NakUwWldabUxXWTBaamt0TkdNeE1TMWhaVGhrTFRJMU1XWXpPREZtTWpreE1DSXNJblI1Y0dVaU9pSkVZWFJsZEdsdFpVRjRhWE1pZlYwc0lteGxablFpT2x0N0ltbGtJam9pWkRObE1UbGlNVFl0T1RKbVppMDBOVGhsTFRrM1lUY3RNV0ppWWpsaE9URXpPR1EzSWl3aWRIbHdaU0k2SWt4cGJtVmhja0Y0YVhNaWZWMHNJbkJzYjNSZmFHVnBaMmgwSWpveU5UQXNJbkJzYjNSZmQybGtkR2dpT2pjMU1Dd2ljbVZ1WkdWeVpYSnpJanBiZXlKcFpDSTZJbUptTWpFMFpXWm1MV1kwWmprdE5HTXhNUzFoWlRoa0xUSTFNV1l6T0RGbU1qa3hNQ0lzSW5SNWNHVWlPaUpFWVhSbGRHbHRaVUY0YVhNaWZTeDdJbWxrSWpvaVpHVmhZbU5rTjJVdFpUVmpaUzAwTkRJd0xXSmxOR010WVRZNVptWTNPR1ZrWVRnMklpd2lkSGx3WlNJNklrZHlhV1FpZlN4N0ltbGtJam9pWkRObE1UbGlNVFl0T1RKbVppMDBOVGhsTFRrM1lUY3RNV0ppWWpsaE9URXpPR1EzSWl3aWRIbHdaU0k2SWt4cGJtVmhja0Y0YVhNaWZTeDdJbWxrSWpvaVlXRXlZMk5oWW1FdFkyRTJZUzAwWWpJekxUa3dZek10TURnMU5XVTBOR1ZsWVdObUlpd2lkSGx3WlNJNklrZHlhV1FpZlN4N0ltbGtJam9pT1RoaE9HWmpPVGN0WXpZNE9DMDBNamd3TFRnM1lXUXRaR1F4WXpCaU9UZ3hOakUySWl3aWRIbHdaU0k2SWtKdmVFRnVibTkwWVhScGIyNGlmU3g3SW1sa0lqb2lOalkwWXpRMU16WXRNRFE1TlMwMFlqSTBMVGc0TXprdE1qQTROemxrTVRoa1pqSmhJaXdpZEhsd1pTSTZJa3hsWjJWdVpDSjlMSHNpYVdRaU9pSmhOemhoT0dNM1l5MWhNREkzTFRReE5HVXRPREk1WWkxbU9XRmtZMlUwWXpJMk1USWlMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOUxIc2lhV1FpT2lKbU56TXlZVEppTXkxbU5HTTBMVFF5TkdJdFlUUmhNQzAzWW1ZNU1qWmxOMk0wT0RZaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5TEhzaWFXUWlPaUkyWWpGbE9UWmlaQzAwT1RZM0xUUmhNell0WW1NM1lTMWxaakl3WlRBME9UVm1OeklpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlMSHNpYVdRaU9pSXhZakUyTmpRNVl5MW1OR1ZrTFRReE5UVXRPVFE0Tmkxak9UWXdPV0ZoTnpjMk9UWWlMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOUxIc2lhV1FpT2lKbFpUVTVaR1l4TnkwMFpXSTNMVFJqT0RrdFlUUXpaQzFrTlRjME5XVTVZVGd5T1RJaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5TEhzaWFXUWlPaUl3TmpjeVlUUTRNUzAyWXpnMExUUTNNbVV0WVRnMU5TMWpObVV6WVRoaE5XUmpZVE1pTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlMSHNpYVdRaU9pSTFZakptTURSa1lTMHlPREk1TFRRNU5XVXRZakJqWWkwM1pqbGtPV1ZrT1RFNFlXTWlMQ0owZVhCbElqb2lSMng1Y0doU1pXNWtaWEpsY2lKOVhTd2lkR2wwYkdVaU9uc2lhV1FpT2lKbE9EVTBOVFpqTnkxbVpqTXdMVFE0TnpJdE9EZGhZUzA0WVRnMk5EYzJZVFJpWTJNaUxDSjBlWEJsSWpvaVZHbDBiR1VpZlN3aWRHOXZiRjlsZG1WdWRITWlPbnNpYVdRaU9pSXlOVGs1TWpjek5TMDROakl6TFRRMU5UQXRPVGt5TVMwMFpHSTJZV1ZsWlRnelkyUWlMQ0owZVhCbElqb2lWRzl2YkVWMlpXNTBjeUo5TENKMGIyOXNZbUZ5SWpwN0ltbGtJam9pTmpRNU5HVXhZakV0Tm1WbU9DMDBNRGRpTFRnNU9UUXROREJrWVRobVlUazNaak0ySWl3aWRIbHdaU0k2SWxSdmIyeGlZWElpZlN3aWRHOXZiR0poY2w5c2IyTmhkR2x2YmlJNkltRmliM1psSWl3aWVGOXlZVzVuWlNJNmV5SnBaQ0k2SW1NMU1tSTBNVFUzTFdJMVkyRXRORFU1TmkxaE1EWmpMVE01TVRKbVpqaGtObUkyWmlJc0luUjVjR1VpT2lKRVlYUmhVbUZ1WjJVeFpDSjlMQ0o0WDNOallXeGxJanA3SW1sa0lqb2lZMlF3T0dZd01qRXRPVGhqTkMwME5EaGtMV0l4WkRZdFpXRTVNelE1TlRneU9USTJJaXdpZEhsd1pTSTZJa3hwYm1WaGNsTmpZV3hsSW4wc0lubGZjbUZ1WjJVaU9uc2lhV1FpT2lJNFpqRXhNamRtTnkwM09ESTRMVFF6TnpjdE9UazVOaTA0WTJFMlpEUmhNbUU1WkRVaUxDSjBlWEJsSWpvaVJHRjBZVkpoYm1kbE1XUWlmU3dpZVY5elkyRnNaU0k2ZXlKcFpDSTZJams0TXpZMk5qWm1MV1UzWVRZdE5EUmxZaTFpTnpkaExXWXlaRFZoWmpsbFlqQTNOaUlzSW5SNWNHVWlPaUpNYVc1bFlYSlRZMkZzWlNKOWZTd2lhV1FpT2lJME1qQTVaVEV3TmkxaU5XWmtMVFJrWkRNdE9EZ3lNUzB5TTJWaFptVTRNRFk0TmpnaUxDSnpkV0owZVhCbElqb2lSbWxuZFhKbElpd2lkSGx3WlNJNklsQnNiM1FpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdsdVpWOWhiSEJvWVNJNmV5SjJZV3gxWlNJNk1DNHhmU3dpYkdsdVpWOWpZWEFpT2lKeWIzVnVaQ0lzSW14cGJtVmZZMjlzYjNJaU9uc2lkbUZzZFdVaU9pSWpNV1kzTjJJMEluMHNJbXhwYm1WZmFtOXBiaUk2SW5KdmRXNWtJaXdpYkdsdVpWOTNhV1IwYUNJNmV5SjJZV3gxWlNJNk5YMHNJbmdpT25zaVptbGxiR1FpT2lKNEluMHNJbmtpT25zaVptbGxiR1FpT2lKNUluMTlMQ0pwWkNJNkltSmtPR1U1TjJGakxUVmpZV1F0TkdaaU5DMDRNakZsTFRBMk1HWTNZekU0TXpBeFlpSXNJblI1Y0dVaU9pSk1hVzVsSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1OaGJHeGlZV05ySWpwdWRXeHNMQ0pqYjJ4MWJXNWZibUZ0WlhNaU9sc2llQ0lzSW5raVhTd2laR0YwWVNJNmV5SjRJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lRVUZFUVdzcmVteGtWVWxCUVV0blF6aFBWakZSWjBGQmEwaEllalZZVmtOQlFVTkJLMVEzYldSVlNVRkJSMmh2VVhWYU1WRm5RVUZWVG1SR05XNVdRMEZCUWtGWU5VaHRaRlZKUVVGRGFrOXNUMW94VVdkQlFVVkVNbGsxYmxaRFFVRkJRWGhsVUcxa1ZVbEJRVTluZWpVcldqRlJaMEZCTUV0TWNUVnVWa05CUVVSQlMycGlibVJWU1VGQlMybGFUMlZrTVZGblFVRnJRV2M1TlROV1EwRkJRMEZyU1dwdVpGVkpRVUZIYWk5cEsyUXhVV2RCUVZWSE5sQTFNMVpEUVVGQ1FUbDBjbTVrVlVsQlFVTm9iRE4xWkRGUlowRkJSVTVVYURVelZrTkJRVUZCV0VNemIyUlZTVUZCVDJwTFRVOW9NVkZuUVVFd1JHc3dOa2hXUTBGQlJFRjNXQzl2WkZWSlFVRkxaM2RuSzJneFVXZEJRV3RLSzBjMlNGWkRJaXdpWkhSNWNHVWlPaUptYkc5aGREWTBJaXdpYzJoaGNHVWlPbHN5TjExOUxDSjVJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lRVUZCUVZGS2FqbE5SVUZCUVVGQ1ozWlFkM2RSUVVGQlFVdEVaeXQ2UWtGQlFVRkJVVkI2YjAxRlFVRkJRVUpCYkdWamQxRkJRVUZCUjBGMU5XcENRVUZCUVVGUlJsaElUVVZCUVVGQlEwRlFZMVYzVVVGQlFVRk5RV3gzZWtKQlFVRkJRVkZDZFZaTlJVRkJRVUZCWjBNMVRYZFJRVUZCUVVGRU4ydEVRa0ZCUVVGQk5FcGthazFGUVVGQlFVUkJZVzFCZDFGQlFVRkJTVUU1V0ZSQ1FVRkJRVUZKUm5kWVRVVkJRVUZCUkdkUFVuTjNVVUZCUVVGTlFWaElla0pCUVVGQlFVbERaREJOUlVGQlFVRkVaM0l6WTNkUlFVRkJRVWxCTkdWNlFrRkJRVUZCTkZCbVNVMUZRVUZCUVVSQk1EaG5kMUZCUVVGQlMwTjJlVVJDUVVGQlFVRlpTbFJHVFVWQlFVRkJRbWRzVFZWM1VVRkJRVUZIUTFWNFZFSkJJaXdpWkhSNWNHVWlPaUptYkc5aGREWTBJaXdpYzJoaGNHVWlPbHN5TjExOWZYMHNJbWxrSWpvaU56ZzVNbUppTlRRdFl6ZzBZUzAwWmpsaUxXSXpNamd0WlRObFpqSm1PRFE0Wm1RM0lpd2lkSGx3WlNJNklrTnZiSFZ0YmtSaGRHRlRiM1Z5WTJVaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW1OdmJIVnRibDl1WVcxbGN5STZXeUo0SWl3aWVTSmRMQ0prWVhSaElqcDdJbmdpT25zaVgxOXVaR0Z5Y21GNVgxOGlPaUpCUVVSQmF5dDZiR1JWU1VGQlMyZERPRTlXTVZGblFVRnJTRWg2TlZoV1EwRkJRalEwVUdKc1pGVkpRVUZIUWxBcmRWWXhVV2RCUVZOTU56azFXRlpEUVVGQmQweFJTRzFrVlVsQlFVSnBZMEpQV2pGUlowRkJRVUZ6U1RWdVZrTkJRVVJ2WlZGMmJXUlZTVUZCVGtSdlJIVmFNVkZuUVVGMVJtTlROVzVXUTBGQlEyZDRhRmh0WkZWSlFVRkpaekZIWlZveFVXZEJRV05MVVdNMWJsWkRRVUZDV1VWNVJHMWtWVWxCUVVWRFEwa3JXakZSWjBGQlMxQkZiVFZ1VmtOQlFVRlJXVU55YldSVlNVRkJVR3BQVEdWYU1WRm5RVUUwUkRCNE5XNVdRMEZCUkVseVJGUnRaRlZKUVVGTVFXSlBUMW94VVdkQlFXMUpiemMxYmxaRFFVRkRRU3RVTjIxa1ZVbEJRVWRvYjFGMVdqRlJaMEZCVlU1a1JqVnVWa05CUVVFMFVtdHViV1JWU1VGQlEwTXhWRTlhTVZGblFVRkRRMUpSTlc1V1EwRkJSSGRyYkZCdFpGVkpRVUZPWjBKV0sxb3hVV2RCUVhkSVFtRTFibFpEUVVGRGJ6TXhNMjFrVlVsQlFVcENUMWxsV2pGUlowRkJaVXd4YXpWdVZrTkJRVUpuVEVkcWJXUlZTVUZCUldsaVlTdGFNVkZuUVVGTlFYQjJOVzVXUTBGQlFWbGxXRXh0WkZWSlFVRkJSRzlrWlZveFVXZEJRVFpHV2pVMWJsWkRRVUZFVVhoWWVtMWtWVWxCUVV4bk1HZFBXakZSWjBGQmIwdFBSRFZ1VmtOQlFVTkpSVzltYldSVlNVRkJTRU5DYVhWYU1WRm5RVUZYVUVOT05XNVdRMEZCUWtGWU5VaHRaRlZKUVVGRGFrOXNUMW94VVdkQlFVVkVNbGsxYmxaRFFVRkVOSEUxZG0xa1ZVbEJRVTlCWVc0cldqRlJaMEZCZVVsdGFUVnVWa05CUVVOM0swdFliV1JWU1VGQlNtaHVjV1ZhTVZGblFVRm5UbUZ6Tlc1V1EwRkJRbTlTWWtSdFpGVkpRVUZHUXpCeksxb3hVV2RCUVU5RFR6TTFibFpEUVVGQloydHljbTFrVlVsQlFVRm5RbloxV2pGUlowRkJPRWN2UWpWdVZrTkJRVVJaTTNOVWJXUlZTVUZCVFVKT2VVOWFNVkZuUVVGeFRIcE1OVzVXUTBGQlExRkxPQzl0WkZWSlFVRklhV0V3ZFZveFVXZEJRVmxCYmxjMWJsWkRRVUZDU1dWT2JtMWtWVWxCUVVSRWJqTlBXakZSWjBGQlIwWmlaelZ1VmtOQlFVRkJlR1ZRYldSVlNVRkJUMmQ2TlN0YU1WRm5RVUV3UzB4eE5XNVdRMEZCUXpSRlpUZHRaRlZKUVVGTFEwRTRaVm94VVdkQlFXbFBMekExYmxaRFFVRkNkMWgyYW0xa1ZVbEJRVVpxVGlzcldqRlJaMEZCVVVSNkx6VnVWa05CUVVGdmNYZE1ibVJWU1VGQlFrRmhRblZrTVZGblFVRXJTV2RLTlROV1EwRkJSR2M1ZDNwdVpGVkpRVUZOYUcxRlQyUXhVV2RCUVhOT1ZWUTFNMVpEUVVGRFdWSkNabTVrVlVsQlFVbERla2QxWkRGUlowRkJZVU5KWlRVelZrTkJRVUpSYTFOSWJtUlZTVUZCUkdkQlNtVmtNVkZuUVVGSlJ6aHZOVE5XUTBGQlFVa3phWFp1WkZWSlFVRlFRazFNSzJReFVXZEJRVEpNYzNrMU0xWkRRVUZFUVV0cVltNWtWVWxCUVV0cFdrOWxaREZSWjBGQmEwRm5PVFV6VmtOQlFVSTBaREJFYm1SVlNVRkJSMFJ0VVN0a01WRm5RVUZUUmxaSU5UTldRMEZCUVhkNFJYSnVaRlZKUVVGQ1ozcFVkV1F4VVdkQlFVRkxTbEkxTTFaRFFVRkViMFZHV0c1a1ZVbEJRVTVDTDFkUFpERlJaMEZCZFU4MVlqVXpWa05CUVVObldGWXZibVJWU1VGQlNXcE5XWFZrTVZGblFVRmpSSFJ0TlROV1EwRkJRbGx4Ylc1dVpGVkpRVUZGUVZwaVpXUXhVV2RCUVV0SmFIYzFNMVpEUVVGQlVUa3pVRzVrVlVsQlFWQm9iR1FyWkRGUlowRkJORTVTTmpVelZrTkJRVVJKVVRNM2JtUlZTVUZCVEVONVoyVmtNVkZuUVVGdFEwZEdOVE5XUTBGQlEwRnJTV3B1WkZWSlFVRkhhaTlwSzJReFVXZEJRVlZITmxBMU0xWkRRVUZCTkROYVRHNWtWVWxCUVVOQ1RXeDFaREZSWjBGQlEweDFXalV6VmtOQlFVUjNTMW96Ym1SVlNVRkJUbWxaYjA5a01WRm5RVUYzUVdWck5UTldRMEZCUTI5a2NXWnVaRlZKUVVGS1JHeHhkV1F4VVdkQlFXVkdVM1UxTTFaRFFVRkNaM2MzU0c1a1ZVbEJRVVZuZVhSbFpERlJaMEZCVFV0SE5EVXpWa05CUVVGWlJVeDZibVJWU1VGQlFVSXZkaXRrTVZGblFVRTJUek5ETlROV1EwRkJSRkZZVFdKdVpGVkpRVUZNYWt4NVpXUXhVV2RCUVc5RWNrNDFNMVpEUVVGRFNYRmtSRzVrVlVsQlFVaEJXVEZQWkRGUlowRkJWMGxtV0RVelZrTWlMQ0prZEhsd1pTSTZJbVpzYjJGME5qUWlMQ0p6YUdGd1pTSTZXekUwTkYxOUxDSjVJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lXbTFhYlZwdFltMU5WVUp0V20xYWJWcDFXWGhSUjFwdFdtMWFiVFZxUmtGYWJWcHRXbTFpYlUxVlFtMWFiVnB0V25WWmVGRk5NMDE2VFhwTmVrUkdRWHBqZWsxNlRYcE5UVlZCZWsxNlRYcE5OMDE0VVVweFdtMWFiVnB0VkVaQmJYQnRXbTFhYlZwTlZVRkJRVUZCUVVGSlFYaFJRVUZCUVVGQlFXZEVSa0ZhYlZwdFdtMWFiVTFWUW0xYWJWcHRXbTFaZUZGQlFVRkJRVUZCWjBSR1FVRkJRVUZCUVVOQlRWVkJlazE2VFhwTk4wMTRVVUZCUVVGQlFVRkJSRXBCUVVGQlFVRkJRMEZOYTBGQlFVRkJRVUZKUVhsUlJFMTZUWHBOZWsxNlNrRnRjRzFhYlZwcldrMXJRVUZCUVVGQlFVRkJlVkZLY1ZwdFdtMWFSMVJLUVcxd2JWcHRXbXRhVFd0RFlXMWFiVnB0VW10NVVVZGFiVnB0V20wMWFrWkJlbU42VFhwTmVrMU5WVVJPZWsxNlRYcE5kM2hSUkUxNlRYcE5lbk42UmtGdGNHMWFiVnB0V2sxVlFVRkJRVUZCUVVsQmVGRkJRVUZCUVVGQlowUkdRVUZCUVVGQlFVTkJUVlZCUVVGQlFVRkJTVUY0VVVkYWJWcHRXbTFhYWtaQldtMWFiVnB0V20xTlZVUk9lazE2VFhwRmQzaFJSMXB0V20xYWJWcHFSa0ZCUVVGQlFVRkRRVTFWUVVGQlFVRkJRVWxCZUZGS2NWcHRXbTFhYlZSR1FYcGplazE2VFhwTlRWVkVUbnBOZWsxNlRYZDRVVWRhYlZwdFdtMDFha1pCUVVGQlFVRkJRVUZOYTBOaGJWcHRXbTFTYTNsUlFVRkJRVUZCUVVGRVNrRjZZM3BOZWsxNlRVMVZRMkZ0V20xYWJWcHJlRkZCUVVGQlFVRkJaMFJHUVZwdFdtMWFiVnB0VFZWQlFVRkJRVUZCU1VGNFVVRkJRVUZCUVVGblJFWkJXbTFhYlZwdFdtMU5WVVJPZWsxNlRYcEZkM2hSU25GYWJWcHRXa2RVUmtGdGNHMWFiVnByV2sxVlFVRkJRVUZCUVVGQmVGRkhXbTFhYlZwdE5XcENRWHBqZWsxNlRYcE5UVVZFVG5wTmVrMTZUWGQzVVUwelRYcE5lazE2UkVKQldtMWFiVnB0WW0xTlJVRkJRVUZCUVVGQlFYaFJUVE5OZWsxNlRWUkVSa0ZhYlZwdFdtMWFiVTFWUVVGQlFVRkJRVWxCZUZGQlFVRkJRVUZCWjBSR1FVRkJRVUZCUVVOQlRWVkJRVUZCUVVGQlNVRjRVVTB6VFhwTmVrMVVSRVpCZW1ONlRYcE5lRTFOVlVKdFdtMWFiVnB0V1hoUlRUTk5lazE2VFZSRVJrRjZZM3BOZWsxNFRVMVZSRTU2VFhwTmVrVjNlRkZFVFhwTmVrMTZUWHBHUVUxNlRYcE5lazE2VFZWRFlXMWFiVnB0VW10NFVVRkJRVUZCUVVGQlJFWkJXbTFhYlZwdFltMU5SVUY2VFhwTmVrMDNUWGRSU25GYWJWcHRXbTFVUWtGdGNHMWFiVnB0V2sxRlFVRkJRVUZCUVVsQmQxRktjVnB0V20xYWJWUkNRWHBqZWsxNlRYcE5UVVZEWVcxYWJWcHRVbXQ0VVVweFdtMWFiVnBIVkVaQldtMWFiVnB0V20xTlZVRjZUWHBOZWswM1RYaFJTbkZhYlZwdFdrZFVTa0Z0Y0cxYWJWcHJXazFyUTJGdFdtMWFiVkpyZVZGRVRYcE5lazE2YzNwR1FVMTZUWHBOZWsxNlRWVkRZVzFhYlZwdFVtdDRVVUZCUVVGQlFVRkJSRVpCV20xYWJWcHRZbTFOUlVST2VrMTZUWHBOZDNkUlRUTk5lazE2VFhwRVFrRk5lazE2VFhwUGVrMUZRWHBOZWsxNlRUZE5kMUZLY1ZwdFdtMWFiVlJDUVcxd2JWcHRXbTFhVFVWRFlXMWFiVnB0V210M1VVcHhXbTFhYlZwdFZFSkJXbTFhYlZwdFdtMU5SVUp0V20xYWJWcHRXWGRSUjFwdFdtMWFiVnBxUWtGdGNHMWFiVnB0V2sxRlJFNTZUWHBOZWsxM2QxRkVUWHBOZWsxNlRYcEdRVUZCUVVGQlFVTkJUV3RCUVVGQlFVRkJTVUY1VVVGQlFVRkJRVUZuUkVwQldtMWFiVnB0WW0xTlZVUk9lazE2VFhwRmQzbFJUVE5OZWsxNlRYcEVSa0Y2WTNwTmVrMTRUVTFWUkU1NlRYcE5la1YzZUZGS2NWcHRXbTFhUjFSR1FYcGplazE2VFhoTlRWVkJlazE2VFhwTmVrMTRVVUZCUVVGQlFVRkJSRVpCV20xYWJWcHRZbTFOUlVST2VrMTZUWHBOZDNkUlJFMTZUWHBOZW5ONlFrRnRjRzFhYlZwdFdrMUZRMkZ0V20xYWJWcHJkMUZCUVVGQlFVRkJaMFJDUVVGQlFVRkJRVU5CVFVWRFlXMWFiVnB0V210M1VVUk5lazE2VFhwemVrSkJiWEJ0V20xYWExcE5WVUZCUVVGQlFVRkpRWGhSVFROTmVrMTZUWHBFUmtGNlkzcE5lazE2VFUxclJFNTZUWHBOZWtWM2VWRk5NMDE2VFhwTmVrUkdRWHBqZWsxNlRYcE5UVlZCUVVGQlFVRkJTVUY0VVVGQlFVRkJRVUZCUkVaQklpd2laSFI1Y0dVaU9pSm1iRzloZERZMElpd2ljMmhoY0dVaU9sc3hORFJkZlgxOUxDSnBaQ0k2SWpkbE56Z3lZamMzTFdWak56TXRORFl4TkMwNU5Ea3pMV1F5WVRJMU5qVm1aV0pqTVNJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW0xdmJuUm9jeUk2V3pBc01pdzBMRFlzT0N3eE1GMTlMQ0pwWkNJNklqUXdZak5sTjJRM0xURXlZelV0Tkdaa1lTMWlOalExTFdObFpHWm1NakEyT0dSbE5DSXNJblI1Y0dVaU9pSk5iMjUwYUhOVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYVhSbGJYTWlPbHQ3SW1sa0lqb2lPVGM1WW1NeE5EUXRaakpqTXkwME9URTVMVGt4Tm1JdE5EYzVOakE1TUdSa05UZ3hJaXdpZEhsd1pTSTZJa3hsWjJWdVpFbDBaVzBpZlN4N0ltbGtJam9pWVdNeFpqQmlaVGt0Tm1KaU9TMDBaV1EwTFRnek1UQXRZMlE1TTJRNU9XRmhZbVZoSWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbWxrSWpvaU5qWTJPV1JqTXpJdFl6Y3hOUzAwWlRjMExUZ3lOVFF0TkRRek5XTmhZV1pqWVRVM0lpd2lkSGx3WlNJNklreGxaMlZ1WkVsMFpXMGlmU3g3SW1sa0lqb2lNR1l4WlRZMU9EVXRPR0l3WmkwME1qTTBMVGxrWWpndFlUYzVOR014T0RJNU9USXpJaXdpZEhsd1pTSTZJa3hsWjJWdVpFbDBaVzBpZlN4N0ltbGtJam9pTldRNVlXRTNNMkV0TVdKbE9DMDBNbUUwTFdFMk1ESXRPV05qTkdVNE1qVXhNVE5sSWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbWxrSWpvaVptTXpZamt3TVRFdE5UbGlNQzAwTnpNM0xUZzNORE10WW1Jek5tSTJNMkV5WTJSaElpd2lkSGx3WlNJNklreGxaMlZ1WkVsMFpXMGlmU3g3SW1sa0lqb2lZV0ppTTJJNU1EQXRaR05qWmkwME5UVXpMV0ZqWlRjdE56SXlNekV3WWpJNE5qY3hJaXdpZEhsd1pTSTZJa3hsWjJWdVpFbDBaVzBpZlYwc0luQnNiM1FpT25zaWFXUWlPaUkwTWpBNVpURXdOaTFpTldaa0xUUmtaRE10T0RneU1TMHlNMlZoWm1VNE1EWTROamdpTENKemRXSjBlWEJsSWpvaVJtbG5kWEpsSWl3aWRIbHdaU0k2SWxCc2IzUWlmWDBzSW1sa0lqb2lOalkwWXpRMU16WXRNRFE1TlMwMFlqSTBMVGc0TXprdE1qQTROemxrTVRoa1pqSmhJaXdpZEhsd1pTSTZJa3hsWjJWdVpDSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmpZV3hzWW1GamF5STZiblZzYkN3aWNHeHZkQ0k2ZXlKcFpDSTZJalF5TURsbE1UQTJMV0kxWm1RdE5HUmtNeTA0T0RJeExUSXpaV0ZtWlRnd05qZzJPQ0lzSW5OMVluUjVjR1VpT2lKR2FXZDFjbVVpTENKMGVYQmxJam9pVUd4dmRDSjlMQ0p5Wlc1a1pYSmxjbk1pT2x0N0ltbGtJam9pTm1JeFpUazJZbVF0TkRrMk55MDBZVE0yTFdKak4yRXRaV1l5TUdVd05EazFaamN5SWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmVjBzSW5SdmIyeDBhWEJ6SWpwYld5Sk9ZVzFsSWl3aVRrVkRUMFpUWDBkUFRUTWlYU3hiSWtKcFlYTWlMQ0l0TUM0ek1TSmRMRnNpVTJ0cGJHd2lMQ0l3TGpNMUlsMWRmU3dpYVdRaU9pSTNZemszT1dRd1pDMDJNamd6TFRRd01EZ3RPVFF6TUMxbVlqazBOams1TW1GaFlUWWlMQ0owZVhCbElqb2lTRzkyWlhKVWIyOXNJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbU5oYkd4aVlXTnJJanB1ZFd4c2ZTd2lhV1FpT2lKak5USmlOREUxTnkxaU5XTmhMVFExT1RZdFlUQTJZeTB6T1RFeVptWTRaRFppTm1ZaUxDSjBlWEJsSWpvaVJHRjBZVkpoYm1kbE1XUWlmU3g3SW1GMGRISnBZblYwWlhNaU9udDlMQ0pwWkNJNklqazFPRE5sTjJKaUxUZGhabVV0TkRKaU1DMDRNbVV4TFRaaU56Tm1OV0kyWldVMVppSXNJblI1Y0dVaU9pSkNZWE5wWTFScFkydEdiM0p0WVhSMFpYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2lZMkZzYkdKaFkyc2lPbTUxYkd3c0luQnNiM1FpT25zaWFXUWlPaUkwTWpBNVpURXdOaTFpTldaa0xUUmtaRE10T0RneU1TMHlNMlZoWm1VNE1EWTROamdpTENKemRXSjBlWEJsSWpvaVJtbG5kWEpsSWl3aWRIbHdaU0k2SWxCc2IzUWlmU3dpY21WdVpHVnlaWEp6SWpwYmV5SnBaQ0k2SW1ZM016SmhNbUl6TFdZMFl6UXROREkwWWkxaE5HRXdMVGRpWmpreU5tVTNZelE0TmlJc0luUjVjR1VpT2lKSGJIbHdhRkpsYm1SbGNtVnlJbjFkTENKMGIyOXNkR2x3Y3lJNlcxc2lUbUZ0WlNJc0lrNUZRMDlHVTE5TllYTnpRbUY1SWwwc1d5SkNhV0Z6SWl3aUxUQXVORElpWFN4YklsTnJhV3hzSWl3aU1DNDBOaUpkWFgwc0ltbGtJam9pT1dGaE5ESXhZVGd0TURoak1DMDBZbUppTFRneVptUXRabVF6T1RsaFpEYzBNR1prSWl3aWRIbHdaU0k2SWtodmRtVnlWRzl2YkNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKallXeHNZbUZqYXlJNmJuVnNiQ3dpWTI5c2RXMXVYMjVoYldWeklqcGJJbmdpTENKNUlsMHNJbVJoZEdFaU9uc2llQ0k2ZXlKZlgyNWtZWEp5WVhsZlh5STZJa0ZCUkVGckszcHNaRlZKUVVGTFowTTRUMVl4VVdkQlFXdElTSG8xV0ZaRFFVRkRRU3RVTjIxa1ZVbEJRVWRvYjFGMVdqRlJaMEZCVlU1a1JqVnVWa05CUVVKQldEVkliV1JWU1VGQlEycFBiRTlhTVZGblFVRkZSREpaTlc1V1EwRkJRVUY0WlZCdFpGVkpRVUZQWjNvMUsxb3hVV2RCUVRCTFRIRTFibFpESWl3aVpIUjVjR1VpT2lKbWJHOWhkRFkwSWl3aWMyaGhjR1VpT2xzeE1sMTlMQ0o1SWpwN0lsOWZibVJoY25KaGVWOWZJam9pUVVGQlFXOVBSelpOVlVGQlFVRkJRVWczVlhoUlFVRkJRVWxDWTNKNlJrRkJRVUZCYjB0TmQwMVZRVUZCUVVGQlRYbHplRkZCUVVGQlJVUkRTbFJHUVVGQlFVRnZRazkxVFVWQlFVRkJRbWREWVRoM1VVRkJRVUZEUkM5eWVrSkJRVUZCUVc5Q00wWk5SVUZCUVVGRFowaGpWWGRSUVVGQlFVdEJaSGhVUWtFaUxDSmtkSGx3WlNJNkltWnNiMkYwTmpRaUxDSnphR0Z3WlNJNld6RXlYWDE5ZlN3aWFXUWlPaUpsT0daak1qUmhNUzB4TVRRd0xUUXhOV1F0T1dKbVl5MDVZalZsTXprMFpqbGlObVVpTENKMGVYQmxJam9pUTI5c2RXMXVSR0YwWVZOdmRYSmpaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZlMzBzSW1sa0lqb2lPREpsT0ROaE16WXRNemhoWmkwME5qQmpMV0V3TVRJdFpXWTNObUV5TnpNMVptWTJJaXdpZEhsd1pTSTZJbGxsWVhKelZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW5Cc2IzUWlPbnNpYVdRaU9pSTBNakE1WlRFd05pMWlOV1prTFRSa1pETXRPRGd5TVMweU0yVmhabVU0TURZNE5qZ2lMQ0p6ZFdKMGVYQmxJam9pUm1sbmRYSmxJaXdpZEhsd1pTSTZJbEJzYjNRaWZYMHNJbWxrSWpvaU5tTXpaRFZrT0RNdE5USTJOUzAwT0RSaUxUaGpZbVF0WkRRM09ERTJNRGszTlRVeElpd2lkSGx3WlNJNklsSmxjMlYwVkc5dmJDSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnRiMjUwYUhNaU9sc3dMRFpkZlN3aWFXUWlPaUl3TXpFd05USTFNaTFoTldVNExUUmtPRGd0T0RBeU1TMHpOVFE0T0Roall6TTFOek1pTENKMGVYQmxJam9pVFc5dWRHaHpWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN2ZTd2lhV1FpT2lKa1pqTmxaR0prWWkxaE9EQm1MVFJtWVRJdE9Ea3dNUzFrTWpNMVpqRTJPV1kyWVRNaUxDSjBlWEJsSWpvaVJHRjBaWFJwYldWVWFXTnJSbTl5YldGMGRHVnlJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbU5oYkd4aVlXTnJJanB1ZFd4c0xDSndiRzkwSWpwN0ltbGtJam9pTkRJd09XVXhNRFl0WWpWbVpDMDBaR1F6TFRnNE1qRXRNak5sWVdabE9EQTJPRFk0SWl3aWMzVmlkSGx3WlNJNklrWnBaM1Z5WlNJc0luUjVjR1VpT2lKUWJHOTBJbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lKbFpUVTVaR1l4TnkwMFpXSTNMVFJqT0RrdFlUUXpaQzFrTlRjME5XVTVZVGd5T1RJaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFN3aWRHOXZiSFJwY0hNaU9sdGJJazVoYldVaUxDSlRSVU5QVDFKQlgwNURVMVZmUTA1QlVGTWlYU3hiSWtKcFlYTWlMQ0l3TGpVeUlsMHNXeUpUYTJsc2JDSXNJakF1TXpraVhWMTlMQ0pwWkNJNkltWmxZakEzTW1GakxUQTNNakl0TkRrMVl5MDRaalE1TFRNek1qQXhZV0prTkdNd055SXNJblI1Y0dVaU9pSkliM1psY2xSdmIyd2lmU3g3SW1GMGRISnBZblYwWlhNaU9uc2laR0YwWVY5emIzVnlZMlVpT25zaWFXUWlPaUpsT0daak1qUmhNUzB4TVRRd0xUUXhOV1F0T1dKbVl5MDVZalZsTXprMFpqbGlObVVpTENKMGVYQmxJam9pUTI5c2RXMXVSR0YwWVZOdmRYSmpaU0o5TENKbmJIbHdhQ0k2ZXlKcFpDSTZJbVpqWmpBMk5qYzRMVGRsTlRRdE5ESXhPUzA1TTJZeExUSmtObUl3WlRGaE1Ea3laaUlzSW5SNWNHVWlPaUpNYVc1bEluMHNJbWh2ZG1WeVgyZHNlWEJvSWpwdWRXeHNMQ0p0ZFhSbFpGOW5iSGx3YUNJNmJuVnNiQ3dpYm05dWMyVnNaV04wYVc5dVgyZHNlWEJvSWpwN0ltbGtJam9pWW1RNFpUazNZV010TldOaFpDMDBabUkwTFRneU1XVXRNRFl3Wmpkak1UZ3pNREZpSWl3aWRIbHdaU0k2SWt4cGJtVWlmU3dpYzJWc1pXTjBhVzl1WDJkc2VYQm9JanB1ZFd4c2ZTd2lhV1FpT2lKaE56aGhPR00zWXkxaE1ESTNMVFF4TkdVdE9ESTVZaTFtT1dGa1kyVTBZekkyTVRJaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5TEhzaVlYUjBjbWxpZFhSbGN5STZleUp0YjI1MGFITWlPbHN3TERRc09GMTlMQ0pwWkNJNkltSmhPRFZpTXpVM0xUWmtOV1F0TkdFMk5DMDROV0kyTFRaaE5UVTVNV1kxWVRNeE15SXNJblI1Y0dVaU9pSk5iMjUwYUhOVWFXTnJaWElpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYkdGaVpXd2lPbnNpZG1Gc2RXVWlPaUpITVY5VFUxUmZSMHhQUWtGTUluMHNJbkpsYm1SbGNtVnljeUk2VzNzaWFXUWlPaUpoTnpoaE9HTTNZeTFoTURJM0xUUXhOR1V0T0RJNVlpMW1PV0ZrWTJVMFl6STJNVElpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlYWDBzSW1sa0lqb2lPVGM1WW1NeE5EUXRaakpqTXkwME9URTVMVGt4Tm1JdE5EYzVOakE1TUdSa05UZ3hJaXdpZEhsd1pTSTZJa3hsWjJWdVpFbDBaVzBpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWTJGc2JHSmhZMnNpT201MWJHd3NJbU52YkhWdGJsOXVZVzFsY3lJNld5SjRJaXdpZVNKZExDSmtZWFJoSWpwN0luZ2lPbnNpWDE5dVpHRnljbUY1WDE4aU9pSkJRVU5uZUdoWWJXUlZTVUZCU1djeFIyVmFNVkZuUVVGalMxRmpOVzVXUTBGQlFtZE1SMnB0WkZWSlFVRkZhV0poSzFveFVXZEJRVTFCY0hZMWJsWkRRVUZCWjJ0eWNtMWtWVWxCUVVGblFuWjFXakZSWjBGQk9FY3ZRalZ1VmtOQlFVUm5PWGQ2Ym1SVlNVRkJUV2h0UlU5a01WRm5RVUZ6VGxWVU5UTldReUlzSW1SMGVYQmxJam9pWm14dllYUTJOQ0lzSW5Ob1lYQmxJanBiTVRKZGZTd2llU0k2ZXlKZlgyNWtZWEp5WVhsZlh5STZJa0ZCUVVGM1ExbGxUVlZCUVVGQlJHZFRhRFI0VVVGQlFVRkJRblpJYWtaQlFVRkJRVkZKYTJoTlZVRkJRVUZCWnl0U01IaFJRVUZCUVVGQ2NFZHFSa0ZCUVVGQk5FRllUVTFGUVVGQlFVRkJXazFuZDFGQlFVRkJRMFJEZUVSQ1FVRkJRVUYzVG1nd1RVVkJRVUZCUkVFeVNGRjNVVUZCUVVGTlJGbGtSRUpCSWl3aVpIUjVjR1VpT2lKbWJHOWhkRFkwSWl3aWMyaGhjR1VpT2xzeE1sMTlmWDBzSW1sa0lqb2lNV1V6WWpnMk56VXRNMkZqTUMwME5EQmtMVGxsTVdRdE5UUTRabVk1WVdVMk5qWXpJaXdpZEhsd1pTSTZJa052YkhWdGJrUmhkR0ZUYjNWeVkyVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2liR2x1WlY5aGJIQm9ZU0k2ZXlKMllXeDFaU0k2TUM0Mk5YMHNJbXhwYm1WZlkyRndJam9pY205MWJtUWlMQ0pzYVc1bFgyTnZiRzl5SWpwN0luWmhiSFZsSWpvaUl6azRaR1k0WVNKOUxDSnNhVzVsWDJwdmFXNGlPaUp5YjNWdVpDSXNJbXhwYm1WZmQybGtkR2dpT25zaWRtRnNkV1VpT2pWOUxDSjRJanA3SW1acFpXeGtJam9pZUNKOUxDSjVJanA3SW1acFpXeGtJam9pZVNKOWZTd2lhV1FpT2lKalpEZGhOalEzTUMwMk1qRXdMVFF4WWpndE9UTTFZUzB4WldRNFpHTmhaREEzTW1NaUxDSjBlWEJsSWpvaVRHbHVaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpoWTNScGRtVmZaSEpoWnlJNkltRjFkRzhpTENKaFkzUnBkbVZmYVc1emNHVmpkQ0k2SW1GMWRHOGlMQ0poWTNScGRtVmZjMk55YjJ4c0lqb2lZWFYwYnlJc0ltRmpkR2wyWlY5MFlYQWlPaUpoZFhSdklpd2lkRzl2YkhNaU9sdDdJbWxrSWpvaU9EWTRNREJqT0dNdE5HWmpOUzAwTjJOa0xXSXdORGd0TXpFNU0yWXpORGcxWXpRNElpd2lkSGx3WlNJNklsQmhibFJ2YjJ3aWZTeDdJbWxrSWpvaVpERTVPRFl5TUdNdE9HUXhZUzAwWVRReUxUaGlaR1l0TkRrNVlqUmlNak14TkRnMklpd2lkSGx3WlNJNklrSnZlRnB2YjIxVWIyOXNJbjBzZXlKcFpDSTZJalpqTTJRMVpEZ3pMVFV5TmpVdE5EZzBZaTA0WTJKa0xXUTBOemd4TmpBNU56VTFNU0lzSW5SNWNHVWlPaUpTWlhObGRGUnZiMndpZlN4N0ltbGtJam9pT0RrMFlqWTBPVE10TTJWbE5DMDBNamhpTFdGbFlUTXRNRFF4WkRRM1pqSXhZekkzSWl3aWRIbHdaU0k2SWtodmRtVnlWRzl2YkNKOUxIc2lhV1FpT2lJNVlXRTBNakZoT0Mwd09HTXdMVFJpWW1JdE9ESm1aQzFtWkRNNU9XRmtOelF3Wm1RaUxDSjBlWEJsSWpvaVNHOTJaWEpVYjI5c0luMHNleUpwWkNJNklqZGpPVGM1WkRCa0xUWXlPRE10TkRBd09DMDVORE13TFdaaU9UUTJPVGt5WVdGaE5pSXNJblI1Y0dVaU9pSkliM1psY2xSdmIyd2lmU3g3SW1sa0lqb2lNalUxWVdNNVpUY3RPRFV3TmkwME1ESXpMV0l4T1RndE1EUm1ObVUzT0dZNE1qTXhJaXdpZEhsd1pTSTZJa2h2ZG1WeVZHOXZiQ0o5TEhzaWFXUWlPaUptWldJd056SmhZeTB3TnpJeUxUUTVOV010T0dZME9TMHpNekl3TVdGaVpEUmpNRGNpTENKMGVYQmxJam9pU0c5MlpYSlViMjlzSW4wc2V5SnBaQ0k2SW1WbFpqTmxNbUkyTFdNeFpURXROREU1TnkwNU9URXhMVE01TW1aaFpEZGtObUUxTmlJc0luUjVjR1VpT2lKSWIzWmxjbFJ2YjJ3aWZTeDdJbWxrSWpvaU0yRTFPVFptTkdNdFpUbGhZaTAwTW1Oa0xUa3hOemN0WkdKbU5ERmxNVEpqTnpobElpd2lkSGx3WlNJNklraHZkbVZ5Vkc5dmJDSjlYWDBzSW1sa0lqb2lOalE1TkdVeFlqRXRObVZtT0MwME1EZGlMVGc1T1RRdE5EQmtZVGhtWVRrM1pqTTJJaXdpZEhsd1pTSTZJbFJ2YjJ4aVlYSWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2laR2x0Wlc1emFXOXVJam94TENKd2JHOTBJanA3SW1sa0lqb2lOREl3T1dVeE1EWXRZalZtWkMwMFpHUXpMVGc0TWpFdE1qTmxZV1psT0RBMk9EWTRJaXdpYzNWaWRIbHdaU0k2SWtacFozVnlaU0lzSW5SNWNHVWlPaUpRYkc5MEluMHNJblJwWTJ0bGNpSTZleUpwWkNJNklqRmtObVZsTXpkakxXTTNaalF0TkdFNFl5MDRNamd5TFRFNE5tRXdOVFExT1RFd09TSXNJblI1Y0dVaU9pSkNZWE5wWTFScFkydGxjaUo5ZlN3aWFXUWlPaUpoWVRKalkyRmlZUzFqWVRaaExUUmlNak10T1RCak15MHdPRFUxWlRRMFpXVmhZMllpTENKMGVYQmxJam9pUjNKcFpDSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNZV0psYkNJNmV5SjJZV3gxWlNJNklrNUZRMDlHVTE5SFQwMHpJbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lJMllqRmxPVFppWkMwME9UWTNMVFJoTXpZdFltTTNZUzFsWmpJd1pUQTBPVFZtTnpJaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFgwc0ltbGtJam9pTmpZMk9XUmpNekl0WXpjeE5TMDBaVGMwTFRneU5UUXRORFF6TldOaFlXWmpZVFUzSWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbUYwZEhKcFluVjBaWE1pT250OUxDSnBaQ0k2SWpJMU9Ua3lOek0xTFRnMk1qTXRORFUxTUMwNU9USXhMVFJrWWpaaFpXVmxPRE5qWkNJc0luUjVjR1VpT2lKVWIyOXNSWFpsYm5SekluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteGhZbVZzSWpwN0luWmhiSFZsSWpvaVUwVkRUMDlTUVY5T1ExTlZYME5PUVZCVEluMHNJbkpsYm1SbGNtVnljeUk2VzNzaWFXUWlPaUpsWlRVNVpHWXhOeTAwWldJM0xUUmpPRGt0WVRRelpDMWtOVGMwTldVNVlUZ3lPVElpTENKMGVYQmxJam9pUjJ4NWNHaFNaVzVrWlhKbGNpSjlYWDBzSW1sa0lqb2lOV1E1WVdFM00yRXRNV0psT0MwME1tRTBMV0UyTURJdE9XTmpOR1U0TWpVeE1UTmxJaXdpZEhsd1pTSTZJa3hsWjJWdVpFbDBaVzBpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpWW1GelpTSTZNalFzSW0xaGJuUnBjM05oY3lJNld6RXNNaXcwTERZc09Dd3hNbDBzSW0xaGVGOXBiblJsY25aaGJDSTZORE15TURBd01EQXVNQ3dpYldsdVgybHVkR1Z5ZG1Gc0lqb3pOakF3TURBd0xqQXNJbTUxYlY5dGFXNXZjbDkwYVdOcmN5STZNSDBzSW1sa0lqb2lOemMxTm1Jd1pHRXROR0V6WWkwME4yUXlMVGd3WWpFdE5UQmlaRFkzWXpFelptTmxJaXdpZEhsd1pTSTZJa0ZrWVhCMGFYWmxWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0ltSnZkSFJ2YlY5MWJtbDBjeUk2SW5OamNtVmxiaUlzSW1acGJHeGZZV3h3YUdFaU9uc2lkbUZzZFdVaU9qQXVOWDBzSW1acGJHeGZZMjlzYjNJaU9uc2lkbUZzZFdVaU9pSnNhV2RvZEdkeVpYa2lmU3dpYkdWbWRGOTFibWwwY3lJNkluTmpjbVZsYmlJc0lteGxkbVZzSWpvaWIzWmxjbXhoZVNJc0lteHBibVZmWVd4d2FHRWlPbnNpZG1Gc2RXVWlPakV1TUgwc0lteHBibVZmWTI5c2IzSWlPbnNpZG1Gc2RXVWlPaUppYkdGamF5SjlMQ0pzYVc1bFgyUmhjMmdpT2xzMExEUmRMQ0pzYVc1bFgzZHBaSFJvSWpwN0luWmhiSFZsSWpveWZTd2ljR3h2ZENJNmJuVnNiQ3dpY21WdVpHVnlYMjF2WkdVaU9pSmpjM01pTENKeWFXZG9kRjkxYm1sMGN5STZJbk5qY21WbGJpSXNJblJ2Y0Y5MWJtbDBjeUk2SW5OamNtVmxiaUo5TENKcFpDSTZJams0WVRobVl6azNMV00yT0RndE5ESTRNQzA0TjJGa0xXUmtNV013WWprNE1UWXhOaUlzSW5SNWNHVWlPaUpDYjNoQmJtNXZkR0YwYVc5dUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteGhZbVZzSWpwN0luWmhiSFZsSWpvaVQySnpaWEoyWVhScGIyNXpJbjBzSW5KbGJtUmxjbVZ5Y3lJNlczc2lhV1FpT2lJeFlqRTJOalE1WXkxbU5HVmtMVFF4TlRVdE9UUTROaTFqT1RZd09XRmhOemMyT1RZaUxDSjBlWEJsSWpvaVIyeDVjR2hTWlc1a1pYSmxjaUo5WFgwc0ltbGtJam9pTUdZeFpUWTFPRFV0T0dJd1ppMDBNak0wTFRsa1lqZ3RZVGM1TkdNeE9ESTVPVEl6SWl3aWRIbHdaU0k2SWt4bFoyVnVaRWwwWlcwaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaWJHbHVaVjloYkhCb1lTSTZleUoyWVd4MVpTSTZNQzQyTlgwc0lteHBibVZmWTJGd0lqb2ljbTkxYm1RaUxDSnNhVzVsWDJOdmJHOXlJanA3SW5aaGJIVmxJam9pSTJabU4yWXdaU0o5TENKc2FXNWxYMnB2YVc0aU9pSnliM1Z1WkNJc0lteHBibVZmZDJsa2RHZ2lPbnNpZG1Gc2RXVWlPalY5TENKNElqcDdJbVpwWld4a0lqb2llQ0o5TENKNUlqcDdJbVpwWld4a0lqb2llU0o5ZlN3aWFXUWlPaUk1TnpabU5EQTRZaTFsWldJM0xUUmpZMkV0WWpBME55MWlabU0wWm1FeE56WXpPR0VpTENKMGVYQmxJam9pVEdsdVpTSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNhVzVsWDJGc2NHaGhJanA3SW5aaGJIVmxJam93TGpGOUxDSnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2ZXlKMllXeDFaU0k2SWlNeFpqYzNZalFpZlN3aWJHbHVaVjlxYjJsdUlqb2ljbTkxYm1RaUxDSnNhVzVsWDNkcFpIUm9JanA3SW5aaGJIVmxJam8xZlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pWm1VM05ERmlOalF0TUdJMk5TMDBPRFEzTFRsak5EZ3RPVGRsWldObFlqRTRZMlEySWl3aWRIbHdaU0k2SWt4cGJtVWlmU3g3SW1GMGRISnBZblYwWlhNaU9uc2ljR3h2ZENJNmV5SnBaQ0k2SWpReU1EbGxNVEEyTFdJMVptUXROR1JrTXkwNE9ESXhMVEl6WldGbVpUZ3dOamcyT0NJc0luTjFZblI1Y0dVaU9pSkdhV2QxY21VaUxDSjBlWEJsSWpvaVVHeHZkQ0o5ZlN3aWFXUWlPaUk0Tmpnd01HTTRZeTAwWm1NMUxUUTNZMlF0WWpBME9DMHpNVGt6WmpNME9EVmpORGdpTENKMGVYQmxJam9pVUdGdVZHOXZiQ0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUprWVhSaFgzTnZkWEpqWlNJNmV5SnBaQ0k2SW1JNU0yRXlOMlZrTFdGak56a3ROR013WlMxaE5tSXhMVGMyWkRneVptSXpZVGcxTWlJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc0ltZHNlWEJvSWpwN0ltbGtJam9pT1RjMlpqUXdPR0l0WldWaU55MDBZMk5oTFdJd05EY3RZbVpqTkdaaE1UYzJNemhoSWl3aWRIbHdaU0k2SWt4cGJtVWlmU3dpYUc5MlpYSmZaMng1Y0dnaU9tNTFiR3dzSW0xMWRHVmtYMmRzZVhCb0lqcHVkV3hzTENKdWIyNXpaV3hsWTNScGIyNWZaMng1Y0dnaU9uc2lhV1FpT2lKbVpUYzBNV0kyTkMwd1lqWTFMVFE0TkRjdE9XTTBPQzA1TjJWbFkyVmlNVGhqWkRZaUxDSjBlWEJsSWpvaVRHbHVaU0o5TENKelpXeGxZM1JwYjI1ZloyeDVjR2dpT201MWJHeDlMQ0pwWkNJNklqWmlNV1U1Tm1Ka0xUUTVOamN0TkdFek5pMWlZemRoTFdWbU1qQmxNRFE1TldZM01pSXNJblI1Y0dVaU9pSkhiSGx3YUZKbGJtUmxjbVZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW14cGJtVmZZV3h3YUdFaU9uc2lkbUZzZFdVaU9qQXVNWDBzSW14cGJtVmZZMkZ3SWpvaWNtOTFibVFpTENKc2FXNWxYMk52Ykc5eUlqcDdJblpoYkhWbElqb2lJekZtTnpkaU5DSjlMQ0pzYVc1bFgycHZhVzRpT2lKeWIzVnVaQ0lzSW14cGJtVmZkMmxrZEdnaU9uc2lkbUZzZFdVaU9qVjlMQ0o0SWpwN0ltWnBaV3hrSWpvaWVDSjlMQ0o1SWpwN0ltWnBaV3hrSWpvaWVTSjlmU3dpYVdRaU9pSmlNbVpoTVdJek9TMHlNakE1TFRReE9HTXRZVFV4TUMwek56RTJaakl6WlRKa05UQWlMQ0owZVhCbElqb2lUR2x1WlNKOUxIc2lZWFIwY21saWRYUmxjeUk2ZXlKa1lYbHpJanBiTVN3eUxETXNOQ3cxTERZc055dzRMRGtzTVRBc01URXNNVElzTVRNc01UUXNNVFVzTVRZc01UY3NNVGdzTVRrc01qQXNNakVzTWpJc01qTXNNalFzTWpVc01qWXNNamNzTWpnc01qa3NNekFzTXpGZGZTd2lhV1FpT2lKa1pUQXhOR1kyWVMwellUazVMVFF5TXpVdE9HWmlNUzB5WXpKalpERmhaV1ptWXpRaUxDSjBlWEJsSWpvaVJHRjVjMVJwWTJ0bGNpSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SmpZV3hzWW1GamF5STZiblZzYkN3aWNHeHZkQ0k2ZXlKcFpDSTZJalF5TURsbE1UQTJMV0kxWm1RdE5HUmtNeTA0T0RJeExUSXpaV0ZtWlRnd05qZzJPQ0lzSW5OMVluUjVjR1VpT2lKR2FXZDFjbVVpTENKMGVYQmxJam9pVUd4dmRDSjlMQ0p5Wlc1a1pYSmxjbk1pT2x0N0ltbGtJam9pTldJeVpqQTBaR0V0TWpneU9TMDBPVFZsTFdJd1kySXROMlk1WkRsbFpEa3hPR0ZqSWl3aWRIbHdaU0k2SWtkc2VYQm9VbVZ1WkdWeVpYSWlmVjBzSW5SdmIyeDBhWEJ6SWpwYld5Sk9ZVzFsSWl3aVNGbERUMDBpWFN4YklrSnBZWE1pTENJdE1DNDVNaUpkTEZzaVUydHBiR3dpTENJd0xqRTRJbDFkZlN3aWFXUWlPaUl6WVRVNU5tWTBZeTFsT1dGaUxUUXlZMlF0T1RFM055MWtZbVkwTVdVeE1tTTNPR1VpTENKMGVYQmxJam9pU0c5MlpYSlViMjlzSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1SaGVYTWlPbHN4TERRc055d3hNQ3d4TXl3eE5pd3hPU3d5TWl3eU5Td3lPRjE5TENKcFpDSTZJalJrTTJFeVpqWXdMV1l5WVdFdE5EUTBOQzA1TVdZd0xUTTFaVGs1TnpabE9HSm1OaUlzSW5SNWNHVWlPaUpFWVhselZHbGphMlZ5SW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW1SaGRHRmZjMjkxY21ObElqcDdJbWxrSWpvaU1UZ3dNamRoTkRJdFptWXdOaTAwT0RVMUxXRTFabVF0TVRNeFptRmhZalExWkRKaElpd2lkSGx3WlNJNklrTnZiSFZ0YmtSaGRHRlRiM1Z5WTJVaWZTd2laMng1Y0dnaU9uc2lhV1FpT2lJd01XSXhPRFpoTnkwMk1UazJMVFExWXpVdE9XUTRPUzB6T1dWa1ltTTVNelZoTVdNaUxDSjBlWEJsSWpvaVRHbHVaU0o5TENKb2IzWmxjbDluYkhsd2FDSTZiblZzYkN3aWJYVjBaV1JmWjJ4NWNHZ2lPbTUxYkd3c0ltNXZibk5sYkdWamRHbHZibDluYkhsd2FDSTZleUpwWkNJNkltSTVaR000TW1JMkxUZGpZVFF0TkdZME5pMWhOamN3TFdJME0yRmpPV0kwTmprNU1pSXNJblI1Y0dVaU9pSk1hVzVsSW4wc0luTmxiR1ZqZEdsdmJsOW5iSGx3YUNJNmJuVnNiSDBzSW1sa0lqb2laV1UxT1dSbU1UY3ROR1ZpTnkwMFl6ZzVMV0UwTTJRdFpEVTNORFZsT1dFNE1qa3lJaXdpZEhsd1pTSTZJa2RzZVhCb1VtVnVaR1Z5WlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVkyRnNiR0poWTJzaU9tNTFiR3dzSW1OdmJIVnRibDl1WVcxbGN5STZXeUo0SWl3aWVTSmRMQ0prWVhSaElqcDdJbmdpT25zaVgxOXVaR0Z5Y21GNVgxOGlPaUpCUVVKQldEVkliV1JWU1VGQlEycFBiRTlhTVZGblFVRkZSREpaTlc1V1EwRkJSRFJ4TlhadFpGVkpRVUZQUVdGdUsxb3hVV2RCUVhsSmJXazFibFpEUVVGRGR5dExXRzFrVlVsQlFVcG9ibkZsV2pGUlowRkJaMDVoY3pWdVZrTkJRVUp2VW1KRWJXUlZTVUZCUmtNd2N5dGFNVkZuUVVGUFEwOHpOVzVXUTBGQlFXZHJjbkp0WkZWSlFVRkJaMEoyZFZveFVXZEJRVGhITDBJMWJsWkRRVUZFV1ROelZHMWtWVWxCUVUxQ1RubFBXakZSWjBGQmNVeDZURFZ1VmtOQlFVTlJTemd2YldSVlNVRkJTR2xoTUhWYU1WRm5RVUZaUVc1WE5XNVdRMEZCUWtsbFRtNXRaRlZKUVVGRVJHNHpUMW94VVdkQlFVZEdZbWMxYmxaRFFVRkJRWGhsVUcxa1ZVbEJRVTluZWpVcldqRlJaMEZCTUV0TWNUVnVWa05CUVVNMFJXVTNiV1JWU1VGQlMwTkJPR1ZhTVZGblFVRnBUeTh3Tlc1V1EwRkJRbmRZZG1wdFpGVkpRVUZHYWs0cksxb3hVV2RCUVZGRWVpODFibFpEUVVGQmIzRjNURzVrVlVsQlFVSkJZVUoxWkRGUlowRkJLMGxuU2pVelZrTkJRVVJuT1hkNmJtUlZTVUZCVFdodFJVOWtNVkZuUVVGelRsVlVOVE5XUTBGQlExbFNRbVp1WkZWSlFVRkpRM3BIZFdReFVXZEJRV0ZEU1dVMU0xWkRRVUZDVVd0VFNHNWtWVWxCUVVSblFVcGxaREZSWjBGQlNVYzRielV6VmtOQlFVRkpNMmwyYm1SVlNVRkJVRUpOVEN0a01WRm5RVUV5VEhONU5UTldRMEZCUkVGTGFtSnVaRlZKUVVGTGFWcFBaV1F4VVdkQlFXdEJaemsxTTFaRFFVRkNOR1F3Ukc1a1ZVbEJRVWRFYlZFclpERlJaMEZCVTBaV1NEVXpWa05CUVVGM2VFVnlibVJWU1VGQlFtZDZWSFZrTVZGblFVRkJTMHBTTlROV1EwRkJSRzlGUmxodVpGVkpRVUZPUWk5WFQyUXhVV2RCUVhWUE5XSTFNMVpEUVVGRFoxaFdMMjVrVlVsQlFVbHFUVmwxWkRGUlowRkJZMFIwYlRVelZrTkJRVUpaY1cxdWJtUlZTVUZCUlVGYVltVmtNVkZuUVVGTFNXaDNOVE5XUTBGQlFWRTVNMUJ1WkZWSlFVRlFhR3hrSzJReFVXZEJRVFJPVWpZMU0xWkRRVUZFU1ZFek4yNWtWVWxCUVV4RGVXZGxaREZSWjBGQmJVTkhSalV6VmtOQlFVTkJhMGxxYm1SVlNVRkJSMm92YVN0a01WRm5RVUZWUnpaUU5UTldRMEZCUVRReldreHVaRlZKUVVGRFFrMXNkV1F4VVdkQlFVTk1kVm8xTTFaRFFVRkVkMHRhTTI1a1ZVbEJRVTVwV1c5UFpERlJaMEZCZDBGbGF6VXpWa05CUVVOdlpIRm1ibVJWU1VGQlNrUnNjWFZrTVZGblFVRmxSbE4xTlROV1EwRkJRbWQzTjBodVpGVkpRVUZGWjNsMFpXUXhVV2RCUVUxTFJ6UTFNMVpEUVVGQldVVk1lbTVrVlVsQlFVRkNMM1lyWkRGUlowRkJOazh6UXpVelZrTkJRVVJSV0UxaWJtUlZTVUZCVEdwTWVXVmtNVkZuUVVGdlJISk9OVE5XUTBGQlEwbHhaRVJ1WkZWSlFVRklRVmt4VDJReFVXZEJRVmRKWmxnMU0xWkRRVUZDUVRsMGNtNWtWVWxCUVVOb2JETjFaREZSWjBGQlJVNVVhRFV6VmtOQlFVUTBVWFZZYm1SVlNVRkJUME40Tms5a01WRm5RVUY1UTBSek5UTldRMEZCUTNkcUt5OXVaRlZKUVVGS2FpczRkV1F4VVdkQlFXZEhNekkxTTFaRFFVRkNiek5RYm01a1ZVbEJRVVpDVEM5bFpERlJaMEZCVDB4dlFUWklWa05CUVVGblMxRlViMlJWU1VGQlFXbFpRaXRvTVZGblFVRTRRVmxNTmtoV1EwRkJSRmxrVVRkdlpGVkpRVUZOUkd0RlpXZ3hVV2RCUVhGR1RWWTJTRlpEUVVGRFVYZG9hbTlrVlVsQlFVaG5lRWhQYURGUlowRkJXVXRCWmpaSVZrTkJRVUpKUkhsUWIyUlZTVUZCUkVJclNuVm9NVkZuUVVGSFR6QndOa2hXUTBGQlFVRllRek52WkZWSlFVRlBha3ROVDJneFVXZEJRVEJFYXpBMlNGWkRRVUZETkhGRVptOWtWVWxCUVV0QldFOHJhREZSWjBGQmFVbFpLelpJVmtOQlFVSjNPVlZJYjJSVlNVRkJSbWhyVW1Wb01WRm5RVUZSVGs1Sk5raFdRMEZCUVc5UmEzcHZaRlZKUVVGQ1EzaFVLMmd4VVdkQlFTdENPVlEyU0ZaRFFVRkVaMnBzWW05a1ZVbEJRVTFxT1ZkbGFERlJaMEZCYzBkNFpEWklWa05CUVVOWk1qSkViMlJWU1VGQlNVSkxXazlvTVZGblFVRmhUR3h1TmtoV1EwRkJRbEZMUjNadlpGVkpRVUZFYVZoaWRXZ3hVV2RCUVVsQlduazJTRlpEUVVGQlNXUllXRzlrVlVsQlFWQkVhbVZQYURGUlowRkJNa1pLT0RaSVZrTWlMQ0prZEhsd1pTSTZJbVpzYjJGME5qUWlMQ0p6YUdGd1pTSTZXekUwTkYxOUxDSjVJanA3SWw5ZmJtUmhjbkpoZVY5Zklqb2lRVUZCUVdkS2RHUk5WVUZCUVVGRVoxaFdhM2hSUVVGQlFVVkJaMVpVUmtGQlFVRkJiMDlLVVUxVlFVRkJRVVJCZFhwamVGRkJRVUZCUVVOV1NHcEdRVUZCUVVGSlJ6UkdUVlZCUVVGQlFVRXZkazEzVVVGQlFVRkJRMDgwYWtKQlFVRkJRVFJDTTFKTlJVRkJRVUZCWjBoaU9IZFJRVUZCUVVWQlkzSlVRa0ZCUVVGQlowSjFZazFGUVVGQlFVSkJVVXRGZDFGQlFVRkJRVUpzY0hwQ1FVRkJRVUYzU1cxMFRVVkJRVUZCUkdkSWNuZDNVVUZCUVVGRFF6QjVha0pCUVVGQlFWRkZibHBOUlVGQlFVRkRaMGhQWjNkUlFVRkJRVUZFZHpscVFrRkJRVUZCV1UxTlJrMVZRVUZCUVVGQlRpOXpkMUZCUVVGQlRVTnhPRVJDUVVGQlFVRlpRamR0VFVWQlFVRkJSRUZHZDFsNFVVRkJRVUZEUVZKS2FrWkJRVUZCUVdkQmNFZE5WVUZCUVVGQlowbDVjM2hSUVVGQlFVMUJOMFZFUmtGQlFVRkJXVVpVTVUxRlFVRkJRVU5CWkRrMGQxRkJRVUZCUzBOaGVIcENRVUZCUVVGM1RESjNUVVZCUVVGQlFVRnZTMk4zVVVGQlFVRkZRME51YWtKQlFVRkJRV2RIVTFaTlJVRkJRVUZDUVRGeFFYZFJRVUZCUVVOQ1NYSkVRa0ZCUVVGQk5FeHRNMDFGUVVGQlFVTm5Samt3ZDFGQlFVRkJSVUl4UVdwR1FVRkJRVUZCVGsxdVRWVkJRVUZCUVdkcmFqQjRVVUZCUVVGRFFsSlZla1pCUVVGQlFWRkNRbkJOVlVGQlFVRkJRV0pXTUhoUlFVRkJRVXRFU2xWVVJrRkJRVUZCV1VOYVIwMVZRVUZCUVVSblJsTnplRkZCUVVGQlIwRkdSVVJHUVVGQlFVRTBVRlF3VFVWQlFVRkJSR2RSWldOM1VVRkJRVUZCUTFBeVZFSkJRVUZCUVVGT2VreE5SVUZCUVVGQ1p6Qk1NSGRSUVVGQlFVOUVSWEo2UWtGQlFVRkJVVXh0YUUxRlFVRkJRVVJCUWpWbmQxRkJRVUZCUjBKWGFtcENRVUZCUVVFMFMxTkZUVVZCUVVGQlJFRkRjRmwzVVVGQlFVRkpRbmR3ZWtKQlFVRkJRVmxPWVRSTlJVRkJRVUZFUVV4T05IZFJRVUZCUVVGRFJFRjZSa0ZCUVVGQldVNXJiMDFWUVVGQlFVSkJlbFIzZUZGQlFVRkJRVVJDVlVSR1FVRkJRVUUwVEZKclRWVkJRVUZCUW1kSE1WRjRVVUZCUVVGQlEwTlJla1pCUVVGQlFXZFBaM2xOVlVGQlFVRkJaM0JDYjNoUlFVRkJRVXRDWmtGcVJrRkJRVUZCVVVKMmNVMUZRVUZCUVVGQmNuUlpkMUZCUVVGQlMwSkJkM3BDUVVGQlFVRlpUazkyVFVWQlFVRkJSRUZuTm1kM1VVRkJRVUZCUVRCdlZFSkJRVUZCUVZsUFUxcE5SVUZCUVVGRFFXSmFTWGRSUVVGQlFVbEVNbWxxUWtGQlFVRkJiMGdyUkUxRlFVRkJRVUZCTTBwWmQxRkJRVUZCUjBFMGNXcENRVUZCUVVGM1NsTTVUVVZCUVVGQlFVRTVkVUYzVVVGQlFVRkhRbGhDUkVaQlFVRkJRVzlNWjI1TlZVRkJRVUZDWjA5VFRYaFJRVUZCUVVORE5raHFSa0ZCUVVGQk5FUnZZVTFWUVVGQlFVUm5iMUZ6ZUZGQlFVRkJUVUZKTDFSQ1FVRkJRVUYzUnk5MVRVVkJRVUZCUkVFd1RqaDNVVUZCUVVGTlFYZ3dWRUpCUVVGQlFYZEtURU5OUlVGQlFVRkRRV1ZNTkhkUlFVRkJRVU5DWlhWcVFrRkJRVUZCTkVWUE1rMUZRVUZCUVVObmR6ZGpkMUZCUVVGQlJVSkVkVlJDUVVGQlFVRkJUVTgyVFVWQlFVRkJRa0V4WW1OM1VVRkJRVUZMUkc1MFJFSkJRVUZCUVRSUWJYaE5SVUZCUVVGQ1oxaHpSWGRSUVVGQlFVMUVRekJFUWtGQlFVRkJVVU5tWjAxRlFVRkJRVUZuYm5aamQxRkJRVUZCUTBGV1JIcEdRVUZCUVVGQlNYZHRUVlZCUVVGQlFXZE5lbFY0VVVGQlFVRkhSR0ZSZWtaQlFVRkJRV2RKUmxOTlZVRkJRVUZFWnpaNlZYaFJRVUZCUVVOQ1YwZFVSa0ZCUVVGQlowMUVPRTFGUVVGQlFVRm5WMlZaZDFGQlFVRkJTMFI0ZW5wQ1FVRkJRVUZSU1hFMVRVVkJRVUZCUVVGNk4xbDNVVUZCUVVGTFFWUjBSRUpCUVVGQlFWbEdhWGhOUlVGQlFVRkNRWFp5U1hkUlFVRkJRVU5CYTNSRVFrRkJRVUZCUVVseE1VMUZRVUZCUVVOQlNreFpkMUZCUVVGQlQwTXJkR3BDUVVGQlFVRlpSbTB6VFVWQlFVRkJSRUZ2WWpSM1VVRkJRVUZEUkhGNFZFSkJRVUZCUVdkRVRFNU5SVUZCUVVGQ1FVcGtXWGRSUVVGQlFVOUJXRE42UWtGQlFVRkJiMEZ5YjAxRlFVRkJRVUZuUVNzNGQxRkJRVUZCU1VRM09WUkNRVUZCUVVGQlVGUTRUVVZCUVVGQlFVRTVVSGQzVVVGQlFVRkJSREF2UkVKQklpd2laSFI1Y0dVaU9pSm1iRzloZERZMElpd2ljMmhoY0dVaU9sc3hORFJkZlgxOUxDSnBaQ0k2SW1JNU0yRXlOMlZrTFdGak56a3ROR013WlMxaE5tSXhMVGMyWkRneVptSXpZVGcxTWlJc0luUjVjR1VpT2lKRGIyeDFiVzVFWVhSaFUyOTFjbU5sSW4wc2V5SmhkSFJ5YVdKMWRHVnpJanA3SW05MlpYSnNZWGtpT25zaWFXUWlPaUk1T0dFNFptTTVOeTFqTmpnNExUUXlPREF0T0RkaFpDMWtaREZqTUdJNU9ERTJNVFlpTENKMGVYQmxJam9pUW05NFFXNXViM1JoZEdsdmJpSjlMQ0p3Ykc5MElqcDdJbWxrSWpvaU5ESXdPV1V4TURZdFlqVm1aQzAwWkdRekxUZzRNakV0TWpObFlXWmxPREEyT0RZNElpd2ljM1ZpZEhsd1pTSTZJa1pwWjNWeVpTSXNJblI1Y0dVaU9pSlFiRzkwSW4xOUxDSnBaQ0k2SW1ReE9UZzJNakJqTFRoa01XRXROR0UwTWkwNFltUm1MVFE1T1dJMFlqSXpNVFE0TmlJc0luUjVjR1VpT2lKQ2IzaGFiMjl0Vkc5dmJDSjlMSHNpWVhSMGNtbGlkWFJsY3lJNmV5SnNhVzVsWDJGc2NHaGhJanA3SW5aaGJIVmxJam93TGpZMWZTd2liR2x1WlY5allYQWlPaUp5YjNWdVpDSXNJbXhwYm1WZlkyOXNiM0lpT25zaWRtRnNkV1VpT2lJak1XWTNOMkkwSW4wc0lteHBibVZmYW05cGJpSTZJbkp2ZFc1a0lpd2liR2x1WlY5M2FXUjBhQ0k2ZXlKMllXeDFaU0k2Tlgwc0luZ2lPbnNpWm1sbGJHUWlPaUo0SW4wc0lua2lPbnNpWm1sbGJHUWlPaUo1SW4xOUxDSnBaQ0k2SW1aalpqQTJOamM0TFRkbE5UUXROREl4T1MwNU0yWXhMVEprTm1Jd1pURmhNRGt5WmlJc0luUjVjR1VpT2lKTWFXNWxJbjBzZXlKaGRIUnlhV0oxZEdWeklqcDdJbXhwYm1WZllXeHdhR0VpT25zaWRtRnNkV1VpT2pBdU1YMHNJbXhwYm1WZlkyRndJam9pY205MWJtUWlMQ0pzYVc1bFgyTnZiRzl5SWpwN0luWmhiSFZsSWpvaUl6Rm1OemRpTkNKOUxDSnNhVzVsWDJwdmFXNGlPaUp5YjNWdVpDSXNJbXhwYm1WZmQybGtkR2dpT25zaWRtRnNkV1VpT2pWOUxDSjRJanA3SW1acFpXeGtJam9pZUNKOUxDSjVJanA3SW1acFpXeGtJam9pZVNKOWZTd2lhV1FpT2lKaU9XUmpPREppTmkwM1kyRTBMVFJtTkRZdFlUWTNNQzFpTkROaFl6bGlORFk1T1RJaUxDSjBlWEJsSWpvaVRHbHVaU0o5TEhzaVlYUjBjbWxpZFhSbGN5STZleUpzYVc1bFgyRnNjR2hoSWpwN0luWmhiSFZsSWpvd0xqRjlMQ0pzYVc1bFgyTmhjQ0k2SW5KdmRXNWtJaXdpYkdsdVpWOWpiMnh2Y2lJNmV5SjJZV3gxWlNJNklpTXhaamMzWWpRaWZTd2liR2x1WlY5cWIybHVJam9pY205MWJtUWlMQ0pzYVc1bFgzZHBaSFJvSWpwN0luWmhiSFZsSWpvMWZTd2llQ0k2ZXlKbWFXVnNaQ0k2SW5naWZTd2llU0k2ZXlKbWFXVnNaQ0k2SW5raWZYMHNJbWxrSWpvaVpEazRNMk5tWVRrdE0yWTFOUzAwWmpkbUxUZ3hZbVF0WkdRd05tVTFPRGc0T0dVMUlpd2lkSGx3WlNJNklreHBibVVpZlN4N0ltRjBkSEpwWW5WMFpYTWlPbnNpYlc5dWRHaHpJanBiTUN3eExESXNNeXcwTERVc05pdzNMRGdzT1N3eE1Dd3hNVjE5TENKcFpDSTZJbVU1T1RkaVpERTNMVFpqWldFdE5ETmlOQzA0WVdKaUxXRXlPVFF3Wm1FNFlqazVNaUlzSW5SNWNHVWlPaUpOYjI1MGFITlVhV05yWlhJaWZTeDdJbUYwZEhKcFluVjBaWE1pT25zaVpHRjVjeUk2V3pFc09Dd3hOU3d5TWwxOUxDSnBaQ0k2SWpBNVpHSTRNbUZsTFRjelpXSXRORE5qT0MxaVptWm1MVGhsWldNek1ESmhNVEV5WWlJc0luUjVjR1VpT2lKRVlYbHpWR2xqYTJWeUluMHNleUpoZEhSeWFXSjFkR1Z6SWpwN0lteHBibVZmWVd4d2FHRWlPbnNpZG1Gc2RXVWlPakF1TmpWOUxDSnNhVzVsWDJOaGNDSTZJbkp2ZFc1a0lpd2liR2x1WlY5amIyeHZjaUk2ZXlKMllXeDFaU0k2SWlObVptSmlOemdpZlN3aWJHbHVaVjlxYjJsdUlqb2ljbTkxYm1RaUxDSnNhVzVsWDNkcFpIUm9JanA3SW5aaGJIVmxJam8xZlN3aWVDSTZleUptYVdWc1pDSTZJbmdpZlN3aWVTSTZleUptYVdWc1pDSTZJbmtpZlgwc0ltbGtJam9pTURGaU1UZzJZVGN0TmpFNU5pMDBOV00xTFRsa09Ea3RNemxsWkdKak9UTTFZVEZqSWl3aWRIbHdaU0k2SWt4cGJtVWlmVjBzSW5KdmIzUmZhV1J6SWpwYklqUXlNRGxsTVRBMkxXSTFabVF0TkdSa015MDRPREl4TFRJelpXRm1aVGd3TmpnMk9DSmRmU3dpZEdsMGJHVWlPaUpDYjJ0bGFDQkJjSEJzYVdOaGRHbHZiaUlzSW5abGNuTnBiMjRpT2lJd0xqRXlMallpZlgwN0NpQWdJQ0FnSUNBZ0lDQWdJQ0FnZG1GeUlISmxibVJsY2w5cGRHVnRjeUE5SUZ0N0ltUnZZMmxrSWpvaU9UVXlOV00yWldNdE16UmlNUzAwTTJRNExXRTBabUV0TW1JeVpqWTNPV1EzWm1Gaklpd2laV3hsYldWdWRHbGtJam9pTWpOa1pXWTVaV1V0WmpNMVpTMDBOR05qTFRsbU5qVXRORGhoT0RObU9USTBZalUySWl3aWJXOWtaV3hwWkNJNklqUXlNRGxsTVRBMkxXSTFabVF0TkdSa015MDRPREl4TFRJelpXRm1aVGd3TmpnMk9DSjlYVHNLSUNBZ0lDQWdJQ0FnSUNBZ0lDQUtJQ0FnSUNBZ0lDQWdJQ0FnSUNCQ2IydGxhQzVsYldKbFpDNWxiV0psWkY5cGRHVnRjeWhrYjJOelgycHpiMjRzSUhKbGJtUmxjbDlwZEdWdGN5azdDaUFnSUNBZ0lDQWdJQ0FnSUgwcE93b2dJQ0FnSUNBZ0lDQWdmVHNLSUNBZ0lDQWdJQ0FnSUdsbUlDaGtiMk4xYldWdWRDNXlaV0ZrZVZOMFlYUmxJQ0U5SUNKc2IyRmthVzVuSWlrZ1ptNG9LVHNLSUNBZ0lDQWdJQ0FnSUdWc2MyVWdaRzlqZFcxbGJuUXVZV1JrUlhabGJuUk1hWE4wWlc1bGNpZ2lSRTlOUTI5dWRHVnVkRXh2WVdSbFpDSXNJR1p1S1RzS0lDQWdJQ0FnSUNCOUtTZ3BPd29nSUNBZ0lDQWdJQW9nSUNBZ0lDQWdJRHd2YzJOeWFYQjBQZ29nSUNBZ1BDOWliMlI1UGdvOEwyaDBiV3crIiB3aWR0aD0iNzkwIiBzdHlsZT0iYm9yZGVyOm5vbmUgIWltcG9ydGFudDsiIGhlaWdodD0iMzMwIj48L2lmcmFtZT4nKVswXTsKICAgICAgICAgICAgICAgIHBvcHVwX2NjMzM2OTRhYjc0OTRiOTViOTk1YWU2YWJhNjY1OGU4LnNldENvbnRlbnQoaV9mcmFtZV9hMzA0MzgyZWI0MjY0OWMwOTY4YjQ4NjAxNjMxM2JmMCk7CiAgICAgICAgICAgIAoKICAgICAgICAgICAgbWFya2VyXzVlMThmYjQ3MDE5NzQxMTJhNDA3NTAxN2Q4NTllMGQ2LmJpbmRQb3B1cChwb3B1cF9jYzMzNjk0YWI3NDk0Yjk1Yjk5NWFlNmFiYTY2NThlOCk7CgogICAgICAgICAgICAKICAgICAgICAKICAgIAogICAgICAgICAgICB2YXIgbGF5ZXJfY29udHJvbF85MjJlNDY3MWRmZmQ0MzU2YmQwZGYxMTNiZDM1OWNiNSA9IHsKICAgICAgICAgICAgICAgIGJhc2VfbGF5ZXJzIDogeyAib3BlbnN0cmVldG1hcCIgOiB0aWxlX2xheWVyXzRmMWY3ZmY1MWE3YTRiMDk5ZGVhMTFkOTg5ZTFhMDkyLCB9LAogICAgICAgICAgICAgICAgb3ZlcmxheXMgOiB7ICJTZWEgU3VyZmFjZSBUZW1wZXJhdHVyZSIgOiBtYWNyb19lbGVtZW50XzAyNTE1Y2RkMWRmYzRjYTk4MGM0Y2Y0MDQxN2Q5ZGI2LCJDbHVzdGVyIiA6IG1hcmtlcl9jbHVzdGVyXzFjYTA3ODUzZTY2OTRmOGVhYWVmY2M0ZTEyOTk4ZmU2LCB9CiAgICAgICAgICAgICAgICB9OwogICAgICAgICAgICBMLmNvbnRyb2wubGF5ZXJzKAogICAgICAgICAgICAgICAgbGF5ZXJfY29udHJvbF85MjJlNDY3MWRmZmQ0MzU2YmQwZGYxMTNiZDM1OWNiNS5iYXNlX2xheWVycywKICAgICAgICAgICAgICAgIGxheWVyX2NvbnRyb2xfOTIyZTQ2NzFkZmZkNDM1NmJkMGRmMTEzYmQzNTljYjUub3ZlcmxheXMsCiAgICAgICAgICAgICAgICB7cG9zaXRpb246ICd0b3ByaWdodCcsCiAgICAgICAgICAgICAgICAgY29sbGFwc2VkOiB0cnVlLAogICAgICAgICAgICAgICAgIGF1dG9aSW5kZXg6IHRydWUKICAgICAgICAgICAgICAgIH0pLmFkZFRvKG1hcF8xNWZhYzA4NmZjODk0NzE0YWRlZWUzYTdlMDUyNDE1OSk7CiAgICAgICAgCjwvc2NyaXB0Pg==" style="position:absolute;width:100%;height:100%;left:0;top:0;border:none !important;" allowfullscreen webkitallowfullscreen mozallowfullscreen></iframe></div></div>



Now we can navigate the map and click on the markers to explorer our findings.

The green markers locate the observations locations. They pop-up an interactive plot with the time-series and scores for the models (hover over the lines to se the scores). The blue markers indicate the nearest model grid point found for the comparison.
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2016-12-22-boston_light_swim.ipynb)
this notebook, or click [here](https://beta.mybinder.org/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2016-12-22-boston_light_swim.ipynb) to run a live instance of this notebook.