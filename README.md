# geoAutorift
ISCE module for finding pixel displacement and motion velocity between two images over user-defined geographic-coordinate grid

This module comprises two parts or sub-modules: "geogrid" and "autorift".

Copyright (C) 2019 California Institute of Technology.  Government Sponsorship Acknowledged.

Citation: https://github.com/leiyangleon/geoAutorift

## 1. Authors

### 1.1 geogrid
Piyush Agram (JPL/Caltech; piyush.agram@jpl.nasa.gov), Yang Lei (GPS/Caltech; ylei@caltech.edu)

### 1.2 autorift

Alex Gardner (JPL/Caltech; alex.s.gardner@jpl.nasa.gov) conceived the algorithm and developed the first version in MATLAB;
Yang Lei (GPS/Caltech; ylei@caltech.edu) translated it to Python, further optimized and incoporated to ISCE.
       
       
## 2. Features


<img src="figures/optical1.png" width="50%">

***Test area and dataset: optical image over the Jakobshavn glacier where the red rectangle marks boundary of the Sentinel-1A/B image pair (20170221-20170227). Input files in this test scenario consist of the Digital Elevation Model (DEM), local surface slope maps (in both x- and y-direction) and coarse motion velocity maps (in both x- and y-direction) over the entire Greenland, where all maps share the same geographic-coordinate grid with 240-m spacing and spatial reference system with EPSG code 3413 (a.k.a WGS 84 / NSIDC Sea Ice Polar Stereographic North).***





### 2.1 geogrid
* user can define a grid in geographic coordinates provided in the form of a DEM with arbitrary EPSG code, 
* the program will extract the portion of the grid that overlaps with the given coregistered radar image pair, 
* return the range and azimuth pixel indices in the radar image pair for each grid point
* return the range and azimuth coarse displacement given the motion velocity maps and the local surface slope maps in the direction of both geographic x- and y-coordinates (they must be provided at the same grid as the DEM)
* return the matrix of conversion coefficients that can convert the range and azimuth displacement between the two radar images (precisely estimated later with the sub-module "autorift") to motion velocity in geographic x- and y-coordinates
* all outputs are in the format of GeoTIFF with the same EPSG code as input

<img src="figures/geogrid.png" width="100%">

***Output of "geogrid" sub-module: (a) range pixel index at each grid point, (b) azimuth pixel index at each grid point, (c) range coarse displacement at each grid point, (d) azimuth coarse displacement at each grid point. Note: only the portion of the grid overlapping with the radar image has been extracted and shown.***


### 2.2 autorift

* fast algorithm that finds displacement between the two images using sparse search and iteratively progressive chip sizes
* faster than the conventional ampcor algorithm in ISCE by almost an order of magnitude
* support various preprocessing modes on the given image pair, e.g. either the raw image (both texture and topography) or the texture only (high-frequency components without the topography) can be used with various choices of the high-pass filter options 
* support data format of either unsigned integer 8 (uint8; faster) or single-precision float (float32)
* user can adjust all of the relevant parameters, e.g. search limit, chip size range, etc
* a Normalized Displacement Coherence (NDC) filter has been developed to filter image chip displacemnt results based on displacement difference thresholds that are scaled to the search limit
* sparse search is used to first eliminate the unreliable chip displacement results that will not be further used for fine search or following chip size iterations
* novel nested grid design that allows chip size to progress
* another chip size that progresses iteratively is used to determine the chip displacement results that have not been estimated from the previous iterations
* a light interpolation is done to fill the missing (unreliable) chip displacement results using bicubic mode (that can remove pixel discrepancy when using other modes) and an interpolation mask is returned
* the core image processing is coded by calling OpenCV's Python and/or C++ functions for efficiency 
* sub-pixel displacement estimation using the pyramid upsampling algorithm
* this sub-module is not only suitable for radar images, but also for optical, etc
* all outputs are in the format of GeoTIFF with the same EPSG code as input


<img src="figures/autorift1.png" width="100%">

***Output of "autorift" sub-module: (a) radar-estimated range pixel displacement, (b) radar-estimated azimuth coarse displacement, (c) light interpolation mask, (b) chip size (x-direction) used. Notes: all maps are established exactly over the same geographic-coordinate grid from input.***



