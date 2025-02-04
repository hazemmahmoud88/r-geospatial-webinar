---
title: "Introduction to Geospatial Analysis in R"
subtitle: "Supplemental Tutorial"
author: 'Presented by the ORNL DAAC  https://daac.ornl.gov'
output:
  html_document:
    keep_md: yes
    number_sections: yes
    toc: yes
  html_notebook:
    number_sections: yes
    toc: yes
editor_options:
  chunk_output_type: inline
---
***

<!--------------------------- LOAD THIS FILE INTO RSTUDIO --------------------------->

# Install and Load Packages  

As with the main portion of the tutorial, the raster and rgdal packages must be installed along with their dependencies.


```r
install.packages("raster", dependencies = TRUE)  
install.packages("rgdal", dependencies = TRUE)  
```

Load the raster and rgdal packages and set options.


```r
library(raster)  # provides most functions
  rasterOptions(progress = "text")  # show the progress of running commands
  rasterOptions(maxmemory = 1e+09)  # increase memory allowance
  rasterOptions(tmpdir = "temp_files")  # folder for temporary storage of large objects
library(rgdal)  # provides writeOGR function
```

# Load Data  

> *Functions featured in this section:*  
> **readOGR {rgdal}**  
> reads an OGR data source and loads it as a SpatialPolygonsDataFrame object  

We must load two objects, *threeStates* and *prjFireInsect*, that were created from manipulations described in the main part of the tutorial. To load a shapefile, use the `readOGR()` function. The third object, *focalFireInsect*, will be created in this supplemental tutorial, but it takes abount an hour to run the code; thus, it is provided here for quicker demonstration.


```r
threeStates <- readOGR("./data/threeStates.shp")
prjFireInsect <- raster("./data/prjFireInsect.tif")
focalFireInsect <- raster("./data/focalFireInsect.tif")  # new object that will be derived in this supplemental tutorial
```

# Perform a Focal Analysis  

> *Functions featured in this section:*  
> **focal {raster}**  
> calculates focal ("moving window") values for the neighborhood of focal cells around a target cell using a matrix  

We begin with a demonstration of how the `focal()` function behaves using a much smaller RasterLayer object than the one used in the main part of the tutorial.

Here, we create a Matrix object to demonstrate a "grid" similar to a RasterLayer object. Like *prjFireInsect*, the Matrix object is made up of cell values (ones code for insect damage and twos code for fire damage) and NAs (non-values). Unlike *prjFireInsect*, it has only eight rows and eight columns.


```r
demo_mat <- matrix(data = c(1, NA, NA, NA, NA, NA, NA, 1,
                            1, NA, NA, NA, 1, NA, 2, 1,
                            NA, 2, 1, NA, 1, NA, 1, NA,
                            NA, 1, NA, NA, NA, 2, NA, 1,
                            2, 2, NA, 1, 2, NA, NA, 1,
                            NA, NA, NA, 1, NA, 2, NA, NA,
                            1, NA, 1, 1, 2, NA, NA, NA,
                            1, NA, NA, NA, NA, NA, NA, 1), 
                   ncol = 8, byrow = TRUE)
print(demo_mat)
```

```
##      [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8]
## [1,]    1   NA   NA   NA   NA   NA   NA    1
## [2,]    1   NA   NA   NA    1   NA    2    1
## [3,]   NA    2    1   NA    1   NA    1   NA
## [4,]   NA    1   NA   NA   NA    2   NA    1
## [5,]    2    2   NA    1    2   NA   NA    1
## [6,]   NA   NA   NA    1   NA    2   NA   NA
## [7,]    1   NA    1    1    2   NA   NA   NA
## [8,]    1   NA   NA   NA   NA   NA   NA    1
```

Printing the Matrix object shows how it resembles an eight by eight grid.

Next, we will save the Matrix object as a RasterLayer object. Using `print()`, we can see we created a RasteLayer object with no CRS.


```r
demo_rast <- raster(demo_mat)
print(demo_rast)
```

```
## class      : RasterLayer 
## dimensions : 8, 8, 64  (nrow, ncol, ncell)
## resolution : 0.125, 0.125  (x, y)
## extent     : 0, 1, 0, 1  (xmin, xmax, ymin, ymax)
## crs        : NA 
## source     : memory
## names      : layer 
## values     : 1, 2  (min, max)
```

