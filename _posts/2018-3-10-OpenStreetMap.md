---
layout: post
title: Parsing and understanding OpenStreetMap data
---

Our goal is to be able to obtain map data from OpenStreetMap (OSM) and then be able to detect the semantic for each zone, e.g., roads, buildings, parcs, etc. I will then be able to tag each area of a neighborhood with tags like walkable / not-walkable, inside / outside.

From my first understanding on how to do get map data, we first have to download some area XML files. Here is what looks like [a good starting point](https://wiki.openstreetmap.org/wiki/Databases_and_data_access_APIs).

## Downloading data

The [OSM downloading page](https://wiki.openstreetmap.org/wiki/Planet.osm#Downloading) explains how to download the whole map data for the planet, which is several dozens of GBs. We want to start small and only want to download a small area like Indiranagar neighborhood in the city of Bangalore, India. Fortunately there is a tool available to [extract a defined geo-fence](https://extract.bbbike.org/?sw_lng=77.64&sw_lat=12.973&ne_lng=77.648&ne_lat=12.987&format=osm.pbf&coords=77.64%2C12.973%7C77.644%2C12.973%7C77.648%2C12.974%7C77.648%2C12.979%7C77.646%2C12.987%7C77.643%2C12.986%7C77.64%2C12.985%7C77.64%2C12.979&city=Indiranagar%2C%20East%20Zone%2C%20Bengaluru%2C%20Bangalore%20Urban%2C%20Karnataka%2C%20560038%2C%20India).

![Extract area]({{site.baseurl}}/images/OSM/extract_area.png)
*Extract OpenStreetMap area interface*

The extract download link is sent by email in PBF format ([Protocolbuffer Binary Format](https://wiki.openstreetmap.org/wiki/PBF_Format)). Here is the saved link to [Indiranagar area pbf file]({{site.baseurl}}/images/OSM/planet_77.64_12.973_4379f04c.osm.pbf).

## Parsing data

Next step is to parse the PBF file with your programming tool of choice. In my case it is `node` with `yarn` module installer.
There is no need to understand the PBF format, we just need to get a module that will parse it for us and give us the different elements. There is a module called [osm-pbf-parser](https://github.com/substack/osm-pbf-parser) that will allow us to easily load a `pbf` file. 

Let's get started and install the modules.
```
yarn add osm-pbf-parser through2;
```
Now, let's iterate over all items in the `pbf` file.
``` javascript
var fs = require('fs');
var through = require('through2');
var parseOSM = require('osm-pbf-parser');

var types = {};

var osm = parseOSM();

var stream = fs.createReadStream(process.argv[2]).pipe(osm).pipe(through.obj(function(items, enc, next) {
  console.log('#', items.length);
  items.forEach(function(item) {
    types[item.type] = types[item.type] || 0;
    ++types[item.type];

    if (item.type === 'relation') {
      console.log('item=', item);
    }
  });

  next();
}));

stream.on('finish', function(){
  console.log('finish', types); 
});
```
Let's run this program.
```
node index.js planet_77.64_12.973_4379f04c.osm.pbf
```
We discover that the Indiranagar neighborhood contains 7682 nodes, 1824 ways and 22 relations.

## OpenStreetMap elements
An OSM file contains a list of elements. An element is either a node, a way or a relation.

![Longitude]({{site.baseurl}}/images/OSM/node_way_relation.png)
*OpenStreeMaps' data contains nodes, ways and relations*

### Nodes

A [node](https://wiki.openstreetmap.org/wiki/Node) is a single point in space defined by a `(longitude, latitude)` pair of coordinates. Nodes in aggreagate form the map's geometry, while ways and relations define its structure.

### Ways

A [way](https://wiki.openstreetmap.org/wiki/Way) is an ordered list of nodes and doesn't contain geometry.

### Relations

A [relation](https://wiki.openstreetmap.org/wiki/Relation) is an element with one or more tags plus an ordered list elements. Similarly to a way, a relation doesn't either contain geometry.

Ways and relations define structure around nodes and don't contain geometry.

## Understanding map data, i.e., re-rendering it.
Parsing the map data and then understand it will allow us to render it into an image tile similarly to what online map software are doing. In order to center the neighborhood on an image with given dimension, we will first compute the bounding box of the map data. 

### Bounding box
The neighborhood's bounding box is the axis-aligned rectangle which encompass all the map data. The OSM elements containing a `geo-location` are the nodes. Each node contains one `(longitude, latitude)` pair of coordinates.  

The axis-aligned bounding box contains only 4 coordinates, which is enough to define a rectangle aligned to the `(lon, lat)` system of coordinates.

To find the extreme points of a set of points, we need to iterate through each of them and check if the current point is an extreme point or not. At the end of the iteration we can be sure to have found the most extreme points as we checked each of them individually. 

```javascript
  items.forEach(item => {
    if (item.type === 'node') {
      if (item.lon < box.lonMin) { box.lonMin = item.lon; }
      if (item.lon > box.lonMax) { box.lonMax = item.lon; }
      if (item.lat < box.latMin) { box.latMin = item.lat; }      
      if (item.lat > box.latMax) { box.latMax = item.lat; }
    } 
  });
```
*Finding the coordinates of the bounding box by iterating through all nodes.*

At the beginning we have to idea about the values of our bounding box. By initialzing the values with positive or negative inifity we ensure we have the bounding box at each step of the iteration. 

```javascript
  /* axis-aligned bounding box */
  var box = { 
    lonMin: Infinity,
    lonMax: -Infinity,
    latMin: Infinity,
    latMax: -Infinity,
  };
```
*Initializing a bounding box data structure*

Indiranagar's bounding box is `{lonMin: 77.6400004, lonMax: 77.6479874, latMin: 12.973003100000001, latMax: 12.9869119}`.

Now that we have found the `(lon, lat)` bounding box, we can start maping it to a `(x, y)` bitmap.

#### Longitudes
A longitude value is the `x` coordinate between `-180` and `180`. The `0` longitude is the line passing through north pole, south pole and Greenwhich. 

![Longitude]({{site.baseurl}}/images/OSM/longitude.png)
*Longitude on a sphere*

#### Latitudes
A latitude value is the `y` coordinate between `+90` at the north pole, `0` at the equator and `-90` at the south pole. Latitude lines are formed where the latitude stays constant.

![Latitude]({{site.baseurl}}/images/OSM/latitude.png)
*Latitude on a sphere*

![A bitmap `(x, y)` coordinate system]({{site.baseurl}}/images/OSM/bitmap.png)
*A bitmap `(x, y)` coordinate system*

```javascript
var bitmap = {
  width: 1024,
  height: 1024,
};
```
Mapping `(lon, lat)` points is done without worrying about the curvature of Earth. And thus we don't need to use a complex projection like the Mercator projection. Indeed, when working with a map with a small surface, we can consider applying `(lon, lat)` onto a `2D` plan and it should be just fine. In our case, the plan is a bitmap whose origin point is the top-left corner. From that origin, `x` axis goes toward the right and `y` axis goes toward the bottom.

![Mapping `(lon, lat)` into a `(x, y)` bitmap]({{site.baseurl}}/images/OSM/map_bb_onto_bitmap.png)
*Mapping `(lon, lat)` into a `(x, y)` bitmap: `y` values are inverted*

```javascript
var mapping = {
  deltaLonMap: box.lonMax - box.lonMin,
  deltaLatMap: box.latMax - box.latMin,
  lon: n.lon,
  lat: n.lat
};
mapping.xNormalized = (mapping.lon - box.lonMin) / mapping.deltaLonMap;
mapping.yNormalized = 1 - ((mapping.lat - box.latMin) / mapping.deltaLatMap);
mapping.x = mapping.xNormalized * bitmap.width;
mapping.y = mapping.yNormalized * bitmap.height;
```
*Each node needs to be mapped onto the bitmap to find its pixel coordinate*

![Left: result of mapping `(lon, lat)` values on a bitmap; right: result as seen on OpenStreetMap website.]({{site.baseurl}}/images/OSM/re_render01_compare.png)
*Left: result of mapping `(lon, lat)` values of nodes on a bitmap; right: result as seen on OpenStreetMap website.*

Our bitmap is of size `1024x1024`, thus a square. However, the Indiranagar's neighborhood has a height approximately two times as large as its width. 


![Left: result of mapping `(lon, lat)` and taking bounding box ratio into account; right: result as seen on OpenStreetMap website.]({{site.baseurl}}/images/OSM/re_render02_compare.png)
*Left: result of mapping `(lon, lat)` and taking bounding box ratio into account; right: result as seen on OpenStreetMap website.*
