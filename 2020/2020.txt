// Accuracy - 0.8995859213250518

// Error Matrix - 
List (5 elements)
0: [782,2,33,0,34]
1: [3,11106,20,5,127]
2: [70,64,1141,0,41]
3: [3,11,0,594,8]
4: [32,1172,24,0,1150]


// Define Geqiugou Basin boundaries (polygon based on coordinates)
// NDVI, Normalised slope and normalised precipitation se predict soil erosion
// Define each area as an ee.FeatureCollection with polygons using all provided coordinates


// Extract individual polygons and convert to FeatureCollection
var geometries = urbanAreaGeo.geometries(); // List of geometries
var features = geometries.map(function(geometry) {
  geometry = ee.Geometry(geometry); // Convert to Geometry object
  var coords = geometry.coordinates();
  if (coords.length < 4) {
    print('Skipping invalid geometry:', geometry);
    return null; // Skip invalid geometry
  }
  
  // Wrap valid geometry into a Feature
  return ee.Feature(ee.Geometry.Polygon(coords), {label: 0});
});

features = features.filter(ee.Filter.notNull(['label']));
// Convert features to FeatureCollection
var urbanArea = ee.FeatureCollection(features);

// Extract individual polygons and convert to FeatureCollection
var geometries1 = denselyForestedAreaGeo.geometries(); // List of geometries
var features1 = geometries1.map(function(geometry) {
  geometry = ee.Geometry(geometry); // Convert to Geometry object
  var coords = geometry.coordinates();
  if (coords.length < 4) {
    print('Skipping invalid geometry:', geometry);
    return null; // Skip invalid geometry
  }
  
  // Wrap valid geometry into a Feature
  return ee.Feature(ee.Geometry.Polygon(coords), {label: 1});
});

features1 = features1.filter(ee.Filter.notNull(['label']));
// Convert features to FeatureCollection
var agricultureArea = ee.FeatureCollection(features1);

// Extract individual polygons and convert to FeatureCollection
var geometries2 = deforestedAreaGeo.geometries(); // List of geometries
var features2 = geometries2.map(function(geometry) {
  geometry = ee.Geometry(geometry); // Convert to Geometry object
  var coords = geometry.coordinates();
  if (coords.length < 4) {
    print('Skipping invalid geometry:', geometry);
    return null; // Skip invalid geometry
  }
  
  // Wrap valid geometry into a Feature
  return ee.Feature(ee.Geometry.Polygon(coords), {label: 2});
});

features2 = features2.filter(ee.Filter.notNull(['label']));
// Convert features to FeatureCollection
var deforestedArea = ee.FeatureCollection(features2);


var geometries3 = riverAreaGeo.geometries(); // List of geometries
var features3 = geometries3.map(function(geometry) {
  geometry = ee.Geometry(geometry); // Convert to Geometry object
  var coords = geometry.coordinates();
  if (coords.length < 4) {
    print('Skipping invalid geometry:', geometry);
    return null; // Skip invalid geometry
  }
  
  // Wrap valid geometry into a Feature
  return ee.Feature(ee.Geometry.Polygon(coords), {label: 3});
});

features3 = features3.filter(ee.Filter.notNull(['label']));
// Convert features to FeatureCollection
var riverbankArea = ee.FeatureCollection(features3);

var geometries4 = sparselyForestedAreaGeo.geometries(); // List of geometries
var features4 = geometries4.map(function(geometry) {
  geometry = ee.Geometry(geometry); // Convert to Geometry object
  var coords = geometry.coordinates();
  if (coords.length < 4) {
    print('Skipping invalid geometry:', geometry);
    return null; // Skip invalid geometry
  }
  
  // Wrap valid geometry into a Feature
  return ee.Feature(ee.Geometry.Polygon(coords), {label: 4});
});

features4 = features4.filter(ee.Filter.notNull(['label']));
// Convert features to FeatureCollection
var sparselyForestedArea = ee.FeatureCollection(features4);

// Function to mask clouds in Landsat 8 data
function maskClouds(image) {
  var qa = image.select('QA_PIXEL');
  var cloudShadowBitMask = (1 << 3);
  var cloudsBitMask = (1 << 5);
  var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
    .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask);
}

// Function to add indices (NDVI, NDBI, NDWI)
function addIndices(img) {
  var ndvi = img.normalizedDifference(['SR_B5', 'SR_B4']).rename('NDVI');
  var ndbi = img.normalizedDifference(['SR_B6', 'SR_B5']).rename('NDBI');
  var ndwi = img.normalizedDifference(['SR_B3', 'SR_B5']).rename('NDWI');
  return img.addBands(ndvi).addBands(ndbi).addBands(ndwi);
}

// Load DEM data and calculate slope
var dem = ee.Image('USGS/SRTMGL1_003');  // SRTM DEM data
var slope = ee.Terrain.slope(dem).rename('Slope'); // Calculate slope

