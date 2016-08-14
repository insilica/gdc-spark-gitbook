# co.insilica.gdc-spark
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

Sometimes it can be helpful to start development using sparks `standalone` cluster. The `standalone` cluster uses your local machine as a cluster.  This allows you to bypass the jar creation/submission process. 

## DatasetBuilder
  Spark analyses often involve two main steps:
  
  1. Munge existing data sources into a **base** spark dataset.
  2. Transform spark datasets which may involve
    3. Joining spark datasets
    4. Applying machine learning or statistics to a dataset and storing the result
  
  [TODO need to write more about this]
## CaseFileEntityBuilder
  `CaseFileEntityBuilder` builds a spark `dataset` from a `co.insilica.gdc.query`. We build a `dataset` for all RNA-Seq files for Colon Adenocarcinoma[^facet_search] that are open access[^gdc_access].

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

  val openColonRNASeq = CaseFileEntityBuilder(query).build()

  openColonRNASeq.show(10)
}
```
##
[^gdc_access]: https://gdc.nci.nih.gov/access-data/data-access-processes-and-tools.
[^facet_search]: https://gdc-api.nci.nih.gov/files?facets=cases.project.disease_type&pretty=true shows disease_types