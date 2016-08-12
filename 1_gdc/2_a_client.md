# Co.Insilica.GDC-Core

  To use the GDC-api from code we need a GDC-client.  One simple java client is available at co.insilica.gdc-core.  To include it in a scala project we can add the following dependency to the `build.sbt` file.  

`librarydependencies += "co.insilica" %% "gdc-core" % "0.1.3"`



## A GDC Context
  GDC-Core allows developers to create a gdc context. To create the default gdc context we call `co.insilica.gdc.GDCContext.default`
  
  ```scala
  package somepackage
  import co.insilica.gdc.GDCContext
  
  object example{
    val gdc = GDCContext.default
  }
  ```
The GDC-core client is in active development.  Much of the more advanced capabilities are incomplete. We will be focusing on two functions **gdc.streamfiles** and **gdc.rawfind**. Creating input streams from the GDC-API is possible with **streamfiles**.  Querying the GDC meta data is possible via **rawfind**.

### GDC InputStreams
  In [GDC](gdc/0_gdc.md) we reviewed the 6 endpoints defined by the genomic data commons. The `data` endpoint allows users to download files through the gdc-api.  When users provide a single uuid receive the referenced the file. We can use the **data** endpoint in the browser to download one or more files by providing a single uuid or comma separated uuids. 
  
  <center>
  <a href=https://gdc-api.nci.nih.gov/data?733d8229-03d2-4237-8f41-1a5fbea4f1f7>https://gdc-api.nci.nih.gov/data?733d8229-03d2-4237-8f41-1a5fbea4f1f7</a><br/>
  https://gdc-api.nci.nih.gov/data?{uuid}<br/>
  or<br/>
  https://gdc-api.nci.nih.gov/data?{uuid,uuid,uuid,...}
  </center>
  <br/>

The same query in scala using a gdcContext is shown below:
  
  ```scala
  package somepackage
  import co.insilica.gdc.GDCContext
  import java.io.InputStream
  
  object example{
    val gdc = GDCContext.default
    
    // xml is an inputstream of one xml file
    val xml : InputStream = gdc.streamFiles("733d8229-03d2-4237-8f41-1a5fbea4f1f7")
    
    //zip is an inputstream for a zip file containing multiple files
    val zip : InputStream = gdc.streamFiles(
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
  {"data":
    {"hits": [ <json_objects> ] },
    {"pagination": {
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
    gdc.rawFind("files")()
  }
  ```
  
  We can show a full example response by adding size and field parameters to the endpoint url. In the below url, we ask the GDC-API to return the "file_id", and "access" fields for 2 files. 
  
  https://gdc-api.nci.nih.gov/files?fields=file_id,access&size=2&pretty=true 
  
  The full response is:
  
  ```json
  {
  "data": {
    "hits": [
      {
        "access": "controlled", 
        "file_id": "84a9c5a8-94f4-4680-b74b-3e743ff6c42d"
      }, 
      {
        "access": "open", 
        "file_id": "97948aac-64c0-411e-853f-e5b208b13565"
      }
    ], 
    "pagination": {
      "count": 2, 
      "sort": "", 
      "from": 1, 
      "page": 1, 
      "total": 262293, 
      "pages": 131147, 
      "size": 2
    }
  }, 
  "warnings": {}
}
```
  GDCContext provides `gdc.rawfFind(endpoint:String)(query:Query)`.  We will discuss what a `Query` object is in a moment but the above 
  
test test