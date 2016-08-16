# co.insilica.gdcSpark
  We use spark to build models on gdc data. `co.insilica.gdcSpark` provides some basic utilities to get started with gdc analyses. 

## Getting Started
  `co.insilica.gdcSpark` provides utilities for building models and datasets from gdc data.  To rely on gdc-spark you must provide apache spark dependencies yourself.  Your own `build.sbt` can rely on the gdc-spark by including the below dependencies:
  
```scala
libraryDependencies ++= Seq(
  
  //gdc-spark
  co.insilica %% "gdcSpark" % "0.1.4" [TODO need to make a version]
  
  // Spark
  "org.apache.spark" %% "spark-core" % "2.0.0" % "provided",
  "org.apache.spark" %% "spark-hive" % "2.0.0" % "provided",
  "org.apache.spark" %% "spark-mllib" % "2.0.0" % "provided"
)
```
We mark "provided" on the `org.apache.spark` dependencies. Spark expects users to submit jobs to an existing spark cluster.  Job submission involves sending a `jar` to an existing cluster. The cluster will provide its own spark dependencies. 

Sometimes it can be helpful to start development using sparks `standalone` cluster. The `standalone` cluster uses your local machine as a cluster.  This allows you to bypass the jar creation/submission process. To use the standalone cluster include spark dependencies as shown above. Follow the [spark documentation](http://spark.apache.org/docs/2.0.0/) for next steps.

## DatasetBuilder
Spark analyses often involve two main steps:
  
  1. **Base** dataset creation from data sources. 
  2. **Transformations** of base datasets. This may involve applying mathematical models or changing the shape of the data.
  
`co.insilica.spark`'s `DatasetBuilder` trait, shown below, identifies classes that build datasets.

```spark
package co.insilica.spark

import org.apache.spark.sql.Dataset
trait DatasetBuilder {
  def build()(implicit se: SparkEnvironment): Dataset[_]
}
```
  
## CaseFileEntityBuilder
  `CaseFileEntityBuilder extends DatasetBuilder` builds a spark `dataset` from a `co.insilica.gdc.query`. Below, we build a `dataset` for all Colon Adenocarcinoma RNA-Seq files [^facet_search] that are open access[^gdc_access].

```scala
"CaseFileEntityBuilder" should "build openColonRNASeq dataset" in {
  import co.insilica.gdc.query.{Filter, Operators, Query}
  import co.insilica.gdcSpark.builders.CaseFileEntityBuilder
  import org.json4s.JString

  implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkEnvironment = co.insilica.spark.SparkEnvironment.local
  implicit val gdcContext = co.insilica.gdc.GDCContext.default

  //build a query for 'Colon Adenocarcinoma' RNA-Seq files that are open access
  val query = Query().withFilter {
    Filter.and(
      Filter(Operators.eq, key="cases.project.disease_type", value=JString("Colon Adenocarcinoma")),
      Filter(Operators.eq, key="experimental_strategy", value=JString("RNA-Seq")),
      Filter(Operators.eq, key="access", value=JString("open"))
    )
  }

  CaseFileEntityBuilder(query)
    .build()
    .show(5,truncate=false) //print 5 lines with no value truncation
}
```
Running this test results in:

| caseId                               | fileId                               | entityType | entityId                             |
|--------------------------------------|--------------------------------------|------------|--------------------------------------|
| c113808a-773f-4179-82d6-9083518404b5 | f4f589af-6e3c-4eb5-92cc-1169a54fbe8d | aliquot    | ae0b0540-fcb6-4c9e-8835-2cb24933a01f |
| 7a481097-14a3-4916-9632-d899c25fd284 | dc4bba2d-8d49-4dcc-aa3c-17688fe73479 | aliquot    | 52c17edc-35f9-484c-949d-62694cfc797a |
| 64bd568d-0509-48fe-8d0a-aef2a85d5c57 | 615ba967-3ca9-42ef-a114-19043ede6ae0 | aliquot    | f9410d08-1525-4bf7-9c7c-939a2abe60ae |
<center>Table of cases, files, and aliquots for colon cancer rna-seq files</center>


Other posts go into more details about these columns but briefly:
* **caseId**: identifies a specific patient
* **fileId**: identifies a specific file (rna-seq file in this case)
* **entityType**: entities associated with a file.  (case or Aliquot)
* **entityId**: universal unique identifier for the entity associated with a file.

Now you can create GDC queries and build datasets. With this knowledge you can start imagining exciting data analysis pipelines.  In the next sections we will explore some simple analyses made possible by  `CaseFileEntityBuilder`.

##Aliquot Transformer
In earlier posts we reviewed the different entities in a **Biospecimen Supplement**. Aliquots are leaf entities in the GDC. They are not composed of any other entities. **Aliquots** are part of **analytes** the full container relationship is below:

<center>
<b>aliquot</b> < <b>analyte</b> < <b>portion</b> < <b>sample</b> < <b>patient</b><br/>
patients have samples which have portions which have analytes which have aliquots.
</center>

`co.insilica.gdcSpark.transformers.AliquotTransformer` identifies patient, sample, and portion ids for a dataset with a column of aliquot_ids. It also collects sample types.  This is useful for discerning normal tissue from tumor tissue. RNA-Seq files are always associated with the aliquot used for sequencing.

Below Aliquot Transformer runs on aliquot ids created in the [CaseFileBuilder Table](#CaseFileBuilderTable) [TODO get reference links working]

```scala
"AliquotTransformer" should "find aliquot information" in {
  import co.insilica.gdcSpark.transformers.AliquotTransformer
  import org.apache.spark.sql.Row
  import org.apache.spark.sql.types.{StringType, StructField, StructType}

  implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
  implicit val sparkEnvironment = co.insilica.spark.SparkEnvironment.local
  implicit val gdcContext = co.insilica.gdc.GDCContext.default

  val ids : org.apache.spark.rdd.RDD[Row] = sparkEnvironment
    .sparkSession
    .sparkContext
    .parallelize( List(
      Row("ae0b0540-fcb6-4c9e-8835-2cb24933a01f"),
      Row("52c17edc-35f9-484c-949d-62694cfc797a"),
      Row("f9410d08-1525-4bf7-9c7c-939a2abe60ae")))

  val schema = StructType(List(StructField("aliquotId",StringType,nullable=false)))
  val aliquotDS = sparkEnvironment.sparkSession.createDataFrame(ids,schema)

  AliquotTransformer(aliquotColumn = "aliquotId")
    .transform(aliquotDS)
    .show()
}
```
which results in
```
+--------------------+--------------------+-------------+--------------------+--------------------+--------------------+--------------------+
|           aliquotId|            sampleId|   sampleType|          sampleDate|           portionId|         portionDate|         aliquotDate|
+--------------------+--------------------+-------------+--------------------+--------------------+--------------------+--------------------+
|f9410d08-1525-4bf...|b481fc53-5b7e-4e2...|Primary Tumor|2016-05-02T14:29:...|3ea9927f-1280-419...|2016-05-02T14:29:...|2016-05-02T14:29:...|
|ae0b0540-fcb6-4c9...|dad24abd-0eb7-453...|Primary Tumor|2016-05-02T14:25:...|43a36de8-d938-411...|2016-05-02T14:25:...|2016-05-02T14:25:...|
|52c17edc-35f9-484...|9ef79e62-54d7-406...|Primary Tumor|2016-05-02T14:42:...|77e17dc2-fe35-4a9...|2016-05-02T14:42:...|2016-05-02T14:42:...|
+--------------------+--------------------+-------------+--------------------+--------------------+--------------------+--------------------+
```
The aliquot transformer uses the gdc-api to generate these results.  

Model design requires knowledge of the origin of data. The aliquot transformer is our first attempt at providing the important aliquot origin features. 

Of particular importance is the **sampleType** feature.  Users must know sample types before creation of models. The most common sample types are "Primary Tumor" and "Blood Derived Normal". See all sample types at https://gdc-api.nci.nih.gov/cases?pretty=true&facets=samples.sample_type. 

## CaseClinicalTransformer
CaseFileEntityBuilder provides information on **fileId** and **caseId**.  We can use **caseId**s to derive information from patient clinical supplements.  We review clinical supplement files in depth in [clinical supplements](https://tomlue.gitbooks.io/gdc-spark-gitbook/content/1_gdc/clinical_supplements.html). 

Clinical supplements are xml files that describe patient disease, treatment, and disease progression.  These xml files generally break down as follows:

```xml
<admin:admin>administrative data</admin:admin>
<coad:patient>
  <??? preferred_name="some_name" cde="some_number">some value</???>
  ...
  <some_nested_fields></some_nested_fields>
</coad:patient>
```
<center>Summarized COAD patient clinical supplement xml file</center>

Most fields in the xml file have the special attributes "cde" which refers to NIH [common data elements](https://www.nlm.nih.gov/cde/). Fields with the "cde" attribute sometimes carry a "preferred_name" attribute which eases interpretation. 
```xml
<clin_shared:icd_o_3_site cde="3226281">C18.5</clin_shared:icd_o_3_site>
```
<center style="color:#800000">Example clinical supplement field with cde but no preferred name. <a href="https://cdebrowser.nci.nih.gov/CDEBrowser/">cdebrowser.nci.nih.gov/CDEBrowser/</a> provides a catalogue of cdes. <a href="https://cdebrowser.nci.nih.gov/CDEBrowser/search?elementDetails=9&FirstTimer=0&PageId=ElementDetailsGroup&publicId=3226281&version=1.0">CDE 3226281</a></center>

Nested xml nodes contain information such as clinical follow-ups and treatment response data:

```xml
<shared_stage:stage_event system="AJCC">
  <shared_stage:pathologic_stage 
  preferred_name="ajcc_pathologic_tumor_stage" 
  cde="3203222">Stage I</shared_stage:pathologic_stage>
</shared_stage:stage_event>
```
<center style="color:#800000">Clinical supplements record stage events in nested nodes</center>

The `CaseClinicalTransformer` works through a two pass system:
```algorithm
1. read caseIds from input dataset
2. download all case clinical supplement xml files from gdc-api
3. First Pass - Record Set of CDEs used in any file
4. Second Pass - Record value of each CDE for each case xml file
```
<center style="color:#800000">CaseClinicalTransformer recipe</center>
The transformer handles nested CDEs by prepending the nested node name and recording an array of values. In the above example the recorded column name is "stage_event_3203222" with value ["Stage I"] if there are no siblings.

[^gdc_access]: https://gdc.nci.nih.gov/access-data/data-access-processes-and-tools.
[^facet_search]: https://gdc-api.nci.nih.gov/files?facets=cases.project.disease_type&pretty=true shows disease_types