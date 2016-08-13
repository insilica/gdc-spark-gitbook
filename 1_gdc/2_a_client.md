# co.insilica.gdc-core

  To use the GDC-API from code we need a GDC-client.  One simple java client is available at co.insilica.gdc-core.  To include it in a scala project we can add the following dependency to the `build.sbt` file.  

`librarydependencies += "co.insilica" %% "gdc-core" % "0.1.4"`

## A GDC Context
  GDC-Core allows developers to create a gdc context. To create the default gdc context we call `co.insilica.gdc.GDCContext.default`
  
  ```scala
  object example{
    import scala.concurrent.ExecutionContext.Implicits.global
    val gdc = co.insilica.gdc.GDCContext.default
  }
  ```
The GDC-core client is in active development.  Much of the more advanced capabilities are incomplete. We will be focusing on two functions **gdc.streamfiles** and **gdc.rawfind**. Creating input streams from the GDC-API is possible with **streamfiles**.  Querying the GDC meta data is possible via **rawfind**.

insilica.gdc-core makes use of scala futures and needs an implicit ExecutionContext in scope.  `import scala.concurrent.ExecutionContext.Implicits.global` is a global execution context.  Documentation about futures is at http://docs.scala-lang.org/overviews/core/futures.html.

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
    import scala.concurrent.ExecutionContext.Implicits.global
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
    import scala.concurrent.ExecutionContext.Implicits.global
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
import org.json4s.jackson.JsonMethods
object example extends App{
  import scala.concurrent.ExecutionContext.Implicits.global
  val gdc = co.insilica.gdc.GDCContext.default
  val query = Query().withFields("file_id","access")
  val jsonValues : Iterator[org.json4s.JsonAST.JObject] = gdc.rawFind("files")(query).take(2)
  jsonValues.foreach{ fileMetaData => println(JsonMethods.pretty(fileMetaData)) }
  sys.exit(0)
}
```
This application prints:
```
{
  "access" : "controlled",
  "file_id" : "84a9c5a8-94f4-4680-b74b-3e743ff6c42d"
}
{
  "access" : "open",
  "file_id" : "97948aac-64c0-411e-853f-e5b208b13565"
}
```
  
  Note that we don't give the query a limit. The iterator created by gdc-core lets us select any number of `JObject`s (until `hasNext == false`). 
  
### GDC Filters
  Please read the GDC-API [documentation on filters](https://gdc-docs.nci.nih.gov/API/Users_Guide/Search_and_Retrieval/#filters). Filters let us reduce the number of documents returned by the GDC-API to those meeting some criteria. There are two types of *filters*. **Content filters** let us define constraints directly on the returned documents. **Nested filters** let us combine operator filters.
  
  Insilica.gdc-core supports both filter types. Below we show how you can find all files that are open access and refer to a case with age at diagnosis less than 20.  You can find a catalogue keys for each GDC endpoint at [GDC Search Appendix A](https://gdc-docs.nci.nih.gov/API/Users_Guide/Appendix_A_Available_Fields/). 
  
  ```scala
object example extends App{

  import scala.concurrent.ExecutionContext.Implicits.global
  val gdc = co.insilica.gdc.GDCContext.default

  //content filters
  val accessFilter = Filter(Operators.eq,ValueContent("access",JString("open")))
  val ageFilter = Filter(Operators.lt,ValueContent("cases.diagnoses.age_at_diagnosis",JInt(20)))

  //nested filters
  val manualNested = Filter(Operators.and,FilterContent(accessFilter,ageFilter))
  val orFilters = accessFilter or ageFilter //open access or file size > 100000
  val andFilter = Filter.and(accessFilter,ageFilter) //open access and file size > 100000

  //print andFilter
  gdc.rawFind("files")(
      Query()
        .withFields("access","cases.diagnoses.age_at_diagnosis")
        .withFilter(andFilter)
    )
    .take(2)
    .foreach{ jobj : JObject => println(JsonMethods.pretty(jobj)) }
}
  ```
  This results in
  ```json
{
  "access" : "open",
  "cases" : [ { "diagnoses" : [ { "age_at_diagnosis" : 18 } ] } ]
}
{
  "access" : "open",
  "cases" : [ { "diagnoses" : [ { "age_at_diagnosis" : 18 } ] } ]
}
  ```
  
  You are now armed with the ability to download tissue data from the GDC and search for files relevant to your needs.  
  
  #### Legacy API
  On a file note, the Genomic Data Commons is in the business of data harmonization. GDC supports distinct genetics projects.  Project data needs modification to fit standards.  If, for example, you know that some TCGA exists but cannot find it on GDC you may be able to access it via the `legacy` api.  

```scala
val legacyApi : GDCContext = GDCContext.legacy
val methylationFilter = 
val it = legacyApi.rawFind("files")(Query())
val fileMeta : JObject = it.next()
println(JsonMethods.pretty(fileMeta))
```

The above shows how to access TCGA methylation data. At the time of this writing, Methylation data had exclusive access through the legacy API.
