
# Loading OSM Data With osm2pgsql   
JinXu
2018-11-28   


## Introduction 
***
OpenStreetMap (OSM) is a collaborative mapping project to create free editable map of the world. User draw maps with GPS, RS images and other free source data. It is a typical source of VGI (Volunteered Geographic Information) providing up-to-date data as an alternative for authoritative geographic data.  
OSM elements, i.e. features on the map, are stored in the database as nodes, ways, and relations:  

- Nodes are basically points on the map. It could be independent points such as points of interest (POI), or points on a line, such as the traffic light.   
- Ways are lines representing independent lines when it is not closed (such as roads), or polygons when it's closed with tags (such as buildings with tag `building = yes`).  
- Relations are logical or spatial relations among OSM elements (whether you can turn right on a road, polygon with a hole, a set of polygons and nodes as one element, etc).    

OSM2pgsql is a tool to import OSM data into PostGIS database. It¡¯s already installed with the OSGeolive. It can convert OSM data to slim and non-slim mode. The slim mode will keep the data structure as node, ways, and relations, but non-slim mode will transfer data into points, lines, and polygons. In this tutorial, we are only dealing with the non-slim mode because it has geometry column.    
![UMLclassOf-osm2pgsql-schema.png](https://wiki.openstreetmap.org/w/images/b/bf/UMLclassOf-osm2pgsql-schema.png)  
The data that we are using here is in pbf format, which is a compressed format similar to xml for geographic data, but it takes less space. Although the OSM database is unstructured, namely it stores attributes with tags other than columns, we can use hstore when loading it with osm2pgsql.  

## Load Data
***
Importing OSM data is based on this webpage: [SWITCH2OSM](https://switch2osm.org/loading-osm-data/), but I skipped some unnecessary steps and made some changes. You can take a look if you are interested.   

### Data  
Original data can be downloaded from [Geofabrik](http://download.geofabrik.de/) where OSM data are updated daily.  
**But** they are usually large files that take a long time to import. So I clipped the data by a bounding box (N:40.1, S:39.9, E:-75.0, W:-75.2) for only Philadelphia.    
**FOR THE PURPOSE OF THIS TUTORIAL, PLEASE DOWNLOAD DATA FROM HERE:[Philadelphia181108.pb](https://drive.google.com/drive/folders/1_UkHzM2ehxoHqdHX0IziDY8WI96Lapn2?usp=sharing)** Put the file in your share folder so that you can have access to it in your virtual machine. Run the virtual machine and drag the file to the desktop to make your life easier.    
**The next steps will all be done on the Linux Terminal.** These codes can not be copied, so you have to type these codes on the black window. To avoid misunderstanding and mistyping, here¡¯s some explanation for some symbols in the codes:    

- `$` : The beginning of the code. It will come after your username so you **DO NOT** have to type it. I just added it here to make it clear that it¡¯s the beginning of the code.  
- `>` : Continuing the code in the previous line. It will appear when you wrap your code to the next line. **DO NOT** type it.   
- `\` : To continue the code in the next line. You **NEED** to type it at the end of the line to wrap the code. But remember **DO NOT** type blank space after typing it, otherwise the code can not be wrapped.    

Remember, Linux Shell is **CASE SENSITIVE.**    

### Getting Ready    
#### Check Version    
Type only the lines after `$` and it will show the result of checking versions. You need to make sure you have **at least** these versions:    
```{bash, eval=FALSE, include=TRUE}
$ proj
Rel. 4.8.0, 6 March 2012
usage: proj [ -beEfiIlormsStTvVwW [args] ] [ +opts[=arg] ] [ files ]
$ psql --version
psql (PostgreSQL) 9.3.4
geos-config --version
3.4.2
$ osm2pgsql --version
osm2pgsql SVN version 0.85.0 (64bit id space)
$ osmconvert -h | head -n2
osmconvert 0.7T  Parameter Overvie
``` 

#### Changing Overcommit (optional)    
This step is optional. It makes the import faster.     
```{bash, eval=FALSE, include=TRUE}
$ sudo tee /etc/sysctl.d/60-overcommit.conf <<EOF
> vm.overcommit_memory=1
> EOF
$ sudo sysctl -p /etc/sysctl.d/60-overcommit.conf
```

### Loading OSM Data   
#### Create Database   
It creates a database called osmphl, and adds the postgis and hstore extensions. The hstore extension is important here because OSM database are unstructured.   
```{bash, eval=FALSE, include=TRUE}
$ createdb osmphl
$ psql -d osmphl -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;' 
```

- `createdb` create database. Put the name of database after it.   
- `-d | --database` database. You can also change the '-d' into '--database' in the code. Put the name of the database you just created here   
- `-c | --create` create. You can also change the '-c' into '--create' in the code. The line behind it is SQL query to create extension.   

If you open your pgAdmin and refresh the server now, it will appear in the database. Make sure hstore is in the extension.    

#### Import    
This code import the pbf file into the database that we just created.   
```{bash, eval=FALSE, include=TRUE}
$ osm2pgsql -c -d osmphl -C 15000 --hstore --extra-attributes --multi-geometry \
> ~/Desktop/Philadelphia181108.pbf
```
- `-c | --create` it create a table in the database.   
- `-d | --database` to specify which database should the data be imported. 
- `-C | --cache` Capital C. It assign some space for importing data. Number after it is the amount of cache assigned. Maximum number is 30,000.   
- `-k | --hstore` import tags as hstore. It's important to include this one.   
- `--extra-attributes` to keep metadata such as version.   
- `--multi-geometry` to keep the multi-geometry in the data. For example, polygon with a hole will not be separated as two polygons.   

Now refresh the local server on your pgAdmin and take a look at the public schema in the database.    There are 4 tables added into public schema. The points, lines, polygons are OSM features, and roads is duplicating some lines that are tags as roads.    
![importpbf.png](G:\OSMtutorial\importpbf.png)   

## Simple Manipulation   
***
These steps are done in SQL so we don¡¯t need the Linux Terminal anymore and you can copy the code into pgAdmin or QGIS.    

### Simple Select   
Take a look for how it looks like:   
```{sql, eval=FALSE, include=TRUE}
SET search_path TO public;
SELECT * FROM planet_osm_line;
```
Notice that he tables that we just imported do not have PKs, and you will notice that there are some repeating ids. Although OSM id is the PK in the database, osm2pgSQL will split some features and the OSM id here is not the PK.   
The way column here is the geometry and the tags records the key and value. Notice that the user and uid keys are missing values. That¡¯s because the data that we are dealing with are from a German website under the EU data protection regulation.    

### Select Hstore   
The tags are stored as hstore. It¡¯s store the value of keys rather than creating columns. The default style for importing OSM data into postgres will create several columns for some keys and store the rest into hstore. Therefore, we can not select as how we usually do if we want to select by the value of a tag.    
Here¡¯s an example for selecting the top 10 roads with the most updates. The tag ¡®version¡¯ represent the time of updates. Once a feature is updated, its version will be added for one more.    
```{sql, eval=FALSE, include=TRUE}
SET search_path TO public;
SELECT osm_id, name,
       CAST(tags -> 'osm_version' as int) AS osm_version, 
       way
FROM planet_osm_roads
WHERE osm_id > 0 
ORDER BY CAST(tags -> 'osm_version' as int) DESC
LIMIT 10
;
```

- `tag -> key` select data based on the key of the tag.   
- `cast(column, AS datatype)` change datatype in the output.   

You can also find help page for hstore on [Postgresql help](https://www.postgresql.org/docs/10/hstore.html).

### Map   
Mapping is done in QGIS.   
Open QGIS and set connect to the database that we just created. In this case it¡¯s called osmphl.   
![connectdb.png](G:\OSMtutorial\connectdb.png)   
Open the SQL scrip and copy the lines that we typed in **Select Hstore** section and set the geometry as way.   
![top10road.png](G:\OSMtutorial\top10road.png)   
Load.   
![top10roadload.png](G:\OSMtutorial\top10roadload.png)   
Here¡¯s where these 10 roads with the most updates locate.   

### Negative OSM ID   
You might notice that there is a 'where' clause in the SQL statement above. If we delete the WHERE line, the result will be different. There will be some records in the table with negative OSM ids.   
![negativeid.png](G:\OSMtutorial\negativeid.png)   
Do they exist? Let¡¯s map it out in the QGIS:   
![negativemap.png](G:\OSMtutorial\negativemap.png)   
What are they? Let¡¯s take ¡®162109¡¯, ¡®224951¡¯, and ¡®417844¡¯ as an example and take a look on the [Overpass Turbo](https://overpass-turbo.eu/). You can select OSM elements here through Overpass API.   
![162109.png](G:\OSMtutorial\162109.png)   
![224951.png](G:\OSMtutorial\224951.png)   
![417844.png](G:\OSMtutorial\417844.png)   
They are relations.   
The tool osm2pgsql will split the boundary of relations and make it into negative to indicate that it¡¯s actually not a road but the boundary of a relation.   

## If You Are Interested   
***
**This part of the workbook is not going to be in the tutorial.** But you can also take a look if you are interested.   

### Slim Mode   
As I mentioned in the introduction, OSM data can be converted in two modes, i.e. slim and non-slim. We are dealing with the slim mode in this tutorial but what about slim mode?   
Load OSM data for the slim mode with a additional ¡®-s | --slim¡¯ in the code after creating database:   
```{bash, eval=FALSE, include=TRUE}
$ osm2pgsql -c -d osmtest -C 15000 --hstore --extra-attributes --multi-geometry \
> ~/Desktop/philadelphia.osm.pbf  
```
Three tables are created in the database. Records in these tables don¡¯t have repeated id but they also missing geometry. Moreover, they store the tags into text strings.   

### Clipping Data With osmconvert   
The data we are using in this tutorial is clipped from OSM PA data by the bounding box with osmconvert. It¡¯s a multipurpose tool for OSM data. It¡¯s not related to osm2pgsql so I did not included in the tutorial. But you can try it if you are interested. More command can be found in a [blog here](https://www.mankier.com/1/osmconvert).   
You might need to install a set of tools in the Linux Terminal first before using it:   
```{bash, eval=FALSE, include=TRUE}
$ sudo apt-get install osmctools
```
To clip the area with bounding box, you can do this:   
```{bash, eval=FALSE, include=TRUE}
$ osmconvert /media/sf_SHAREFOLDERS_Linux_OSGEO/history-latest.osm.pbf \
> -b=116.08,39.69,116.71,40.18 -o=beijinghistory.pbf
```

- This code is an example for clipping data within Beijing 6th Ring highway from OSM history dump    stored in my sharefolder.   
- `-b` means bounding box, put the coordinate in this order: W,S,E,N   
- `-o` is the output file.   

### Export as csv Data With osmconvert   
Osmconvert can also be used in exporting only intended data in a csv format, but it might not contain geometry.   
```{bash, eval=FALSE, include=TRUE}
$ osmconvert /media/sf_SHAREFOLDERS_Linux_OSGEO/history-latest.osm.pbf \
> --out-csv --csv="@id @user @uid @timestamp @version" --csv-headline --csv-separator=, \
> -o=beijinghistory.csv
```

- This code is an example for exporting csv data within Beijing 6th Ring highway from OSM history dump stored in my sharefolder.   
- `--csv-headline` is followed by the header in the csv    
- `--csv-separator=` defines what is the separator for csv columns. In this case we used comma as the separator.   

The import of csv file can be done with psql. Type the first line before continuing the second line.   
```{bash, eval=FALSE, include=FALSE}
$ psql osmhistory_planet
  copy beijingh from '/media/jin/Elements/OSMHistoryDump/beijingh.csv' delimiter ',' csv header;
```

- This code is an example for importing csv data within Beijing 6th Ring highway to a database preciously created called osmhistory_planet.   
- `delimiter` is used to specify the separator for columns. In this case we used comma as the separator.   
