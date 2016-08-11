# GDC-Core

  To use the GDC-api from code we need a GDC-client.  One simple client is available from `co.insilica.gdc-core`.  To include it in a scala project we can add the following dependency to the `build.sbt` file.  

`librarydependencies += "co.insilica" %% "gdc-core" % "0.1.3"`

The GDC-core client is in active development.  Much of the more advanced capabilities is incomplete and we will be focusing on two main functions.

## A GDC Context
  GDC-Core allows developers to create a gdc context. Using scala implicits we can pass the same GDC Context implicitly to different functions. To create the default gdc context we call `co.insilica.gdc.GDCContext.default`
  
  ```scala
  import co.insilica.gdc.GDCContext
  
  object 
  object example{
    implicit val gdcContext = GDCContext.default
  }
  ```