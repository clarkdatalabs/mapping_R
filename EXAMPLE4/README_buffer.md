# EXAMPLE 4: Join data, create choropleth, and create buffer 

Transportation equity has long been a concern in the city of Detroit. How well it is that the bus system in
Detroit is taking people of different income/social stratus into consideration? What can be done to
improve the current situation? This study examines how well the bus transit system has served the
Detroit disadvantaged neighborhood based on walkability criteria.

The examining standard is based on a 5-min walking distance(0.25mile/400m) buffer around both the
neighborhoods, which are defined according to census tract units, and bus stops of DDOT , which contains
most bus routes in Detroit. The study identifies the spatial distribution of disadvantaged neighborhoods
based on poverty level and educational attainment, and showcases their spatial relationship with bus
stops.

In the example, you'll learn how to clip a shapefile according to a certain boundary shapefile,
                             how to check and change the coordinate system of the shapefile,
                             how to join data from a .csv file to a shapefile,
                             how to create choropleth using ggplot,
                             how to create buffer around points and ploygons,
                             etc.
Now let's get started!
 
 ## 1. Install packages
```
install.packages("tidyverse")
```
This is going to get us all the transformation and visualization packages we need, before we start with the spatial-specific ones below.
```
install.packages("rgeos")
```
This is a link b/w R and an open source geography engine
```
install.packages("rgdal") 
```
This links R to a library for transforming shapefiles
```
install.packages("ggmap")
```
ggmap adds on to ggplot2 to make it more useful for mapping
```
install.packages("maptools")
```
This is a set of functions for manipulating spatial data
```
install.packages("sf")
```
This is a revised/faster version of "sp", a package for shapefile transformation and editing.
```
install.packages("raster")
```
This is a package to deal with raster mapping.

## 2. Getting set up

```
library(ggplot2) # for viz!
library(readr) # for reading in files
library(haven) # for reading in DTA files
library(rgeos) # This is how we get to load in shapefiles (the old way)
library(rgdal) 
library(ggmap) # Additional functions for ggplot
library(maptools) # Additional functions for mapping
library(sf) # Reading and transforming shapefiles (the newer way)
library(dplyr) # for merging/transforming the data
library(sp)
library(raster)
 ```

## 3. Map the Detroit disadvantaged neighborhoods
### Trim the shapefile 
The census tract shapefile contains tracts out of the boundary of Detroit city.
We want to trim the shapefile first so that only census tracts within the boundary of Detroit remain.

First, import both shapefiles 
```
Detroit_ct <- st_read("data_Downloads/2010CensusTracts.shp",
                      stringsAsFactors = F)
```
Use the same methond to read the boundary shpfile again so that the two dataframe are of the same class
```
Detroit_b_st <- st_read("data_Downloads/City of Detroit Boundary/geo_export_d1d47202-15e7-4013-8fdf-3b4bad0e2de7.shp",
                        stringsAsFactors = F) 
```
Check the coordinate system of the both shapefiles
```
st_crs(Detroit_b_st)
st_crs(Detroit_ct)
```
Transform them into the same coordinate system
```
Detroit_b_st <- st_transform(Detroit_b_st, 4326)
Detroit_ct <- st_transform(Detroit_ct, 4326)
```
Clip 
```
Detroit_census_st <- st_intersection(Detroit_b_st, Detroit_ct)
```
 Now we've got the shapefile we wanted

### Import the first group data we want to visualize
Import education data
```
Detroithsperc <- read_csv("./data_tidied/S1501_hsperc.csv")
```
Join the data
```
Detroithsperc_full <- left_join(Detroit_census_st, Detroithsperc, by= c("GEOID10" = "GEO.id2"))
```
It shows: Error: Can't join on 'GEO.id2' x 'GEOID10' because of incompatible types (numeric / character)
So try to coerce the data type to character
```
Detroithsperc$GEO.id2 <- as.character(Detroithsperc$GEO.id2)
```
Join again
```
Detroithsperc_full <- left_join(Detroit_census_st, Detroithsperc, by= c("GEOID10" = "GEO.id2"))
```
Plotting
```
Edu1 <- ggplot()+
  geom_sf(data = Detroithsperc_full, aes(fill = as.numeric(HC02_EST_VC17)),  size = 0.25)+
  scale_fill_gradient(name = "Percent High School Graduate or Higher", na.value = "gray65") +
  labs(title = "Detroit Census Tracts Educational Attainment", size = 20)
Edu1
```

