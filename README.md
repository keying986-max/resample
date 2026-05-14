resample
/******************************************************************************
var selectors = ['lon', 'lat'].concat(metadataFields).concat(variableNames);
var requiredFields = ['lon', 'lat'].concat(variableNames);

var samples = outputImage.sample({
  region: studyArea,
  projection: targetProjection,
  geometries: false,
  dropNulls: false,
  tileScale: CONFIG.tileScale
}).map(addRunMetadata);

samples = samples.filter(ee.Filter.notNull(requiredFields));

print('Sample count after removing missing rows:', samples.size());
print('First sample:', samples.first());


/******************************************************************************
 * 12. Visualization
 ******************************************************************************/

Map.centerObject(studyArea, CONFIG.mapZoom);

Map.addLayer(
  stacked.select('NDVI'),
  {min: 0, max: 0.8},
  'NDVI'
);

Map.addLayer(
  stacked.select('NPP'),
  {min: 0, max: 600},
  'NPP'
);

Map.addLayer(
  stacked.select('Precipitation'),
  {min: 0, max: 1000},
  'Precipitation'
);

Map.addLayer(
  stacked.select('ShortwaveRadiation'),
  {min: 0, max: 9000000000},
  'Shortwave Radiation'
);

Map.addLayer(
  stacked.select('Temperature'),
  {min: -10, max: 30},
  'Temperature'
);

Map.addLayer(
  stacked.select('Elevation'),
  {min: 0, max: 6000},
  'Elevation'
);

Map.addLayer(
  stacked.select('LandCover'),
  {min: 1, max: 17},
  'Land Cover'
);

Map.addLayer(
  stacked.select('SOC'),
  {min: 0, max: 150},
  'SOC'
);


/******************************************************************************
 * 13. Export table
 ******************************************************************************/

Export.table.toDrive({
  collection: samples,
  description: exportName,
  folder: CONFIG.exportFolder,
  fileNamePrefix: exportName,
  fileFormat: 'CSV',
  selectors: selectors
});

print('Export task has been created. Please run it in the Tasks panel.');
print('Export file name prefix:', exportName);