***Runtime comparison for this test:***
* ___geoAutorift: 10 mins___
* ___dense ampcor from ISCE: 90 mins___



<img src="figures/autorift2.png" width="100%">

***Final motion velocity results by combining outputs from "geogrid" and "autorift" sub-modules: (a) estimated motion velocity from Sentinel-1 data (x-direction; in m/yr), (b) coarse motion velocity from input data (x-direction; in m/yr), (c) estimated motion velocity from Sentinel-1 data (y-direction; in m/yr), (b) coarse motion velocity from input data (y-direction; in m/yr). Notes: all maps are established exactly over the same geographic-coordinate grid from input.***


## 3. Install

* First install ISCE
* Put the "geoAutorift" folder and the "Sconscript" file under the "contrib" folder that is one level down ISCE's source directory (denoted as "isce-version"; where you started installing ISCE), i.e. "isce-version/contrib/" (see the snapshot below)

<img src="figures/install_snapshot.png" width="35%">

* run "scons install" again from ISCE's source directory "isce-version" using command line


## 4. Instructions

### 4.1 geogrid

* It is recommended to run ISCE up to the step where coregistered SLC's are done, e.g. "mergebursts" for using topsApp.

For quick use:
* Refer to the file "testGeogrid.py" for the usage of the sub-module and modify it for your own purpose
* Input files include the master image folder (required), slave image folder (required), a DEM (required), local surface slope maps, velocity maps

For modular use:
* In Python environment, type the following to import the "geogrid" sub-module and initialize the "geogrid" object

       import isce
       from components.contrib.geoAutorift.geogrid.Geogrid import Geogrid
       obj = Geogrid()
       obj.configure()

* The "geogrid" object has several parameters that have to be set up (listed below; can also be obtained by referring to "testGeogrid.py"): 

       ------------------radar parameters------------------
       startingRange:       starting range
       rangePixelSize:      range pixel size
       sensingStart:        starting azimuth time
       prf:                 pulse repition frequency 
       lookSide:            look side, e.g. -1 for right looking 
       repeatTime:          time period between the acquisition of the two radar images
       numberOfLines:       number of lines (in azimuth)
       numberOfSamples:     number of samples (in range)
       orbit:               ISCE orbit data structure
       ------------------input file names------------------
       demname:             (input) name of the DEM file
       dhdxname:            (input; not required) name of the local surface slope in x-coodinate file
       dhdyname:            (input; not required) name of the local surface slope in y-coodinate file
       vxname:              (input; not required) name of the motion velocity in x-coodinate file
       vyname:              (input; not required) name of the motion velocity in y-coodinate file
       ------------------output file names------------------
       winlocname:          (output) name of the range and azimuth pixel indices (at each grid point) file
       winoffname:          (output) name of the range and azimuth coarse displacement (at each grid point) file
       winro2vxname:        (output) name of the conversion coefficients from radar displacement (range and azimuth) to motion velocity in x-coordinate (at each grid point) file
       winro2vyname:        (output) name of the conversion coefficients from radar displacement (range and azimuth) to motion velocity in y-coordinate (at each grid point) file

* After the above parameters are set, run the sub-module as below to create the output files

       obj.geogrid()


### 4.2 autorift

* It is recommended to run the "geogrid" sub-module first before running "autorift". In other words, the outputs from "testGeogrid.py" (a.k.a winlocname, winoffname, winro2vxname, winro2vyname) will serve as the inputs for running "autorift".

For quick use:
* Refer to the file "testAutorift.py" for the usage of the sub-module and modify it for your own purpose
* Input files include the master image (required), slave image (required), and the four outputs from running "testGeogrid.py"

For modular use:
* In Python environment, type the following to import the "autorift" sub-module and initialize the "autorift" object

       import isce
       from components.contrib.geoAutorift.autorift.Autorift import Autorift
       obj = Autorift()
       obj.configure()

