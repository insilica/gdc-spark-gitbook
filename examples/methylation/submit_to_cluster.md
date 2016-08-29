# Submit to the Cluster
  In the [last section](/examples/methylation/drugs_and_methylation.md) we demonstrated how to get:
  1. Case ids, file ids and tissue data for all Illumina 450k files on the **legacy** gdc-api
  2. Extract beta values for illumina **composite ref elements** {add explanation link | todo}
  3. Extract drugnames and responses for each case from clinical supplements
  4. Put it all in a Dataset

Analysis of the resulting dataset gives us relationships between drugs, epigenetics and responses.  Alas, the scale of the data is too large for quick computation on a standalone spark cluster.  Insilica's client for the GDC-API can operate on worker nodes in a spark cluster. This means that we can perform analyses at scale while relying on the Genomic Data Commons to store the bulk of the data. 

[![Foo](https://spark.apache.org/docs/1.1.1/img/cluster-overview.png)](https://spark.apache.org/docs/1.1.1/img/cluster-overview.png) 

{can I actually use this image? | todo}

To show cluster analyses we will start with simple tasks and work our way up to more complex:

1. Count number methylation files on GDC
2. What is the distribution of drugnames over cases with methylation data?
3. Pick the most common drug.  How does methylation status relate to patient response for that drug?

## Counting Methylation Files
  This is a simple task which doesn't need the use of a spark cluster. Despite it's ease, this is a good example to show how to build and submit a job to a spark cluster.  
  
###Build.sbt
  To 

###Simple Application
```scala
object{

}
```