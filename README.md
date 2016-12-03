# how-to-geocode-and-vis-your-data-with-cartodb
post from www.ramiroaznar.com website

In the last post we geocode our data with R. But the geocoding process did not work enterely right. Some points were badly geolocated. In this post, I will explain how to subset these outliers, correct their coordenates and finally, visualize the entire dataset both using R and CartoDB.

As always, set your working directory and load your data. You can download the fires_lonlat csv dataset here. Also I have upload a GitHub repository for this tutorial.

## Setting your working directory & loading data

```{r}
setwd("D:/data/")
fires_lonlat <- read.csv("fires_lonlat.csv", header = TRUE, sep = ",", stringsAsFactors=FALSE)
fires_lonlat$X <- NULL #Get rid of X column (we have already a id variable)
```

Now it is turn to subset those outliers. You can do it easily using the select tool with a GIS desktop software. But here I will take a longer but more interesting methodology. In fact, you will learn along this path a few useful tips, functions and applications for GIS programming. First, we have to figure out what the extreme geographic points are. Wikipedia can help us with this. Secondly, we have to identify those datapoints outside these extreme points. Finally, we must subset them from the original dataset.

## Setting extreme points of Spain, converting coordinates

```{r}
library("sp")
n <- as.numeric(char2dms("43d47'38\"N"))
s <- as.numeric(char2dms("36d00'38\"N"))
o <- as.numeric(char2dms("9d18'05\"O"))*(-1)
e <- as.numeric(char2dms("3d19'55\"E"))
```

It is worth to point out the utility of the char2dms() function from the sp package. It allows us to translate a string into a degrees minutes seconds format.

## Subsetting outliers

```{r}
outliersN <- subset(fires_lonlat, fires_lonlat$lat > n)
outliersS <- subset(fires_lonlat, fires_lonlat$lat < s)
outliersO <- subset(fires_lonlat, fires_lonlat$lon < o)
outliersE <- subset(fires_lonlat, fires_lonlat$lon > e)

outliersNSOE <- rbind(outliersN, outliersS, outliersO, outliersE) #Merge datasets
outliers <- outliersNSOE[order(outliersNSOE$id),] #Order dataset
outliersUnique <- unique(outliers) #Remove repeated columns
write.csv(outliersUnique, "outliersUnique.csv") #Save dataset
```

## Saving outliers without coordinates

```{r}
outliersNew <- outliersUnique #Rename dataset
outliersNew$lon <- NULL #Remove wrong coordinates
outliersNew$lat <- NULL
outliersNew$country <- "SPAIN" #Add a country column to help CartoDB's georeferencing tool
write.csv(outliersNew, "outliersNew.csv") #Save dataset
```

We have our outliers dataset. Now it is time to correct these rebel datapoints. Normally, geocoding algoriths fail to get the right coordinates because there more than one place in The Earth with the same name. It looks like this our case. In order to avoid this, we will try CartoDB's georeferencing tool. First, load and connect the outliersNew csv dataset. Then click on "the_geom" column to launch the georeferencing application and select the "City Names" option. And choose "town_name", "region" and "country" in those three tabs as shown in the following image.

![cartodb georef](cartodb_georef.jpeg)

Sadly CartoDB has only georeferenced half of our points. The other half we will have to edit it manually. To export this new dataset, we have to split the_geom column into two separate ones, one from the latitude values and one for the longitude values. First, create a new number type column for latitude, and one for longitude. Second, with the CartoDB Editor enter the following SQL query:

```sql
UPDATE 
  outliersnew 
SET 
  longitude = ST_X(the_geom), latitude = ST_Y(the_geom);
```
  
Apply and export this new dataset as outliersCDB.csv. Load it again in R and clean the useless columns that CartoDB has added:

## Loading, cleaning and editing

```{r}
outliersCDB <- read.csv("outliersCDB.csv", header = TRUE, sep = ",", stringsAsFactors=FALSE)
outliersCDB$cartodb_id <- NULL #Get rid of useless columns
outliersCDB$the_geom <- NULL
outliersCDB$cartodb_georef_status <- NULL
outliersCDB$country <- NULL
outliersCDB$lon <- outliersCDB$longitude #Rename longitude/latitude columns
outliersCDB$lat <- outliersCDB$latitude
outliersCDB$longitude <- NULL
outliersCDB$latitude <- NULL
```

As you remember, we need to find 5 pairs of coordinates. For this purpuse, we utilize the Latitude Longitude Finder. Write down the values and introduce them using these simple lines of code:

## Adding lon/lat coordinates

```{r}
#Lat
outliersCDB$lat[5] <- 42.0028
outliersCDB$lat[6] <- 42.6898
outliersCDB$lat[8] <- 41.9739
outliersCDB$lat[9] <- 42.9038
outliersCDB$lat[10] <- 36.7583

#Lon

outliersCDB$lon[5] <- -8.7260
outliersCDB$lon[6] <- -8.4909
outliersCDB$lon[8] <- -7.2821
outliersCDB$lon[9] <- -8.6583
outliersCDB$lon[10] <- -4.2416
```

Finally, the last step before visualizing consits in replacing the wrong coordinates in the fires_lonlat dataset with the right ones from the outliersCDB dataset:

## Replacing outliers coordinates

```{r}
fires_lonlat[match(outliersCDB$id, fires_lonlat$id), ] <- outliersCDB
firesFinal <- fires_lonlat
write.csv(firesFinal, "firesFinal.csv")
```

Time to map! First, with R. Use the symbol() function to draw proportional circles in relation to the number of fires per town.

## Mapping fires - Spain

```{r}
xlim <- c(-11.10, 5.76) #Set x/longitude limits
ylim <- c(35.44, 44.18) #Set y/latitude limits

map("world", col="#1ABC9C", fill=TRUE, bg="#F8F8FF", lwd=0.05, xlim=xlim, ylim=ylim) #Plot a Spain map

symbols(firesFinal$lon, firesFinal$lat, circles=sqrt(firesFinal$num_fires),
        add=TRUE, inches=0.15, fg="#F8F8FF", bg="#CA99CE") #Plot proportional circles
        
```

[spainmap.jpeg]

Nice. But have a try with CartoDB. You will discover that CartoDB has more elegant ways to visualize your data. Once more, load and connect your dataset. Then, create a map. Now it is up to you how to display the data. As the dataset consists in fires and it is arranged in clear clusters, I seems a good idea to make a heatmap. So go to the wizard tab and select heatmap and change your color gradient with the CartoCSS editor according to your preferences. These are mines:

#firesfinal
{
  image-filters: colorize-alpha(#2fe2bf, #51e7c9, #74ecd4, #96f0df , #b9f5e9, #dbfaf4);
  marker-file: url(http://s3.amazonaws.com/com.cartodb.assets.static/alphamarker.png);
  marker-fill-opacity: 0.4*[value];
  marker-width: 35;
}

The result is shown above. You can see clearly the two clusters I was talking about. One in Galicia, and the other in Asturias. Time to take decisions. Any comment?
