# rayshader_animation_experiment
R script to create an animated hillshade image.
```
require(av)
require(jpeg)
require(rayshader)
require(raster)
require(rayrender)

#Load image
image_url<-"https://github.com/cverdel/rayshader_experiment/raw/main/Hermannsburg_map.tif"
temp<-tempfile()
download.file(image_url, temp, mode="wb")

rgb = raster::brick(temp) #Map

#Load elevation data
DEM_url<-"https://github.com/cverdel/rayshader_experiment/raw/main/Hermannsburg_DEM.tif"
temp2<-tempfile()
download.file(DEM_url, temp2, mode="wb")

elevation1 = raster::raster(temp2) #Elevation data
elevation<-aggregate(elevation1,fact=2) #Reduces dimensions of elevation data for faster processing

#Splits image into rgb and creates an array
names(rgb) = c("r","g","b", "t")
rgb_r = rayshader::raster_to_matrix(rgb$r)
rgb_g = rayshader::raster_to_matrix(rgb$g)
rgb_b = rayshader::raster_to_matrix(rgb$b)

map_array = array(0,dim=c(nrow(rgb_r),ncol(rgb_r),3))
map_array[,,1] = rgb_r/255 #Red 
map_array[,,2] = rgb_g/255 #Blue 
map_array[,,3] = rgb_b/255 #Green 
map_array = aperm(map_array, c(2,1,3))

#Loop
steps=120  #Half the number of total frames in the animation
num<-c(1:steps, steps:1) #Sequence for frames
width=1800
height=1200
zscale1=12 #Smaller value=more vertical exaggeration
elevation_cutoff= seq(maxValue(elevation), minValue(elevation), length.out = steps) #Elevation intervals used in loop
for(i in 1:steps) 
{
  #Elevation cutoffs
  map_raster <- elevation #Starts each time with the "normal" elevation data
  map_raster[map_raster[] < elevation_cutoff[i] ] = minValue(elevation) # Removes data below a cutoff value, creates "new" elevation data
  
  #Raster to matrix conversion of elevation data
  el_matrix = rayshader::raster_to_matrix(map_raster)
  
  #Calculate shading 
  ambient_layer = ambient_shade(el_matrix, zscale = zscale1, multicore = TRUE, maxsearch = 50)
  ray_layer = ray_shade(el_matrix, sunangle=340, zscale = zscale1, multicore = TRUE)
  
  #Plot in 2D
  png(file= sprintf("el_plot%i.png", i), width=width, height=height) #Filename of frame
  (map_array) %>%
    add_shadow(ray_layer,0.3) %>%
    add_shadow(ambient_layer,0) %>%
    plot_map(asp=asp)
  dev.off()
}

#Read sequence of frames and create video
av::av_encode_video(sprintf("el_plot%i.png",num), vfilter = "scale=trunc(iw/2)*2:trunc(ih/2)*2", framerate = 20, output = "el_video.mp4")

```
