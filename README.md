# BON-in-a-Box-2.0---Block-2-eBird-data-workflow

## Organize workflow environment

1.  Load packages to use in the script.

```{r include=FALSE, eval= T}

# name libraries
packagesList<-list("rstudioapi", "magrittr", "dplyr", "plyr", "purrr", "raster", "terra", "auk", "sf","tools", "gdalUtilities", "tibble", "reshape2", "vapour", "unmarked")

# load libraries
lapply(packagesList, library, character.only = TRUE)
```

2.  Set working directory.

```{r,  warning=FALSE, message=FALSE, results='hide'}

# name working directory
dirfolder<- "~/Bloque2/draft_ebird_bloque2_06_12_22"

# set working directory
setwd(dirfolder)
```

## Specify the basic parameters for eBird data cleaning

1.  Create a reference to an eBird Basic Dataset.

```{r,  warning=FALSE, message=FALSE, results='hide'}

# Name ebird records database file. This file is available for download as relist at https://ebird.org/data/download
file_data<- "ebd_CO_relFeb-2022.txt"

# Name ebird sampling lists database file. This file is available for download as sampling relist at https://ebird.org/data/download
file_lists<- "ebd_sampling_relFeb-2022.txt"

# Create a reference file for the ebird database
setwd(dirfolder)
ebd <- auk_ebd(file = file_data, file_sampling = file_lists) 
```

2.  Filter the ebird database

```{r,  warning=FALSE, message=FALSE, results='hide'}

#### Define filters

## Define filter by Date
date_filter<- c("2010-01-01", "2020-12-31")

##  Define filter by duration - sampling time
duration_filter<- c(0, 300)

## Define filter by protocol sampling
protocol_filter<- c("Stationary", "Traveling")

## Define filter by sampling effort - distance in km
distance_filter<- c(0,10)

##  Define filter by study area

# set spatial parameters
dir_basemap = "~/Bloque2/draft_ebird_bloque2_06_12_22/Caldas.shp" # set the directory path of shape file
resolution_basemap<- 1000 # set the resolution grid in meters
crs_basemap<- CRS("+init=epsg:3395") # set the projection system. We use EPSG 3395 as global coordinate system

setwd(dirfolder)
info_layer<- st_read(dir_basemap)
extentBase<- info_layer%>% st_bbox() %>% st_as_sfc() %>% st_transform(crs = crs_basemap) %>% st_bbox() %>% extent()
rasterbase<- raster(extentBase,crs = crs_basemap, res= resolution_basemap )


# create base grid 
tname2 = "base_grid.tif"; t_file = writeStart(rasterbase, filename = tname2,  overwrite = T); writeStop(t_file);
gdalUtilities::gdal_rasterize(dir_basemap, tname2, burn =1, at=T)
raster_area = rast(t_file) %>% {terra::mask(setValues(., seq(ncell(.))), .)} 

# create copy extent in WGS84 EPSG 4326 coordinate system This is necessary because the ebrid data is in that projection.
area_4326<-  raster(raster_area) %>% projectRaster(crs = st_crs(4326)$proj4string, method = "ngb")
ebd_bbox<- st_bbox(area_4326)


#### Specify the function with filters
filter_bbox<- ebd %>% auk_bbox(ebd_bbox) %>% auk_protocol(protocol = c("Stationary", "Traveling")) %>%
  auk_distance(distance = c(0,10), distance_units= "km") %>% auk::auk_duration(duration = c(0, 300)) %>% auk_date(date = c("2010-01-01", "2020-12-31"))  %>% auk_complete()
```

```{r,  warning=FALSE, message=FALSE, results='hide'}

# plot covars_raster
plot(raster_area)
```