The `focal()` function is very useful for raster analyses because it allows one to change the value of a cell based upon the values of nearby cells. It is often used in place of "distance-based" calculations that are common for vector-type objects, like shapefiles.

We will first define a function that will tell R how we would like `focal()` to behave. In the code, we tell R that for each focal cell of our RasterLayer object (which is every non-NA cell because "na.rm = TRUE"), we want to know the maximum value of the "neighborhood" around that focal cell.


```r
demo_fun <- function(x) { max(x, na.rm = TRUE) }
```

To use `focal()`, we also must define what the "neighborhood" is for the focal value; that is, how many surrounding cells do we consider a "neighbor" of the focal cell. We will use a neighborhood of eight cells so that every cell adjacent to the focal cell is considered a neighbor. You can see this neighborhood represented in the code following "w = " below. We use ones to represent all the neighboring cells because we want all cells weighted equally, and we also consider the focal cell in our analysis. "pad = TRUE" and "padValue = 0" are used because we want to consider all cells in *demo_rast*, even the "edge" cells that do not have a complete neighborhood of cells. In the case of edges, the "missing" neighbors will be considered zeros. Notice that we use *demo_fun* as an argument in the `focal()` function.


```r
demo_foc <- focal(demo_rast, 
                  w = matrix(c(1, 1, 1, 
                               1, 1, 1, 
                               1, 1, 1), ncol = 3), 
                  fun = demo_fun, pad = TRUE, padValue = 0)
print(as.matrix(demo_foc))
```

```
##      [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8]
## [1,]    1    1    0    1    1    2    2    2
## [2,]    2    2    2    1    1    2    2    2
## [3,]    2    2    2    1    2    2    2    2
## [4,]    2    2    2    2    2    2    2    1
## [5,]    2    2    2    2    2    2    2    1
## [6,]    2    2    2    2    2    2    2    1
## [7,]    1    1    1    2    2    2    2    1
## [8,]    1    1    1    2    2    2    1    1
```

Compare the resultant output with *demo_mat*. If a focal cell was one, but adjacent to a two, the focal cell became a two. A focal cell that was a two remains a two because two is the maximum value of the RasterLayer object.

Now we combine *demo_rast* and *demo_foc* using raster algebra (i.e., adding the two RasterLayer objects).


```r
demo_temp <- demo_rast + demo_foc
print(as.matrix(demo_temp))
```

```
##      [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8]
## [1,]    2   NA   NA   NA   NA   NA   NA    3
## [2,]    3   NA   NA   NA    2   NA    4    3
## [3,]   NA    4    3   NA    3   NA    3   NA
## [4,]   NA    3   NA   NA   NA    4   NA    2
## [5,]    4    4   NA    3    4   NA   NA    2
## [6,]   NA   NA   NA    3   NA    4   NA   NA
## [7,]    2   NA    2    3    4   NA   NA   NA
## [8,]    2   NA   NA   NA   NA   NA   NA    2
```

Every cell that was a one in *demo_rast* but changed to a two in *demo_foc* is now a three (e.g., *demo_rast* = 1 and *demo_foc* = 2; therefore, *demo_rast* + *demo_foc* = 1 + 2 = 3). A cell that was a two in *demo_rast* and stayed a two in *demo_foc* is now a four.

We will use the `calc()` function to reclassify the values of *demo_temp* to get the final values we'd like to see represented. In the code below, we tell R that if a value is three, change it to a one, and every other value becomes NA.


```r
demo_tar <- calc(demo_temp, 
                 fun = function(x) {
                       ifelse( x == 3, 1, NA) } )
print(as.matrix(demo_tar))
```

```
##      [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8]
## [1,]   NA   NA   NA   NA   NA   NA   NA    1
## [2,]    1   NA   NA   NA   NA   NA   NA    1
## [3,]   NA   NA    1   NA    1   NA    1   NA
## [4,]   NA    1   NA   NA   NA   NA   NA   NA
## [5,]   NA   NA   NA    1   NA   NA   NA   NA
## [6,]   NA   NA   NA    1   NA   NA   NA   NA
## [7,]   NA   NA   NA    1   NA   NA   NA   NA
## [8,]   NA   NA   NA   NA   NA   NA   NA   NA
```

Our final product is a grid of NAs and ones that were adjacent to a two in *demo_rast*. Now we will apply this same logic to the "real" data.

