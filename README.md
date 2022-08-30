# Covid-19 Elastic Stack Visualisation - Slovakia

**Please be aware that this is not an official source of information on infection. If you find an issue in the data set or want to improve
the quality of the set, open an issue or PR**

**Repository is still work in progress and will involve work over the next days/weeks**

**Live Dashboard based on the data and the setup is located at [covid-19.radoondas.io](https://covid-19.radoondas.io)**

**Tested with Elasticsearch v7.17.5 and Logstash v7.8.1**

## What
With the current coronavirus outbreak, there are many visualizations and dashboards, which track how the virus spreads across the world. I  focus here on dataset and visualizations in Slovakia.

## Data
Since the beginning of the outbreak, we had only a few sources available, which were technically merged into one source over time. Government webpage dedicated to [covid-19](https://korona.gov.sk/). Data are provided by [National health information centre](http://www.nczisk.sk/en/Pages/default.aspx). Access to any machine-readable and open data source is impossible, or I am not aware of it. I consider this as unfortunate. Another source is news that scrapes the official data and possibly enhances with other reports from their resources. This approach does not make for a good and reliable data source - primarily if not published freely. Everyone keeps data for themselves as far as I know about (please correct me if I am wrong).

I combine reports and news to create a [dataset](/data/covid-19-slovensko-*.csv) that is simple to consume by machine.

### Data description
The data set consists CSV report with following header:

`date;city;infected;gender;note_1;note_2;healthy;died;region;age;district`

| Column name | Description                                                        |
|:------------|:-------------------------------------------------------------------|
| date        | Date - the date of the record                                      |
| city        | City - the location of the person infected by covid-19             |
| infected    | Infected - number of infected                                      |
| gender      | Gender, `M` - male, `Ž` - female, `D` - children, `X` - unknown    |
| note_1      | Note 1                                                             |
| note_2      | Note 2                                                             |
| healthy     | Healthy - number of people who recovered from the virus            |
| died        | Dead - number people who died                                      |
| region      | Region                                                             |
| age         | Age                                                                |
| district    | District                                                           |

### Geo data
For this occasion I generated `geojson` source file for all towns/cities/villages and also regions. Files are located in 
`data` folder ([obce.json](/data/obce.json) and [kraje.json](/data/obce.json)). JSON files are generated from [Geoportal](https://www.geoportal.sk/sk/zbgis_smd/na-stiahnutie/) using the GDAL library for format conversion.

## Howto - full version
The complete tutorial on how to build the dashboard is located on my [personal blog](https://radoondas.io/posts/2020/visualise-covid-19-using-elastic-stack/). Please read through and let me know if you have issues.

## Howto - compact version
The following are minimal steps to reproduce. I will also write a longer blog post and update the link here when available.

1. Clone this repository to get all data and configuration
2. Start up your Elasticsearch cluster. You either have your running cluster, and you know how it operates or you can use prepared docker configuration.

To use Docker environment run following command (I expect you have Docker up and running), and it will spin up 1 node cluster with version 7.8.1.
```docker
docker-compose up -d
```

Verify if your cluster is up and running. Navigate to your browser and open Kibana url `http://127.0.0.1:5601/`.

3. Import GeoJSON data for [Obce](/data/obce.json), [Kraje](/data/kraje.json) and [Okresy](/data/okresy.json). You now have 3 indices in your cluster.
   I am using GDAL library for the task which will be described in blog post later. For the reference, this is the command I use:
```bash
ogr2ogr -lco INDEX_NAME=kraje -lco NOT_ANALYZED_FIELDS={ALL} "ES:http://elastic:changeme@localhost:9200"  "$(pwd)/kraje.json"
ogr2ogr -lco INDEX_NAME=obce -lco NOT_ANALYZED_FIELDS={ALL} "ES:http://elastic:changeme@localhost:9200"  "$(pwd)/obce.json"
ogr2ogr -lco INDEX_NAME=okresy -lco NOT_ANALYZED_FIELDS={ALL} "ES:http://elastic:changeme@localhost:9200"  "$(pwd)/okresy.json"
```

4. Index data into Elasticsearch and stop docker container when finished

 ```docker
docker run --rm -it --network=host \
-v $(pwd)/template.json:/tmp/template.json \
-v $(pwd)/data/:/usr/share/logstash/covid/ \
-v $(pwd)/ls.conf:/usr/share/logstash/pipeline/logstash.conf \
-e MONITORING_ENABLED=false \
docker.elastic.co/logstash/logstash:7.8.1
```

5. Import the template and actual data for annotations used in visualizations
```bash
   cd data
   # Import index template
   curl -s -H "Content-Type: application/x-ndjson" -XPUT "localhost:9208/_template/milestones" \
     --data-binary "@template_milestones.json"; echo
   # You should see message: {"acknowledged":true}
   # Index actual data
   curl -s -H "Content-Type: application/x-ndjson" -XPOST localhost:9200/_bulk --data-binary "@milestones.bulk"; echo
```

6. [Import](https://www.elastic.co/guide/en/kibana/current/managing-saved-objects.html#managing-saved-objects-export-objects) `Saved Objects` from [visualisations file](data/visualisations.ndjson) and adjust patterns appropriately.

7. Navigate to `Dashboards` and check the data. ![Dashboard overview](/images/dashboard.png)

Note: The adjustment of all saved objects should not be necessary if you used `ogr2ogr` for the importing of the geojson data. The mapping and index name will match those in Saved objects. If you use different setup, then you will need to adjust ID's of pattern inside of the `visualisations.ndjson` file.
