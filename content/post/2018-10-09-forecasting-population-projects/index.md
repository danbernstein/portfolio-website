---
title: 'Forecasting: Population Projections'
author: Dan
date: '2018-10-09'
slug: []
categories: []
tags:
  - R
  - forecasting
  - analysis
description: ''
lastmod: ''
image: ''
author_twitter: ''
---


*This is the second in a series of posts that seek to recreate the methods used in [Future ozone-related acute excess mortality under climate and population change scenarios in China: A modeling study](https://journals.plos.org/plosmedicine/article?id=10.1371/journal.pmed.1002598#sec014). All the posts build on my previous post on [forecasting](https://danbernstein.netlify.com/post/learning-forecasting/), by using data available from national and international research initatives to project future scenarios for complex systems including [atmospheric chemistry](https://danbernstein.netlify.com/post/climate-modeling/), [population dynamics](https://danbernstein.netlify.com/post/population-projections/), and mortality. This post focuses on the second projection: population dynamics.*

The previous post discussed how to project future ozone concentrations using global climate models of historical and future climate scnearios and observational monitoring data. This post will describe methods for projecting future population dynamics. In the next post, the population projections will be related to age-specific mortality rates, which will ultimately be attributed to various causes to identify the number of additional deaths due to projected ozone concentration changes. 

## Methods: Projecting Future Population Dynamics under Various Scenarios

### *Overview of Analysis*

The workflow for this analysis involves recreating the methods used in the PLOS Medicine article.

*"Chinese population size projections between 2010 and 2050 for all ages under the 5 SSPs at 0.125° × 0.125° resolution were extracted from the global projections [44]. City-level population projections were then calculated by summing the populations of each grid cell that fell within a certain city boundary. Chinese population age structure changes between 2010 and 2050 under the 5 SSPs were obtained from the SSP Database (https://tntcat.iiasa.ac.at/SspDb)."*

Shared socioeconomic pathways (SSPs) are agreed-upon assumptions concerning future changes in societal characteristics, such as migration, demographics, etc. The referenced article focuses on urban populations, thus the authors used a collection of SSPs that consider varying levels of fertility, mortality, migration, education, and urbanization.

To break down this excerpt into discrete steps:

#### For city population projections: 
* load the appropriate packages 
* Collect population size projection data for the 5 SSPs at 0.125 resolution 
* Overlay the population projection data with city geographic extent data to sum the projected city population

#### For national population age structure projections:
* Collect age structure data for the 5 SSPs from SSP Database


## Analysis - City Population Projections 

This post is less about analysis and more about finding the appropriate dataset and processing it in an efficient manner. Concerning city population projections, there are five files within the downloaded data (available from the National Center for Atmospheric Research [here](http://www.cgd.ucar.edu/iam/modeling/spatial-population-scenarios.html)) which represent the 5 SSPs. Within each SSP file, there are population projections for urban, rural, and total population; in this situation, we are interested in total population because we will filter the data geospatially. 


```{r file structure}

library(tidyverse)
library(raster) # to load NetCDF (.nc) files
library(spdplyr) # to process spatial* data with the tools in dplyr

# the path to main directory 
data.path <- "/home/dan/data/climate_tro3/raw_data/NCAR_population/NetCDF"

list.files(data.path)
```

Following the file path for one of these SSPs yields more files, there is one file for each ten-year dataset; in this case, ten files for the years 2010-2100. Between the five SSPs and the ten time points within each SSP, there are 50 separate projections to consider and analyze.

```{r, eval=FALSE}

data.path.ssp <- "/home/dan/data/climate_tro3/raw_data/NCAR_population/NetCDF/SSP1_NetCDF/total/NetCDF"

list.files(data.path.ssp)
```

Understanding the file structure and which ones are of interest, I can filter the listed files for only those that are .nc files and contain the total population projection, rather than urban or rural alone. 

```{r, eval=FALSE}
# prepare data frame of files to loop across
files <- list.files(data.path, all.files = T, recursive = T,
                    full.names = T,
                    pattern = ".nc") %>% 
  tibble() %>% 
  set_names(c("file.path")) %>% 
  filter(grepl("total", file.path) == TRUE,
         grepl("aux.xml", file.path) == FALSE)

head(files)
```

I can then prepare a data frame containing the name of each projection (SSP number and year time point) which will soon be populated by the populations contained in the population projection files. 
```{r prepare data frame to fill}
file.names <- str_extract(string = files$file.path, 
                          pattern = "(?<=total/NetCDF/).*(?=.nc)") %>% 
  data_frame("ssp" = ., "pop" = vector(mode = "numeric", length = 50))


head(file.names)
```


I created a function ```getPop()``` which takes in the name of a city and will take the geographic coordinates from a [shapefile](https://geo.nyu.edu/catalog/stanford-yk247bg4748) of world urban areas at 1:10 million scale (available from the [NYU spatial data repository](https://geo.nyu.edu/)). The function loads the urban areas shapefile if it does not already exist, and then filters for the city of interest. This filtered shapefile is then looped through the 50 NetCDF files to extract the sum of the grid cells that fall within the shapefile's geographic coordinates. The final dataframe ("city.return") contains the 50 scenarios and the associated population that was extracted at each time point. 

```{r getPop}
getPop <- function(city){
  require(spdplyr)

  if (!exists("urban_areas", where = -1)) {
    urban_areas <- raster::shapefile("/home/dan/data/climate_tro3/raw_data/NCAR_population/urban_area_global_nyu/ne_10m_urban_areas_landscan")
    } else {
      urban_areas <- get("urban_areas")
      }
  
  loc <- filter(urban_areas, name_conve == city)
  ssp.pop <- file.names
  
  for (i in 1:nrow(files)){
    # read in each raster
    pop <- raster(files$file.path[i])
    
    # retrieve the total population within the geospatial locations of the city
    ssp.pop$pop[i] <- extract(pop, loc, fun=sum, na.rm=TRUE)
    
    # remove the raster of the ssp to preserve memory
    remove(pop)
  }
  
  return(ssp.pop)
}

city.return <- getPop(city = "New York")

print(city.return)
```

We can plot the five population projections in ggplot, with an additional horizontal line that represents no population change. Clearly SSP5 includes large population growth, while SSP3 and SSP4 show the population of New York City peaking at 2050 and 2075, respectively, and then declining. The other scenarios assume a plateau effect. 
```{r population projection ggplot}
city.return %>% 
  separate(ssp, c("ssp", "year"), sep = "_", convert = T) %>% 
  ggplot(., aes(x = year, y = pop, colour = ssp))+
  ylab("population")+
  geom_line()+
  geom_segment(aes(x=min(year),xend=max(year),y=pop[1],yend=pop[1]),
               colour = "red")
```



## Analysis - National Age Structure Projections 

Data for national age structure projections are readily available from the SSP Database. While the website provides a great, user-friendly interface for filtering the data before you download a subset, in this instances, we want to get data for every age bracket for every SSP at once, rather than downloading numerous files. To do so, we navigate to the "Download" tab and download the country-level csv zip file "SspDb_country_data_2013-06-12.csv.zip". 
```{r, eval=FALSE}
agestr <- read_csv("/home/dan/data/climate_tro3/raw_data/NCAR_population/china_agestructure/SspDb_country_data_2013-06-12.csv")

```

With a little data wrangling, I have cleaned up the data including: filtering for the United States ("USA"), cleaning the SSP names, and filtering the rows for only those that pertain to age brackets. 
```{r, eval=FALSE}
agestr.country <- 
  agestr %>% 
  filter(REGION == "USA") %>% 
  # reduce ssp names to SSP1-5 (also remove one instance where the letter "d" is appended)
  mutate(ssp = str_replace_all(SCENARIO, "\\_[a-z,0-9]*|d", "")) %>% 
  # select relevant variables and only years with future population projections
  dplyr::select(REGION, VARIABLE, UNIT, ssp,
    one_of(as.character(seq(2010, 2100, by = 5)))) %>% 
  # filter rows with age bracket projections
  filter(str_detect(VARIABLE, "Aged"),
          !str_detect(VARIABLE, "Education|PPP|Share")) %>% 
  # clean up variable names
  mutate(VARIABLE = str_remove_all(VARIABLE, "Population\\||Aged")) %>% 
  separate(VARIABLE, c("gender", "age_group"), sep = "\\|") %>% 
  gather(key = year, value = population, -c(REGION, gender, age_group, UNIT, ssp))


head(agestr.country)
```

We can visualize the population dynamics for women for every age bracket under all five SSP scenarios using ```facet_grid()```. Similar to the results from the urban population projections, SSP5 shows large population growth in most age brackets, while SSP3 and SSP4 shows a peak and then a decline for most age brackets.

```{r, eval=FALSE}
agestr.country %>% 
  filter(gender == "Female") %>% 
  ggplot(., aes(x = year, y = population, group = age_group, colour = age_group)) +
  theme(axis.text.x = element_blank(),
        legend.title = element_text("Age Group"))+
  labs(title = "Population Change by Age Group Under 5 Shared Socioeconomic Pathways",
       color = "Age Group")+
  geom_line() +
  facet_grid(vars(ssp))
```

