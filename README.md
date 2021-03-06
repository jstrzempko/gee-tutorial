# Tutorial: Habitat Suitability Modeling using Random Forest Classification in Google Earth Engine

This tutorial was created on May 1st, 2020 by Jess Strzempko as a final project for GEOG 296 Advanced Raster GIS at Clark University taught by Professor Florencia Sangermano. The full project text can be found in the repository file titled `JeStrzempko_geog296FinalProject.pdf`. A simpler, more accessible version of the tutorial can be followed in this repository's README. 

# Objectives

1. Familiarize users with Google Earth Engine modeling applications
2. Walk users through a simple application of the Random Forest machine learning algorithm to habitat biodiversity modeling (HBM)
3. Highlight opportunities for users to explore beyond the scope of the tutorial by building off existing scripts

# Background

In recent decades, machine learning models have emerged as a prevalent mode of inductive modeling in the GIS community. With habitat biodiversity modeling (HBM), empirical species sightings are combined with bioclimatic data to extrapolate the potential habitat of a species and produce potential range maps. 

The technique this tutorial will use is a form of supervised classification called Random forest where the model is trained using species observations [to generate an output map of potential species presence](https://komodowebhost.com/index.php/features-mainmenu-47/geographic-information-system/289-proposed-workflow-to-for-habitat-modelling-using-gbif-mediated-data-and-google-earth-engine). [Random decision forests](https://en.wikipedia.org/wiki/Random_forest) are an extension of [Classification Tree Analysis (CTA)](https://clarklabs.org/classification-tree-analysis/) which are most often applied to remotely sensed data to classify land cover. In a single classification tree, the input data (pixels) is partitioned into increasingly homogeneous groups (classes) based on their relation to a set of predictor variables (spectral bands). While this method is robust to outliers, it is unstable allowing small changes in the input to produce vastly different decision trees. 

In response, Random forest classification was developed where multiple decision trees are generated using random, but equally sized subsets of the data with majority voting deciding the final class of a pixel. In this tutorial, we will use Google Earth Engine's Random forest algorithm to output both hard and soft classifications of a species' habitat. GEE's Random forest classifier has many parameters to modify including the number of decision trees to create, the fraction of the input to “bag” per tree (the random subset selected), and the maximum number of lead nodes (classes) in each tree. With the ability to easily invoke and run classification algorithms across large, openly sourced datasets, GEE presents ample geoprocessing opportunities to explore. 

# Data

* **`data/vicugna_final.zip`**: Point shapefile of observations of vicugna in South America sourced from TerrSet tutorial data (Chapter 6 Exercise 6-3 HBM: Maxent). 
  + converted from `.vct` to `.shp` in TerrSet's SHAPEIDR
  + "absence" points added in ArcMap - representing regions where it would be extremely unlikely to find vicugna (e.g. Amazon rainforest, lake, or coast)
  + "absence" points given a value of 0, "presence" points given a value of 1
  + sent to a zipped folder so GEE will read as a single file
* **`WorldClim 2`**: open source bioclimatic variables available through the Google Earth Engine search bar
  + formally `WorldClim BIO Variables V1`, a set of 19 global “bioclimatic” variables, compiled by [Fick & Hijmans at University of California, Berkeley](https://rmets.onlinelibrary.wiley.com/doi/full/10.1002/joc.5086)
  + average values of bioclimatic variables for 1970-2000  at 1-kilometer spatial resolution
  + we will select bands corresponding to annual mean temperature and annual precipitation
* **`Landsat 8 Data`**: atmospherically corrected surface reflectance available through the Google Earth Engine search bar
  + formally `USGS Landsat 8 Surface Reflectance Tier 1 data`, collected by [USGS EROS](https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C01_T1_SR)
  + 30-meters spatial resolution, we will filter for low cloud cover across 2019 to create composite bands
  + we will select the visible, near-infrared, short-wave infrared, and thermal infrared bands
* **`Country Boundaries`**: international boundary polygons available from Google Earth Engine search bar
  + formally `LSIB: Large Scale International Boundary Polygons`, collected by the [US Dept of State](https://developers.google.com/earth-engine/datasets/catalog/USDOS_LSIB_2017)
  + we will use to select our region of interest and visualize country boundaries in relation to analysis outputs

# Set-Up

Users will need an Earth Engine account, which can be set up through an existing google account. Google “Sign up for Google Earth Engine” or visit [https://signup.earthengine.google.com](https://signup.earthengine.google.com) and an email will be sent to your gmail inbox when you are approved. Visit [code.earthengine.google.com](https://code.earthengine.google.com/) to get started. 

It is recommended that users have a working knowledge of JavaScript for Earth Engine before starting this tutorial. Sample code (that can be cut and pasted into the code editor) will be provided to guide the user along, but previous exposure to this mode of analysis and relevant syntax will enhance learning. Provided below are several introductory links to get started.

[**Introduction to JavaScript for Earth Engine**](https://developers.google.com/earth-engine/tutorials/tutorial_js_01) *(good overview of commonly used statements and basic data types)*

[**Get Started with Earth Engine**](https://developers.google.com/earth-engine/guides/playground) *(helpful in exploring/locating parts of the Google Earth Engine Application Programming Interface (API))*

# Tutorial

**[Link to Final Code](https://code.earthengine.google.com/10a6b8ef6aca75843c27ec161cf16a92)**

## Import Vicguna Data

* To begin, import the vicugna shapefile from a zipped folder. Navigate to the **Asset tab** on the left side of the screen and click the large red drop-down menu labeled **New**. Select Shapefile (.shp, .shx, .dbf, .prj, .zip) under the Table Upload option and the window *Upload a new shapefile asset* will appear as shown below.

<p align="center">
  <img src="images/assets.PNG" width="300" />
</p>

<p align="center">
  <img src="images/assets2.PNG" width="600" />
</p>

* You can upload multiple files at one time, but they must all correspond to the same shapefile. Rather than selecting each file separately, we will be uploading a zipped folder which contains all relevant files. Under source files, hit the **select button** and navigate to where your vicugna zipped folder is on your computer and select to open it. Name the asset *vicguna*, leave all other properties as default, and scroll down to **select upload**. 

* At this point, your Tasks tab on the right side of thescreenmay blink orange indicating that there are unseen tasks occurring. Click on this tab and you should see your asset being ingested. This may take a few minutes. 

* When Earth Engine is done ingesting your asset, it will appear blue in the **Tasks tab**. If it hasn’t already appeared in your **Assets tab** then hit the refresh button and *vicguna* should be added to the list. When you hover over an asset, three icons will appear: Share, Rename, Import into Script. Select the arrow to **Import into Script** and the *vicguna* table with its pathname will be added to your list of imports at the top of the code editor. Change the variable name to *vicugna*.

* Visualize this data by adding this statement to your code and running the script (Ctrl + Enter) or hitting the **run button** at the top of the code editor:

```javascript
Map.addLayer(vicugna, {}, ‘Vicugna Data’);
```
*(if you need help understanding this code, type Map.addLayer into the **Docs search bar** on the left side of the screen. This documentation gives the inputs, outputs, and arguments to a statement – for instance, we want to add our layer “vicugna” to the map with default palette {}, and name it “Vicugna Data”)*

* On the right side of the screen, select the **Inspector tab**. Now, when you click on features in the map, information about them will appear in the **Console tab**. Investigate the data by clicking on points. Pay particular attention to the value of the *presence* property. 

* Next, we will create a variable called *label* in which we will store the property *presence*. When the data was created, locations where vicugna were observed were 
given a value of 1, while areas where the species was extremely unlikely to be found were given a value of 0 in a column labeled *presence*. We will base the creation of our classification off this property. 
```javascript
var label = 'presence';
```

## Decide on a Region of Interest

* Next, we will import data on world country boundaries and select a region of interest to bound our classification. In the **search bar** at the top of the screen (search places and datasets), type in *LSIB: Large Scale International Boundary Polygons, Detailed* and hit enter. Select the first option to appear (should be for the year 2017). Read the description and look through the table schema for this data before selecting the **blue import button**. 

* Similar to the vicugna table, this data should appear in your imports. Name it *world*.

* To minimize runtime and increase simplicity of the tutorial, we will select only the country of Peru as our region of interest. To do so, filter the world layer by selecting the feature where the property 'COUNTRY_NA' = 'Peru.'

```javascript
var roi = world.filter(ee.Filter.eq('COUNTRY_NA', 'Peru'));
```

* Center the map on this region of interest and set it to a zoom level of 5.

```javascript
Map.centerObject(roi, 5);
```

* You can also try out a larger analysis region defined by the user rather than country boundaries as shown below. To do so, navigate to the map below and look to the upper left-hand corner. Here you can manage and create geometry imports (points, lines, polygons, and rectangles). Zoom out so your map includes all of Peru and Bolivia and draw a rectangle that encompasses these countries. Under geometry imports, select *edit layer properties* and rename the layer *vicguna_region*. It will appear with your *world* and *vicugna* imports from previous steps. Comment out the previous roi variable (by adding // before the statement) and add this code: 

```javascript
var roi = vicugna_region;
//Visualize this new roi: 
Map.addLayer(roi, {}, 'Region of Interest');
```

<p align="center">
  <img src="images/roi_sq.PNG" width="400" />
</p>

* Return to the original roi by removing the // and adding them in front of the the vicugna_region roi. You can only define a variable once. 

<p align="center">
  <img src="images/roi_peru.PNG" width="400" />
</p>

## Import Environmental Variables

* Similar to before, search for *WorldClim BIO Variables V1* in the **search bar**. Explore this data and the bands it contains. We will be selecting *bio01* and *bio12* for analysis as shown below.

<p align="center">
  <img src="images/rain.PNG" width="200" />
</p>
<p align="center">
  <img src="images/temp.PNG" width="200" />
</p>

* Import the image and name it *bioclim*.

* Next, we will create a new variable called *temp_bioclim* from the mean annual temperature band (bio01). Type and run this code:

```javascript
var temp_bioclim = bioclim.select('bio01');
```

* We can also create visualization parameters with which to explore the data, by setting minimum and maximum display values and assigning a palette of colors. This visualization variable can be added to our Map.addLayer function.
```javascript
var temp_vis = {min: -230.0, max: 300.0,
  palette: ['blue', 'purple', 'cyan', 'green', 'yellow', 'red']};
Map.addLayer(temp_bioclim, temp_vis, 'Annual Mean Temperature');
```

* Once you’ve explore the worldwide temperature trends, we will clip this layer to our roi to increase processing speed. Comment out the previous statement to remove the worldwide data from the map. 

```javascript
var temp = temp_bioclim.clip(roi);
```

* Conduct a similar process to obtain and visualize annual precipitation (bio12). Make sure to comment out extraneous statements when done exploring. 

```javascript
var precip_bioclim = bioclim.select('bio12');
var precip_vis = {min: 0, max: 10000,
  palette: ['white', '00008B', 'black']};
var precip = precip_bioclim.clip(roi);iv.Map.addLayer(precip, precip_vis, 'Annual Precipitation');
```
  
***(Optional Addition of Landsat Data)***

* Similar to previous steps, search for *USGS Landsat 8 Surface Reflectance Tier 1* and import this data. If you’ve ever worked with Landsat data before, Earth Engine gives easy access to all Landsat image collections. Name this import *L8*.

* Next, we want to create a variable of Landsat 8 data filtered for 2019 over our roi.

```javascript
var L8_filter = ee.ImageCollection('LANDSAT/LC08/C01/T1_SR')
  .filterBounds(roi)
  .filterDate('2019-01-01', '2019-12-31');
```

* Currently, we have an image collection consisting of surface reflectance for all Landsat bands for all of 2019 over our roi. We will want to create a function to filter out cloudy imagery using a cloud mask. We will use bits 3 and 5 (cloud shadow and cloud) in the pixel quality assessment (QA) band to filter for images where the flags are set to 0 indicating clear conditions. We will scale the image, so values are lie on a scale of 0 to 1.

```javascript
function maskL8(image) {
  var cloudShadowBitMask = ee.Number(2).pow(3).int();
  var cloudsBitMask = ee.Number(2).pow(5).int();
  var qa = image.select('pixel_qa');
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
      .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask).divide(10000);}
```

* We will then map this function over the 2019 data, take the median pixel values (so as to avoid the influence of outliers), and clip to our roi. 

```javascript
var L8_median = L8_filter.map(maskL8)
                    .reduce(ee.Reducer.median())
                    .clip(roi);
```

* Create a true color composite of the median bands to visualize the images you have just created. We can utilize the visualization parameters to create a composite of 3 bands. 

```javascript
Map.addLayer(L8_median, 
  {bands: ['B4_median', 'B3_median', 'B2_median'], min: 0, max: 0.2},
  'Landsat 8 Composite'); 
```

<p align="center">
  <img src="images/landsat8.PNG" width="400" />
</p>

## Create a Composite of the Layers

* In order to run the classification, we will need all of our environmental variables in one variable that we can reference. If you included the Landsat 8 bands, you need to create a new variable called *L8_bands* which excludes the pixel qa, aerosol, and radarsat bands. If you did not run the Landsat 8 steps, then you can simply add the temperature data to the precipitation data to create a new composite variable. In both cases, we need a variable *bands* which includes the names of all the bands in this composite variable. 
  + *First option* 
  
```javascript
var enviro = temp.addBands(precip);
var bands = enviro.bandNames();
```

  + *Second option*
  
 ```javascript
 var L8_bands = L8_median
  .select('B1_median', 'B2_median', 'B3_median', 'B4_median',
  'B5_median', 'B6_median', 'B7_median', 'B10_median', 'B11_median');
var enviro = L8_bands.addBands(temp).addBands(precip); 
var bands = enviro.bandNames();
 ```

## Sample the Input Imagery to get Training Data

* Next, we will need to create training data by sampling our environmental variables at the points in the vicugna dataset. This will create a feature collection of training data which we can use to train the classifier. We will use the sampleRegions statement across our vicguna selection, making sure that our data is separated across the *label* property (representative of our *presence* classes). 

```javascript
var training = enviro.select(bands).sampleRegions({collection: vicugna,properties: [label],scale: 30});
```

## Train and Run a Discrete Classifier

* In this first section, we will train and run a discrete Random Forest classifier, meaning that the output will contain presence (value = 1) and absence (value = 0) regions. In this case, if greater than 50% of the decision trees mark a pixel a certain value, then it will be given this value. The output map will be a Boolean/binary image. This can be useful when generating species range maps as it can generate a range perimeter.

* For our run of the Random Forest classifier (called smileRandomForest in GEE), we will set the classification to use 20 decision trees with all other parameters at default.

```javascript
var classifier_disc = ee.Classifier.smileRandomForest(20)
  .train({
  features: training,
  classProperty: label,
  inputProperties: bands});
```

* Once we have created the discrete classifier, we can run it on our *enviro* image and display the resulting Map. 

```javascript
var classified_disc = enviro.select(bands).classify(classifier_disc);
Map.addLayer(classified_disc, 
  {min: 0, max: 1, palette: ['FFFFFF','031A00']}, 
  'Random Forest Discrete');
```

* Explore the output. Use the inspector tool and the pan option to move around your image and select regions to investigate their value. When might a map like this be useful?

<p align="center">
  <img src="images/output2.PNG" width="300" />
</p>

<p align="center">
  <img src="images/output4.PNG" width="300" />
</p>

## Train and Run a Soft Classifier (probabilities)

* To compare outputs, we will also run a soft classifier and scale the output probabilities from 0 to 100 to see gradients of habitat suitability for the vicugna. The same color palette was used for both classifiers to produce acontrast between results.

```javascript
var classifier_prob = ee.Classifier.smileRandomForest(20).
setOutputMode('PROBABILITY')
.train({
  features: training,
  classProperty: label,
  inputProperties: bands});
```

* Again, we will run the classifier on the data and display the output. 

```javascript
var classified_prob = enviro.select(bands).classify(classifier_prob).multiply(100);
Map.addLayer(classified_prob,
  {min:0, max:100, palette: ['FFFFFF','C6AC94','8D8846','395315','031A00']},
  "Random Forest Probability");
```

* Explore the output. Use the **inspector tool** to select certain regions. Compare their values to the discretely classified image. What advantages or disadvantages does this soft classification provide in comparison to the hard classification?

<p align="center">
  <img src="images/output.PNG" width="300" />
</p>

<p align="center">
  <img src="images/output3.PNG" width="300" />
</p>

## Confusion Matrix representing Re-substitution Accuracy

* GEE recommends in its supervised classification tutorial/help documents that you create a confusion matrix (table that describes performance of the algorithm by comparing output data with test data for which values are known). Unfortunately, this data does not have another dataset available for validation (in the way land cover classes might) meaning that resubstitution error is simply the error rate obtained from comparing the training data to the output. The output confusion matrix and accuracy measurement will appear in your **Console tab** on the right side of the screen.

```javascript
var trainAccuracy = classifier_disc.confusionMatrix();
print('RF Resubstitution error matrix: ', trainAccuracy);
print('RF Training overall accuracy: ', trainAccuracy.accuracy());
```

* Because each pixel has to be in a certain class for comparison, we can only conduct this on our hard/discrete classified image. Resubstitution error is optimistic and often unrealistic. Further validation data would be needed to generate a cross-validation matrix, which would provide more accurate assessments. 

## Additional Step
* For visualization purposes, create an empty image to paint the roi features into. This will allow us to see country boundaries as outlines (which will be less disruptive to the analysis). 

```javascript
Map.addLayer(ee.Image().paint(roi, 1, 2), {}, 'Region of Interest');
```

## Export Image

* To export the image to your google drive, set the parameters as follows:

```javascript
Export.image.toDrive({
  image: classified_prob,
  description: 'Random Forest Probability',
  maxPixels: 1e20,
  scale: 500,
  region: roi,
  folder: 'Advanced_Raster'});
```

* Once you’ve run the script with this statement, a separate option for you to run the export will appear in your **Tasks** tab where you can decide on resolution, location, and name of the output image.

<p align="center">
  <img src="images/export.PNG" width="400" />
</p>

# Takeaways

The Random Forest classifier is one of the most easily understood machine learning algorithm because of its basis in simpler methods. The simple construction and modification of scripts combined with easy access to open source data via the Earth Engine search bar makes this program especially useful in this geoprocessing application. The script from this tutorial can be used with different data or can be expanded upon to run the classification methods with a set of new parameters. Ultimately, the application of Random Forest classification to habitat suitability/biodiversity modeling offers abundant opportunity to explore predictions of species distribution through the lens of bioclimatic factors.