* The "autorift" object has several inputs that have to be assigned (listed below; can also be obtained by referring to "testAutorift.py"): 
       
       ------------------input------------------
       I1:                  reference image
       I2:                  test image (displacement = motion vector of I2 relative to I1)
       xGrid:               range pixel index at each grid point
       yGrid:               azimuth pixel index at each grid point
       (if xGrid and yGrid not provided, a regular grid spanning the entire image will be automatically set up, which is similar to the conventional ISCE module, ampcor)
       Dx0:                 range coarse displacement at each grid point
       Dy0:                 azimuth coarse displacement at each grid point
       (if Dx0 and Dy0 not provided, an array with zero values will be automatically assigned for them)

* After the inputs are specified, run the sub-module as below
       
       obj.preprocess_filt_XXX() or obj.preprocess_db()
       obj.uniform_data_type()
       obj.runAutorift()

where "XXX" can be "wal" for the Wallis filter, "hps" for the trivial high-pass filter, "sob" for the Sobel filter, "lap" for the Laplacian filter, and also a logarithmic operator without filtering is adopted for occasions where low-frequency components (i.e. topography) are desired, i.e. "obj.preprocess_db()".

* The "autorift" object has the following four outputs: 
       
       ------------------output------------------
       Dx:                  estimated range displacement
       Dy:                  estimated azimuth displacement
       InterpMask:          light interpolation mask
       ChipSizeX:           iteratively progressive chip size used (range-direction; different chip sizes allowed for range and azimuth)

* The "autorift" object has many parameters that can be flexibly tweaked by the users for their own purpose (listed below; can also be obtained by referring to "geoAutorift/autorift/Autorift.py"):

       ------------------parameter list: general function------------------
       ChipSizeMinX:               Minimum size (in X direction) of the image template (chip) to correlate (default = 32; could be scalar or array with same dimension as xGrid)
       ChipSizeMaxX:               Maximum size (in X direction) of the image template (chip) to correlate (default = 64; could be scalar or array with same dimension as xGrid)
       ChipSize0X:                 Minimum acceptable size (in X direction) of the image template (chip) to correlate (default = 32)
       ScaleChipSizeY              Scaling factor to get the Y-directed chip size in reference to the X-directed sizes (default = 1)
       SearchLimitX                Range (in X direction) to search for displacement in the image (default = 25; could be scalar or array with same dimension as xGrid; when provided in array, set its elements to 0 if excluded for finding displacement)
       SearchLimitY                Range (in Y direction) to search for displacement in the image (default = 25; could be scalar or array with same dimension as xGrid; when provided in array, set its elements to 0 if excluded for finding displacement)
       SkipSampleX                 Number of samples to skip between windows in X (range) direction for automatically creating the grid that is not specified by the user (default = 32)
       SkipSampleY                 Number of lines to skip between windows in Y ( "-" azimuth) direction for automatically creating the grid that is not specified by the user (default = 32)
       minSearch                   Minimum search limit (default = 6)
       
       ------------------parameter list: about Normalized Displacement Coherence (NDC) filter ------------------
       FracValid                   Fraction of valid displacements (default = 8/25) to be multiplied by filter window size "FiltWidth^2" and then used for thresholding the number of chip displacements that have passed the "FracSearch" disparity filtering
       FracSearch                  Fraction of search limit used as threshold for disparity filtering of the chip displacement difference that is normalized by the search limit (default = 0.25)
       FiltWidth                   Disparity filter width (default = 5)
       Iter                        Number of iterations (default = 3)
       MadScalar                   Scalar to be multiplied by Mad used as threshold for disparity filtering of the chip displacement deviation from the median (default = 4)
       
       ------------------parameter list: miscellaneous------------------
       WallisFilterWidth:          Width of the filter to be used for the preprocessing (default = 21)
       fillFiltWidth               Light interpolation filling filter width (default = 3)
       sparseSearchSampleRate      Sparse search sample rate (default = 4)
       BuffDistanceC               Buffer coarse correlation mask by this many pixels for use as fine search mask (default = 8)
       CoarseCorCutoff             Coarse correlation search cutoff (default = 0.01)
       OverSampleRatio             Factor for pyramid upsampling for sub-pixel level offset refinement (default = 16)
       DataTypeInput               Image data type: 0 -> uint8, 1 -> float32 (default = 0)
       zeroMask                    Force the margin (no data) to zeros which is useful for Wallis-filter-preprocessed images (default = None; 1 for Wallis filter)
