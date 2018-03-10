---
layout: post
title: Parsing and understanding OpenStreetMap data
---

Our goal is to be able to obtain map data from OpenStreetMap (OSM) and then be able to detect the semantic for each zone, e.g., roads, buildings, parcs, etc. I will then be able to tag each area of a neighborhood with tags like walkable / not-walkable, inside / outside.

From my first understanding on how to do get map data, we first have to download some area XML files. Here is what looks like [a good starting point](https://wiki.openstreetmap.org/wiki/Databases_and_data_access_APIs).

## Downloading data

The [OSM downloading page](https://wiki.openstreetmap.org/wiki/Planet.osm#Downloading) explains how to download the whole map data for the planet, which is several dozens of GBs. We want to start small and only want to download a small area like Indiranagar neighborhood in the city of Bangalore, India. Fortunately there is a tool available to [extract a defined geo-fence](https://extract.bbbike.org/).

![Extract area]({{ site.baseurl }}/images/OSM/extract_area.png)

The extract download link is sent by email and is in PBF format ([Protocolbuffer Binary Format](https://wiki.openstreetmap.org/wiki/PBF_Format)).


