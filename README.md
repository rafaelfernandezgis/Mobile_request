Migration of services to new authentication schema (Coresight 2.0)
====================

*[15th May 2017]*

We are going to show the general process to migrate from Coresight 1.0 to Coresight 2.0. One of the big changes in 2.0 is the new authentication schema. Here, we'll apply that migration process to the *"Infrastructure"* service but it is applicable to any other service. 

On this page:


* [Auth0](#markdown-header-auth0)
	* [Manually](#manually)

	* [By signup](#markdown-header-by-signup)
	
	* [User management site](#markdown-header-user-management-site)

* [Code thread from coresight-geoserver.js](#markdown-header-code-thread-from-coresight-geoserver.js)

* [PostGIS](#markdown-header-postgis)

* [Geoserver](#markdown-header-geoserver)

	1. [wmsRasterType name for infrastructure](#wmsrastertype-name-for-infrastructure)

	2. [Add infrastructure in main.js](#markdown-header-add-infrastructure-in-mainjs)

	* [Migrate the table in PostGIS and the layer in geoserver](#markdown-header-migrate-the-table-in-postgis-and-the-layer-in-geoserver)

	* [wmsRasterType name for pipeline](#markdown-header-wmsrastertype-name-for-pipeline)
	
	* [The pipelineGeometryPromise](#markdown-header-the-pipelinegeometrypromise)

	* [Modifying the 15AUG_PS2_RGB_p2.tif raster file](#markdown-header-modifying-the-15aug_ps2_rgb_p2.tif-raster-file)

	* [Publishing the 15AUG_PS2_RGB_p2.tif raster file](#markdown-header-publishing-the-15aug_ps2_rgb_p2.tif-raster-file)
	
		7.1 [Adding a new store in geoserver](#markdown-header-adding-a-new-store-in-geoserver)
		
		7.2 [Adding the layer for the 15AUG_PS2_RGB_p2 tif raster file](#markdown-header-adding-the-layer-for-the-15aug_ps2_rgb_p2-tif-raster-file)
		
		7.3 [Container considerations](#markdown-header-container-considerations)

- - -


# Auth0

We are using [external][auth0] services for authentication. It is based is in emails where we inject metadata. There are 3 ways to create them:
 
## Manually

You'll need to setup the *app_metadata* and *user_metadata* for any specific service. Something like this:

![Alt text](https://c1.staticflickr.com/5/4181/34532381145_b3a2710e48_b.jpg "User_metadata and app_metadata in Auth0")

## By signup

We use the last release of Lock.js to let users create their own accounts (only demos so far). More information about how it works [here][lock].
Since we are using demos, you'll still need to create a rule in Auth0 like this:

![Alt text](https://c1.staticflickr.com/5/4190/34371571402_9cc3a25d28_z.jpg "rule in Auth0")

## User management site

We already have an user management site working, but it still needs to be modified by using the API we are building.

![Alt text](https://c1.staticflickr.com/5/4186/33743145983_7873be4dc6.jpg "user management site")

- - -

# Code thread from coresight-geoserver.js

First thing you'll need to do is to take a look at the *coresight-geoserver.js* file in our project. There you'll find the definiton of a special object that the project will use for building the service. For the *Infrastucture* service, you'll be able to see:

![Alt text](https://c1.staticflickr.com/5/4167/34152606120_52c9ea6700_c.jpg "coresight-geoserver.js")

Into the object we have different functions, usually the first one will be the wms geoserver is serving, so in this case *"QueryInfrastructureCountingRaster"*. We'll change this name later for one more properly; but now, following this thread, we'll take a look at geoserver looking for the *"QueryInfrastructureCountingRaster"* layer.

![Alt text](https://c1.staticflickr.com/5/4170/34538253935_1669225519_c.jpg "Geoserver layer")

You'll realize that layer is actually calling to the table *"InfrastructureCounting"* in the PostGIS *coresight* database.

![Alt text](https://c1.staticflickr.com/5/4184/34408261061_028c25a81f_c.jpg "PostGIS coresight database")

This is the general workflow for all our services. The whole solution works in the same way by following these rules.
- - -

# PostGIS

Once we have the table identified into the *coresight* database, we are going to create a replica in *coresight2*, with some considerations about the convection names.

```SQL
-- Table: public."Infrastructure"

-- DROP TABLE public."Infrastructure";

CREATE TABLE public."Infrastructure"
(
    "UID" bigint NOT NULL,
    "timestamp" timestamp with time zone NOT NULL,
    wkb_geometry geometry,
    name character varying(50) COLLATE pg_catalog."default" NOT NULL,
    description character varying(75) COLLATE pg_catalog."default",
    CONSTRAINT "InfrastructurePkey" PRIMARY KEY ("UID")
)
WITH (
    OIDS = FALSE
)
TABLESPACE pg_default;

ALTER TABLE public."Infrastructure"
    OWNER to postgres;

-- Index: InfrastructureGeom_idx

-- DROP INDEX public."InfrastructureGeom_idx";

CREATE INDEX "InfrastructureGeom_idx"
    ON public."Infrastructure" USING gist
    (wkb_geometry)
    TABLESPACE pg_default;

-- Index: InfrastructureTimestamp_idx

-- DROP INDEX public."InfrastructureTimestamp_idx";

CREATE INDEX "InfrastructureTimestamp_idx"
    ON public."Infrastructure" USING btree
    (timestamp)
    TABLESPACE pg_default; 		
```
Once we have ready the *"Infrastructure"* table in coresight2 database, you'll need to populate it copying its records from the old *"InfrastructureCounting"* table by using a script like this:

```SQL
pg_dump -C --file "C:\Rafael\backup\postgres\infra.sql" --host "coresight-02.chtbrszez1rh.us-west-2.rds.amazonaws.com" --port "5432" --username "postgres" --verbose --role "postgres" --format=p --data-only --encoding "UTF8" --table "public.\"InfrastructureCounting\"" "coresight"
```
You'll still need to modify the SQL you just generated:
1. Comment the line in **```SET idle_in_transaction_session_timeout = 0;```**
2. Comment  the line in **```SET row_security = off;```**
3. Change the table and column names in **```COPY "Infrastructure" ("UID", "timestamp", wkb_geometry, "name", "description") FROM stdin;```**
4. Change the table name in **```SELECT pg_catalog.setval('"Infrastructure_UID_seq"', 10003, true);```**

Also create this sequence:

```SQL
CREATE SEQUENCE public."Infrastructure_UID_seq"
    INCREMENT 1
    START 1
    MINVALUE 1
    MAXVALUE 9223372036854775807
    CACHE 1;

ALTER SEQUENCE public."Infrastructure_UID_seq"
    OWNER TO postgres;
```

And then launch the next script in order to populate the table you just created:

```SQL
psql --host "coresight-02.chtbrszez1rh.us-west-2.rds.amazonaws.com" --port "5432" --username "postgres" --dbname "coresight2" -f "C:\Rafael\backup\postgres\infra.sql" -L "C:\Rafael\backup\postgres\logfile.log"
```
With this last step done, you'll have the *"Infrastructure"* table created and populated in the *coresight2* database.


![Alt text](https://c1.staticflickr.com/5/4159/33746568783_d1df71eb30_z.jpg "PostGIS coresight2 Infrastructure table")

---

# Geoserver

In this point, we need to start migrating layers from [geoserver][geoserver1] in ```coresight 1.0``` server to [geoserver][geoserver2] in ```coresight 2.0```.

Therefore we need a flow to keep going the thread of the changes. You will always follow the Chrome console, it'll let you know what is next.

We are changing the infrastructure services since it is the hardest one but again, it is totally applicable for any other service.

>1. Change the WMS name in the object we are trying to initialize in **coresight-geoserver.js**.
>2.  Add *infrastructure* into the products function *"infrastructure: {'module': Infrastructure, 'client': client.infrastructure}"* in **main.js**.
>3. Change any other layer in the same way we already did for *infrastructure*. Following the console, we know we also need to change the pipeline object, which is using the *"QueryInfrastructureTrailersVegetationPipelineRaster"* layer in ```coresight 1.0``` and the *"PipelineSegment"* table in PostGIS.
>4. Once the previous step is done, change the *wmsRasterType* name in **coresight-geoserver.js** for the pipeline object. Must be now *InfrastructurePipelineWMS*  instead *QueryInfrastructureTrailersVegetationPipelineRaster*. The infrastructure service is a very special one. It requires an extra work as we just saw . The main app needs the pipeline layer already created before the user 'calls' the infrastructure service. This is because the service has two sources of geometries, one of them is a pipeline.
>5. This service also needs feeding from some data before its initialization in order to create the graphs and the circular buffer around a *map click*, so we would need to modify **main.js** and **infrastructure.js** again.
>6. As a complementary basemap layer, we need to modify the raster *15AUG_PS2_RGB_p2.tif* in geoserver.
>7. Finally we'll publish the raster file

## wmsRasterType name for infrastructure
First function into the object is the WMS feature. It gets the raster that we usually see on the map:

Object		  | WMS in the map
------------- | -------------
![WMS Raster object](https://c1.staticflickr.com/5/4176/33761756743_da07395250.jpg "WMS Raster object")  | ![WMS Raster on the map](https://c1.staticflickr.com/5/4162/34185925110_1456a6c19b.jpg "WMS Raster on the map")

## 2. Add infrastructure in main.js

![Add infrastructure](https://c1.staticflickr.com/5/4187/34410833132_1653710973_z.jpg "Add infrastructure")

## 3. Migrate the "PipelineSegment" table in PostGIS and the layer "QueryInfrastructureTrailersVegetationPipelineRaster" in geoserver

We are going to follow the same method than we already used in PostGIS section.

--- Create the table *"Pipeline_infrastructure"* in ```coresight 2.0``` by using the next script:

```sql
-- Table: public."Infrastructure_pipeline"

-- DROP TABLE public."Infrastructure_pipeline";

CREATE TABLE public."Infrastructure_pipeline"
(
    ogc_fid integer NOT NULL,
    wkb_geometry geometry(LINESTRING,4326),
    actualresidentialcount integer,
    actualcommercialcount integer,
    calculatedresidentialcount integer,
    calculatedcommercialcount integer,
    CONSTRAINT infrastructure_pipelinePkey PRIMARY KEY (ogc_fid)
)
WITH (
    OIDS = FALSE
)
TABLESPACE pg_default;

ALTER TABLE public."Infrastructure_pipeline"
    OWNER to postgres;

-- Index: infrastructure_pipeline_geom_idx

-- DROP INDEX public.infrastructure_pipeline_geom_idx;

CREATE INDEX infrastructure_pipeline_geom_idx
    ON public."Infrastructure_pipeline" USING gist
    (wkb_geometry)
    TABLESPACE pg_default;
    
CREATE SEQUENCE public."Infrastructure_pipeline_UID_seq"
    INCREMENT 1
    START 1
    MINVALUE 1
    MAXVALUE 9223372036854775807
    CACHE 1;

ALTER SEQUENCE public."Infrastructure_pipeline_ogc_fid_seq"
    OWNER TO postgres;
```
--- Dump the content of the table *"PipelineSegment"* in ```coresight 1.0```

```
pg_dump -C --file "C:\Rafael\backup\postgres\pipelineInfra.sql" --host "coresight-02.chtbrszez1rh.us-west-2.rds.amazonaws.com" --port "5432" --username "postgres" --verbose --role "postgres" --format=p --data-only --encoding "UTF8" --table "public.\"PipelineSegment\"" "coresight"
```
The output should be something similar to this:

![Dump the PipelineSegment table](https://c1.staticflickr.com/5/4179/34187267030_a01d371c69_z.jpg "Dump the PipelineSegment table")

--- Then modify the ```sql output file``` according said in the PostGIS section using **notepad++** or any text editor.
--- Eventually, restore (populate) the table *"Infrastructure_pipeline"* by using ```psql```:
```sql
psql --host "coresight-02.chtbrszez1rh.us-west-2.rds.amazonaws.com" --port "5432" --username "postgres" --dbname "coresight2" -f "C:\Rafael\backup\postgres\pipelineInfra.sql" -L "C:\Rafael\backup\postgres\logfile.log"
```
--- In geoserver for ```coresight 2.0```, create a new layer called *"InfrastructurePipelineWMS"* following the same pattern than *"QueryInfrastructureTrailersVegetationPipelineRaster"* in ```coresight 1.0```. Note about styling too.

## 4. wmsRasterType name for pipeline

![Change wms name for pipeline](https://c1.staticflickr.com/5/4160/34574601075_a77dc6f58c_z.jpg "Change wms name for pipeline")

Also for this specific case, we need to get the pipeline geometry before the user calls to the infrastructure service since it is a parameter in the ```open() function```. Therefore we need to modify again the **main.js** file in different areas:

--- Getting the Pipeline geometry

```javascript
var pipelineGeometryPromise = null;

    for (var i in userProjects) {
        if(typeof userProjects[i].layers.infrastructure !== 'undefined') {
            if(userProjects[i].layers.infrastructure === true) {
                pipelineGeometryPromise = client.pipeline.queryGeometry().then(function (data) {
                    if (data) {
                        return data.features.map(function (feat) {
                            var line = feat.geometry.coordinates;
                            return {
                                props: feat.properties,
                                start: L.latLng([line[0][1], line[0][0]]),
                                end: L.latLng([line[1][1], line[1][0]])
                            };
                        });
                    } else {
                        return Promise.reject();
                    }
                });
            }
        }
    }
```
--- Initialize the module properly by passing the ```pipelineGeometryPromise``` we got in the previous step.
```
function products(name, bounds, start, end){
        var products = {
            watertemp: {'module': WaterTemperature, 'client': client.watertemp},
            watersediment: {'module': WaterSediment, 'client': client.watersediment},
            riverice: {'module': RiverIce, 'client': client.riverice},
            smartice: {'module': SmartIce, 'client': client.smartice},
            infrastructure: {'module': Infrastructure, 'client': client.infrastructure}
        };
        var module = products[name].module;
        var client_obj = products[name].client;
        if(name === "infrastructure") {
            client.infrastructure.queryGeometry(new Date(0), new Date(), bounds, {maxFeatures: 100000}).then(function (data) {
                var geoTable = GeometryTable(data, map);
                var product = module(client.pipeline, client_obj, map, geoTable, pipelineGeometryPromise, bounds, start, end);
                product.productName = name;
                
                $("#product-" + product.productName).css('display', 'block');
                product.open();
                //return product;
            });
        }
        else {
            var product = module(client_obj, map, bounds, start, end);
            product.productName = name;

             $("#product-" + product.productName).css('display', 'block');
            product.open();
            //return product;
        }
    }
```
In this point, if we take a look at the point entry for the **infrastructure.js** file:

```function (clientPipeline, clientInfrastructure, map, infTable, pipelineGeometryPromise, bounds, start, end)```

we can see there are more arguments then other services have:
1. *clientPipeline:* Comes from coresight-geoserver, object pipeline.
2. *clientInfrastructure:* Comes from coresight-geoserver, object infrastructure.
3. *map:* This is a common one for any service.
4. *infTable:* we need to know why this argument is here. It takes the times from the database in order to build the timeline.
5.  *pipelineGeometryPromise:* The service needs some data before its initialization. We'll see it in the next section.
6. *bounds, start and end:* With the new schema in ```coresight 2.0```, these are common arguments.

## 5. The pipelineGeometryPromise

This service would need some data before its initialization in order to create the graphs and the circular buffer around a *map click*, so we would need to modify **main.js** and **infrastructure.js** files again.

--- In **main.js** there are two special functions ```open()``` and ```products()```, so any time the user selects a product in the tab navigation, the app passes trough *open* and then *products*. The key in this change was to get data before the *infrastructure* initialization dealing with the Asynchronous ```client.infrastructure.queryGeometry``` function:
```
function products(name, bounds, start, end){
        var products = {
            watertemp: {'module': WaterTemperature, 'client': client.watertemp},
            watersediment: {'module': WaterSediment, 'client': client.watersediment},
            riverice: {'module': RiverIce, 'client': client.riverice},
            smartice: {'module': SmartIce, 'client': client.smartice},
            infrastructure: {'module': Infrastructure, 'client': client.infrastructure}
        };
        var module = products[name].module;
        var client_obj = products[name].client;
        if(name === "infrastructure") {
            client.infrastructure.queryGeometry(new Date(0), new Date(), bounds, {maxFeatures: 100000}).then(function (data) {
                var geoTable = GeometryTable(data, map);
                var product = module(client.pipeline, client_obj, map, geoTable, pipelineGeometryPromise, bounds, start, end);
                product.productName = name;
                
                $("#product-" + product.productName).css('display', 'block');
                currentOpenProduct = product;
                currentOpenProduct.open();
                //return product;
            });
        }
        else {
            var product = module(client_obj, map, bounds, start, end);
            product.productName = name;

             $("#product-" + product.productName).css('display', 'block');
            currentOpenProduct = product;
            currentOpenProduct.open();
            //return product;
        }
    }

    function open(productName, bounds, start, end) {
        if (currentOpenProduct && currentOpenProduct.productName == productName) {
            return;
        }
        if (currentOpenProduct) {
            currentOpenProduct.close();
            currentOpenProduct = null;
        }
        products(productName, bounds, start, end);
        
        //$("#product-" + productName).css('display', 'block');
        //currentOpenProduct.open();
    }
```
--- In **infrastructure.js**, once we have the data, we feed the timeline:
```
if (pipelineGeometryPromise) {
            pipelineGeometryPromise.then(function (data) {
                infPipelineChart = InfrastructurePipelineClassificationChart('#product-infrastructure-pipeline-classifications-chart', '#product-infrastructure-percent-of-pipeline-chart', data, map);
                pipelineRadius = CircleAlongPipeline(data, map, {
                    onRadiusMove: function (circ, segment) {
                        var ts = timeline.getSelectedTimeBounds();
                        clientInfrastructure.queryCountsWithinRadius(ts.minTime, ts.maxTime, circ.origin, circ.radius).then(infRadiusChart.updateChart);
                        infrastructureClassificationTable('#product-infrastructure-counting-selected-point', segment);
                    }
                });
            });
        }
```
## 6. Modifying the 15AUG_PS2_RGB_p2.tif raster file

First thing we need to do is the setup Geotiff file for fast rendering by adding the following capabilities:
>-- inner tiling
>-- overviews

Inner tiling sets up the image layout so that it�s organized in tiles instead of simple stripes (rows). This allows much quicker access to a certain area of the geotiff, and the GeoServer readers will leverage this by accessing only the tiles needed to render the current display area. The following command instructs gdal_translate to create a tiled geotiff.
```
gdal_translate -of GTiff -co "TILED=YES" "15AUG_PS2_RGB_p2.tif" "15AUG_PS2_RGB_p2_tiled.tif"
```

An overview is a downsampled version of the same image, that is, a zoomed out version, which is usually much smaller. When GeoServer needs to render the Geotiff, it�ll look for the most appropriate overview as a starting point, thus reading and converting way less data. Overviews can be added using gdaladdo, or the the OverviewsEmbedded command included in Geotools. Here is a sample of using gdaladdo to add overviews that are downsampled 2, 4, 8 and 16 times compared to the original:

```
gdaladdo -r average 15AUG_PS2_RGB_p2_tiled.tif 2 4 8 16
```
As a final note, Geotiff supports various kinds of compression, but we do suggest to not use it. Whilst it allows for much smaller files, the decompression process is expensive and will be performed on each data access, significantly slowing down rendering. In our experience, the decompression time is higher than the pure disk data reading.

## 7. Publishing the 15AUG_PS2_RGB_p2.tif raster file
### Adding a new store in geoserver

![Add a store](https://c1.staticflickr.com/5/4170/34511096832_3986020268_z.jpg "Add a store")


![Add a store](https://c1.staticflickr.com/5/4157/33830580064_d69e6e2fd0.jpg "Add a store")

### Adding the layer for the 15AUG_PS2_RGB_p2.tif raster file
We do as usual adding the new layer from the store we created earlier.


![Add a layer](https://c1.staticflickr.com/5/4174/33830626234_8e6304d89d_z.jpg "Add a layer")

Some parameters in the tile caching tab should be:

![Parameters in tile caching tab](https://c1.staticflickr.com/5/4177/34542821871_d05a79ec64_z.jpg "Parameters in tile caching tab")

Since this layer will be tiled, geoserver will spend a lot of memory. Therefore we'll probably see an error like this in the geoserver web console:
>**"geoserver GC overhead limit exceeded"**

Running Java applications in computers takes some memory during the process which is known as Java memory (Java heap). Frequently, it is necessary to increase that heap to prevent throttling the performance of our application.

Set the following performance settings in the Java virtual machine (JVM) for your container by using ```-Xmx1024M``` which defines an upper limit on how much heap memory Java will request from the operating system (use more if you have excess memory). By default, the JVM will use 1/4 of available system memory. The setting -Xms1024m allocates 1024MB of memory to GeoServer, usually enough for a tiled geotiff file.


![JVM](https://c1.staticflickr.com/5/4192/33863510603_cbc1198c38_z.jpg "JVM")

### Container considerations
Java web containers such as Tomcat or **Jetty** ship with configurations that allow for fast startup, but don�t always deliver the best performance.
Please consider to change the file ```C:\Program Files (x86)\GeoServer 2.10.1\wrapper\wrapper.conf``` in the server to this parameters:
```
--Maximum Java Heap Size (in MB)--
wrapper.java.maxmemory=1024
```
Then, when an user will request the raster file, geoserver will start to tile properly, showing something like this:

![Raster file working](https://c1.staticflickr.com/5/4192/34543231361_182065c77c_z.jpg "Raster file working")

[auth0]: https://manage.auth0.com
[lock]: https://bitbucket.org/coresight709/coresight/src/50ae8872b15f7be4067a09142a77da7065012651/docs/Lock_setup.md?at=master&fileviewer=file-view-default
[geoserver1]: http://ec2-35-163-86-82.us-west-2.compute.amazonaws.com:8085/geoserver/web
[geoserver2]: http://ec2-34-209-121-134.us-west-2.compute.amazonaws.com:8085/geoserver/web
