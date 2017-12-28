---
date: 2016-03-09T00:11:02+01:00
title: Getting started
weight: 10
---

## 1. Download Tegola
Choose the binary that matches the operating system Tegola will run on. Quick links are available below for your convenience:

- [OSX](https://github.com/terranodo/tegola/releases/download/v0.3.2/tegola_darwin_amd64)
- [Windows](https://github.com/terranodo/tegola/releases/download/v0.3.2/tegola_windows_amd64.exe)
- [Linux](https://github.com/terranodo/tegola/releases/download/v0.3.2/tegola_linux_amd64)

Additional binaries for other operating systems and versions are [here](https://github.com/terranodo/tegola/releases).

Find the Tegola file that was downloaded and move it into a fresh directory. Rename this file `tegola`.

## 2. Set up a data provider

Tegola needs geospatial data to run. Currently, Tegola supports PostGIS which is a geospatial extension for PostgreSQL. If you don't have PostGIS installed, [download PostGIS](http://postgis.net/install/).

You'll need to load your data provider with data. For your convenience, you can download [PostGIS data for Bonn, Germany](https://s3-us-west-2.amazonaws.com/tegola/bonn_osm.sql.tgz). Unzip this archive to extract the file `bonn_osm.sql`.

Create a new database named `bonn`, and use a restore command to import the unzipped sql file into the database. Documentation can be found [here](https://www.postgresql.org/docs/current/static/backup.html) under the section titled "Restoring the dump". The command should look something like this:

```sh
psql bonn < bonn_osm.sql
```

To enable the Tegola application to connect to the database, create a database user named `tegola` and grant the privileges required to read the tables in the `public` schema of the `bonn` database, using these commands:

```sh
psql -c "CREATE USER tegola;"
psql -d bonn -c "GRANT SELECT ON ALL TABLES IN SCHEMA public TO tegola;"
```

## 3. Create a configuration file

Tegola utilizes a single configuration file to coordinate with data provider(s). This configuration file is written in [TOML format](https://github.com/toml-lang/toml).

Create your configuration file in the same directory as the Tegola binary and name it `config.toml`. Next, copy and paste the following into this file:

```toml
[webserver]
port = ":8080"

# register data providers
[[providers]]
name = "bonn"           # provider name is referenced from map layers
type = "postgis"        # the type of data provider. currently only supports postgis
host = "localhost"      # postgis database host
port = 5432             # postgis database port
database = "bonn"       # postgis database name
user = "tegola"         # postgis database user
password = ""           # postgis database password
srid = 3857             # The default srid for this provider. If not provided it will be WebMercator (3857)

  [[providers.layers]]
  name = "road"
  geometry_fieldname = "wkb_geometry"
  id_fieldname = "ogc_fid"
  sql = "SELECT ST_AsBinary(wkb_geometry) AS wkb_geometry, name, ogc_fid FROM all_roads_3857 WHERE wkb_geometry && !BBOX!"

  [[providers.layers]]
  name = "main_roads"
  geometry_fieldname = "wkb_geometry"
  id_fieldname = "ogc_fid"
  sql = "SELECT ST_AsBinary(wkb_geometry) AS wkb_geometry, name, ogc_fid FROM main_roads_3857 WHERE wkb_geometry && !BBOX!"

  [[providers.layers]]
  name = "lakes"
  geometry_fieldname = "wkb_geometry"
  id_fieldname = "ogc_fid"
  sql = "SELECT ST_AsBinary(wkb_geometry) AS wkb_geometry, name, ogc_fid FROM lakes_3857 WHERE wkb_geometry && !BBOX!"

[[maps]]
name = "zoning"

  [[maps.layers]]
  provider_layer = "bonn.road"
  min_zoom = 10
  max_zoom = 20

  [[maps.layers]]
  provider_layer = "bonn.main_roads"
  min_zoom = 5
  max_zoom = 20

  [[maps.layers]]
  provider_layer = "bonn.lakes"
  min_zoom = 5
  max_zoom = 20
```

Note: This configuration file is specific to the Bonn data provided in step 2. If you're using another dataset, reference the [Configuration Documentation](/configuration).

## 4. Start Tegola

Navigate to the Tegola directory in your computer's terminal and run this command:

```sh
./tegola --config=config.toml
```

You should see a message confirming the config file load and Tegola being started on port 8080. If your computer's port 8080 is being used by another process, change the port in the config file to an open port.

## 5. Create an HTML page

Tegola delivers geospatial vector tile data to any requesting client. For simplicity, we'll be setting up a basic HTML page as our client that will display the rendered map. We'll be using the [OpenLayers](http://openlayers.org/) client side library to display and style the vector tile content.

Create a new directory called `static` in the same directory as the Tegola binary. In `static`, create a new HTML file called `index.html`, copy in the contents below, and open in a browser:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Tegola Sample</title>
    <link rel="stylesheet" href="http://openlayers.org/en/v4.3.1/css/ol.css" type="text/css">
    <script src="http://openlayers.org/en/v4.3.1/build/ol.js"></script>
    <style>
      #map {
        width: 100%;
        height: 100%;
        position: absolute;
        background: #f8f4f0;
      }
    </style>
  </head>
  <body>
    <div id="map"></div>
    <script>
      var map = new ol.Map({
        layers: [
          new ol.layer.VectorTile({
            source: new ol.source.VectorTile({
              attributions: '© <a href="https://www.mapbox.com/map-feedback/">Mapbox</a> ' +
              '© <a href="http://www.openstreetmap.org/copyright">' +
              'OpenStreetMap contributors</a>',
              format: new ol.format.MVT(),
              tileGrid: ol.tilegrid.createXYZ({maxZoom: 22}),
              tilePixelRatio: 16,
              url:'/maps/zoning/{z}/{x}/{y}.vector.pbf?debug=true'
            })
          })
        ],
        target: 'map',
        view: new ol.View({
          center: [790793, 6574927], //coordinates the map will center on initially
          zoom: 14
        })
      });
    </script>
  </body>
</html>
```

In your browser, navigate to [http://localhost:8080](http://localhost:8080). If everything was successful, you should see a map:

![Bonn, Germany](/images/bonn.png)
