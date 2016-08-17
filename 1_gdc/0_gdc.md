# Basic Concepts
  The Genomic Data Commons provides an API for accessing many aspects of the underlying cancer data files.  The GDC API documentation is at https://gdc-docs.nci.nih.gov/API/Users_Guide/Getting_Started/. 
  
# A Simple API
  In this book we will be making use of the Insilica scala client for the Genomic Data Commons REST API.  To include the Insilica in scala api `co.insilica.gdc-core.0.1.4-SNAPSHOT` in a scala project via:

```scala
libraryDependencies ++= "co.insilica" %% "gdc-core" % "0.1.4-SNAPSHOT"
```
{include in build.sbt | caption}

## Search and Retrieval
  GDC-API documentation goes into more detailed discussion of all the API capabilities. We will only discuss Four 
  
# Files, Cases and Samples oh my!
  The Genomic Data Commons is an enormous repository of cancer data.  Exploring this data can overwhelm even well-seasoned bioinformaticians.  We will focus on a subset of the available functionality and thus provide a more gentle introduction.  
  
## Files
  To search for files contained in GDC we can search for them via the GDC-API. The search functionality br
## Cases
## Samples