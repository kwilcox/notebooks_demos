---
layout: notebook
title: ""
---
## Shore Station Compliance Checker Script

The IOOS Compliance Checker is a Python-based tool that helps users check the meta data compliance of a netCDF file. This software can be run in a web interface here: https://data.ioos.us/compliance/index.html The checker can also be run as a Python tool either on the command line or in a Python script.  This notebook demonstrates the python usage of the Compliance Checker.


### Purpose: 
Run the compliance checker python tool on a Scipps Pier shore station dataset to check for the metadata compliance.

The Scripps Pier automated shore station operated by Southern California Coastal Ocean Observing System (SCCOOS) at Scripps Institution of Oceanography (SIO) is mounted at a nominal depth of 5 meters MLLW. The instrument package includes a Seabird SBE 16plus SEACAT Conductivity, Temperature, and Pressure recorder, and a Seapoint Chlorophyll Fluorometer with a 0-50 ug/L gain setting.

### Dependencies: 
This script must be run in the "IOOS" environment for the compliance checker to work properly.

Written by: J.Bosch Feb. 10, 2017



<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
# First import the compliance checker and test that it is installed properly.
from compliance_checker.runner import ComplianceChecker, CheckSuite

# Load all available checker classes.
check_suite = CheckSuite()
check_suite.load_all_available_checkers()
```

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
# Path to the Scripps Pier Data.

buoy_path = 'https://data.nodc.noaa.gov/thredds/dodsC/ioos/sccoos/scripps_pier/scripps_pier-2016.nc'
```

### Running Compliance Checker on the Scripps Pier shore station data
This code is written with all the arguments spelled out, following the usage instructions on the README section of compliance checker github page: https://github.com/ioos/compliance-checker

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
output_file = 'buoy_testCC.txt'

return_value, errors = ComplianceChecker.run_checker(
    ds_loc=buoy_path,
    checker_names=['cf', 'acdd'],
    verbose=True,
    criteria='normal',
    skip_checks=None,
    output_filename=output_file,
    output_format='text'
)
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Error fetching standard name table. Using packaged v36

</pre>
</div><div class="warning" style="border:thin solid red">
    WARNING: The following exceptions occured during the acdd checker (possibly
indicate compliance checker issues):
    acdd.check_vertical_extents: ufunc 'isfinite' not supported for the input
types, and the inputs could not be safely coerced to any supported types
according to the casting rule ''safe''
      File "/home/filipe/miniconda3/envs/IOOS3/lib/python3.5/site-
packages/compliance_checker/acdd.py", line 445, in check_vertical_extents
        return self._check_scalar_vertical_extents(ds, z_variable)
      File "/home/filipe/miniconda3/envs/IOOS3/lib/python3.5/site-
packages/compliance_checker/acdd.py", line 411, in
_check_scalar_vertical_extents
        if not np.isclose(vert_min, vert_max):
      File "/home/filipe/miniconda3/envs/IOOS3/lib/python3.5/site-
packages/numpy/core/numeric.py", line 2450, in isclose
        xfin = isfinite(x)


</div>
<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
with open(output_file, 'r') as f:
    print(f.read())
