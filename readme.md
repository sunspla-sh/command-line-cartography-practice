# About
Learn to make a population density map of your state. This Github repository contains a condensed set of instructions and commands. I'm not the original creator of this process, I only adapted it for Florida as practice while I was learning about SHP files, GeoJSON, and TopoJSON. There's a much larger and more detailed tutorial [(link)](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c) written by @mbostock (Mike Bostock) on Medium, who originally created the process for the state of California, which I later adapted for Florida, so give credit to him and big thanks from me because it would've taken me a lot longer without his writeups.

This tutorial includes basic info for working with shapefiles, geojson,
topojson, and d3.

![Florida Population Density Map](/fl_cb_2014_12_tract_500k/fl_albers_tracts_counties_color_ylgn_final.svg)

Technologies used for this project include the following:

![List of technologies](tech.png "List of technologies")

# Warning for PNPM
I used ```pnpm``` as my package manager, but at some points it failed and I
needed to do some manual configuration and re-installations of global packages
in the local store. You probably should stick to ```npm```, because I did not
include the fixes I used for ```pnpm``` in this readme/list of commands.

# Following @mbostock's tutorial
https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c

# FIPS Codes
https://en.wikipedia.org/wiki/Federal_Information_Processing_Standard_state_code
CA = 06
FL = 12

# CURL required shp files from census.gov
```curl 'https://www2.census.gov/geo/tiger/GENZ2014/shp/cb_2014_06_tract_500k.zip' -o cb_2014_06_tract_500k.zip```
```curl 'https://www2.census.gov/geo/tiger/GENZ2014/shp/cb_2014_12_tract_500k.zip' -o cb_2014_12_tract_500k.zip```

# Install shp2json cli tool
```pnpm add -g shapefile```

# Convert shp files into GeoJSON
```shp2json cb_2014_06_tract_500k.shp -o ca_geo.json```
```shp2json cb_2014_12_tract_500k.shp -o fl_geo.json```

# Install d3-geo-projection cli tool
```pnpm add -g d3-geo-projection```

# Apply geographic projections to geojson files now to avoid expensive runtime computations
```geoproject 'd3.geoConicEqualArea().parallels([34, 40.5]).rotate([120, 0]).fitSize([960, 960], d)' < ca_geo.json > ca_albers_geo.json```
```geoproject 'd3.geoConicEqualArea().parallels([24, 31.5]).rotate([84, 0]).fitSize([960, 960], d)' < fl_geo.json > fl_albers_geo.json```

# Preview projected geometry with geo2svg cli from d3-geo-projection
```geo2svg -w 960 -h 960 < ca_albers_geo.json > ca_albers_geo.svg```
```geo2svg -w 960 -h 960 < fl_albers_geo.json > fl_albers_geo.svg```

# Install ndjson cli tool
```pnpm add -g ndjson-cli```

# Convert GeoJSON feature collection into newline-delimited stream of GeoJSON features
```ndjson-split 'd.features' < ca_albers_geo.json > ca_albers_geo.ndjson```
```ndjson-split 'd.features' < fl_albers_geo.json > fl_albers_geo.ndjson```

# Manipulate each feature individually and set each feature's id using ndjson-map
```ndjson-map 'd.id = d.properties.GEOID.slice(2), d' < ca_albers_geo.ndjson > ca_albers_geo_id.ndjson```
```ndjson-map 'd.id = d.properties.GEOID.slice(2), d' < fl_albers_geo.ndjson > fl_albers_geo_id.ndjson```

# Download Census Tract Data
```url 'https://api.census.gov/data/2014/acs/acs5?get=B01003_001E&for=tract:*&in=state:06&key=YOUR_KEY_HERE' -o cb_2014_06_tract_B01003.json```
```url 'https://api.census.gov/data/2014/acs/acs5?get=B01003_001E&for=tract:*&in=state:12&key=YOUR_KEY_HERE' -o cb_2014_12_tract_B01003.json```

