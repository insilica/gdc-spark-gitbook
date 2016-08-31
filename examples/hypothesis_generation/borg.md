# BORG - Bioinformatics Orphan Gene Rescue Graphical Models
  The BORG (bioinformatics orphan gene rescue graphical models) is an approach to hypothesis generation. BORG searches for little-known biological entities strongly related to a clinical endpoint.  In this example we use the BORG pipeline to find genes related to a clinical endpoint such as colon cancer tumor stage.  
  
## Blueprint
  Borg finds biological entities that are promising for clinical research via a simple blueprint:
  
  1. **Collect** data on biological entities for patients
  2. **Group** patients according to one or more clinical targets
  3. **Derive** biological entity importance for predicting clinical target 
  4. **Filter** by current knowledge of biological entities

In this example we perform these steps for finding genes in colorectal cancer:

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
  //next examples begin here...
```
This code results in a dataset of `CaseFileEntity` objects. Each file has a case (with a **uuid** or universally unique identifier).  

### Finding Aliquot Information
RNA-Seq experiments use aliquots. Aliquots are either **normal tissue** or **primary tumor** tissue taken from patients. The resulting table is shown below:

|caseId|fileId|entityType|entityId|
|------|------|----------|--------|
|c0b8c55c-b993-481d-aeea-9ebfa64ee20e|1a7ab72c-ccbe-4ffe-b43d-d8570cb62c0b|aliquot   |c30ce88d-5dff-4503-b090-01b4b6aa0b80|
|565e2726-4942-4726-89d3-c5e3797f7204|046af5c1-b645-4338-be64-a8f2e08a9f2e|aliquot   |b8290920-9642-4137-ad13-88590a6694e8|

The `AliquotTransformer` transforms aliquotIds into tissue data:

```scala
//...previous example starts
  "BORG" should "finding aliquot information" in {

    val aliquotIds : RDD[Row] = sparkEnvironment
      .sparkSession
      .sparkContext
      .parallelize( List(
        Row("ae0b0540-fcb6-4c9e-8835-2cb24933a01f"),
        Row("52c17edc-35f9-484c-949d-62694cfc797a"),
        Row("f9410d08-1525-4bf7-9c7c-939a2abe60ae")))

    val schema = StructType(List(StructField("aliquotId",StringType,nullable=false)))
    val aliquotDS = sparkEnvironment.sparkSession.createDataFrame(aliquotIds,schema)

    AliquotTransformer(aliquotColumn = "aliquotId")
      .transform(aliquotDS)
      .show()
  }
//next example starts here...
```
This code gives us the below table:

|aliquotId|sampleId|sampleType|sample_createdDate|portionId|portionCreatedDate|aliquotCreatedDate|
|---------|---------|---------|---------|---------|---------|---------|---------|
|f9410d08-1525-4bf...|b481fc53-5b7e-4e2...|**Primary Tumor**|2016-05-02T14:29:...|3ea9927f-1280-419...|2016-05-02T14:29:...|2016-05-02T14:29:...|

Note  the sampleType of **Primary Tumor** for the first example in the table.  In this example we're interested in relating genes to tumor stage. This means we should focus on biospecimens of **Primary Tumor** rather than **Normal** tissue.