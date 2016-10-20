# IR and GSTP Methylation

## Create a dataset
  We need a dataset to show a link between interventional radiation and GSTP methylation. This dataset needs a 2 critical values:
  
  1. IR response information
  2. GSTP methylation information

## IR Response Data
  To collect IR response and methylation information we aggregate the below:
  
  1. Collect all GDC cases where the patient received IR. 
  2. Collect all open access Illumina Human Methylation 450k for these cases.
 
The `DatasetBuilder` called `IRCases` builds this dataset. Its code is given below in **Appendix - IRCases**.  The first rows of this dataset are below:

|fileId|cgref|beta_value|caseId|radiation_response|sampleType|
|---------|----|-------|-----|------------------|-----------------|-------------------|
|292...|cg...29|4.4|8cc1...|Complete...|Solid Tissue Normal|
|292...|cg...108|26.1|8cc1...|Complete...|Solid Tissue Normal|
|292...|cg...109|9.1|8cc1...|Complete...|Solid Tissue Normal|
<center> IRCases dataset (truncated values and some columns left out)</center>

**Cgref** above is short for **composite_element_ref** which is an identifier used by Illumina. **Cgref** identifies a **CG** base pair measured in the 450k array. Illumina annotates genomic location, associated genes and other information related to these cgrefs.  The annotation file, HumanMethylation450_15017482_v1-2.csv, is available at [the illumina ftp](ftp://webdata2:webdata2@ussd-ftp.illumina.com/downloads/ProductFiles/HumanMethylation450/).
## GSTP Methylation Data
  GSTP methylation data can be accessed through the Illumina 450k TCGA experiments:
  
  ```scala
  ```
  
  | | cg1 | cg2 | cg2 |
  |-|-----|-----|-----|
  |res1|----|----|----|
  |res2
  
  #Appendix - IRCases
  IRCases builds the dataset shown in **IR Response Data**
  ```scala
  object IRCases extends DatasetBuilder{

  override def name: String = "IRCases"

  object columns{
    val cgref = FileMethylationTransformer.columns.compositeElementRef
    val beta = FileMethylationTransformer.columns.beta
    val radiation_response = "radiation_response"
    val fileId = "fileId"
  }

  override def build()(implicit se: SparkEnvironment): Dataset[_] = {
    import CaseFileEntityBuilder.{columns => CFEB}
    import columns._
    import se.sparkSession.implicits._

    Query()
      .withFilter(Filter(Operators.eq, "platform", "Illumina Human Methylation 450")) //select illumina 450k data
      .withFilter(Filter(Operators.eq, "access", "open")) //only use open access data
      .withFilter(Filter(Operators.eq, "data_format","TXT"))
      .|> { query => CaseFileEntityBuilder()
          .withQuery(query)
          .withLegacy()
          .withFields(List("data_format"))
          .build()
      }
      .<|{ _.show(truncate = false) }
      .transform{ df =>
        println("transforming aliquots")
        AliquotTransformer()
        .withAliquotIdColumn(CFEB.entityId)
        .transform(df)
      }
      .<|{ _.show()}
      .transform{ df =>
        println("getting clinical data")
        CaseClinicalTransformer().withCaseId(CFEB.caseId).transform(df)
      }
      .withColumnRenamed("radiation_therapy@2005312","radiation_therapy")
      .withColumnRenamed("radiation_therapy@3427615","radiation")
      .withColumnRenamed("radiations@radiation@measure_of_response@2857291",
        "radiation_response")
      .|>{ df =>
        println("columns renamed")
        val firstUDF = functions.udf{ arr :Seq[String] => arr.headOption.orNull }
        df.withColumn("radiation_response",firstUDF($"radiation_response"))
      }
      .transform{ df => //drop all unnecessary columns
        println("dropping unnecessary columns")
        import CaseFileEntityBuilder.columns.{fileId,caseId}
        val keepColumns = Seq(fileId, caseId,
          AliquotTransformer.columns.sampleType,
          "radiation_therapy",
          "radiation",
          radiation_response
        )
        df.drop((df.columns.toSet[String] -- keepColumns).toSeq :_*)
      }
      .|>{ df =>
        println("selecting radiation_therapy patients")
        df.where(df("radiation_therapy")==="YES")
      }
      .transform{ df =>
        println("building methylation")
        FileMethylationTransformer()
        .withFileIdColumn(CaseFileEntityBuilder.columns.fileId)
        .transform(df)
      }
  }
}
  ```