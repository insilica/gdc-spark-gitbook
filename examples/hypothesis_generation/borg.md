# BORG - Bioinformatics Orphan Gene Rescue Graphical Models
  The BORG (bioinformatics orphan gene rescue graphical models) is an approach to hypothesis generation. BORG searches for little-known biological entities strongly related to a clinical endpoint.  In this example we use the BORG pipeline to find genes related to a clinical endpoint such as colon cancer tumor stage.  
  
## Blueprint
  Borg finds biological entities that are promising for clinical research via a simple blueprint:
  
  1. **Collect** experimental data on biological entities for patients
  2. **Group** experimental data according to one or more clinical targets
  3. **Derive** biological entity importance for predicting clinical target 
  4. **Filter** by current knowledge of biological entities

In this example we perform these steps for finding genes in colorectal cancer:

1. **Collect** RNA-Seq biospecimen data for patients with Colorectal cancer
2. **Group** patients according to tumor stage
3. **Derive** genetic importance for predicting tumor stage
4. **Filter** out genes with many existing publications

In [Building the Table](#Building the Table) we go through these steps and show how to derive the below table.  In the next chapter we describe how to use this table to create a ranked list of hypothesis genes.

######GDCTable

## Building The Table
  co.insilica.gdcSpark provides the `CaseFileEntityBuilder` for building a table of case-file-aliquots. We document our progress through this example in excerpts from [bitbucket.BORGTest]({provide link to bitbucket BORG test|todo}). Below the builder collects rna-seq data for patients with colorectal cancer:
    
```scala
import co.insilica.functional._ //provides implicit |> function on all objects

import co.insilica.gdc.query.{Query,Filter,Operators}
import co.insilica.gdcSpark.builders.{CaseFileEntityBuilder,CaseFileEntity}
import co.insilica.gdcSpark.transformers.AliquotTransformer

import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{Row,Dataset}
import org.apache.spark.sql.types.{StringType, StructField, StructType}

class BORG extends FlatSpec{

  implicit val sparkEnvironment = co.insilica.spark.SparkEnvironment.local
  import sparkEnvironment.sparkSession.implicits._
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

### Grouping Aliquots by Tumor Stage
  To generate hypotheses we need a target.  The literature for cancer models targets diverse metrics for cancer aggression.  Some researchers focus on metastatic potential, tumor size, survival time, and others combine targets. To keep things simple we will focus on tumor stage:
  
```scala
//...previous example ends here
"BORG" should "group aliquots by tumor stage" in {

  val caseIds : org.apache.spark.sql.DataFrame = sparkEnvironment
    .sparkSession
    .sparkContext
    .parallelize(
      List(
        "c113808a-773f-4179-82d6-9083518404b5",
        "7a481097-14a3-4916-9632-d899c25fd284",
        "64bd568d-0509-48fe-8d0a-aef2a85d5c57"
      )
    ).toDF("caseId")

  co.insilica.gdcSpark.transformers.clinical.CaseClinicalTransformer()
    .withCaseId("caseId")
    .transform(caseIds)
    .show(truncate=false)
}
//next example starts here...
```
This code allows us to relate each caseId to a tumor stage at diagnosis.  

|caseId|tumor_stage|
|--------------------|-----------|
|c113808a-773f-417...|    Stage I|
|7a481097-14a3-491...|   Stage II|
|64bd568d-0509-48f...|    Stage I|


### Get RNA-SEQ Data
  To determine gene importance we need to characterize genes in relation to the target (tumor stage in this case). RNA-SEQ experiments count gene transcripts in a biospecimen.  These gene transcript counts characterize the genetic expression - biological sample relationship. The below code illustrates how to get this data:
  
```scala
"BORG" should "get RNA-SEQ data" in {
  List("1a7ab72c-ccbe-4ffe-b43d-d8570cb62c0b","046af5c1-b645-4338-be64-a8f2e08a9f2e")
    .|>{ sparkEnvironment.sparkSession.sparkContext.parallelize(_)}
    .toDF("fileId")
    .|> { df => EntityRNATransformer()
      .withFileId(CaseFileEntityBuilder.columns.fileId)
      .transform(df)
      .show(10)
    }
}
```
This code results in the table:

|              fileId|       ensembl_id|   expression|
|--------------------|-----------------|-------------|
|046af5c1-b645-433...|ENSG00000164182.9| 255335.26804|
|046af5c1-b645-433...|ENSG00000123364.4|          0.0|
|046af5c1-b645-433...|ENSG00000082068.7|70393.4578711|

In this table we can see that all ensembl_ids start with **ENSG** which means they are homo-sapien genes.  Ensembl provides information on every identifier (see [ENSG00000164182.9](http://useast.ensembl.org/Homo_sapiens/Gene/Splice?db=core;g=ENSG00000164182;r=5:60945129-61153037;t=ENST00000296597) with HUGO identifier **NDUFAF2**). The page also informs us that NDUFAF2 has 4 splice variants.  The suffix in ENSG00000164182**.9**  identifies a protein coding splice variant.

### Get Gene Publication Counts
  To 'rescue' orphan genes we need a definition of 'orphan'.  Here we define an orphan gene to be any gene that has less than 10 publications. The National Library of Medicine provides a useful [FTP repository](ftp://ftp.ncbi.nlm.nih.gov/gene/DATA) with a map of genes to publications. `Gene2PubmedBuilder` allows us to find publications for individual genes:
  
```scala
"BORG" should "get gene publication counts" in {
  val ds = Gene2PubmedBuilder.build()
  import Gene2PubmedBuilder.{columns => g2p}
  ds.select(g2p.ensembl_gene,g2p.pmid).show(3)

  ds.groupBy(Gene2PubmedBuilder.columns.ensembl_gene)
    .agg(functions.count(Gene2PubmedBuilder.columns.ensembl_gene).as("count"))
    .show(5)
}
//...next code starts here
```
This simple code prints the tables:

|Ensembl_gene|PubMed_ID|
|------------------|---------|
|ENSDARG00000062467| 12477932|
|ENSDARG00000055731| 12618376|
|ENSDARG00000056281| 12477932|

and 

|      Ensembl_gene|count|
|------------------|-----|
|ENSFCAG00000028791|    1|
|       FBgn0036474|   86|
|ENSBTAG00000008238|    4|

Counts for ensembl gene publications provide us with a filtering criteria.  We can select those genes that have a low publication count.

### Put it together
  We can now stitch together all the components needed to run a BORG process:
  
  ```scala
"BORG" should "build the gdc table" in {
  import CaseFileEntityBuilder.{columns => CFEB} //we'll use these column names

  Query()  //The query is our entry point.  It tells us what to get from gdc
    .withFilter { Filter.and(
      Filter(Operators.eq, key="cases.project.disease_type", "Colon Adenocarcinoma"),
      Filter(Operators.eq, key="experimental_strategy", "RNA-Seq"),
      Filter(Operators.eq, key="access", "open")
    )}
    .|> { CaseFileEntityBuilder() //download/links caseIds,fileIds,entityIds (aliquots here)
      .withQuery(_)
      .withLimit(10) //limit to make this test go quicker
      .build()
    }
    .transform{ AliquotTransformer() // get that aliquot info!
      .withAliquotIdColumn(CFEB.entityId)
      .transform
    }
    .|>{ df => CaseClinicalTransformer() //get tumor stage info.
      .withCaseId(CFEB.caseId)
      .transform(df)
      .|> { tumorDF => tumorDF.select(tumorDF(CFEB.caseId), //Select the tumor stage data (which is in an array) here.
        functions.explode($"stage_event@pathologic_stage@3203222").as("tumor_stage"))
        .join(df, CFEB.caseId) //join back with input
      }
    }
    .|>{ df => FileRNATransformer() //stitch in the RNA-SEQ data
      .withFileId(CFEB.fileId)
      .transform(df)
      .withColumn("root_ensembl", $"ensembl_id".substr(0,15)) //first 15 characters are the root gene id
    }.|> { df => //build gene2pubmed and join on 'root_ensembl'
      import Gene2PubmedBuilder.{columns => g2p}
      Gene2PubmedBuilder
        .build()
        .|>{ g2pDF => g2pDF.filter( g2pDF(g2p.ensembl_gene).startsWith("ENSG") ) } //select human genes
        .groupBy(g2p.ensembl_gene)
        .agg(functions.count(g2p.pmid).as("publication_count"))
        .withColumnRenamed(g2p.ensembl_gene,"root_ensembl")
        .|> { g2pDF => g2pDF.join(df,
            usingColumns = Seq("root_ensembl"),
            joinType = "right_outer")
        }.na.fill(0.0,Seq("publication_count")) //set 0 publications if count is null
    }
    .|> { df => //select caseId,entityId,tumor_stage,root_ensembl,expression,publicationCount
      import FileRNATransformer.{columns => FRT}
      df.sort("publication_count")
        .select(
          CFEB.entityId,
          "tumor_stage",
          "root_ensembl",
          FRT.ensemblId,
          FRT.expression,
          "publication_count"
        )
    }.|>{ df => //report a few things
      df.show(10) //show!
      df.select(CFEB.entityId,"tumor_stage")
        .distinct()
        .groupBy("tumor_stage") //number of cases with each tumor stage
        .agg(functions.count(CFEB.entityId))
        .show(truncate = false)
    }
}
  ```
  
  The reporting block in this code prints the requested table:
  
|entityId|tumor_stage|root_ensembl|ensembl_id|expression|publication_count|
|--------|-----------|------------|----------|----------|-----------------|
|99cb42da-b846-446...|Stage IIB|ENSG00000162888|ENSG00000162888.4|22.0|0|
|99cb42da-b846-446...|Stage IIB|ENSG00000176268|ENSG00000176268.5|168.0|0|
|9c737eba-552c-495...|Stage IIIB|ENSG00000162888|ENSG00000162888.4|3182.42|0|
|9c737eba-552c-495...|Stage IIIB|ENSG00000020219|ENSG00000020219.9|0.0|0

And shows us the number of aliquots for each tumor_stage:

|tumor_stage|count|
|-----------|-----|

From this we can see that there are some genes with 0 publications.  We now have all the information we need to complete the BORG pipeline. All we need is some method to derive gene **importance** from its relationship to tumor_stage. 

###Deriving Importance
  The easiest way to derive importance is by univariate correlation.  This means we will look for the pearson correlation of each genetic variant count with the clinical target.  Of course clinical target is not a numeric value, so lets fix that:
