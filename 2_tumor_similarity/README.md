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
  import SampleDataset.{columns=>SD} //import sampledataset column namespace
  object columns{
    val ensembl_id = SD.ensembl_id
    val ensembl_fp = "ensembl_fingerprint"
  }
  import columns._

  //build a map to define each gene index in the aggregated sparse vectors
  def buildGeneIdxMap() : Map[String,Long] = SampleDataset
    .loadOrBuild() //load the sample dataset
    .select(SD.ensembl_id)
    .distinct() //select distinct genes
    .rdd
    .zipWithIndex //associate a number with each gene
    .map{ case (Row(entity:String),idx:Long) => (entity,idx) }
    .collect() //collect onto driver
    .toMap //build a map

 override protected def build()(implicit se: SparkEnvironment): Dataset[_] = {
    val sampleRNA = SampleDataset.loadOrBuild() //load the sample dataset
    val geneIdxMap : Map[String,Long] = buildGeneIdxMap()
    val sva = new SparseVectorAgg(geneIdxMap) //initialize the sparse vector aggregator
    sampleRNA //aggregate sparse vectors for each aliquot
      .groupBy(SD.entity_id)
      .agg(sva(sampleRNA(SD.ensembl_id),sampleRNA(SD.fpkm)).as(ensembl_fp))
      .sort(SD.entity_id)
  }
  def buildAliquotGeneMatrix() : CoordinateMatrix  = ??? //next section
}
```

Note the line `val sva = new SparseVectorAgg(geneIdxMap)` requires reference to a map from gene ensembl_ids to a number.  This map tells SparseVectorAgg where to put each feature in the aggregated vector.  

```scala
  "Tumor Similarity" should "build a sparse vector for each aliquot" in{
    TumorSimilarityBuilder
      .loadOrBuild()
      .show() //show the results

    //print the first entry in the geneMap
    val idxGeneMap = TumorSimilarityBuilder.buildGeneIdxMap().map{x => (x._2,x._1)}
    print(s"gene map has size: ${idxGeneMap.size} the first value is ${idxGeneMap(0)}")
  }
```

This test gives:

|entity_id|ensembl_fingerprint|
|---------|-------------------|
|8af61a8a-17b0-402...|(60488,[54852,171...|
|ba2ba71d-2d57-415...|(60488,[18433,108...|
|17d328ce-367a-47c...|(60488,[30724,180...|
```gene map has size: 60488 the first value is ENSG00000212206.1```

These results show us that the aliquot `8af61a8a-17b0...` has an **fpkm** value for of `54852` for [ENSG00000212206.1](http://useast.ensembl.org/Homo_sapiens/Gene/Summary?g=ENSG00000212206;r=17:8329583-8329719;t=ENST00000390904) which happens to be a small nucleolar rna.  To start working with similarity approaches we will want to build a `Matrix`. In the next section we complete the `TumorSimilarityBuilder`.

###Coordinate Matrix
  Linear algebra is pervasive in bioinformatics.  It finds uses in feature generation/reduction (Principle Component Analysis, Singular Value Decomposition) and similarity analysis.  However, bioinformatics matrices can get very large. Wherever possible it is ideal to store matrices in a sparse and distributed manner.
  
  Spark's coordinate matrices let us store sparse distributed data.  A `CoordinateMatrix` is really just a wrapper around an `RDD` filled with `MatrixEntry` values.  A `MatrixEntry` tells us the *value* associated with a given *row* and *column*.  If a row/column does not have a `MatrixEntry` then the value, usually 0, is a default value.  
  
  We complete the `def buildAliquotGeneMatrix() : CoordinateMatrix` function defined in `TumorSimilarityBuilder` below:
  
  ```scala
  object TumorSimilarityBuilder extends DatasetBuilder{
    // see previous example for other code in object  
    /**
      * build a map from aliquotIds to index
      * This allows us to find the aliquot represented by each row in the coordinate matrix.
      */
    def buildAliquotIdxMap() : Map[String,Long] = SampleDataset
      .loadOrBuild() //load the sample dataset
      .select(SD.entity_id)
      .distinct()
      .sort(SD.entity_id)
      .rdd
      .zipWithIndex //associate a number with each gene
      .map{ case (Row(entity:String),idx:Long) => (entity,idx) }
      .collect() //collect onto driver
      .toMap //build a map
    
    /** makes coordinate matrix. rows = aliquot_ids, cols = ensembl_ids, values = fpkm */
    def buildAliquotGeneMatrix() : CoordinateMatrix = {
      val matrixEntries : RDD[MatrixEntry] = this
        .loadOrBuild()
        .rdd
        .zipWithIndex
        .flatMap{ case (Row(entity:String,ensembl_fingerprint:SparseVector),i) =>
            ensembl_fingerprint.toArray.zipWithIndex.filter{ _._1 != 0 }
              .map{ case (value,j) => MatrixEntry(i,j,value)}
        }
      new CoordinateMatrix(matrixEntries)
    }
  }
  ```
  <center> Demonstrate how to build a sparse distributed matrix `CoordinateMatrix` </center>

Now that we can build sparse distributed matrices, we can do some similarity calculations.  Fortunately, spark provides a built in similarity function.  This function operates on columns, so we need to transpose our matrix first:

```scala
"Tumor Similarity" should "demonstrate aliquot-aliquot similarity" in {
  val similarityMatrix = TumorSimilarityBuilder
    .buildAliquotGeneMatrix()
    .transpose() //aliquotGeneMatrix has genes in columns we want aliquots in columns
    .toIndexedRowMatrix() // CoordinateMatrices cannot do similarity so we transform
    .columnSimilarities() // build aliquot-aliquot similarity

  //reverse the aliquotIdx map so we can map row indices to aliquots
  val rowToAliquotMap = TumorSimilarityBuilder.buildAliquotIdxMap().map{ x => (x._2,x._1)}
  val aliquotLookup = udf{ x : Long => rowToAliquotMap(x)}

  similarityMatrix.entries.toDF("aliquot_i","aliquot_j","value")
    .withColumn("aliquot_i",aliquotLookup($"aliquot_i")) //look up aliquot ids for 1st aliquot index
    .withColumn("aliquot_j",aliquotLookup($"aliquot_j")) //look up aliquot ids for 2nd aliquot index
    .sort($"value".desc)
    .show(truncate=false)
}
```
This test prints:

|aliquot_i|aliquot_j|similarity|
|---------|---------|----------|
|9ecc833d-b92d-474a-a981-d6e0c12292e0|ba2ba71d-2d57-4154-a712-84a3ef19247d|0.9907634886968388|
|07215ddc-3201-5eac-ba1c-5e834b4bbf81|6deb1bf0-273a-47ba-9698-3d38ab05f02c|0.9495898788486178|
|07215ddc-3201-5eac-ba1c-5e834b4bbf81|5cf30542-0745-4cd5-9193-14ef995218b9|0.9466598114076444|