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
<a id="CaseFileBuilderTable">test</a>
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

Below we run the Aliquot Transformer on a aliquot ids formed in the [CaseFileBuilder Table](#CaseFileBuilderTable)

```scala

```

##
[^gdc_access]: https://gdc.nci.nih.gov/access-data/data-access-processes-and-tools.
[^facet_search]: https://gdc-api.nci.nih.gov/files?facets=cases.project.disease_type&pretty=true shows disease_types