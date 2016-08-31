# BORG - Bioinformatics Orphan Gene Rescue Graphical Models
  The BORG (bioinformatics orphan gene rescue graphical models) is an approach to hypothesis generation. BORG searches for little-known biological entities strongly related to a clinical endpoint.  In this example we use the BORG pipeline to find genes related to a clinical endpoint such as colon cancer tumor stage.  
  
## Blueprint
  Borg finds biological entities that are promising for clinical research via a simple blueprint:
  
  1. **Collect** data on biological entities for patients
  2. **Group** patients according to one or more clinical targets
  3. **Derive** biological entity importance for predicting clinical target 
  4. **Filter** by current knowledge of biological entities

In this example we perform these steps by:

1. **Collect** RNA-Seq biospecimen data for patients with Colorectal cancer
2. **Group** patients according to tumor stage
3. **Derive** genetic importance for predicting tumor stage
4. **Filter** out genes with many existing publications
  
## Collect Patient RNA-SEQ data
  co.insilica.gdcSpark provides the `CaseFileEntityBuilder` for building a table of case-file-aliquots. We document our progress through this example in excerpts from [bitbucket.BORGTest]({provide link to bitbucket BORG test|todo}). Below the builder collects rna-seq data for patients with colorectal cancer:
    
```scala
class BORG extends FlatSpec{

  import co.insilica.gdc.query.{Query,Filter,Operators}
  import co.insilica.gdcSpark.builders.{CaseFileEntityBuilder,CaseFileEntity}

  implicit val sparkEnvironment = co.insilica.spark.SparkEnvironment.local
  implicit val gdc = co.insilica.gdc.GDCContext.default

  "BORG" should "collect patient rna-seq data" in {
    //build a query for 'Colon Adenocarcinoma' RNA-Seq files that are open access
    val query = Query().withFilter {
      Filter.and(
        Filter(Operators.eq, key="cases.project.disease_type", "Colon Adenocarcinoma"),
        Filter(Operators.eq, key="experimental_strategy", "RNA-Seq"),
        Filter(Operators.eq, key="access", "open")
      )
    }
    val dataset : org.apache.spark.sql.Dataset[CaseFileEntity] = {
      CaseFileEntityBuilder(query)
        .withLimit(10)
        .build()
    }

    dataset.show(truncate=false)
  }
```
This code results in a dataset of `CaseFileEntity` objects. Each file has a case (with a **uuid** or universally unique identifier).  RNA-Seq experiments use aliquots. Aliquots are either **normal tissue** or **primary tumor** tissue taken from patients. The resulting table is shown below:

|caseId|fileId|entityType|entityId|
