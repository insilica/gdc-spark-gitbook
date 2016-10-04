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

Table 1 represents a "long-form" dataset. For similarity purposes we would prefer to associate a 'fingerprint' vector with each aliquot. To do so we use the `SparseVectorAgg` aggregation function.  

###Sparse Vector Aggregation
  To collect a sparse vector for each aliquot we create a **user defined aggregation function** or **UDAF** called `SparseVectorAgg`.  This aggregation function associates a sparse vector with each value in a grouped column.  We are going to re-use this dataset so we create it in a `DatasetBuilder`:

```scala
  object TumorSimilarityBuilder extends DatasetBuilder{

    override def name: String = "Tumor_Similarity_Builder"

    import SampleDataset.{columns=>SD}

    //build a map to define each gene index in the aggregated sparse vectors
    def buildGeneIdxMap() : Map[String,Long] = {
      val sampleRNA = SampleDataset.loadOrBuild() //load the sample dataset
      sampleRNA.select(SD.ensembl_id)
        .distinct() //select distinct genes
        .rdd
        .zipWithIndex //associate a number with each gene
        .map{ case (Row(entity:String),idx:Long) => (entity,idx) }
        .collect() //collect onto driver
        .toMap //build a map
    }

    override protected def build()(implicit se: SparkEnvironment): Dataset[_] = {

      val sampleRNA = SampleDataset.loadOrBuild() //load the sample dataset
      val geneIdxMap : Map[String,Long] = buildGeneIdxMap()
      val sva = new SparseVectorAgg(geneIdxMap) //initialize the sparse vector aggregator
      sampleRNA //aggregate sparse vectors for each aliquot
        .groupBy(SD.entity_id)
        .agg(sva(sampleRNA(SD.ensembl_id),sampleRNA(SD.fpkm)).as("ensembl_fingerprint"))
    }

    def aliquotGeneMatrix() : CoordinateMatrix = ??? //next section
  }

  "Tumor Similarity" should "build a sparse vector for each aliquot" in{
    TumorSimilarityBuilder
      .loadOrBuild()
      .show() //show the results

    //print the first entry in the geneMap
    val idxGeneMap = TumorSimilarityBuilder.buildGeneIdxMap().map{x => (x._2,x._1)}
    print(s"gene map has size: ${idxGeneMap.size} the first value is ${idxGeneMap(0)}")
  }
```

You will note the line `val sva = new SparseVectorAgg(entityIdxMap)` requires reference to a map from gene ensembl_ids to a number.  This map tells SparseVectorAgg where to put each feature in the aggregated vector.  

This test results in:

|entity_id|ensembl_fingerprint|
|---------|-------------------|
|8af61a8a-17b0-402...|(60488,[54852,171...|
|ba2ba71d-2d57-415...|(60488,[18433,108...|
|17d328ce-367a-47c...|(60488,[30724,180...|
```gene map has size 60488. the first value is ...```

These results show us that the aliquot 8af61a8a-17b0... has an **fpkm** value for of 54852.  To start working with similarity approaches we will want to build a `Matrix`

###Coordinate Matrix
  Linear algebra is pervasive in bioinformatics.  It finds uses in feature generation/reduction (Principle Component Analysis, Singular Value Decomposition) and similarity analysis.  However, bioinformatics matrices can get very large. Wherever possible it is ideal to store matrices in a sparse and distributed manner.
  
  Spark's coordinate matrices let us store sparse distributed data.  A `CoordinateMatrix` is really just a wrapper around an `RDD` filled with `MatrixEntry` values.  A `MatrixEntry` tells us the *value* associated with a given *row* and *column*.  If a row/column does not have a `MatrixEntry` then the value, usually 0, is a default value.  
  
  Since we are going to re-use this distributed matrix we create a dataset-builder for it:
  
  ```scala
  ```

To achieve this goal we will create a new `DatasetBuilder` that modifies the `SampleDataset` dataset builder. First we ne
```scala
  object FingerprintDataset extends DatasetBuilder{
    override def name: String = "Tumor_Similarity.FingerprintDataset"

    //import columns from sample dataset to guarantee namespace agreement
    import SampleDataset.{columns => SD}

    //define column namespace for easy reference
    object columns{
      val entity_id = SD.entity_id
      val fingerprint = "fingerprint"
      def apply() = List(entity_id,fingerprint)
    }

  //we define this in the next section
  override protected def build()(implicit se: SparkEnvironment): Dataset[_] = ???
}
```
<center>Fingerprint Dataset Builder</center>

This fingerprint vector may contain ensembl_ids for which there were zero reads.  We can represent a vector with zero-reads more compactly as a sparse-vector.  


  requires reference to a map from feature names to an identifier. This map tells the aggregator where to place each feature in the vector.  

  
```scala
//implementation of build function for FingerprintDataset
def build()
```
  <center> Initialize a sparse vector aggregation for each aliquot </center>
  
  the length of the vector to associate with each aliquot.  This length is equal to the number of genes in our dataset:

```scala

```

We can reload this dataset from file via:
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