### Import the second group data we want to visualize
Import education data
```
Detroitpovperc <- read_csv("./data_tidied/S1701_povperc.csv")
```
Coerce the data type to character
```
Detroitpovperc$GEO.id2 <- as.character(Detroitpovperc$GEO.id2)
```
Join again
```
Detroitpovperc_full <- left_join(Detroit_census_st, Detroitpovperc, by= c("GEOID10" = "GEO.id2"))
```
Plotting
```
Pov1 <- ggplot()+
  geom_sf(data = Detroitpovperc_full, aes(fill = as.numeric(HC03_EST_VC01)),  size = 0.25)+
  scale_fill_gradient(name = "Poverty Percentage", low = "#FFFFCC", high = "#FF6600", na.value = "gray65") +
  labs(title = "Detroit Census Tracts Poverty Level", size = 40)
Pov1
```

### Create a new data frame to identify the disadvantaged neighborhood
Census tracts that have both a percentage poverty level over 34.5%,
and a percentage of high school graduattion less than 81.2%, 
which are the overall average level of Detroit city,
are considered to be the disadvantaged census tracts.

Join both data into the same file
```
disadvan <- left_join(Detroitpovperc_full, Detroithsperc, by= c("GEOID10" = "GEO.id2"))
```
Create indicator to identify disadvantaged census tracts
```
disadvan$indicator <- ifelse(disadvan$HC02_EST_VC17 <81.2, ifelse(disadvan$HC03_EST_VC01 >34.5, 1, 0), 0)
```
Plotting
```
disadvan1 <- ggplot()+
  geom_sf(data = disadvan, aes(fill = as.character(indicator)),  size = 0.25)+
  scale_fill_manual(values = c("#174a7c", "#f0573e"), name = "Categary", labels = c("Comparison","Disadvantaged")) +
  labs(title = "Disadvantaged Census Tracts", size = 40)
disadvan1
```

## 4. Map the busstop distribution
Add bus stop shapefile as an additional layer
```
busstop <- st_read("data_Downloads/DDOT_Bus_Stops_2017/DDOT_Bus_Stops_2017.shp",
                      stringsAsFactors = F)
```
Adjust the coordinate system
```
busstop <- st_transform(busstop, 4326)
```
Clip 
```
busstop_clip <- st_intersection(Detroit_b_st, busstop)
```

## 5. Create different indicators for buffering
Coerce the data type to character
```
disadvan$indicator <- as.character(disadvan$indicator)
```
Create different indicators for comparison/disadvantaged census tracts
```
disadvan$comparison <- ifelse(disadvan$indicator == 0, 0, NA)
disadvan$disadvantaged <- ifelse(disadvan$indicator == 1, 1, NA)
```
Create new data frame to differiate comparison/disadvantaged census tracts
```
geom <- disadvan[disadvan$disadvantaged ==1,]
geom <- geom[order(disadvan$disadvantaged),]
geom[,(NA:NA.201)] <- NULL
```
Create data frame for buffering
bf <- st_buffer(geom, dist = 0.005)
busbf <- st_buffer(busstop_clip, dist = 0.005)

## 6. Mapping in ggplot
Create buffer arounds disadvantaged census tracts
```
buffer1 <- ggplot()+
  geom_sf(data = disadvan, aes(fill = as.character(comparison),),  size = 0.25)+
  geom_sf(data = bf, fill = "#bcbec0",  size = 0.25, alpha = 0.3, colour=NA)+
  geom_sf(data = geom, aes(fill = as.character(disadvantaged)),  size = 0.25)+
  scale_fill_manual(values = c("#174a7c", "#f0573e","#fcb64b"), name = "Categary", labels = c("Comparison","Disadvantaged")) +
  labs(title = "5-min Walking Distance from the Edge of Census Tracts", size = 40)
buffer1
```
```
buffer2 <- ggplot()+
  geom_sf(data = disadvan, aes(fill = as.character(comparison),),  size = 0.25)+
  geom_sf(data = bf, fill = "#bcbec0",  size = 0.25, alpha = 0.3, colour=NA)+
  geom_sf(data = geom, aes(fill = as.character(disadvantaged)),  size = 0.25)+
  geom_sf(data = busstop_clip)+
  scale_fill_manual(values = c("#174a7c", "#f0573e","#fcb64b"), name = "Categary", labels = c("Comparison","Disadvantaged")) +
  labs(title = "DDOT Bus Stops Distribution around Neighborhoods' 5-min Walking Distance", size = 40)
buffer2
```
Create buffer around bus stops
```
buffer3 <- ggplot()+
  geom_sf(data = disadvan, aes(fill = as.character(comparison),),  size = 0.25)+
  geom_sf(data = geom, aes(fill = as.character(disadvantaged)),  size = 0.25)+
  geom_sf(data = busbf, fill = "#bcbec0", size = 0.25, alpha = 0.08, colour=NA )+
  geom_sf(data = busstop_clip)+
  scale_fill_manual(values = c("#174a7c", "#f0573e","#fcb64b"), name = "Categary", labels = c("Comparison","Disadvantaged")) +
  labs(title = "5-min Walking Buffer around DDOT Bus Stops", size = 40)
buffer3
```
                           