# Convert Census Tract Data to NDJSON stream (remove newlines, separate array into multiple lines, reformat as objects)
```ndjson-cat cb_2014_06_tract_B01003.json | ndjson-split 'd.slice(1)' | ndjson-map '{id: d[2] + d[3], B01003: +d[0]}' > cb_2014_06_tract_B01003.ndjson```
```ndjson-cat cb_2014_12_tract_B01003.json | ndjson-split 'd.slice(1)' | ndjson-map '{id: d[2] + d[3], B01003: +d[0]}' > cb_2014_12_tract_B01003.ndjson```

# Join population data to geometry data on the id property using ndjson-join
```ndjson-join 'd.id' ca_albers_geo_id.ndjson cb_2014_06_tract_B01003.ndjson > ca_albers_geo_join.ndjson```
```ndjson-join 'd.id' fl_albers_geo_id.ndjson cb_2014_12_tract_B01003.ndjson > fl_albers_geo_join.ndjson```

# Compute population density and remove properties we no longer need (constant 2589975.2356 converts m2 to mi2)
```ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' < ca_albers_geo_join.ndjson > ca_albers_geo_density.ndjson```
```ndjson-map 'd[0].properties = {density: Math.floor(d[1].B01003 / d[0].properties.ALAND * 2589975.2356)}, d[0]' < fl_albers_geo_join.ndjson > fl_albers_geo_density.ndjson```

# Convert back to GeoJSON
```ndjson-reduce < ca_albers_geo_density.ndjson | ndjson-map '{type: "FeatureCollection", features: d}' > ca_albers_geo_density.json```
```ndjson-reduce < fl_albers_geo_density.ndjson | ndjson-map '{type: "FeatureCollection", features: d}' > fl_albers_geo_density.json```

# Install older version of d3@5.16.0 (non-ESM) because newer version wasn't working with CommonJS require()
```pnpm add -g d3@5.16.0```

# Define a fill property using a sequential scale with Viridis color scheme
```ndjson-map -r d3 '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0,4000])(d.properties.density), d)' < ca_albers_geo_density.ndjson > ca_albers_geo_color.ndjson```
```ndjson-map -r d3 '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0,4000])(d.properties.density), d)' < fl_albers_geo_density.ndjson > fl_albers_geo_color.ndjson```

# Convert newline delimited GeoJSON to SVG using geo2svg
```geo2svg -n --stroke none -p 1 -w 960 -h 960 < ca_albers_geo_color.ndjson > ca_albers_geo_color.svg```
```geo2svg -n --stroke none -p 1 -w 960 -h 960 < fl_albers_geo_color.ndjson > fl_albers_geo_color.svg```

# Install TopoJSON cli
```pnpm add -g topojson```

# Use geo2topo to convert to TopoJSON
```geo2topo -n tracts=ca_albers_geo_density.ndjson > ca_albers_tracts_topo.json```
```geo2topo -n tracts=fl_albers_geo_density.ndjson > fl_albers_tracts_topo.json```

# Use toposimplify to further reduce size
```toposimplify -p 1 -f < ca_albers_tracts_topo.json > ca_albers_simple_topo.json```
```toposimplify -p 1 -f < fl_albers_tracts_topo.json > fl_albers_simple_topo.json```

# Use topoquantize to further reduce size
```topoquantize 1e5 < ca_albers_simple_topo.json > ca_albers_quantized_topo.json```
```topoquantize 1e5 < fl_albers_simple_topo.json > fl_albers_quantized_topo.json```

# Use topomerge to derive country geometries
```topomerge -k 'd.id.slice(0,3)' counties=tracts < ca_albers_quantized_topo.json > ca_albers_counties_merge_topo.json```
```topomerge -k 'd.id.slice(0,3)' counties=tracts < fl_albers_quantized_topo.json > fl_albers_counties_merge_topo.json```

# Use topomerge to derive only internal county borders - we don't need the county borders overlapping with the existing state border
```topomerge --mesh -f 'a !== b' counties=counties < ca_albers_counties_merge_topo.json > ca_albers_counties_internal_topo.json```
```topomerge --mesh -f 'a !== b' counties=counties < fl_albers_counties_merge_topo.json > fl_albers_counties_internal_topo.json```

# Use topo2geo to extract simplified tracts, pipe to ndjson-map to assign fill of each tract, pipe to ndjson-split to break collection into features, lastly pipe to geo2svg
```topo2geo tracts=- < ca_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0,4000]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > ca_albers_tracts_counties_color.svg```
```topo2geo tracts=- < fl_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0,4000]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > fl_albers_tracts_counties_color.svg```

