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
"Drugs and Methylation" should "find methylation files on GDC" in {
  import co.insilica.gdc.query.{Filter,Query,Operators} //gdc queries
  implicit val executionContext = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
  implicit val gdcContext = co.insilica.gdc.GDCContext.legacy //legacy api

  //query will find all methylation files that are open access
  val query = Query()
    .withFilter(Filter(Operators.eq, "platform", "Illumina Human Methylation 450"))
    .withFilter(Filter(Operators.eq, "access", "open"))

  co.insilica.gdcSpark.builders.CaseFileEntityBuilder()
    .withQuery(query)
    .withLimit(10)
    .build()
    .show(3,truncate=false)
}
```
<center style="color:#800000">Methylation data should be downloadable from GDC</center>  

  The above code prints the below table:
  
| caseId | fileId | entityType | entityId |
|--------------------------------------|--------------------------------------|------------|--------------------------------------|
| abaf757b-f79f-40bd-96e3-e2c0f63061f0 | 4e19c35d-2ec7-444c-ac1d-d71b4ea7d4ce | aliquot | a4ac969d-0698-432f-9e0c-44e6232815ff |
| f31c21b6-0f7f-435b-9e24-97c909755c36 | c766fcc4-76d6-4460-9fce-5575089fbb72 | aliquot | 247b89d7-05c2-49ca-8f96-08786b03a511 |
| 75dbc8fb-4db8-4764-824c-eccf3a223884 | 686d00b2-2bf0-4560-9fc8-923934e556b9 | aliquot | bea6a21c-a9ce-464d-b4fa-4a93afdc18f6 |

The **GDC-API Data** endpoint [gdc-api.nci.nih.gov/legacy/data/**fileId**](gdc-api.nci.nih.gov/legacy/data/) allows us to inspect one of these files:

| Composite Element REF | Beta_value                   | Gene_Symbol                  | Chromosome                   | Genomic_Coordinate           |
|-----------------------|------------------------------|------------------------------|------------------------------|------------------------------|
| cg00000029            | 0.466309829264718            | RBL2                         | 16                           | 53468112                     |
| cg00000236            | 0.910532917344955            | VDAC3                        | 8                            | 42263294                     |
<center style="color:#800000">Excerpt from file at <a href=>http://gdc-api.nci.nih.gov/legacy/data/4e19c35d-2ec7-444c-ac1d-d71b4ea7d4ce</a>. First line removed and line 2-4 shown </center>
  Column Descriptions:
  * **Composite Element REF**
  * **Beta_value**
  * **Gene_Symbol**
  * **Chromosome**
  * **Genomic_Coordinate**

Illumina provides some [videos describing methylation array analysis](http://www.illumina.com/techniques/microarrays/methylation-arrays/methylation-array-data-analysis-tips.html). Methylation array normalization is of particular importance and we will come back to it later. 

Now that we have file identifiers we can build a large spark `Dataset` of all the methylation data for our cases:

```scala
"Drugs and Methylation" should "transform fileIds into methylation data" in {
  import co.insilica.functional._ //scalaz |> pipe
  implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
  implicit val gdcContext = co.insilica.gdc.GDCContext.legacy //legacy api

  import sparkSession.sqlContext.implicits._
  //files derived from last test "Methylation data" should "be downloadable from GDC"
  List("4e19c35d-2ec7-444c-ac1d-d71b4ea7d4ce",
    "c766fcc4-76d6-4460-9fce-5575089fbb72",
    "686d00b2-2bf0-4560-9fc8-923934e556b9")
    .|>{sparkSession.sparkContext.parallelize(_).toDF("fileId") }
    .transform {
      co.insilica.gdcSpark.transformers.FileMethylationTransformer()
        .withFileIdColumn("fileId")
        .transform
    }
    .show(10)
}
```
**FileMethylationTransformer transforms fileIds into methylation data**

This code creates the dataset below:

|fileId|composite_element_ref|beta_value|
|---------------------------------------------|
|4e19c3...|cg00000029|0.435|
|4e19c3...|cg00000108|0.903|
|4e19c3...|cg00000109|0.69|
|4e19c3...|cg00000165|NA|
This dataset matches our structure expectations from the inspected methylation file. 

We demonstrated how to:
  1. Find methylation files (with associated cases and biospecimens)
  2. Build a methylation dataset from those files.  

Next we find patient treatments and responses. You could stop here to apply these approaches to different analyses.

###Finding Patient Treatments
  The `CaseClinicalTransformer` can extract clinical supplement data for each **caseId**.  We provide the code below, but there is a more complete discussion in {add link to other section |todo}.
  
```scala
"Drugs and Methylation" should "find drug names and responses from caseIds" in {
  import co.insilica.functional._ //used for |> which is the scalaZ pipe
  import co.insilica.gdcSpark.transformers.clinical.CaseClinicalTransformer //extracts clinical supplement data

  implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
  implicit val gdcContext = co.insilica.gdc.GDCContext.legacy //legacy api

  import sparkSession.implicits._ //allows rdd.toDF()

  //files derived from last test "Methylation data" should "be downloadable from GDC"
  List("abaf757b-f79f-40bd-96e3-e2c0f63061f0",
    "f31c21b6-0f7f-435b-9e24-97c909755c36",
    "75dbc8fb-4db8-4764-824c-eccf3a223884")
    .|>{ sparkSession.sparkContext.parallelize(_).toDF("caseId") } //transform list into a single column Dataset
    .|>{ CaseClinicalTransformer().withCaseId("caseId").transform } //extract clinical supplement data from cases
    .withColumnRenamed("drugs@drug@drug_name@2975232","drugnames") //rename cde names to something more legible
    .withColumnRenamed("drugs@drug@measure_of_response@2857291","responses")
    .select( "caseId","drugnames","responses" ) //select just the columns with drugnames and drugresponses
    .|> { df => //drugNames / responses stored in arrays with equal orders. zip them up
      val zipUDF = org.apache.spark.sql.functions.udf{ (col1: Seq[String], col2: Seq[String]) => col1.zip(col2) }
      df
        .withColumn("drugname_response", zipUDF(df("drugnames"), df("responses")))
        .drop("drugname_response")
    }
  .show(truncate=false)
}
```
  <center style="color:#800000">folding out drug use and response for patients </center>
  This code results in a drug response table.
  
|caseId|drugname_response|
|------|---------|
|f31c21b6-0f7f-435b-9e24-97c909755c36|[[Arimidex,null]]|
|abaf757b-f79f-40bd-96e3-e2c0f63061f0|[]|
|75dbc8fb-4db8-4764-824c-eccf3a223884|[[Temodar,Stable Disease],[CCNU,Stable Disease], [Temodar,Clinical Progressive Disease]]|

<center style="color:#800000">drug responses for cases with methylation data </center>+
In the above example we chose `"drugs@drug@drug_name@2975232"` and `"drugs@drug@measure_of_response@2857291"` to create our drug response table. You can recall the structure of these column names from our section on parsing clinical supplements. {link section | TODO}.  There are other drug common data elements which we list at the bottom of the page [Appendix Drug Data Elements](#Appendix Drug Data elements).

### Tissue Type
  We extracted methylation data and drug response data. Now we need to check the tissue type of the aliquot associated with each methylation file.  
  
```scala
"Drugs and Methylation" should "find aliquot information for files" in {
  import co.insilica.gdcSpark.transformers.AliquotTransformer

  implicit val executionContext = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
  implicit val gdcContext = co.insilica.gdc.GDCContext.default
  import sparkSession.implicits._

  sparkSession
    .sparkContext
    .parallelize( List( //list of aliquot ids for methylation files (see above table)
      "a4ac969d-0698-432f-9e0c-44e6232815ff",
      "247b89d7-05c2-49ca-8f96-08786b03a511",
      "bea6a21c-a9ce-464d-b4fa-4a93afdc18f6"))
    .toDF("aliquotId")
    .transform{ AliquotTransformer(aliquotColumn = "aliquotId").transform }
    .select("aliquotId",AliquotTransformer.columns.sampleType)
    .show()
}
```
This code prints 

| aliquotId | sampleType |
|-----------|------------|
| a4ac969d-0698-432... |Solid Tissue Normal|
|bea6a21c-a9ce-464...|Primary Tumor |
|247b89d7-05c2-49c...|Primary Tumor|

### Bringing it all together
  We can find methylation files, patient treatment data and tissue information.  Lets see if we can do it all at once.  Then we will build a spark task and submit it to a cluster.
  
```scala 
"Drugs and Methylation" should "find methylation, treatment, and tissue information" in {

  import co.insilica.gdc.query.{Filter,Query,Operators}
  import co.insilica.gdcSpark.transformers.{AliquotTransformer,FileMethylationTransformer}
  import co.insilica.gdcSpark.transformers.clinical.CaseClinicalTransformer
  import co.insilica.gdcSpark.builders.CaseFileEntityBuilder
  import co.insilica.functional._ //used for scalaZ .|> pipe

  import org.apache.spark.sql.functions.{udf,size}
  implicit val executionContext = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
  implicit val gdcContext = co.insilica.gdc.GDCContext.legacy

  import sparkSession.implicits._

  //query will find all methylation files that are open access
  Query()
    .withFilter(Filter(Operators.eq, "platform", "Illumina Human Methylation 450")) //select illumina 450k data
    .withFilter(Filter(Operators.eq, "access", "open")) //only use open access data
    .|>{ CaseFileEntityBuilder() //find all fileIds and aliquotIds associated with cases fitting query
      .withQuery(_) //apply query
      .withLimit(10) //only look at 10 cases
      .build() //build the dataset
    }
    .transform{ AliquotTransformer()
      .withAliquotIdColumn(CaseFileEntityBuilder.columns.entityId)
      .transform
    }
    .transform{ CaseClinicalTransformer().withCaseId("caseId").transform }
    .withColumnRenamed("drugs@drug@drug_name@2975232","drugnames") //rename cde names to something more legible
    .withColumnRenamed("drugs@drug@measure_of_response@2857291","responses")
    .filter{ size($"drugnames") > 0 } //filter out instances without any drug-response information
    .transform { df => //drugnames and responses are stored in arrays with equal order
      //Create a udf for zipping drugnames and response names (which are in arrays)
      val zipUDF = org.apache.spark.sql.functions.udf { (col1: Seq[String], col2: Seq[String]) => col1.zip(col2) }
      df
        .withColumn("drugname_response", zipUDF($"drugnames", $"responses"))
        .drop("drugnames","responses")
    }
    .transform{ df => //drop all unnecessary columns
      import CaseFileEntityBuilder.columns.{fileId,caseId}
      val keepColumns = Seq(fileId, caseId, AliquotTransformer.columns.sampleType,"drugname_response")
      df.drop((df.columns.toSet[String] -- keepColumns).toSeq :_*)
    }
    .transform{ FileMethylationTransformer()
      .withFileIdColumn(CaseFileEntityBuilder.columns.fileId)
      .withCachePath("/home/thomas/mnt/ext/data/clinicaltrials/borg/methylationfiles")
      .transform
    }
    .show(20)
}
```
  This code builds our final dataset:
  
|fileId|composite_element_ref|beta_value|caseId|sample_type|drugname_response|
|------|------|---------------------|----------|-----------|-----------------|
|cf8de69a-e1ab-403...|cg00000029| 0.583|f02b66f4-3f20-443...|Primary Tumor|[[Taxol,Complete Response],..
|cf8de69a-e1ab-403...|cg00000108|  null|f02b66f4-3f20-443...|Primary Tumor|[[Taxol,Complete Response],..

Now that we understand the basics with these examples, we can start digging into some slightly bigger data.  To do that, we'll need to learn how to submit these jobs to a spark cluster.


### Appendix Drug Data Elements
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