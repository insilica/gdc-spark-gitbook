# Drugs and Methylation

## Prerequisites
  To complete this tutorial you will need knowledge of:
  1. [gdc-core](1_gdc/2_a_client.md): a client for connecting to the [GDC-API](https://gdc-docs.nci.nih.gov/API/Users_Guide/Getting_Started/)
  2. [clinical-supplements](1_gdc/clinical_supplements.md): an explanation of TCGA clinical supplements

##Getting Started
Epigenetics affect drug toxicity and efficacy.  In some cases, specific epigenetic marks  make a good prognostic biomarker for cancer treatment. {Need a citation |todo}. In this example we will:

1. Use gdc-core to download [TCGA methylation data](#abcd)
2. Find patient treatments / response
3. Search for relationships between methylation and response in the context of drug treatment.

These steps use example data on a standalone spark cluster.  In the next section we build a functioning spark application that performs all of these steps.

  
###Downloading TCGA methylation data<a name="abcd"></a>
  At the time of writing, the GDC had not completed harmonizing methylation data. When the GDC incorporates a new data type it undergoes a harmonization procedure.  Different  projects must conform to the same standards for harmonized data.
  
  The `legacy` gdc-api provides access to unharmonized data. An example of this kind of legacy data is available at https://gdc-api.nci.nih.gov/legacy/files?pretty=true. To perform a legacy query one can prepend `legacy` to a url as in `gdc-api.nci.nih.gov/legacy/files?...` to the GDC-API endpoint. GDC-Core provides a client for the legacy api which we use below to access Illumina 450k methylation data:
  
```scala
"Methylation data" should "be downloadable from GDC" in {
  import co.insilica.gdc.query.{Filter,Query,Operators} //gdc queries

  implicit val executionContext = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
  implicit val gdcContext = co.insilica.gdc.GDCContext.legacy //legacy api

  //query will find all methylation files that are open access
  val query = Query()
    .withFilter(Filter(Operators.eq, "platform", "Illumina Human Methylation 450"))
    .withFilter(Filter(Operators.eq, "access", "open"))

  val df = co.insilica.gdcSpark.builders.CaseFileEntityBuilder()
    .withQuery(query)
    .withLimit(10)
    .build()

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

The **GDC-API Data** endpoint (<a href=>gdc-api.nci.nih.gov/legacy/data/**fileId**</a>) allows us to download one of these files for inspection:

| Composite Element REF | Beta_value                   | Gene_Symbol                  | Chromosome                   | Genomic_Coordinate           |
|-----------------------|------------------------------|------------------------------|------------------------------|------------------------------|
| cg00000029            | 0.466309829264718            | RBL2                         | 16                           | 53468112                     |
| cg00000236            | 0.910532917344955            | VDAC3                        | 8                            | 42263294                     |
| cg00000289            | 0.663497467565074            | ACTN1                        | 14                           | 69341139                     |
| cg00000292            | 0.772453089249052            | ATP2A1                       | 16                           | 28890100                     |
| cg00000321            | 0.331188919383951            | SFRP1                        | 8                            | 41167802                     |
<center style="color:#800000">Excerpt from file at <a href=>http://gdc-api.nci.nih.gov/legacy/data/4e19c35d-2ec7-444c-ac1d-d71b4ea7d4ce</a>. First line removed and line 2-10 shown </center>
  Column Descriptions:
  * **Composite Element REF**
  * **Beta_value**
  * **Gene_Symbol**
  * **Chromosome**
  * **Genomic_Coordinate**

Illumina provides some [videos describing methylation array analysis](http://www.illumina.com/techniques/microarrays/methylation-arrays/methylation-array-data-analysis-tips.html). Methylation array normalization is of particular importance and we will come back to it later. 

Now that we have file identifiers and caseIds we should build a large `Dataset` of all the Methylation data for our cases:

```scala
  "FileMethylationTransformer" should "transform fileIds into methylation data" in {
    import co.insilica.gdcSpark.builders.CaseFileEntityBuilder

    implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
    implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
    implicit val gdcContext = co.insilica.gdc.GDCContext.legacy //legacy api

    //Note that we are using the same filepath used in the last test to load our data
    val caseFilesPath = new java.io.File("resources/methylation data should be downloadable from gdc")
    if(!caseFilesPath.exists()) throw new Exception("run 'Methylation data should be downloadable from GDC' test")

    //just read a few fileIds for testing
    val methylationFiles = sparkSession.read.parquet(caseFilesPath.getPath).limit(5)

    co.insilica.gdcSpark.transformers.FileMethylationTransformer()
      .withFileIdColumn(CaseFileEntityBuilder.columns.fileId)
      .transform(methylationFiles)
      .show(10)

    /** output is
      * +------+---------------------+----------+-----------+----------+------------------+------+----------+--------+
      * |fileId|composite_element_ref|beta_value|gene_symbol|chromosome|genomic_coordinate|caseId|entityType|entityId|
      * +------+---------------------+----------+-----------+----------+------------------+------+----------+--------+
      * |498...|           cg00000029| 0.4356648|       RBL2|        16|          53468112|490...|   aliquot|f-49e...|
      * |498...|           cg00000108|        NA|    C3orf35|         3|          37459206|490...|   aliquot|f-49e...|
      * |498...|           cg00000109|        NA|     FNDC3B|         3|         171916037|490...|   aliquot|f-49e...|
      * |498...|           cg00000165| 0.0631504|           |         1|          91194674|490...|   aliquot|f-49e...|
      */
  }
```
<center style="color:#800000">Downloading methylation data for all files. Uses dataset derived by CaseFileEntityBuilder in last example { need link | TODO }</center>
With this code we can find methylation files and their associated aliquots and cases.

###Finding Patient Treatments
  The last section demonstrates how to find patients whose biological samples have methylation data.  It is straightforward to build a table of patient clinical data from caseIds.  We provide the code below, but there is a more complete discussion in {add link to other section |todo}.
  
  ```scala
  "Patient drug data" should "derive from methylation case files" in {
    import co.insilica.gdcSpark.builders.CaseFileEntityBuilder
    import org.apache.spark.sql.types.{StringType, StructField, StructType}
    import org.apache.spark.sql.Row
    
    implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
    implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
    implicit val gdcContext = co.insilica.gdc.GDCContext.legacy //legacy api

    //Note that we are using the same filepath used in the last test to load our data
    val caseFilesPath = new java.io.File("resources/methylation data should be downloadable from gdc")
    if(!caseFilesPath.exists()) throw new Exception("run 'Methylation data should be downloadable from GDC' test")

    val methylationFiles = sparkSession.read.parquet(caseFilesPath.getPath)

    val df = co.insilica.gdcSpark.transformers.clinical.CaseClinicalTransformer()
      .withCaseId(CaseFileEntityBuilder.columns.caseId)
      .transform(methylationFiles)
      .toDF()
      
    //patient drugs are listed in an array i.e. drugs = [Taxol,Carboplatin,...]
    //we copy the patient and make one row per drug
    val rdd = co.insilica.gdcSpark.transformers.clinical.CaseClinicalTransformer()
      .withCaseId(CaseFileEntityBuilder.columns.caseId)
      .transform(methylationFiles)
      .toDF()
      .select(CaseFileEntityBuilder.columns.caseId,
        "drugs@drug@drug_name@2975232", //drugs are embedded under the 'drugs' field in clinical supplements
        "drugs@drug@measure_of_response@2857291"
      )
      .rdd
      .flatMap{ (row: Row) =>
        val drugNames = row.getAs[mutable.WrappedArray[String]]("drugs@drug@drug_name@2975232")
        val responses = row.getAs[mutable.WrappedArray[String]]("drugs@drug@measure_of_response@2857291")
        drugNames.zip(responses).map{ case (name,response) =>
            Row(row(0),name,response)
        }
      }

    val drugResponse = sparkSession.createDataFrame(rdd,StructType(Array(
      StructField("caseId",StringType,nullable=false),
      StructField("caseDrugName",StringType,nullable=true),
      StructField("caseResponse",StringType,nullable=true)
    )))

    drugResponse
      .where(drugResponse("caseResponse").isNotNull)
      .join(methylationFiles,CaseFileEntityBuilder.columns.caseId)
      .show(3)
      
    //save for future use
    drugResponse.write.parquet("/resources/patient drug data should derive from methylation case files")
  }
  ```
  <center style="color:#800000">folding out drug use and response for patients </center>
  This code results in a drug response table.
  
| caseId | caseDrugName | caseResponse | fileId | entityType | entityId |
|----------------------|-------------|-------------------|----------------------|------------|----------------------|
| ac0d7a82-82cb-4ae... | Taxol | Complete Response | 44d4a138-b76e-489... | aliquot | dc48f578-193c-474... |
| ac0d7a82-82cb-4ae... | Carboplatin | Complete Response | 44d4a138-b76e-489... | aliquot | dc48f578-193c-474... |
| 23f438bd-1dbb-4d4... | Adriamycin | Stable Disease | a1e710d7-6ec1-430... | aliquot | 14c87534-87eb-472... |
<center style="color:#800000">drug responses for cases with methylation data </center>
In the above example we chose `"drugs@drug@drug_name@2975232"` and `"drugs@drug@measure_of_response@2857291"` to create our drug response table. You can recall the structure of these column names from our section on parsing clinical supplements. {link section | TODO}.  There are other drug common data elements which we list at the bottom of the page. {section links | TODO}.

Now that we have drug response data we should check what the possible drugNames are:
```scala
//following from last example

```

| drugName | count |
|----------------------|---------------|
| asdf | asdf |

### Appendix Drug data elements
* route_of_administrations
* therapy_ongoing
* days_to_stem_cell_transplantation
* regimen_number
* number_cycles
* day_of_form_completion
* total_dose_units
* prescribed_dose
* total_dose
* therapy_types
* month_of_form_completion
* days_to_drug_therapy_start
* therapy_types
* drug_name
* pharm_regimen_other
* year_of_form_completion
* regimen_indication
* stem_cell_transplantation
* measure_of_response
* days_to_drug_therapy_end
* tx_on_clinical_trial
* stem_cell_transplantation_type
* clinical_trail_drug_classification
* prescribed_dose_units
* regimen_indication_notes
* pharm_regimen