---
title: "Basic Mapping"
output: 
  html_document:
    keep_md: true
self_contained: yes
---

<script src="../../../js/hideoutput.js"></script>



### Goals and Motivation

Now that we've covered vectors, rasters, and geoprocessing in R, it's time to discuss how we can make some nice plots and maps of these data.  The distributed network of R packages has continued to grow over the years and now offers many easy options for mapping.  In addition, base R graphics and those provided by the powerful ggplot2 package can also be used to plot spatial data.  We'll discuss how we can leverage generic plotting functions in R and more specific packages to help us explore spatial data in more detail. 

By the end of this lesson, you should be able to answer these questions:

* How can I use base graphics to create simple vector and raster plots?
* How can I use ggplot2 to create simple vector and raster plots?
* How and when would I use mapview?
* How and when would I use tmap?
* How and when would I use micromap?

### Exercise

Let's first make sure we have an R Markdown file setup and the correct packages loaded.

1. In the project you created for this workshop, open a fresh R Markdown file from the File menu.  Name it "Basic mapping" and save it in your project.

1. Remove all the template content below the YAML header.  Add some descriptive text using Markdown below the YAML header that briefly describes the lesson plan (e.g., "This lessons covers base graphics, ggplot, and other R packags for mapping spatial data.")

1. Below this text, add a code chunk and write some script to load the following packages: `tidyverse`, `maps`, `sf`, `raster`, `tmap`, `micromap`.

1. When you're done, compile the document by pressing the knit button at the top.

<details> 
  <summary>Click here to cheat!</summary>
   <script src="https://gist.github.com/fawda123/e3f1d19471652194e29f519b42b054f8.js"></script>
</details>

### A primer on mapping

Mapping spatial data can be generically described as simple data vizualization.  The same motivation for creating a simple scatterplot can apply to creating a map.  The overall goal is to develop insight into your data by visualizing patterns that cannot be seen in tabular format.  Of course the added complication with spatial data is the inclusion of a location.  How you handle this spatial component is up to you and depends on whether location is relevant for the plot or map.  

Spatial data by definition always include a reference location for each observation.  More often than not, additional variables not related to space may be collected for each observation.  For example, multiple measurements of water quality taken at different locations in a lake can be indexed by latitude, longitude, and whatever water quality data were taken at a sample site.  The only piece of information that makes the water quality data spatial is the location. Whatever your data look like, you have to choose if space is an important variable to consider given your question.  For mapping spatial data, consider the following:

* Do I care about the spatial arrangement of my data?
* Would I expect my non-spatial data to vary by space?  
* Are there other spatial units for aggregating my data that can help understand patterns?

The answer to these questions can help you decide what type of visualization is important for the data.  We'll explore different approaches to mapping spatial in this lesson that will be guided by the answers to these questions.

### Basic maps

Let's start by taking the simplest type of spatial data and creating the simplest type of map.  We'll use the same dataset when we first started the workshop.  


```r
cities <- c('Ashland', 'Corvallis', 'Bend', 'Portland', 'Newport')
longitude <- c(-122.699, -123.275, -121.313, -122.670, -124.054)
latitude <- c(42.189, 44.57, 44.061, 45.523, 44.652)
population <- c(20062, 50297, 61362, 537557, 9603)
dat <- data.frame(cities, longitude, latitude, population)
plot(latitude ~ longitude, data = dat)
```

![](../../../img/citybs1-1.png)<!-- -->

Neat, we've created a map in cartesian space.  As we've learned, we need to convert our data into a spatial object with the correct coordinate system.  This will let us correctly view the georeferenced data.  Let's convert our data frame to an `sf` object.


```r
dat_sf <- st_as_sf(dat, coords = c("longitude", "latitude"))
st_crs(dat_sf) <- "+proj=longlat +ellps=WGS84 +datum=WGS84 +no_defs"
head(dat_sf)
```

```
## Simple feature collection with 5 features and 2 fields
## geometry type:  POINT
## dimension:      XY
## bbox:           xmin: -124.054 ymin: 42.189 xmax: -121.313 ymax: 45.523
## epsg (SRID):    4326
## proj4string:    +proj=longlat +datum=WGS84 +no_defs
##      cities population                geometry
## 1   Ashland      20062 POINT (-122.699 42.189)
## 2 Corvallis      50297  POINT (-123.275 44.57)
## 3      Bend      61362 POINT (-121.313 44.061)
## 4  Portland     537557  POINT (-122.67 45.523)
## 5   Newport       9603 POINT (-124.054 44.652)
```

Now we can just plot the locations of our `sf` object by telling R to only plot the geometry attribute.

```r
plot(st_geometry(dat_sf))
```

![](../../../img/citybs2-1.png)<!-- -->

Although it's a bland plot, the points are now georeferenced.  We can use some base maps from the `maps` package to add some context for the locations (note the `add = T` argument). 

```r
map('county', region = 'oregon')
plot(st_geometry(dat_sf), add = T)
```

![](../../../img/citybs3-1.png)<!-- -->

There really isn't anything too interesting about the locations of these cities, so maybe we want to overlay an additional attribute from our dataset.  Remember that we have the population of each city as well.  We can change the point types and scale the size accordingly.

```r
map('county', region = 'oregon')
plot(st_geometry(dat_sf), add = T, pch = 16, cex = sqrt(dat_sf$population * 0.0002))
```

![](../../../img/citybs4-1.png)<!-- -->

This is kind of tedious.  We can alternatively make the same plot using ggplot2 to map some of the data attributes to the plot aesthetics.  We'll take advantage of the `geom_sf` function to plot the `sf` object.  This will be much easier once we have all of our map objects the correct format for `sf`.  Our cities dataset is already an `sf` object, so all we have to do is convert our base state map to an `sf` object.


```r
state <- st_as_sf(map('county', region = 'oregon', plot = F, fill = T))
```

Now we make the plot with the `geom_sf` function for both `sf` objects.

```r
ggplot() +
  geom_sf(data = state) +
  geom_sf(data = dat_sf)
```

![](../../../img/citygg1-1.png)<!-- -->

We can also map population to size and colour easily and add some text labels.

```r
ggplot(dat_sf) +
  geom_sf(data = state) +
  geom_sf(aes(size = population, colour = population))
```

![](../../../img/citygg2-1.png)<!-- -->

There are a few things we might want to modify with this plot, so let's assign it to a variable in our workspace that we can easily modify.

```r
p <- last_plot()
```

Now let's make the size gradients more noticeable, remove the weird size legend, and change the color ramp.

```r
p <- p +
  scale_size(range = c(5, 15), guide = F) + 
  scale_colour_distiller(type = 'div', palette = 9)
p
```

![](../../../img/citygg3-1.png)<!-- -->

We can also add labels using `geom_text`.  Let's plot the city names over the points.

```r
p + geom_text(aes(x = longitude, y = latitude, label = cities))
```

![](../../../img/citygg4-1.png)<!-- -->

If we want to prevent the labels from overlapping the points, we can use the `geom_text_repel` function from the ggrepel package. You'll have to play around with the values for `point.padding` to get the distances you like. The `segment.alpha` value will also remove the lines joining the points to the labels.

```r
p + geom_text_repel(aes(x = longitude, y = latitude, label = cities), point.padding = 0.5, segment.alpha = 0)
```

![](../../../img/citygg5-1.png)<!-- -->

### Chloropleth maps

<!-- http://strimas.com/r/tidy-sf/ -->

### Mapview

### tmap

### micromap