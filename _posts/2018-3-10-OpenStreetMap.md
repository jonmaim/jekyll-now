---
layout: post
title: Parsing and understanding OpenStreetMap data
---

Our goal is to be able to obtain map data from OpenStreetMap (OSM) and then be able to detect the semantic for each zone, e.g., roads, buildings, parcs, etc. I will then be able to tag each area of a neighborhood with tags like walkable / not-walkable, inside / outside.

From my first understanding on how to do get map data, we first have to download some area XML files. Here is what looks like [a good starting point](https://wiki.openstreetmap.org/wiki/Databases_and_data_access_APIs).

## Downloading data

The [OSM downloading page](https://wiki.openstreetmap.org/wiki/Planet.osm#Downloading) explains how to download the whole map data for the planet, which is several dozens of GBs. We want to start small and only want to download a small area like Indiranagar neighborhood in the city of Bangalore, India. Fortunately there is a tool available to [extract a defined geo-fence](https://extract.bbbike.org/).

![Extract area]({{site.baseurl}}/images/OSM/extract_area.png)
*Extract OpenStreetMap area interface*

The extract download link is sent by email and is in PBF format ([Protocolbuffer Binary Format](https://wiki.openstreetmap.org/wiki/PBF_Format)). Here is the saved link to [Indiranagar area pbf file]({{site.baseurl}}/images/OSM/planet_77.64_12.973_4379f04c.osm.pbf).

## Parsing data

Next step is to parse the PBF file with your programming tool of choice. In my case it is NodeJS with the help of the [osm-pbf-parser](https://github.com/substack/osm-pbf-parser) module.