![raster_area](https://user-images.githubusercontent.com/108538775/218140939-2b5cfe89-74eb-45a7-a612-e63044a9ece3.png)

## Load and organize eBird data file

1.  Set output folder and name files for save results.

```{r,  warning=FALSE, message=FALSE, results='hide'}

## Set folder to save the results
# Define name of output_auk_ebird folder
outputs_auk_ebird<- "outputs_auk_ebird"

# Check and create output_auk_ebird folder
setwd(dirfolder)
if(!dir.exists(outputs_auk_ebird)){dir.create(outputs_auk_ebird)}

# Set output_auk_ebird directory
setwd(dirfolder); setwd(outputs_auk_ebird)

## Set specific subfolder to save results
# Define name of output folder
output_folder<- basename(file_path_sans_ext(dir_basemap)) # In this case we set the name according the study area file name

# set output folder
setwd(dirfolder); setwd(outputs_auk_ebird)
if(!dir.exists(output_folder)){dir.create(output_folder)}; setwd(output_folder)

# Define name to save filter ebird records database file
output_studyarea_ebd<- paste(output_folder, file_data, sep= "_")

# Define name to save filter ebird sampling lists database file
output_studyarea_lists<- paste(output_folder, file_lists, sep= "_")
```

2.  Load filtered ebird data files according the parameter filters.

```{r,  warning=FALSE, message=FALSE, results='hide'}

## Load filtered ebird data files
auk_filter(filter_bbox, file = output_studyarea_ebd, file_sampling = output_studyarea_lists, overwrite = T)
```

## Loading study area spatial covariates

1.  Define folder in which the spatial files of the covariates of interest are saved. The script supports files in format "tif" or "shp".

```{r,  warning=FALSE, message=FALSE, results='hide'}

# Specify de covariates folder
setwd(dirfolder)
folder_covars<- "defaultcovars"

# Get files names of folder covars
dir_covars<- list.files(folder_covars, pattern =  c("\\.tif$", "\\.shp$")) 

```

2.  Load and adjust covariates according study area.

```{r,  warning=FALSE, message=FALSE, results='hide'}

covars_raster<- lapply(dir_covars, function(x) {setwd(dirfolder); setwd(folder_covars); print(x)
  tname2<- tempfile(fileext = '.tif')
  t_file<- writeStart(rasterbase, filename = tname2,  overwrite=T); writeStop(t_file);
  
  gdalUtilities::gdalwarp(srcfile = x, dstfile= tname2, 
                          tr= res(rasterbase), te = st_bbox(extent(rasterbase)),
                          overwrite=TRUE, r= "near")
  rast(tname2) %>% setNames(gsub(".tif", "",x))
}) %>% setNames(sapply(., function(x) gsub(" ", "_", names(x)) ))
```

```{r,  warning=FALSE, message=FALSE, results='hide'}

# plot covars_raster
plot(rast(covars_raster))
```


![covers_raster](https://user-images.githubusercontent.com/108538775/218141145-ed7a9564-f9fa-4cc3-a793-715aa4bf50b0.png)

3.  Organize covariates table.

```{r,  warning=FALSE, message=FALSE, results='hide'}

# Create table with cells of study area
covars_data<-  data.frame(Pixel= cells(raster_area))

# Add covariates values to table
for(i in names(covars_raster)){
  covars_data[,i]<-covars_raster[[i]][covars_data$Pixel]
}
```

## **Estimation of species occupancy models**

1.  Specify the interest species to estimate the model

```{r,  warning=FALSE, message=FALSE, results='hide'}

sp<- "Zonotrichia capensis"
```

2.  Filter species database by interest species

```{r,  warning=FALSE, message=FALSE, results='hide'}

ebird_data <- auk_zerofill(output_studyarea_ebd, output_studyarea_lists, species = sp , collapse = T, complete= F) %>%
  mutate(observation_count= ifelse(observation_count == "X", 1, observation_count)) %>% 
  mutate(observation_count= as.numeric(observation_count))
```

3.  Filter species database by study area

```{r,  warning=FALSE, message=FALSE, results='hide'}

# Spatialize the data
ebird_spatial<- ebird_data %>%  st_as_sf(coords = c("longitude", "latitude"), crs = 4326)

# Filter by mask of study area
ebird_spatial_mask<- as.data.frame(ebird_spatial) %>% mutate(Pixel= raster::extract(area_4326, ebird_spatial) ) %>% 
  dplyr::filter(!is.na(Pixel))  %>%  st_as_sf() %>% st_transform(crs_basemap)

ebird_data_mask<- st_drop_geometry(ebird_spatial_mask)

```

```{r,  warning=FALSE, message=FALSE, results='hide'}

plot(raster_area)
plot(ebird_spatial_mask[, "geometry"], add= T)

```
![ebird_spatial_mask](https://user-images.githubusercontent.com/108538775/218140400-bfc0d972-e3ae-4c51-940f-599125be2fb5.png)


4.  Organize matrix by pixels and covariates

```{r,  warning=FALSE, message=FALSE, results='hide'}

# Join species database with covariates data
summ_sp<- ebird_data_mask %>% mutate(month= format(observation_date,'%Y') ) %>%
  group_by(Pixel, month) %>% dplyr::summarise(occurrence= ifelse(sum(observation_count)>0,1,0)  ) %>% 
  data.frame(xyFromCell(raster_area, .$Pixel)) %>% list(covars_data) %>% join_all()

# Organize data as matrix as pixels as rows and covariates as columns
matriz_input<- reshape2::acast(summ_sp, Pixel~month, value.var = "occurrence", fill=0)

matriz_input_covars<- list(rownames_to_column(as.data.frame(matriz_input), "Pixel"), dplyr::select(summ_sp, c("Pixel", names(covars_raster))  ) ) %>% 
  join_all() %>% dplyr::select(-"Pixel")
```

5.  Explore detection probabilities

```{r,  warning=FALSE, message=FALSE, results='hide'}

# Organize data for the single season occupancy models
Data.1 <- unmarkedFrameOccu(y = dplyr::select(matriz_input_covars, -names(covars_raster)), siteCovs = dplyr::select(matriz_input_covars, names(covars_raster)), obsCovs = NULL)  

DataMod <- occu(~1 ~ 1, Data.1)

# Original scale of data
ests <- plogis(coef(DataMod))

# Get results at normal scale with standard error
psiSE <- backTransform(DataMod, type="state")
pSE <- backTransform(DataMod, type="det")

# Get confidence intervals
ciPsi <- confint(psiSE)
ciP <- confint(pSE)

# Organize results
resultsTable <- rbind(psi = c(ests[1], ciPsi), p = c(ests[2], ciP))
colnames(resultsTable) <- c("Estimate", "lowerCI", "upperCI")
```

```{r,  warning=FALSE, message=FALSE, results='hide'}

# plot resutls
plot(1:2, resultsTable[,"Estimate"], xlim=c(0.5, 2.5), ylim=0:1, 
     col=c("blue", "darkgreen"), pch=16, cex=2, cex.lab=1.5,
     xaxt="n", ann=F)
axis(1, 1:2, labels=c(expression(psi), expression(italic(p))), cex.axis=1.5)
arrows(1:2, resultsTable[,"lowerCI"], 1:2, resultsTable[,"upperCI"], 
       angle=90, length=0.1, code=3, col=c("blue", "darkgreen"), lwd=2)
```

![resultsTable](https://user-images.githubusercontent.com/108538775/218141503-90b6a25c-38f7-4576-9437-0e662717c05d.png)


6.  Estimate occupancy model according (Mackenzy et al 2002)

```{r,  warning=FALSE, message=FALSE, results='hide'}

# Estimate occupancy
m1<-occu(~1 ~altitud,Data.1)

# create table of variables attribute to test
newData2 <- data.frame(altitud=  sort(unique(covars_data$altitud)))
E.psi <- predict(m1, type="state", newdata=newData2, appendData=TRUE)

```

```{r,  warning=FALSE, message=FALSE, results='hide'}

# plot resutls
plot(Predicted ~ altitud, E.psi, type="l", col="blue",
     xlab="Altitud",
     ylab="Probabilidad de ocupacion esperada")
```
![predicted](https://user-images.githubusercontent.com/108538775/218140160-2c27db53-3559-431e-b9dd-9c7c199269b2.png)


```{r,  warning=FALSE, message=FALSE, results='hide'}

# create data for map predict
match_pixels<- list(covars_data, E.psi) %>% join_all()
occupancy_model_raster<- raster_area
occupancy_model_raster[match_pixels$Pixel]<- match_pixels$Predicted
```

```{r,  warning=FALSE, message=FALSE, results='hide'}

# plot resutls
plot(occupancy_model_raster)
```
![occupancy_model_raster](https://user-images.githubusercontent.com/108538775/218140010-d74f9c29-bf1f-46c3-8355-945b77b09e07.png)
