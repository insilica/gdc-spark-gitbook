# Co.Insilica.GDC-Core

  To use the GDC-api from code we need a GDC-client.  One simple java client is available at co.insilica.gdc-core.  To include it in a scala project we can add the following dependency to the `build.sbt` file.  

`librarydependencies += "co.insilica" %% "gdc-core" % "0.1.3"`

The GDC-core client is in active development.  Much of the more advanced capabilities is incomplete and we will be focusing on two main functions. 

## A GDC Context
  GDC-Core allows developers to create a gdc context. To create the default gdc context we call `co.insilica.gdc.GDCContext.default`
  
  ```scala
  package somepackage
  import co.insilica.gdc.GDCContext
  
  object example{
    implicit val gdcContext = GDCContext.default
  }
  ```
  
  GDCContext makes two functions available that will be our focus.