// Load and prepare Landsat 8 data
var l8 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
  .filterBounds(agricultureArea.geometry().union(urbanArea.geometry())
  .union(deforestedArea.geometry()).union(riverbankArea.geometry()))
  .filterDate('2020-01-01', '2020-12-31')
  .map(maskClouds)
  .map(addIndices);

// Prepare Landsat image with NDVI, NDBI, NDWI, and slope
var image = l8.median()
  .select(['SR_B5', 'SR_B6', 'SR_B4', 'NDVI', 'NDBI', 'NDWI'])
  .addBands(slope); // Add slope as a band

// Function to calculate the overall GMD (Geometric Mean Diameter)
function calculateOverallGMD() {
  var gmdSandySoil = 0.5; // Example GMD value for Sandy Soil
  var gmdLoessSoil = 0.1; // Example GMD value for Loess Soil
  var gmdCalcareousSoil = 0.2; // Example GMD value for Calcareous Soil
  
  // Calculate the geometric mean of the three GMD values
  var gmdOverall = Math.pow(
    gmdSandySoil * gmdLoessSoil * gmdCalcareousSoil,
    1 / 3
  );
  
  return gmdOverall;
}

// Formula to calculate the specific value based on GMD
function applyFormulaToGMD(gmdOverall) {
  var logGMD = Math.log(gmdOverall);
  var formulaResult = 7.954 * (0.0017 + 0.0494 * Math.exp(
    -0.5 * Math.pow((logGMD + 1.675) / 0.6986, 2)
  ));
  return formulaResult;
}

// Calculate the overall GMD
var gmdOverall = calculateOverallGMD();

// Apply the formula to the GMD
var formulaResult = applyFormulaToGMD(gmdOverall);

// Log the formula result for checking
print('Formula Result:', formulaResult);

// Calculate NDVI range (min and max) within the area of interest
var ndviStats = image.select('NDVI').reduceRegion({
  reducer: ee.Reducer.minMax(),
  geometry: agricultureArea.geometry().union(urbanArea.geometry())
    .union(deforestedArea.geometry()).union(riverbankArea.geometry()),
  scale: 30,
  bestEffort: true
});
print('NDVI Range:', ndviStats);

// Load CHIRPS precipitation data for 2020
var chirps = ee.ImageCollection('UCSB-CHG/CHIRPS/DAILY')
  .filterBounds(agricultureArea.geometry().union(urbanArea.geometry())
  .union(deforestedArea.geometry()).union(riverbankArea.geometry()))
  .filterDate('2020-01-01', '2020-12-31');

// Calculate annual precipitation
var annualPrecipitation = chirps.sum().rename('Precipitation');

// Normalize precipitation data to 0-1 scale
var normalizedPrecipitation = annualPrecipitation.unitScale(0, 2000).rename('NormalizedPrecipitation');

// Calculate Dynamic Erosion Risk Index
var normalizedSlope = slope.unitScale(0, 90).rename('NormalizedSlope');
var normalizedNDVI = image.select('NDVI').unitScale(-1, 1).rename('NormalizedNDVI');

// Erosion risk = (transformed NDVI) * slope * precipitation
var erosionIndex = normalizedSlope
  .multiply(normalizedNDVI)
  .multiply(normalizedPrecipitation)
  .rename('ErosionRisk');

// Multiply the result of the formula with the erosionIndex
var finalErosionRisk = erosionIndex.multiply(formulaResult).rename('FinalErosionRisk');

// Add final erosion risk to the map
// Map.addLayer(finalErosionRisk, {min: 0, max: 1, palette: ['white', 'red']}, 'Final Erosion Risk');
// Add final erosion risk to the map with adjusted palette
// Map.addLayer(finalErosionRisk, {
//   min: 0,
//   max: 1,
//   palette: ['white','red']
// }, 'Final Erosion Risk');

// var finalErosionRisk = erosionIndex.multiply(formulaResult).rename('FinalErosionRisk');

// Add final erosion risk to the map
// Map.addLayer(finalErosionRisk, {min: 0, max: 1, palette: ['white', 'red']}, 'Final Erosion Risk');

// // Create legend panel
// var legend = ui.Panel({
//   style: {
//     position: 'bottom-left',
//     padding: '8px 15px'
//   }
// });

// // Add title to legend
// var title = ui.Label({
//   value: 'Erosion Risk Legend',
//   style: {fontWeight: 'bold', fontSize: '18px', margin: '0 0 4px 0'}
// });
// legend.add(title);

// // Define legend colors and labels
// var palette = ['white', 'red'];
// var names = ['Low Risk (0 - 0.5)', 'High Risk (0.5-1)'];