# Same as before but using a square root of density (a non-linear transform) to distribute colors more equitably
```topo2geo tracts=- < ca_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0,100]), d.features.forEach(f => f.properties.fill = z(Math.sqrt(f.properties.density))), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > ca_albers_tracts_counties_color_sqrt.svg```
```topo2geo tracts=- < fl_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleSequential(d3.interpolateViridis).domain([0,100]), d.features.forEach(f => f.properties.fill = z(Math.sqrt(f.properties.density))), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > fl_albers_tracts_counties_color_sqrt.svg```

# Same as before but we could try a log transform instead of square root
```topo2geo tracts=- < ca_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleLog().domain(d3.extent(d.features.filter(f => f.properties.density), f => f.properties.density)).interpolate(() => d3.interpolateViridis), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p1 -w 960 -h 960 > ca_albers_tracts_counties_color_log.svg```
```topo2geo tracts=- < fl_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleLog().domain(d3.extent(d.features.filter(f => f.properties.density), f => f.properties.density)).interpolate(() => d3.interpolateViridis), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p1 -w 960 -h 960 > fl_albers_tracts_counties_color_log.svg```

# Same as before but using density's p-quantile instead
```topo2geo tracts=- < ca_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleQuantile().domain(d.features.map(f => f.properties.density)).range(d3.quantize(d3.interpolateViridis, 256)), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > ca_albers_tracts_counties_color_quantile.svg```
```topo2geo tracts=- < fl_albers_counties_internal_topo.json | ndjson-map -r d3 'z = d3.scaleQuantile().domain(d.features.map(f => f.properties.density)).range(d3.quantize(d3.interpolateViridis, 256)), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > fl_albers_tracts_counties_color_quantile.svg```

# Install d3-scale-chromatic for discrete color scales
```pnpm add -g d3-scale-chromatic```

# Same as before but implement a threshold scale with OrRd color scheme instead of p-quantiles or log transforms etc
```topo2geo tracts=- < ca_albers_counties_internal_topo.json | ndjson-map -r d3 -r d3_scale_chromatic=d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3_scale_chromatic.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > ca_albers_tracts_counties_color_threshold.svg```
```topo2geo tracts=- < fl_albers_counties_internal_topo.json | ndjson-map -r d3 -r d3_scale_chromatic=d3-scale-chromatic 'z = d3.scaleThreshold().domain([1, 10, 50, 200, 500, 1000, 2000, 4000]).range(d3_scale_chromatic.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features' | geo2svg -n --stroke none -p 1 -w 960 -h 960 > fl_albers_tracts_counties_color_threshold.svg```

# Add our internal county borders to our threshold scale svg
```(topo2geo tracts=- < ca_albers_counties_internal_topo.json | ndjson-map -r d3 -r d3_chromatic_scale=d3-chromatic-scale 'z = d3.scaleThreshold().domain([1,10,50,200,500,1000,2000,4000]).range(d3_scale_chromatic.schemeOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features'; topo2geo counties=- < ca_albers_counties_internal_topo.json | ndjson-map 'd.properties = {"stroke": "#000", "stroke-opacity": 0.3}, d') | geo2svg -n --stroke none -p 1 -w 960 -h 960 > ca_albers_tracts_counties_color_ylgn_final.svg```
```(topo2geo tracts=- < fl_albers_counties_internal_topo.json | ndjson-map -r d3 -r d3_chromatic_scale=d3-chromatic-scale 'z = d3.scaleThreshold().domain([1,10,50,200,500,1000,2000,4000]).range(d3_scale_chromatic.schemeYlGn[9]), d.features.forEach(f => f.properties.fill = z(f.properties.density)), d' | ndjson-split 'd.features'; topo2geo counties=- < fl_albers_counties_internal_topo.json | ndjson-map 'd.properties = {"stroke": "#000", "stroke-opacity": 0.3}, d') | geo2svg -n --stroke none -p 1 -w 960 -h 960 > fl_albers_tracts_counties_color_ylgn_final.svg```
