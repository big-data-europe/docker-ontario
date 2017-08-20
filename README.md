# Ontario: Ontology-based Architecture for Semantic Data Lakes

Ontario is a Semantic Data Lake capable of storing and querying heterogeneous data (e.g., csv, json, rdf) in its original format. Ontario uses the RDF molecules approach as a logical representation of the heterogeneous data. MULDER federated query engine leverages RDF molecules metadata to efficiently perform query decomposition, source selection, query planning, and query execution.

## Setting up a single container Ontario

One can test Ontario using a self contained Ontario container for small data. 
Self-contained Ontario contains:
* MongoDB 3.4
* Spark 2.1.1
* Ontario endpoint: `http://youraddress:5001/sparql`

To test on your local machine, do the following:

1. Pull Ontario from docker hub

  ```
   docker pull kemele/ontario:0.1-spark-2.1.1-hadoop2.7-mongodb_3.4
  ```

2. Run Ontario:

Use sample data (BSBM Person data):
The image contains a sample data of person.csv in /datasets and person collection within bsbm100 dataset in  mongodb. To run this:
  ```
   docker run -d --name ontario-demo -p 5001:5000 -p 27017:27017 kemele/ontario:0.1-spark-2.1.1-hadoop2.7-mongodb_3.4
  ```

To use your own data:
  * To add raw files, do either of the following:
    * use docker copy to put files: 
      ```
        docker cp /path/to/yourfile.csv.json:/datasets
      ```
    * mount your data folder to `/datasets` as: 
       ```
        -v /path/to/csv/json/filesfolder:/datasets
       ```
    * use mongoimport to load data to mongodb: 
       ```
        docker exec -it ontario-demo mongoimport --type csv|json [--headerline] --db [yourdatabase] --collection [collectionname] --file [path-to-json-or-csv-file]
       ```
  * Create RDF molecule templates for your dataset. 
    RDF molecule templates file contains the following elements:
    * `rootType`: RDF type (rdf:type) or arbitry name of a molecule
    * `predicates`: list of predicates with `range` (if available)
    * `linkedTo`: list of `range` values (if available in `predicates` element)
    * `wrappers`: list of wrapper that provide a certain set of predicates of this RDF molecule template
   Example: `person-template.json`
    ```json
     {
     "rootType": "http://xmlns.com/foaf/0.1/Person",
     "linkedTo": [],
     "predicates": [ { "predicate": "http://xmlns.com/foaf/0.1/mbox_sh1sum", "range": [] },
                    { "predicate": "http://www4.wiwiss.fu-berlin.de/bizer/bsbm/v01/vocabulary/country", "range": [] },
                    { "predicate": "http://purl.org/dc/elements/1.1/date", "range": [] },
                    { "predicate": "http://purl.org/dc/elements/1.1/publisher", "range": [] },
                    { "predicate": "http://www.w3.org/1999/02/22-rdf-syntax-ns#type", "range": [] }
                  ],
     "wrappers": [
           {
            "url": "localhost:27017",
            "urlparam": "",
            "wrapperType": "MongoDB",
            "predicates": [ "http://www4.wiwiss.fu-berlin.de/bizer/bsbm/v01/vocabulary/country" ]
           },
           {
            "url": "local[*]",
            "urlparam": "",
            "wrapperType": "SPARKCSV",
            "predicates": [
                 "http://xmlns.com/foaf/0.1/mbox_sh1sum",
                 "http://purl.org/dc/elements/1.1/date",
                 "http://purl.org/dc/elements/1.1/publisher",
                 "http://www.w3.org/1999/02/22-rdf-syntax-ns#type"
             ]
           }
       ]
    }
    ```
  * Create RML mapping for csv, json, or mongodb collection.
    Example: `sparkcsvmapping.ttl`
    ```
    @prefix rr:   <http://www.w3.org/ns/r2rml#>.
    @prefix rml:  <http://semweb.mmlab.be/ns/rml#>.
    @prefix ql:   <http://semweb.mmlab.be/ns/ql#>.
    @prefix bsbm: <http://www4.wiwiss.fu-berlin.de/bizer/bsbm/v01/vocabulary/> .
    @prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
    @prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
    @prefix foaf: <http://xmlns.com/foaf/0.1/> .
    @prefix dc:   <http://purl.org/dc/elements/1.1/> .
    @prefix rev:  <http://purl.org/stuff/rev#> .
    @prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
    @prefix base: <http://eis.iai.uni-bonn.de/ontario/mapping#> .

    #PERSON mappings
    <#PersonMappings>
    rml:logicalSource [
      rml:source "file:///datasets/person.csv" ;
      rml:referenceFormulation ql:CSV
    ];
    rr:subjectMap [
      rr:template "{person}";
      rr:class foaf:Person
    ];

    rr:predicateObjectMap [
      rr:predicate dc:date;
      rr:objectMap [
        rml:reference "date";
        rr:datatype xsd:date
        ]
      ];

    rr:predicateObjectMap [
      rr:predicate foaf:mbox_sha1sum;
      rr:objectMap [
        rml:reference "mbox_sha1sum";
        rr:datatype xsd:string
      ]
    ];

    rr:predicateObjectMap [
      rr:predicate dc:publisher ;
      rr:objectMap [
        rml:reference "publisher";
        rr:datatype xsd:anyURI
      ]
    ];
    rr:predicateObjectMap [
          rr:predicate rdf:type ;
          rr:objectMap [
            rml:reference "type";
            rr:datatype xsd:anyURI
          ]
     ].  
    ```
  * Create configuration file:
   Configuration file points to templates and mappings. In addition, you can specify different parameters to spark context based on your system capacity.
   Example: `config.json`
   ```json
   {
   "MoleculeTemplates": [
     {
       "type": "filepath",
       "path": "/ontario/templates/person-template.json"
     }
   ],
   "WrappersConfig": {
     "MappingFolder": "/ontario/mappings",
     "MongoDB": {
       "type": "MongoDB",
       "url": "localhost:27017",
       "mappingfile": "mongodbmapping.ttl",
       "params": {
       }
     },
     "SPARKCSV": {
       "type": "SPARK",
       "url": "local[*]",
       "mappingfile": "sparkcsvmapping.ttl",
       "params": {
         "spark.driver.cores": "4",
         "spark.executor.cores": "4",
         "spark.cores.max": "4",
         "spark.default.parallelism": "4",
         "spark.executor.memory": "4g",
         "spark.driver.memory": "4g",
         "spark.driver.maxResultSize": "1g"
       }
     },
     "SPARKJSON": {
       "type": "SPARK",
       "url": "local[*]",
       "mappingfile": "sparkjsonmapping.ttl",
       "params": {
         "spark.driver.cores": "4",
         "spark.executor.cores": "4",
         "spark.cores.max": "4",
         "spark.default.parallelism": "4",
         "spark.executor.memory": "4g",
         "spark.driver.memory": "4g",
         "spark.driver.maxResultSize": "1g"
       }
     }
    }
   }
   ``` 
