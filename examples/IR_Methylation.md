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

**Cgref** above is short for **composite_element_ref** which is an identifier used by Illumina. **Cgref** identifies a **CG** base pair measured in the 450k array. Illumina annotates genomic location, associated genes and other information related to these cgrefs.  The annotation file, HumanMethylation450_15017482_v1-2.csv, is available at [the illumina ftp](ftp://webdata2:webdata2@ussd-ftp.illumina.com/downloads/ProductFiles/HumanMethylation450/).  This dataset is available in `co.insilica.genetics.builders.IlluminaHuman450k` and when loaded gives:

|    IlmnID|Chromosome_36|Coordinate_36|   UCSC_RefGene_Name|
|----------|-------------|-------------|--------------------|
|cg25610294|           14|     23847750|[LTB4R2, CIDEB, L...|
|cg25611736|           14|    103700265|            [KIF26A]|
|cg25611750|           14|    100825409|                  []|
<center>co.insilica.genetics.builders.IlluminaHuman450k genomic location and associated genes</center>

With both of these datasets we can select all the Illumina450k cgrefs associated with UCSC gene GSTP1:

```scala
  import IlluminaHuman450k.{columns => IH}
  import IRCases.{columns => IR}
  
  val illumina = IlluminaHuman450k.loadOrBuild()
  val cgrefList = illumina
    .where(functions.array_contains(df(IH.UCSC_RefGene_Name),"GSTP1"))
    .select(IH.IlmnID,IH.Coordinate_36)
    .distinct()
    .collect()
    .map{ row => row.getAs[String](0)}
    .toList
 
 IRCases
    .loadOrBuild()
    .join(illumina, illumina(IH.IlmnID) === $"${IR.cgref}")
    .where($"${IR.cgref}".isin(gstpElements:_*))
    .where($"${IR.beta}".isNotNull)
```
Which yields a table of methylation data for GSTP1 with patient radiation response:

|gene_coord|cgref|          beta_value|radiation_response|              fileId|         sampleType|
|-------------|---------------------|--------------------|------------------|--------------------|-------------------|
|     67107552|           cg02659086|0.02| Complete Response|2920cd97-aa05-4c5...|Solid Tissue Normal|
|     67107847|           cg04920951|0.011| Complete Response|2920cd97-aa05-4c5...|Solid Tissue Normal|

  From this table we can start graphing methylation relationships to sample types and radiation response.
  
## Global Relationships
  Before identifying specific relationships between methylation and treatment/disease, we should set baselines. Baseline methylation is defined here as the level of methylation across all samples. Methylation is measured by **beta value**:
  
  $$\beta = \dfrac{Methylated Intensity}{Unmethylated Intensity}$$
  
  First we want to know about the general distribution on each CGref:
  ![alt text](../images/globalUnivariate.png)
  <center> boxplot of beta values across all GSTP1 cgres </center>
  
  So it is clear that there are differing regions of methylation within cgrefs associated with GSTP1.  But where are all these genomic locations?  We paste the UCSC Genome Browser visualization of the CGREF genomic locations [chr11:67106166-67110550](https://genome.ucsc.edu/cgi-bin/hgTracks?db=hg18&lastVirtModeType=default&lastVirtModeExtraState=&virtModeType=default&virtMode=0&nonVirtPosition=&position=chr11%3A67106166-67110550&hgsid=551918241_4VXMGIhTqKseXUYgS5zXbD5XcTyv) below:
  
  ![ucsc browser](../images/ucscGSTP1.png)
  <center>UCSC Browser visualiation of CGRef genomic locations. Red points are approximate CGREF locations</center>
  
  Since beta value cannot fall below 0 we use a log transformation to enable a closer to normal distribution of beta values.
  
  ![log transform of methylation values](../images/logTransform.png)
  
  While it is tempting to demonstrate methylation relationships purely on univariate CGREF-methylation relationships with radiation-response, we should concede that methylation of a CG base pair is never in a vacuum.  Below each line corresponds to an illumina 450k assay on a single sample:
  
  ![lines](../images/generalDistributionLines.png)
  <center>Log transformed methylation of CG pairs.  Each line represents one Illumina450k assay</center>
  
  This graph demonstrates that methylation undergoes branching structures.  For instance, at genomic location 6710550 there appear to be two clusters of methylation levels.  This kind of clustering is lost in the earlier box plots.  Additionally both of these clusters seem to derive almost entirely from similar methylation levels in the 5' direction.
  
  It is difficult to see changes in methylation level in CG pairs with close genomic locations.  **67107067**, **67107075**, **67107087** are each within 20 base pairs and cannot be clearly discerned.  The below graph equispaces each cgref:
  
  ![equispaced lines](../images/equispacedGeneralDistributionLines.png)
  <center>Log transformed methylation of CG pairs. Pairs are equispaced despite different relative distances.</center>
  
  Looking at this equispaced data makes it clear that many patients do not have methylation data for the 5' cgrefs.  It also seems that these left censored patients are somehow clustered differently.  For the following analyses we filter out patients that do no thave measurements for all cgrefs:
  
  ![filtered equispaced lines](../images/filteredGeneralLines.png)
  
  Removing the non-fully measured experiments reveals much less branching. We use this dataset for the remaining analysis.
  
## Tissue Relationships
  We interrogate tissue relationships to methylation by looking at individual cgref methylation as a function of sampleType and then overall methylation trends as a function of sample type.
  
  Below we see 
![sample type boxplots](../images/sampleType_boxplots.png)

![tissue relationships](../images/sampleType_lines.png)
## Radiation Response Relationships
  To evaluate radiation_response we filter the sample type to only 'Primary Tumor.' 
  
  ![radiation response boxplots](../images/radiation_response_box.png)
  
  This figure shows that the 'Stable Disease' radiation response seems 
   ![Lines stable disease](../images/lines_stable_disease.png)
   
   It helps see the difference if we overlap these figures:
   ![Overlapping lines stable disease](../images/overlapping_stable_disease_lines.png)
  
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