Using *prjFireInsect* we will determine which forest locations were damaged by insects and were adjacent to forest locations damaged by fire. In other words, we will locate cells with the value one that are next to cells with the value two. As explained in the main part of the tutorial, no cells that have values overlap between the original *fire* and *insect* RasterLayer objects, which is why we will use `focal()` here.

The code below is the same as demonstrated above, but uses a much larger RasterLayer object. We loaded *focalFireInsect* earlier so that it would not be necessary to run the `focal()` function, but we can still examine the results.


```r
# this will take about an hour to run
funt <- function(x) { max(x, na.rm = TRUE) }

if(file.exists("./data/focalFireInsect.Rds")) { 
  focalFireInsect <- readRDS("./data/focalFireInsect.rds") 
  print(focalFireInsect)
  }else{ 
    focalFireInsect <- focal(prjFireInsect, 
                             w = matrix(c(1, 1, 1,
                                          1, 1, 1,
                                          1, 1, 1), ncol = 3), 
                             fun = funt, pad = TRUE, padValue = 0)
    print(focalFireInsect)
  }
```

```
## class      : RasterLayer 
## dimensions : 12086, 12796, 154652456  (nrow, ncol, ncell)
## resolution : 0.00126, 0.000888  (x, y)
## extent     : -119.2667, -103.1437, 39.60561, 50.33798  (xmin, xmax, ymin, ymax)
## crs        : +proj=longlat +datum=WGS84 +no_defs 
## source     : r_tmp_2021-12-03_091211_13400_20419.grd 
## names      : layer 
## values     : 0, 2  (min, max)
```


```r
# this will take a minute to run
temp <- focalFireInsect + prjFireInsect
tarFireInsect <- calc(temp, 
                      fun = function(x) { 
                            ifelse(x == 3, 1, NA) } )
```


```r
print(tarFireInsect)
```

```
## class      : RasterLayer 
## dimensions : 12086, 12796, 154652456  (nrow, ncol, ncell)
## resolution : 0.00126, 0.000888  (x, y)
## extent     : -119.2667, -103.1437, 39.60561, 50.33798  (xmin, xmax, ymin, ymax)
## crs        : +proj=longlat +datum=WGS84 +no_defs 
## source     : r_tmp_2021-12-03_113324_13224_81808.grd 
## names      : layer 
## values     : 1, 1  (min, max)
```

Notice that the *tarFireInsect* has only the value one and NAs, just like in our demonstration.

# Get Cell Coordinates  

> *Functions featured in this section:*  
> **xyFromCell {raster}**  
> gets coordinates of the center of raster cells for a RasterLayer object  

In this section, we will get the actual geographic coordinates of the cells that we identified in the last section. To do this we create a vector that stores the index of *tarFireInsect* cells that are one. That is, we store the "locations" of the values across the extent of the RasterLayer object, as if counting from one to the total number of cells across the "grid".


```r
val_tar <- which(tarFireInsect[]==1)
head(val_tar)
```

```
## [1] 22663564 22999067 23037512 23254953 23306132 23318928
```

We use the function `head()` to view the first six values of *val_tar*. The output tells us that the cell at position 22,663,564 of the "grid" has the value one.

Using the cell "locations" provided by *val_tar* we can extract the coordinates of those cells according to the CRS of *tarFireInsect*. *loc_tarFireInsect* is a Matrix object, and we name its columns.


```r
loc_tarFireInsect <- xyFromCell(tarFireInsect, val_tar, spatial = FALSE)
colnames(loc_tarFireInsect) <- c("lon","lat")
head(loc_tarFireInsect)
```

```
##            lon      lat
## [1,] -116.9388 48.76489
## [2,] -113.4020 48.74180
## [3,] -113.3302 48.73914
## [4,] -113.4449 48.72404
## [5,] -113.4512 48.72049
## [6,] -113.4512 48.71960
```

The first six rows of *loc_traFireInsect* shows the longitude and latitude of cells that have the value one. In other words, we now know the exact locations of forested areas throughtout Idaho, Montana, and Wyoming that had insect damage AND were adjacent to a location that had fire damage.

We end by saving the coordiantes in \*.csv format. If we wanted to, we could visit the locations in person! Try looking up one of these locations using a web-based map, like Google Maps.


```r
write.csv(loc_tarFireInsect, file = "loc_tarFireInsect.csv", row.names = FALSE, overwrite = TRUE)
```

***

<!--------------------------------- END OF TUTORIAL --------------------------------->
