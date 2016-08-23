# Drugs and Methylation

## Prerequisites
  To complete this tutorial you will need knowledge of:
  1. [gdc-core](1_gdc/2_a_client.md): a client for connecting to the [GDC-API](https://gdc-docs.nci.nih.gov/API/Users_Guide/Getting_Started/)
  2. [clinical-supplements](1_gdc/clinical_supplements.md): an explanation of TCGA clinical supplements

##Getting Started
Epigenetics affect drug toxicity and efficacy.  In some cases, specific epigenetic marks  make a good prognostic biomarker for cancer treatment. {Need a citation |todo}. In this example we will:

1. Use gdc-core to download TCGA methylation data
2. Find patient treatments
3. Find patient adverse events
4. Search for relationships between methylation and adverse events in the context of drug treatment.

###Downloading TCGA methylation data
  At the time of writing, the GDC had not completed harmonizing methylation data. When the GDC incorporates a new data type it undergoes a harmonization procedure.  Different  projects must conform to the same standards for harmonized data.
  
  The `legacy` gdc-api provides access to unharmonized data. An example of this kind of legacy data is available at https://gdc-api.nci.nih.gov/legacy/files?pretty=true. To perform a legacy query one can prepend `/legacy/` the GDC-API endpoint. GDC-Core provides a client for the legacy api as well:
  
```scala
"Methylation data" should "be downloadable from GDC" in {
  import co.insilica.gdc.query.{Query,Operators}
  import co.insilica.gdcSpark.builders.CaseFileEntityBuilder
  import org.json4s.JString

  implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
  implicit val gdcContext = co.insilica.gdc.GDCContext.legacy //legacy api

  //query will find all methylation files that are open access
  val query = Query()
    .withFilter(Filter(Operators.eq, "data_type", JString("Methylation beta value")))
    .withFilter(Filter(Operators.eq, "access", JString("open")))

  val df = CaseFileEntityBuilder()
    .withQuery(query)
    .withLimit(1000)
    .build()

  //save for later
  df.write.parquet("resources/methylation data should be downloadable from gdc")

  //print results
  df.show(3,truncate=false)
}
```
  The above code prints the below table:
  
| caseId | fileId | entityType | entityId |
|--------------------------------------|--------------------------------------|------------|--------------------------------------|
| abaf757b-f79f-40bd-96e3-e2c0f63061f0 | 4e19c35d-2ec7-444c-ac1d-d71b4ea7d4ce | aliquot | a4ac969d-0698-432f-9e0c-44e6232815ff |
| f31c21b6-0f7f-435b-9e24-97c909755c36 | c766fcc4-76d6-4460-9fce-5575089fbb72 | aliquot | 247b89d7-05c2-49ca-8f96-08786b03a511 |
| 75dbc8fb-4db8-4764-824c-eccf3a223884 | 686d00b2-2bf0-4560-9fc8-923934e556b9 | aliquot | bea6a21c-a9ce-464d-b4fa-4a93afdc18f6 |

We can build a spark `Dataset` from the fileIds associated with each methylation experiment. To download one of these files we use the **GDC-API Data** endpoint (<a href=>gdc-api.nci.nih.gov/legacy/data?**fileId**</a>):

```
asdf
```
<center style="color:#800000">Excert from file at <a href=>gdc-api.nci.nih.gov/legacy/data/4e19c35d-2ec7-444c-ac1d-d71b4ea7d4ce</a> </center>

With this code we can find methylation files and their associated aliquots and cases. The next steps will use these identifiers to:
1. Find patient treatment data
2. Download methylation data
3. Derive experimental properties.  

###Finding Patient Treatments
  The last section demonstrates how to find patients whose biological samples have methylation data.  It is straightforward to build a table of patient clinical data from caseIds.  We provide the code below, but there is a more complete discussion in {add link to other section |todo}.
  
  ```scala
  ```