```
<div class="output_area"><div class="prompt"></div>
<pre>
    
    
    --------------------------------------------------------------------------------
                        The dataset scored 648 out of 650 points                    
                                  during the cf check                               
    --------------------------------------------------------------------------------
                               Verbose Scoring Breakdown:                            
    
                                     High Priority                                  
    --------------------------------------------------------------------------------
        Name                            :Priority: Score
    §2.2 Valid netCDF data types            :3:    30/30
    §2.4 Unique dimensions                  :3:    30/30
    §3.1 Variable aux1 contains valid CF un :3:     3/3
    §3.1 Variable aux3 contains valid CF un :3:     3/3
    §3.1 Variable aux4 contains valid CF un :3:     3/3
    §3.1 Variable chlorophyll contains vali :3:     3/3
    §3.1 Variable chlorophyll's units are a :3:     1/1
    §3.1 Variable conductivity contains val :3:     3/3
    §3.1 Variable conductivity's units are  :3:     1/1
    §3.1 Variable currentDraw contains vali :3:     3/3
    §3.1 Variable depth contains valid CF u :3:     3/3
    §3.1 Variable depth's units are appropr :3:     1/1
    §3.1 Variable diagnosticVoltage contain :3:     3/3
    §3.1 Variable lat contains valid CF uni :3:     3/3
    §3.1 Variable lat's units are appropria :3:     1/1
    §3.1 Variable lon contains valid CF uni :3:     3/3
    §3.1 Variable lon's units are appropria :3:     1/1
    §3.1 Variable pressure contains valid C :3:     3/3
    §3.1 Variable pressure's units are appr :3:     1/1
    §3.1 Variable salinity contains valid C :3:     3/3
    §3.1 Variable salinity's units are appr :3:     1/1
    §3.1 Variable sigmat contains valid CF  :3:     3/3
    §3.1 Variable sigmat's units are approp :3:     1/1
    §3.1 Variable temperature contains vali :3:     3/3
    §3.1 Variable temperature's units are a :3:     1/1
    §3.1 Variable time contains valid CF un :3:     3/3
    §3.1 Variable time's units are appropri :3:     1/1
    §3.3 Variable chlorophyll has valid sta :3:     2/2
    §3.3 Variable chlorophyll_flagPrimary h :3:     2/2
    §3.3 Variable conductivity has valid st :3:     2/2
    §3.3 Variable conductivity_flagPrimary  :3:     2/2
    §3.3 Variable depth has valid standard_ :3:     2/2
    §3.3 Variable lat has valid standard_na :3:     2/2
    §3.3 Variable lon has valid standard_na :3:     2/2
    §3.3 Variable pressure has valid standa :3:     2/2
    §3.3 Variable pressure_flagPrimary has  :3:     2/2
    §3.3 Variable salinity has valid standa :3:     2/2
    §3.3 Variable salinity_flagPrimary has  :3:     2/2
    §3.3 Variable sigmat has valid standard :3:     2/2
    §3.3 Variable temperature has valid sta :3:     2/2
    §3.3 Variable temperature_flagPrimary h :3:     2/2
    §3.3 Variable time has valid standard_n :3:     2/2
    §3.3 standard_name modifier for chlorop :3:     1/1
    §3.3 standard_name modifier for conduct :3:     1/1
    §3.3 standard_name modifier for pressur :3:     1/1
    §3.3 standard_name modifier for salinit :3:     1/1
    §3.3 standard_name modifier for tempera :3:     1/1
    §3.5 chlorophyll_flagPrimary is a valid :3:     1/1
    §3.5 chlorophyll_flagSecondary is a val :3:     1/1
    §3.5 conductivity_flagPrimary is a vali :3:     1/1
    §3.5 conductivity_flagSecondary is a va :3:     1/1
    §3.5 flag_meanings for chlorophyll_flag :3:     3/3
    §3.5 flag_meanings for chlorophyll_flag :3:     3/3
    §3.5 flag_meanings for conductivity_fla :3:     3/3
    §3.5 flag_meanings for conductivity_fla :3:     3/3
    §3.5 flag_meanings for pressure_flagPri :3:     3/3
    §3.5 flag_meanings for pressure_flagSec :3:     3/3
    §3.5 flag_meanings for salinity_flagPri :3:     3/3
    §3.5 flag_meanings for salinity_flagSec :3:     3/3
    §3.5 flag_meanings for temperature_flag :3:     3/3
    §3.5 flag_meanings for temperature_flag :3:     3/3
    §3.5 flag_values for chlorophyll_flagPr :3:     4/4
    §3.5 flag_values for chlorophyll_flagSe :3:     4/4
    §3.5 flag_values for conductivity_flagP :3:     4/4
    §3.5 flag_values for conductivity_flagS :3:     4/4
    §3.5 flag_values for pressure_flagPrima :3:     4/4
    §3.5 flag_values for pressure_flagSecon :3:     4/4
    §3.5 flag_values for salinity_flagPrima :3:     4/4
    §3.5 flag_values for salinity_flagSecon :3:     4/4
    §3.5 flag_values for temperature_flagPr :3:     4/4
    §3.5 flag_values for temperature_flagSe :3:     4/4
    §3.5 pressure_flagPrimary is a valid fl :3:     1/1
    §3.5 pressure_flagSecondary is a valid  :3:     1/1
    §3.5 salinity_flagPrimary is a valid fl :3:     1/1
    §3.5 salinity_flagSecondary is a valid  :3:     1/1
    §3.5 temperature_flagPrimary is a valid :3:     1/1
    §3.5 temperature_flagSecondary is a val :3:     1/1
    §4 depth contains a valid axis          :3:     2/2
    §4 lat contains a valid axis            :3:     2/2
    §4 lon contains a valid axis            :3:     2/2
    §4 time contains a valid axis           :3:     2/2
    §4.1 Latitude variable lat has required :3:     1/1
    §4.1 Longitude variable lon has require :3:     1/1
    §4.3.1 depth is a valid vertical coordi :3:     2/2
    §4.4 Time coordinate variable and attri :3:     2/2
    §5.0 Auxiliary Coordinates of aux1 must :3:     8/8
    §5.0 Auxiliary Coordinates of aux3 must :3:     8/8
    §5.0 Auxiliary Coordinates of aux4 must :3:     8/8
    §5.0 Auxiliary Coordinates of chlorophy :3:     8/8
    §5.0 Auxiliary Coordinates of conductiv :3:     8/8
    §5.0 Auxiliary Coordinates of currentDr :3:     8/8
    §5.0 Auxiliary Coordinates of diagnosti :3:     8/8
    §5.0 Auxiliary Coordinates of pressure  :3:     8/8
    §5.0 Auxiliary Coordinates of salinity  :3:     8/8
    §5.0 Auxiliary Coordinates of sigmat mu :3:     8/8
    §5.0 Auxiliary Coordinates of temperatu :3:     8/8
    §5.0 Variable aux1 does not contain dup :3:     4/4
    §5.0 Variable aux3 does not contain dup :3:     4/4
    §5.0 Variable aux4 does not contain dup :3:     4/4
    §5.0 Variable chlorophyll does not cont :3:     4/4
    §5.0 Variable conductivity does not con :3:     4/4
    §5.0 Variable currentDraw does not cont :3:     4/4
    §5.0 Variable diagnosticVoltage does no :3:     4/4
    §5.0 Variable pressure does not contain :3:     4/4
    §5.0 Variable salinity does not contain :3:     4/4
    §5.0 Variable sigmat does not contain d :3:     4/4
    §5.0 Variable temperature does not cont :3:     4/4
    §9.1 Dataset contains a valid featureTy :3:     1/1
    §9.1 Feature Types are all the same     :3:     1/1
    
    
                                    Medium Priority                                 
    --------------------------------------------------------------------------------
        Name                            :Priority: Score
    §2.3 Naming Conventions for attributes  :2:   161/162
    §2.3 Naming Conventions for dimensions  :2:     2/2
    §2.3 Naming Conventions for variables   :2:    30/30
    §2.3 Unique variable names              :2:    30/30
    §2.4 Dimension Order                    :2:    22/22
    §2.5.1 Fill Values should be outside th :2:     0/0
    §2.6.1 Global Attribute Conventions inc :2:     1/1
    §2.6.2 Recommended Attributes           :2:     7/8
    §2.6.2 Recommended Global Attributes    :2:     2/2
    §4.1 Latitude variable lat defines eith :2:     1/1
    §4.1 Latitude variable lat uses recomme :2:     1/1
    §4.1 Longitude variable lon defines eit :2:     1/1
    §4.1 Longitude variable lon uses recomm :2:     1/1
    §9.1 Feature Type for aux1 is valid tim :2:     1/1
    §9.1 Feature Type for aux3 is valid tim :2:     1/1
    §9.1 Feature Type for aux4 is valid tim :2:     1/1
    §9.1 Feature Type for chlorophyll is va :2:     1/1
    §9.1 Feature Type for conductivity is v :2:     1/1
    §9.1 Feature Type for currentDraw is va :2:     1/1
    §9.1 Feature Type for diagnosticVoltage :2:     1/1
    §9.1 Feature Type for pressure is valid :2:     1/1
    §9.1 Feature Type for salinity is valid :2:     1/1
    §9.1 Feature Type for sigmat is valid t :2:     1/1
    §9.1 Feature Type for temperature is va :2:     1/1
    
    
                                      Low Priority                                  
    --------------------------------------------------------------------------------
        Name                            :Priority: Score
    §3.1 Variable aux1's units are containe :1:     1/1
    §3.1 Variable aux3's units are containe :1:     1/1
    §3.1 Variable aux4's units are containe :1:     1/1
    §3.1 Variable chlorophyll's units are c :1:     1/1
    §3.1 Variable conductivity's units are  :1:     1/1
    §3.1 Variable currentDraw's units are c :1:     1/1
    §3.1 Variable depth's units are contain :1:     1/1
    §3.1 Variable diagnosticVoltage's units :1:     1/1
    §3.1 Variable lat's units are contained :1:     1/1
    §3.1 Variable lon's units are contained :1:     1/1
    §3.1 Variable pressure's units are cont :1:     1/1
    §3.1 Variable salinity's units are cont :1:     1/1
    §3.1 Variable sigmat's units are contai :1:     1/1
    §3.1 Variable temperature's units are c :1:     1/1
    §3.1 Variable time's units are containe :1:     1/1
    §4.4.1 Time and calendar                :1:     1/1
    
    
    --------------------------------------------------------------------------------
                      Reasoning for the failed tests given below:                   
    
    
    Name                             Priority:     Score:Reasoning
    --------------------------------------------------------------------------------
    §2.3 Naming Conventions for attributes :2:   161/162 : attribute
                                                          time:_Netcdf4Dimid should
                                                          begin with a letter and be
                                                          composed of letters,
                                                          digits, and underscores
    §2.6.2 Recommended Attributes          :2:     7/ 8 : references should be
                                                          defined
    
    
    --------------------------------------------------------------------------------
                         The dataset scored 77 out of 99 points                     
                                 during the acdd check                              
    --------------------------------------------------------------------------------
                               Verbose Scoring Breakdown:                            
    
                                     High Priority                                  
    --------------------------------------------------------------------------------
        Name                            :Priority: Score
    Conventions                             :3:     1/2
    keywords                                :3:     1/1
    summary                                 :3:     1/1
    title                                   :3:     1/1
    varattr                                 :3:    37/53
        aux1                                :3:       2/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         1/2
            var_units                       :3:         1/1
        aux3                                :3:       2/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         1/2
            var_units                       :3:         1/1
        aux4                                :3:       2/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         1/2
            var_units                       :3:         1/1
        chlorophyll                         :3:       3/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         2/2
            var_units                       :3:         1/1
        conductivity                        :3:       3/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         2/2
            var_units                       :3:         1/1
        currentDraw                         :3:       2/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         1/2
            var_units                       :3:         1/1
        depth                               :3:       2/2
            var_std_name                    :3:         2/2
        diagnosticVoltage                   :3:       2/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         1/2
            var_units                       :3:         1/1
        lat                                 :3:       2/2
            var_std_name                    :3:         2/2
        lon                                 :3:       2/2
            var_std_name                    :3:         2/2
        pressure                            :3:       3/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         2/2
            var_units                       :3:         1/1
        salinity                            :3:       3/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         2/2
            var_units                       :3:         1/1
        sigmat                              :3:       3/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         2/2
            var_units                       :3:         1/1
        temperature                         :3:       3/4
            coverage_content_type           :3:         0/1
            var_std_name                    :3:         2/2
            var_units                       :3:         1/1
        time                                :3:       3/3
            var_std_name                    :3:         2/2
            var_units                       :3:         1/1
    
    
                                    Medium Priority                                 
    --------------------------------------------------------------------------------
        Name                            :Priority: Score
    acknowledgment/acknowledgement          :2:     1/1
    comment                                 :2:     1/1
    creator_email                           :2:     0/1
    creator_name                            :2:     1/1
    creator_url                             :2:     1/1
    date_created                            :2:     1/1
    date_created_is_iso                     :2:     1/1
    date_issued_is_iso                      :2:     1/1
    date_metadata_modified_is_iso           :2:     0/0
    date_modified_is_iso                    :2:     1/1
    geospatial_bounds                       :2:     0/1
    geospatial_bounds_crs                   :2:     0/1
    geospatial_bounds_vertical_crs          :2:     0/1
    geospatial_lat_extents_match            :2:     2/2
    geospatial_lat_max                      :2:     1/1
    geospatial_lat_min                      :2:     1/1
    geospatial_lon_extents_match            :2:     2/2
    geospatial_lon_max                      :2:     1/1
    geospatial_lon_min                      :2:     1/1
    geospatial_vertical_max                 :2:     1/1
    geospatial_vertical_min                 :2:     1/1
    geospatial_vertical_positive            :2:     1/1
    history                                 :2:     1/1
    id                                      :2:     0/1
    id_has_no_blanks                        :2:     0/0
    institution                             :2:     1/1
    license                                 :2:     1/1
    naming_authority                        :2:     1/1
    processing_level                        :2:     1/1
    project                                 :2:     1/1
    publisher_email                         :2:     1/1
    publisher_name                          :2:     1/1
    publisher_url                           :2:     1/1
    source                                  :2:     1/1
    standard_name_vocabulary                :2:     1/1
    time_coverage_duration                  :2:     1/1
    time_coverage_end                       :2:     1/1
    time_coverage_extents_match             :2:     2/2
    time_coverage_resolution                :2:     1/1
    time_coverage_start                     :2:     1/1
    
    
                                      Low Priority                                  
    --------------------------------------------------------------------------------
        Name                            :Priority: Score
    contributor_name                        :1:     1/1
    contributor_role                        :1:     1/1
    creator_institution                     :1:     0/1
    creator_type                            :1:     0/2
    date_issued                             :1:     1/1
    date_metadata_modified                  :1:     0/1
    date_modified                           :1:     1/1
    geospatial_lat_resolution               :1:     1/1
    geospatial_lat_units                    :1:     1/1
    geospatial_lon_resolution               :1:     1/1
    geospatial_lon_units                    :1:     1/1
    geospatial_vertical_resolution          :1:     1/1
    geospatial_vertical_units               :1:     1/1
    instrument                              :1:     0/1
    instrument_vocabulary                   :1:     0/1
    keywords_vocabulary                     :1:     1/1
    metadata_link                           :1:     1/1
    metadata_link_valid                     :1:     0/1
    platform                                :1:     0/1
    platform_vocabulary                     :1:     0/1
    product_version                         :1:     0/1
    program                                 :1:     0/1
    publisher_institution                   :1:     0/1
    publisher_type                          :1:     0/2
    references                              :1:     0/1
    
    
    --------------------------------------------------------------------------------
                      Reasoning for the failed tests given below:                   
    
    
    Name                             Priority:     Score:Reasoning
    --------------------------------------------------------------------------------
    Conventions                            :3:     1/ 2 : Attr Conventions does not
                                                          contain 'ACDD-1.3'
    varattr                                :3:    37/53 :  
        aux1                               :3:     2/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var aux1 missing attr
                                                          coverage_content_type
            var_std_name                   :3:     1/ 2 : Var aux1 missing attr
                                                          standard_name
        aux3                               :3:     2/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var aux3 missing attr
                                                          coverage_content_type
            var_std_name                   :3:     1/ 2 : Var aux3 missing attr
                                                          standard_name
        aux4                               :3:     2/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var aux4 missing attr
                                                          coverage_content_type
            var_std_name                   :3:     1/ 2 : Var aux4 missing attr
                                                          standard_name
        chlorophyll                        :3:     3/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var chlorophyll missing
                                                          attr coverage_content_type
        conductivity                       :3:     3/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var conductivity missing
                                                          attr coverage_content_type
        currentDraw                        :3:     2/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var currentDraw missing
                                                          attr coverage_content_type
            var_std_name                   :3:     1/ 2 : Var currentDraw missing
                                                          attr standard_name
        diagnosticVoltage                  :3:     2/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var diagnosticVoltage
                                                          missing attr
                                                          coverage_content_type
            var_std_name                   :3:     1/ 2 : Var diagnosticVoltage
                                                          missing attr standard_name
        pressure                           :3:     3/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var pressure missing attr
                                                          coverage_content_type
        salinity                           :3:     3/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var salinity missing attr
                                                          coverage_content_type
        sigmat                             :3:     3/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var sigmat missing attr
                                                          coverage_content_type
        temperature                        :3:     3/ 4 :  
            coverage_content_type          :3:     0/ 1 : Var temperature missing
                                                          attr coverage_content_type
    creator_email                          :2:     0/ 1 : Attr creator_email not
                                                          present
    geospatial_bounds                      :2:     0/ 1 : Attr geospatial_bounds not
                                                          present
    geospatial_bounds_crs                  :2:     0/ 1 : Attr geospatial_bounds_crs
                                                          not present
    geospatial_bounds_vertical_crs         :2:     0/ 1 : Attr
                                                          geospatial_bounds_vertical
                                                          _crs not present
    id                                     :2:     0/ 1 : Attr id not present
    creator_institution                    :1:     0/ 1 : Attr creator_institution
                                                          not present
    creator_type                           :1:     0/ 2 : Attr creator_type not
                                                          present
    date_metadata_modified                 :1:     0/ 1 : Attr
                                                          date_metadata_modified not
                                                          present
    instrument                             :1:     0/ 1 : Attr instrument not
                                                          present
    instrument_vocabulary                  :1:     0/ 1 : Attr instrument_vocabulary
                                                          not present
    metadata_link_valid                    :1:     0/ 1 : Metadata URL should
                                                          include http:// or
                                                          https://
    platform                               :1:     0/ 1 : Attr platform not present
    platform_vocabulary                    :1:     0/ 1 : Attr platform_vocabulary
                                                          not present
    product_version                        :1:     0/ 1 : Attr product_version not
                                                          present
    program                                :1:     0/ 1 : Attr program not present
    publisher_institution                  :1:     0/ 1 : Attr publisher_institution
                                                          not present
    publisher_type                         :1:     0/ 2 : Attr publisher_type not
                                                          present
    references                             :1:     0/ 1 : Attr references not
                                                          present
    

</pre>
</div>
This Compliance Checker Report can be used to identify where file meta data can be improved.  A strong meta data record allows for greater utility of the data for a broader audience of data analysts.
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2017-02-10-running_compliance_checker.ipynb)
this notebook, or click [here](https://beta.mybinder.org/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2017-02-10-running_compliance_checker.ipynb) to run a live instance of this notebook.