Then, run the following with -v options pointing to the above files:
  ```
   docker run -d --name ontario-demo -v /path/to/csv/or/json/filesfolder:/datasets -v /path/to/config.json:/ontario/config/config.json -v /path/to/templatesfolder:/ontario/templates -v /path/to/mappingsfolder:/ontario/mappings  -p 5001:5000 -p 27017:27017 kemele/ontario:0.1-spark-2.1.1-hadoop2.7-mongodb_3.4
  ```
 Check the status of the mongo and ontario services:
  ```
   docker logs -f ontario-demo
  ```

3. Run queries 
* Use `curl`:
  ```
  curl -G --data-urlencode "query=select ?person where {?person a <http://xmlns.com/foaf/0.1/Person>}limit 10" http://0.0.0.0:5001/sparql
  ```
* Use `python` code: 
  ```python
  import urllib
  import httplib
  
  query = """
	    PREFIX foaf: <http://xmlns.com/foaf/0.1/>
            PREFIX bsbm: <http://www4.wiwiss.fu-berlin.de/bizer/bsbm/v01/vocabulary/>
            PREFIX dc: <http://purl.org/dc/elements/1.1/>

            SELECT DISTINCT ?person ?mbox ?country ?publisher
            where{
                ?person a foaf:Person.
                ?person dc:publisher ?publisher.
                ?person bsbm:country ?country.
                ?person foaf:mbox_sh1sum ?mbox
            } limit 10
       """
  params = urllib.urlencode({'query': prodq})
  headers = {"Accept": "*/*"}
  conn = httplib.HTTPConnection('0.0.0.0:5001')
  conn.request("GET", "/sparql" + "?" + params, None, headers)
  response = conn.getresponse()
  if response.status == httplib.OK:
        res = response.read()
        res = res.replace("false", "False")
        res = res.replace("true", "True")
        res = eval(res)
        print "results", res['result']
        print 'execTime', res['execTime']
        print 'totalRows', res['totalRows']
        print 'firstResult', res['firstResult']
  ```

## Setting up Ontario cluser using `docker-compose`
