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

## Parsing and understanding data

Next step is to parse the PBF file with your programming tool of choice. In my case it is NodeJS with the help of the [osm-pbf-parser](https://github.com/substack/osm-pbf-parser) module. 

Let's get started!
```
yarn add osm-pbf-parser;
```

Indiranagar contains 7682 nodes, 1824 ways and 22 relations.

```
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

    //console.log('.');
  });

  next();
}));

stream.on('error', function(err){
  console.log('error', err);
});
stream.on('finish', function(){
  console.log('finish', types);
});
```

Here is how to reproduce the results:
```
git clone https://github.com/jonmaim/indiranagar_bangalore_openstreetmap;
cd indiranagar_bangalore_openstreetmap;
yarn;
./index.js planet_77.64_12.973_4379f04c.osm.pbf 
```

