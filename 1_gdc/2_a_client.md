# co.insilica.gdc-core

  To use the GDC-API from code we need a GDC-client.  One simple java client is available at co.insilica.gdc-core.  To include it in a scala project we can add the following dependency to the `build.sbt` file.  

`librarydependencies += "co.insilica" %% "gdc-core" % "0.1.3"`



## A GDC Context
  GDC-Core allows developers to create a gdc context. To create the default gdc context we call `co.insilica.gdc.GDCContext.default`
  
  ```scala
  object example{
    val gdc = co.insilica.gdc.GDCContext.default
  }
  ```
The GDC-core client is in active development.  Much of the more advanced capabilities are incomplete. We will be focusing on two functions **gdc.streamfiles** and **gdc.rawfind**. Creating input streams from the GDC-API is possible with **streamfiles**.  Querying the GDC meta data is possible via **rawfind**.

### GDC InputStreams
  In [GDC](gdc/0_gdc.md) we reviewed the 6 endpoints defined by the genomic data commons. The `data` endpoint allows users to download files through the gdc-api.  When users provide a single uuid receive the referenced the file. We can use the **data** endpoint in the browser to download one or more files by providing a single uuid or comma separated uuids. 
  
  <center>
  <a href=https://gdc-api.nci.nih.gov/data?733d8229-03d2-4237-8f41-1a5fbea4f1f7>https://gdc-api.nci.nih.gov/data?733d8229-03d2-4237-8f41-1a5fbea4f1f7</a><br/>
  https://gdc-api.nci.nih.gov/data?{uuid}<br/>
  https://gdc-api.nci.nih.gov/data?{uuid,uuid,uuid,...}
  </center>
  <br/>

The same query in scala using a gdcContext is shown below:
  
  ```scala
  object example{
    val gdc = co.insilica.gdc.GDCContext.default    
    // xml is an inputstream of one xml file
    val xml : java.io.InputStream = gdc.streamFiles("733d8229-03d2-4237-8f41-1a5fbea4f1f7")
    //zip is an inputstream for a zip file containing multiple files
    val zip : java.io.InputStream = gdc.streamFiles(
      "733d8229-03d2-4237-8f41-1a5fbea4f1f7",
      "77e73cc4-ff31-449e-8e3c-7ae5ce57838c"
    )
  }
  ```


### GDC Queries
  The GDC-API allow retrieval of json for 4 main **endpoints** *cases*,*files*,*projects* and *annotations*. The base url for each endpoint is:
  
  `https://gdc-api.nci.nih/{endpoint}?...` for example https://gdc-api.nci.nih.gov/files?pretty=true
  
  Putting this url into your browser returns a json object with the below basic skeleton:
  
  ```json
  { "data":
    { "hits": [ <json_objects> ] },
    { "pagination": {
      "count": <number of objects shown>, 
      "sort": "", 
      "from": 1, 
      "page": <current page>, 
      "total": <number of objects in query>, 
      "pages": <number of pages in query>, 
      "size": 2
      }
    },
    "warnings":{}
  }
  ```
  
  A GDCContext can reproduce this query as follows:
  ```scala
  object example{
    val gdc = GDCContext.default
    val filesQueryBuilder : (Query => Iterator[JObject]) = gdc.rawFind("files")
    filesQueryBuilder(Query())
  }
  ```
  The function `gdc.rawFind("files")` returns a function that takes a `Query` object to an iterator of `JObject`s. We discuss `Query` objects in the next section.

### co.insilica.gdc.Query
  
  The GDC-API allows users to add parameters to an endpoint query.  **insilica.gdc-core** supports the below parameters:

| Parameter | Default | Description        |
|--------|------|----------------------------------------------------|
| fields | null | Query option to specify which fields to include in the response |
| size   | 10   | Specifies the number of results to return |
| from   | 1    | Skips all records before from |
| filters| null | Query option filters specify criteria for the returned response |
summarized from GDC-API [documentation](https://gdc-docs.nci.nih.gov/API/Users_Guide/Search_and_Retrieval/#query-parameters)

We can show a full example response by adding size and field parameters to the endpoint url. In the below url, we ask the GDC-API to return the "file_id", and "access" fields for 2 files. 
  
  https://gdc-api.nci.nih.gov/files?fields=file_id,access&size=2&pretty=true 
  
  The response (with pagination and warnings omitted):
  
  ```json
  {
  "data": {
    "hits": [
      { "access": "controlled", "file_id": "84a9c5a8-94f4-4680-b74b-3e743ff6c42d" }, 
      { "access": "open", "file_id": "97948aac-64c0-411e-853f-e5b208b13565" }
    ],...
}
```
<center><a>https://gdc-api.nci.nih.gov/files?fields=file_id,access&size=2&pretty=true</a></center><br/>
  The below example describes how to reproduce the above example with a `GDCContext`
  
  ```scala
  object example{
   val gdc = co.insilica.gdc.GDCContext.default
   val query = Query()
     .withFields("file_id","access")
     
   val jsonValues : Seq[JSONObject] = gdc.rawFind("files")(query).take(2).toSeq
  }
  ```
  
  Note that we don't give the query a limit. The iterator created by gdc-core lets us select any number of `JObject`s (until `hasNext == false`). 