// // Add color boxes and labels to legend
// palette.forEach(function(color, index) {
//   var colorBox = ui.Label({
//     style: {
//       backgroundColor: color,
//       padding: '8px',
//       margin: '0 8px 4px 0'
//     }
//   });
//   var description = ui.Label({
//     value: names[index],
//     style: {margin: '0 0 4px 0'}
//   });
//   var legendItem = ui.Panel({
//     widgets: [colorBox, description],
//     layout: ui.Panel.Layout.Flow('horizontal')
//   });
//   legend.add(legendItem);
// });

// // Add legend to the map
// Map.add(legend);

// Add the erosion index layer
Map.addLayer(finalErosionRisk, {min: 0, mid: 0.05, max: 0.082, palette: ['lightblue', 'purple', 'red']}, 'Dynamic Erosion Risk');

// Create a panel for the legend
var legend = ui.Panel({
  style: {
    position: 'bottom-left',
    padding: '8px 15px'
  }
});

// Create a title for the legend
var legendTitle = ui.Label({
  value: 'Erosion Risk Legend',
  style: {
    fontWeight: 'bold',
    fontSize: '16px',
    margin: '0 0 4px 0',
    padding: '0'
  }
});
legend.add(legendTitle);

// Define the colors and labels
var colors = ['lightblue', 'purple', 'red'];
var labels = ['Low Erosion', 'Medium Erosion', 'High Erosion'];

// Add color swatches and labels to the legend
for (var i = 0; i < colors.length; i++) {
  var colorBox = ui.Label({
    style: {
      backgroundColor: colors[i],
      padding: '8px',
      margin: '0 0 4px 0'
    }
  });
  
  var description = ui.Label({
    value: labels[i],
    style: {
      margin: '0 0 4px 6px'
    }
  });
  
  // Create a horizontal panel for each color and label
  var legendRow = ui.Panel({
    widgets: [colorBox, description],
    layout: ui.Panel.Layout.Flow('horizontal')
  });
  legend.add(legendRow);
}

// Add the legend to the map
Map.add(legend);


// Merge labeled regions into a single FeatureCollection for training
var trainingData = agricultureArea.merge(urbanArea).merge(deforestedArea).merge(riverbankArea).merge(sparselyForestedArea);

// Sample regions for training
var samples = image.sampleRegions({
  collection: trainingData,
  properties: ['label'],
  scale: 30
}).randomColumn('random');

// Split samples into training and testing sets
var split = 0.7;
var trainingSet = samples.filter(ee.Filter.lt('random', split));
var testingSet = samples.filter(ee.Filter.gte('random', split));

// Train a Random Forest classifier
var classifier = ee.Classifier.smileRandomForest(10).train({
  features: trainingSet,
  classProperty: 'label',
  inputProperties: image.bandNames()
});

// var classifiedImage = /* your classified image */;Map.addLayer(classifiedImage, imageVisParam, 'Classified Image');

// Center the map
// Map.centerObject(classifiedImage, 10);

// Classify the image
var classifiedImage = image.classify(classifier);
Map.addLayer(classifiedImage, imageVisParam2, 'Classified Image');
// Classify the testing set and calculate accuracy
var validation = testingSet.classify(classifier);
var testAccuracy = validation.errorMatrix('label', 'classification');
print('Error matrix:', testAccuracy);
print('Overall accuracy:', testAccuracy.accuracy());
// Map.centerObject(classifiedImage, 10);
// Add the classified image to the map
// Map.addLayer(classifiedImage, {
//   min: 0,
//   max: 3,
//   palette: ['green', 'gray', 'brown', 'blue']
// }, 'Classified Image');

// Export the final erosion risk to Google Drive
Export.image.toDrive({
  image: finalErosionRisk,
  description: 'Final_Erosion_Risk_Index',
  scale: 30,
  region: agricultureArea.geometry().union(urbanArea.geometry())
    .union(deforestedArea.geometry()).union(riverbankArea.geometry()).union(sparselyForestedArea.geometry()),
  fileFormat: 'GeoTIFF'
});

// Export the classified image to Google Drive
Export.image.toDrive({
  image: classifiedImage,
  description: 'Classified_Image',
  scale: 30,
  region: agricultureArea.geometry().union(urbanArea.geometry())
    .union(deforestedArea.geometry()).union(riverbankArea.geometry()).union(sparselyForestedArea.geometry()),
  fileFormat: 'GeoTIFF'
});

// Export the erosion index to Google Drive
Export.image.toDrive({
  image: erosionIndex,
  description: 'Dynamic_Erosion_Risk_Index',
  scale: 30,
  region: agricultureArea.geometry().union(urbanArea.geometry())
    .union(deforestedArea.geometry()).union(riverbankArea.geometry()).union(sparselyForestedArea.geometry()),
  fileFormat: 'GeoTIFF'
});