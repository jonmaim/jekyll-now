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

Next step is to parse the PBF file with your programming tool of choice. In my case it is `node` with `yarn` module installer. There is a module called [osm-pbf-parser](https://github.com/substack/osm-pbf-parser) that will allow us to easily load a `pbf` file. 

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

### Nodes

| [![OSM node]({{site.baseurl}}/images/OSM/node.png)](https://wiki.openstreetmap.org/wiki/Node) | A node is a single point in space defined by a `(latitude, longitude)` pair of coordinates. Nodes in aggreagate form the map's geometry, while ways and relations define its structure. |

### Ways

| [![OSM node]({{site.baseurl}}/images/OSM/way.png)](https://wiki.openstreetmap.org/wiki/Way) | A way is an ordered list of nodes and doesn't contain geometry. |

### Relations

| [![OSM relation]({{site.baseurl}}/images/OSM/relation.png)](https://wiki.openstreetmap.org/wiki/Relation) | A relation is an element with one or more tags plus an ordered list elements. Similarly to a way, a relation doesn't either contain geometry. |

Ways and relations define structure around nodes and don't contain geometry.

## Understanding map data, i.e., re-rendering it.
Parsing the map data and then understand it will allow us to render it into an image tile similarly to what online map software are doing. In order to center the neighborhood on an image with given dimension, we will first compute the bounding box of the map data. 

### Bounding box
The neighborhood's bounding box is the axis-aligned rectangle which encompass all the map data. The OSM elements containing a `geo-location` are the nodes. Each node contains one `(longitude, latitude)` pair of coordinates.  

The axis-aligned bounding box contains only 4 coordinates, which is enough to define a rectangle aligned to the `(lon, lat)` system of coordinates.

```javascript
  /* axis-aligned bounding box */
  var box = { 
    lonMin: Infinity,
    lonPos: -Infinity,
    latMin: Infinity,
    latPos: -Infinity,
  };
```
Finding the coordinates of the bounding box by iterating through all nodes.

```javascript
  items.forEach(item => {
    if (item.type === 'node') {
      if (item.lon > box.lonPos) { box.lonPos = item.lon; }
      if (item.lat > box.latPos) { box.latPos = item.lat; }
      if (item.lon < box.lonMin) { box.lonMin = item.lon; }
      if (item.lat < box.latMin) { box.latMin = item.lat; }
    } 
  });
```
Indiranagar's bounding box is `{lonMin: 77.6400004, lonPos: 77.6479874, latMin: 12.973003100000001, latPos: 12.9869119}`.

Now that we have found the bounding box in `(lon, lat)`, we have to map it to a bitmap with a `(x, y)` coordinate's system.

#### Longitudes
A longitude value is the `x` coordinate between `-180` and `180`. The `0` longitude is the line passing through north pole, south pole and Greenwhich. 

#### Latitudes
A latitude value is the `y` coordinate between `+90` at the north pole, `0` at the equator and `-90` at the south pole. Latitude lines are formed where the latitude stays constant.

![Longitude and latitude on a sphere]({{site.baseurl}}/images/OSM/lon_lat.png)
*Longitude and latitude on a sphere*

![A bitmap `(x, y)` coordinate system]({{site.baseurl}}/images/OSM/bitmap.png)
*A bitmap `(x, y)` coordinate system*

![Mapping `(lon, lat)` into a `(x, y)` bitmap]({{site.baseurl}}/images/OSM/map_bb_onto_bitmap.png)
*Mapping `(lon, lat)` into a `(x, y)` bitmap: `y` values are inverted*


![Left: first visual result of mapping `(lon, lat)` values on a bitmap; right: result as seen on OpenStreetMap website.]({{site.baseurl}}/images/OSM/re_render01_compare.png)
*Left: first visual result of mapping `(lon, lat)` values on a bitmap; right: result as seen on OpenStreetMap website.*

