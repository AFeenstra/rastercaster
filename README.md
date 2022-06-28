# RasterCaster
**Make DEMs and other rasters from polygons with a specific gradient definition for each polygon.**

_Made by Leendert van Wolfswinkel, Nelen & Schuurmans in partnership with Deltares (2018-2019)_

## Introduction
The RasterCaster allows the user to create digital elevation models geotiffs from vector data (points, lines and polygons). Elevation is a continous variable and therefore better represented by a raster than polygons or other vector formats. However, raster data is much harder to edit than vector data. RasterCaster bridges this gap. The user makes a set of polygons that cover the area of interest and do not overlap. The polygons may be hand-drawn directly into GIS, but can also be derived from landscape desings (usually CAD drawings). 

The polygons can be seen as ‘molds’, that are used to ‘cast’ the elevation map into. Each polygon contains a definition of the altitude variations within it. Simple elevation profiles (e.g. the flat floor level within a building) can be cast using a simple definition, while at the same time the RasterCaster offers the user all flexibility to add more complex profiles (e.g. a sloping road with a convex cross section). Currently, molds can be defined in four different way - see under Definition Types.

The RasterCaster lives in a PostGIS database, so you need to install PostGIS or have access to an existing PostGIS database. All vector data are stored there. The first time you _cast_, for each polygon a raster is created and stored in the database. The next time you cast, rasters are only generated for polygons that have been edited (geometry and/or attributes). All data edits are done directly in the PostGIS tables (using QGIS for example). Installing the RasterCaster into your database, casting your raster and a few other things are done via a Python script that takes command line arguments. Depending on available funding, a QGIS plugin may be developed in the future. 

## Requirements
Postgres >= 10.0

PostGIS >= 2.2

Python 2 or 3

QGIS 3.x


Currently only the Dutch projection RD New (EPSG:28992) is supported.

## Command line user interface

The below command line examples assume that you first `cd` to the folder where you cloned the rastercaster to, in Command Prompt or OSGeo4W. 

### Install
1. Clone or download the GitHub repo
2. Copy settings.ini and fill with the values specific to your project.
3. If your database server uses PostGIS 3.x, you will need to manually activate postgis_raster. Execute the following query `CREATE EXTENSION postgis_raster;`
4. Run the following command:

```
python rastercaster.py -install path\to\your\settings.ini
```

### Cast
Exports the raster from the database to the output file location specified in the .ini file.

```
python rastercaster.py -cast path\to\your\settings.ini
```

### Autogenerate auxiliary lines
Fills the table rc.auxiliary lines with the main axis and sides of a polygon. 

Works well for elongated polygons like roads or waterways. Performance is less for weirdly shaped polygons.

```
python rastercaster.py -aux path\to\your\settings.ini
```
## Tables
After installation, the database contains an 'rc' schema. The tables in this schema represent the input data for the RasterCaster.

### Surface
The rc.surface table contains the polygons that are converted to rasters by the RasterCaster. The elevation gradient within a surface is defined by the attributes of the polygon, in a way that depends on the definition_type (see below). This table has the following columns:

Column | Data type | Description | Remarks
-------|-----------|-------------|-------------
id | integer | Unique identifier | Filled automatically if left blank
definition | text | SQL definition for custom elevation gradients | Defaults to 'NULL'
definition_type | text | See Definition Types | NOT NULL, defaults to 'custom'
comment | text | user-defined comment | No restrictions, NULL allowed
geom | geometry (Polygon in EPSG:28992) | area for which to generate the raster | NOT NULL, must be a valid geometry > pixel size
aux_1 .. aux_6 | integer | References to the id of auxiliary lines | Referenced aux line must exist if used 
param_1 .. param_6 | double precision | Parameters to use in custom definitions, param_1 is used in constant definitions | NOT NULL if used
   
### Elevation Point
You must place this within the elevation_point_search_radius (in the .ini) of a line. If you want to make sure that the point is included, turn on Snapping in QGIS.

The rc.elevation_point table has the following columns:

Column | Data type | Description | Remarks
-------|-----------|-------------|-------------
id | integer | Unique identifier | Filled automatically if left blank
comment | text | user-defined comment | No restrictions, NULL allowed
elevation | double precision | Elevation (m.a.s.l.)
geom | geometry (Point in EPGS:28992) | Location of the elevation point | Point without z-coordinate
geom_3d | geometry (PointZ in EPGS:28992) | Autogenerated Point with Z coördinate taken from the 'elevation' field | Do not fill this field.
in_polygon_only | boolean | If 'true' the elevation point applies only to the surface in which the point is located. Applies only to definition type 'tin' | Defaults to false.

### Auxiliary Line
You can use the rc.auxiliary_line table to define complex elevation gradients.

If you want to create a complicated elevation gradient or your source data contains too little detail (in rc.surface) then you can choose to draw auxiliary lines. For example, if you want to create a road with a convex cross section, you can define the elevation of the road based on the distance between the pixel and the middle of the road. The auxiliary line should then be located in the middle of the road, and your custom definition could be `ST_Distance(pixel, aux_1)*0.01`. Auxiliary lines can be automatically generated on the command line, see Autogenerate Auxiliary Lines.

This table has the following columns:

Column | Data type | Description | Remarks
-------|-----------|-------------|-------------
id | integer | Unique identifier | Filled automatically if left blank
comment | text | user-defined comment | No restrictions, NULL allowed
autogenerated | boolean | True if generated with -aux from the command line | Defaults to False
geom | geometry (MultiLinestringZ in EPGS:28992) | Location and elevation of the line | If filling with a SQL query, use `ST_Multi(ST_Force3D(geom))`

## Definition Types
The definition_type field in the rc.surface table determines how the elevation gradient of the relevant polygon is defined. The following types can currently be entered in that field:

### 'constant'
The same elevation in the entire polygon. Enter the desired elevation in the column _param_1_. A usecase can be defining floor levels of buildings in the DEM.

### 'tin'
Elevation data is retrieved from elevation_points that lie on or near the inner or outer edge of the polygon. The elevation is specified in the elevation column of the elevation_point. An elevation_point is included in the determination of the elevation gradient if it is closer to the edge than the elevation_point_search_radius in the settings. If you want two adjacent polygons to have a different elevations at their shared boundary (for example at a curb), you can place the elevation point in the polygon (close enough to the edge) and set the attribute in_polygon_only to True.

### 'filler'
The elevation gradient is interpolated based on the surrounding raster values. The surrounding raster is sampled along the edge at an interval determined by filler_seg_dist in settings.ini.

### 'custom'
This definition type can be used to define complex profiles, such as a road or waterway with a convex or concave cross section and a longitudinal gradient. The elevation is determined by a formula that is specified in the `definition` field. The language of the formula is PostgreSQL. The following terms are treated in a special way:

- pixel: the dot representation of the pixel in question
- geom: the geometry of the surface polygon
- aux_1 to aux_6: the auxiliary geometry from the rc.auxiliary_line table associated with the surface polygon
- param_1 to param_6: these constants allow you to apply the same definition to many polygons, but with different parameters per polygon

Examples of custom definitions can be found in the [RasterCaster Mold Library] (https://docs.google.com/spreadsheets/d/1nrRuSO89Rfs1AuXtvw6QpmeLq1P5CMJCD1IZiEAK1dg/edit?usp=sharing).
