# Covid-19 Elastic Stack Visualisation - Slovakia

**Please be aware that this is not an official source of information on infection. If you find an issue in the data set or want to improve
the quality of the set, open an issue or PR**

**Repository is still work in progress and will involve work over the next days/weeks**

**Live Dashboard based on the data and the setup is located at [covid-19.radoondas.io](https://covid-19.radoondas.io)**

## What
With the current coronavirus outbreak, there are many visualizations and dashboards, which track how the virus spreads across the world. I  focus here on dataset and visualizations in Slovakia.

## Data
The data is hard to get as there is no complete data source in proper format as far as I know. The only official source of information is [Public Health Authority of Slovak Republic](http://www.uvzsr.sk/en/) and their
[reports](http://www.uvzsr.sk/index.php?option=com_content&view=category&layout=blog&id=250&Itemid=153) on the virus. Another source of information is news where I could find additional or missing information.

I take reports and news to combine a [dataset](/data/covid-19-slovensko.csv) that is simple to consume by machine.

### Data description
The data set consists CSV report with following header:

`datum;mesto;infikovani;pohlavie;poznamka1;poznamka2;zdravi;mrtvi;kraj`

| Column name | Description |
|-------------|-------------|
| datum | Date - the date of the record |
| mesto | City - the location of the person infected by covid-19 |
| infikovani| Infected - number of infected |
| pohlavie| Gender, `M` - male, `Ž` - female, `D` - children, `X` - unknown |
| poznamka1 | Note 1 |
| poznamka2 | Note 2 |
| zdravi | Helthy - nymber of people who healed from the virus |
| mrtvi | Dead - number people who died |
| kraj | Region |

### Geo data
For this occasion I generated `geojson` source file for all towns/cities/villages and also regions. Files are located in 
`data` folder ([obce.json](/data/obce.json) and [kraje.json](/data/obce.json)). JSON files are generated from [Geoportal](https://www.geoportal.sk/sk/zbgis_smd/na-stiahnutie/)  
using GDAL library for format conversion.

## How - compact version
The following are minimal steps to reproduce. I will also write a longer blog post and update the link here when available.

1. Clone this repository to get all data and configuration
2. Start up your Elasticsearch cluster. You either have your running cluster, and you know how it operates or you can use prepared docker configuration.

To use Docker environment run following command (I expect you have Docker up and running) and it will spin up 1 node cluster with version 7.6.1.
```docker
docker-compose up
```

Verify if your cluster is up and running. Navigate to your browser and open Kibana url `http://127.0.0.1:5601/`.

3. Import GeoJSON data for [Obce](/data/obce.json) and [Kraje](/data/kraje.json). You now have 2 indices in your cluster.  I am using GDAL library for the task which will be described in blog post later.
For the reference, this is the command I use:
```bash
ogr2ogr -lco INDEX_NAME=kraje -lco NOT_ANALYZED_FIELDS={ALL} "ES:http://elastic:changeme@localhost:9200"  "$(pwd)/kraje.json"
ogr2ogr -lco INDEX_NAME=obce -lco NOT_ANALYZED_FIELDS={ALL} "ES:http://elastic:changeme@localhost:9200"  "$(pwd)/obce.json"
```

4. Index data into Elasticsearch and stop docker container when finished

 ```docker
docker run --rm -it --network=host \
-v $(pwd)/template.json:/tmp/template.json \
-v $(pwd)/data/covid-19-slovensko.csv:/tmp/covid-19-slovensko.csv \
-v $(pwd)/ls.conf:/usr/share/logstash/pipeline/logstash.conf \
docker.elastic.co/logstash/logstash:7.6.1
```

5. [Import](https://www.elastic.co/guide/en/kibana/current/managing-saved-objects.html#managing-saved-objects-export-objects) `Saved Objects` from [visualisations file](data/visualisations.ndjson) and adjust patterns appropriately.
6. Navigate to `Dashboards` and check the data.
![Dashboard overview](/images/dashboard.png)

Note: The adjustment of all saved objects should not be necessary if you used `ogr2ogr` for the importing of the geojson data. The mapping and index name will match those in Saved objects. If you use different setup, then you will need to adjust ID's of pattern inside of the `visualisations.ndjson` file.
