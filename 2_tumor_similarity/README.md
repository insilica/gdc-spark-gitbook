# Tumor Similarity
  Tumor similarity is an important method to enable data exploration and model creation. Similarity models rely on two components **fingerprints** and **similarity metrics**. 
  
  The code in this section is on bitbucket (TODO link) and makes up a FlatSpec class in scala:
  
  ```scala
class Tumor_Similarity extends FlatSpec{
  import co.insilica.gdc.query.{Filter, Operators, Query}
  import co.insilica.gdcSpark.builders.{CaseFileEntityBuilder,CaseFileEntity}
  import co.insilica.gdcSpark.transformers.rna.FileRNATransformer
  import co.insilica.spark.udaf.SparseVectorAgg

  //following examples start here...
  ```
  
  ## Tumor Fingerprinting
 **Tumor fingerprinting** is the creation of numeric vectors from tumor samples. Biological assays exist to measure diverse tumor and normal tissue features.  Here we will focus on rna-sequencing as a method to estimate protein expression.
  
  In this example we represent a tumor as a vector of gene expression values. Ensembl identifiers identify genes and FPKM {cite this | todo} encodes numeric expression values.
  
| Tumor_Sample | ENSG | FPKM |
| -- | -- | -- | -- | -- |
| c30ce88d-5dff-450... | ENSG00000200842.1 | 0.0 | 
| c30ce88d-5dff-450... | ENSG00000240097.1 | 0.23133 | 
<center>Table 1 </center>

For our example we will create a `DatasetBuilder` that builds a ten sample spark `Dataset`:
```scala
//Build a 10 sample dataset
object SampleDataset extends DatasetBuilder{

  override def name: String = "Tumor_Similarity.Dataset"

  override protected def build()(implicit se: SparkEnvironment): Dataset[_] = {

    val query = Query().withFilter {
      Filter.and(
        Filter(Operators.eq, key="experimental_strategy", value="RNA-Seq"),
        Filter(Operators.eq, key="access", value="open")
      )
    }

    CaseFileEntityBuilder(query) //enumerates case, file and aliquot ids
      .withLimit(10) //limits to 10 examples
      .build()
      .transform{ FileRNATransformer() //transforms into rna-seq data
        .withFileId(CaseFileEntityBuilder.columns.fileId)
        .transform
      }.select(columns.entity_id,columns.ensembl_id,columns.fpkm)
  }
}
```
This builder generates a table akin to Table 1. Entity_ids are tumor samples, ensembl_id identifies genes and fpkm quantifies expression.  To build and store this dataset we write a test:

```scala
"Tumor Similarity" should "build a simple dataset" in {
  SampleDataset
    .buildThenCache(org.apache.spark.sql.SaveMode.Overwrite)
    .show()
}
```
We can quickly reload this dataset from file via:
```scala
"Tumor Similarity" should "load sample dataset" in {
  SampleDataset
    .loadOrBuild()
    .show()
}
```

  //build a query for open access RNA-Seq files
  val query = Query().withFilter {
    Filter.and(
      Filter(Operators.eq, key="experimental_strategy", value=JString("RNA-Seq")),
      Filter(Operators.eq, key="access", value=JString("open"))
    )
  }

  val caseFileEntities: org.apache.spark.sql.Dataset[CaseFileEntity] = CaseFileEntityBuilder(query)
    .withLimit(10)
    .build()

  val df = EntityRNATransformer()
    .withEntityId(CaseFileEntityBuilder.columns.entityId)
    .withFileId(CaseFileEntityBuilder.columns.fileId)
    .transform(caseFileEntities)

  //lets save this dataframe for later. You can use any other file name you like
  df.write.parquet("resources/RNA Datasets should build from gdc-core")
  df.show(10)

/** results in 
* |            entityId|              caseId|              fileId|entityType|       ensembl_id|     expression|
* +--------------------+--------------------+--------------------+----------+-----------------+---------------+
* |8b1695b3-8abd-4bf...|9fcdccae-676e-407...|e1f2cb27-b78b-43b...|   aliquot|ENSG00000225215.1|            0.0|
* |8b1695b3-8abd-4bf...|9fcdccae-676e-407...|e1f2cb27-b78b-43b...|   aliquot|ENSG00000275261.1|            0.0|
* |8b1695b3-8abd-4bf...|9fcdccae-676e-407...|e1f2cb27-b78b-43b...|   aliquot|ENSG00000269680.1|  4.74034915892|
*/
}
```
{entityRNATransformer needs to take a cache directory | todo}

We now have gene variant expression data for 10 aliquots.  This expression data makes our tumor fingerprints.  We will come back and use these fingerprints in the next example {link next example | todo}.

 ###RNA-Sequencing
 The central dogma of molecular biology states 
 ####method
 ####fpkm

  
  ## Similarity Metrics
  Similarity metrics compare vectors. Specifically a similarity metric is a function $$f:R^n \times R^n \rightarrow R$$.  Similarity metrics generate large values for dissimilar vectors and small values for similar vectors.  There is rich literature around similarity metrics {citation for similarity literature | todo}. We use cosine similarity in this example:
  
  <center> $$\text{cosineSimilarity}(\vec{A},\vec{B}) = cos(\theta) = \dfrac{\vec{A} \cdot \vec{B}}{\lVert \vec{A} \rVert \lVert \vec{B} \rVert}$$ </center>
  
  Cosine similarity has a value of 1 for identical vectors and 0 for perpendicular vectors. Similarity metrics are a special case of distance metrics which all hold common traits:

  1. **Identity**  
  $$f(A,B) = 0 \iff A = B$$
  2. **symmetry**  
  $$f(A,B) = f(B,A)$$
  3. **triangle inequality**  
  $$f(A,C) \le f(A,B) + f(B,C)$$
  4. **non-negativity**  
  $$\forall A,B : f(A,B) \ge 0 $$

We end our discussion of similarity metrics here but encourage the reader to read further.

## A Pivot
  In the above sections we describe how to build a dataset containing rna-seq data for aliquots akin to:
  
  |aliquot_id|ensembl_id|fpkm|
  |----------|----------|----|
  |111-111-1|ENSG0001|3.2|
  |111-111-1|ENSG0002|5.0|
  |222-222-2|ENSG0001|3.2|
  |222-222-2|ENSG0002|5.0|
  <center> Long-form dataset for rna-seq files from the genomic data commons</center>

To start performing similarity analyses we need to build a feature matrix.  In this matrix each row represents an ensembl_id and each column an aliquot_id.  Cell values are the FPKM values for each aliquot-ensembl identifier pair.  

```scala
  "RNA Datasets" should "allow tumor tumor similarity" in {
    implicit val executionContex = scala.concurrent.ExecutionContext.Implicits.global
    implicit val sparkSession = co.insilica.spark.SparkEnvironment.local.sparkSession
    implicit val gdcContext = co.insilica.gdc.GDCContext.default

    //load the dataset created in last example
    val rnaDS = sparkSession.read.parquet("resources/RNA Datasets should build from gdc-core")

    //reshape
  }
```