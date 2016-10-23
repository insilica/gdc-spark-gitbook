# Tumor Aggression Tests
  In this example we will go through all the steps to generate per-sample aggression and per-gene aggression. These steps involve:

1. Build a dataset: We will need a dataset on which to define aggression
2. Per gene aggression: We will generate an aggresion value for each gene
3. Per sample aggression: We will generate an aggression value for each sample.

The test structure is as below in `co.insilica.booktests.TumorAggression.scala`: 
```scala
package co.insilica.booktests
import co.insilica.gdcSpark.builders.CaseFileEntityBuilder
import co.insilica.gdcSpark.transformers.clinical.CaseClinicalTransformer
import co.insilica.spark.{DatasetBuilder, SparkEnvironment}
import co.insilica.gdc.query.{Filter, Operators, Query}
import org.apache.spark
import org.apache.spark.sql.{DataFrame,Dataset}
import org.apache.spark.sql.types.StructType

class Tumor_Aggression extends org.scalatest.FlatSpec{

  //ClinicalOutcomes build a toy dataset for this test
  object ClinicalOutcomes extends DatasetBuilder{...}
  "Tumor Aggression" should "preview ClinicalOutcomes dataset" in {...}
  
  //we implement a transformer that derives per-gene aggression from ClinicalOutcomes
  object PerGeneAggressionTF extends spark.ml.Transformer{...}
  "Tumor Aggression" should "generate per-gene aggression" in {...}

  //we implement a transformer that dervies per-sample aggression from ClinicalOutcomes
  object PerSampleAggressionTF extends spark.ml.Transformer{...}
  "Tumor Aggresion" should "generate per-sample aggression" in {...}
}
```
This test class implements `ClinicalOutcomes` which generates our 'base' dataset. [GDC-Spark](../1_gdc/3_gdc-spark.md) describes `DatasetBuilder`s. In short, the dataset builder implements a `build` method and a `name` method.  These methods allow saving of the generated dataset in hadoop.

The `PerGeneAggressionTF` and `PerSampleAggressionTF` and [spark.ml.Transformer](http://spark.apache.org/docs/latest/ml-features.html) objects transform the ClinicalOutcomes dataset. PerGeneAggressionTF creates a numeric aggression value for each gene.  PerSampleAggressionTF creates a numeric aggression value for each sample.



## Build a dataset
   In these examples we use a toy data set for cancer aggression.  TCGA clinical supplements define clinical outcomes.  The [Clinical Supplements](./1_gdc/clinical_supplements.md) section describes how co.insilica.gdcSpark converts TCGA clinical supplements into spark `Dataset`s. To build our dataset we implement `ClinicalOutcomes extends DatasetBuilder` and preview the result.
  
```scala
object ClinicalOutcomes extends DatasetBuilder{

  override def name: String = "ClinicalOutcomes"

  object CommonDataElements{
    val lymph_node_examined = "lymph_node_examined_count@3"
    val lymph_nodes_positive_he = "number_of_lymphnodes_positive_by_he@3086388"
    val lymph_nodes_positive_ihc = "number_of_lymphnodes_positive_by_ihc@3086383"
    val vascular_invasion = "venous_invasion@64358"
    val lymphovascular_invasion = "lymphatic_invasion@64171"
    val tumor_stage = "stage_event@pathologic_stage@3203222"
    val metastasis = "stage_event@tnm_categories@pathologic_categories@pathologic_M@3045439"

    def apply() : List[String] = List(
      this.lymph_nodes_positive_he, this.lymph_nodes_positive_ihc, this.lymph_node_examined,
      this.vascular_invasion, this.lymphovascular_invasion, this.tumor_stage, this.metastasis)
  }

  override def build()(implicit se : SparkEnvironment) : Dataset[_] = {
    val query = Query()  //The query is our entry point.  It tells us what to get from gdc
      .withFilter { Filter.and(
        Filter(Operators.eq, key="cases.project.disease_type", "Colon Adenocarcinoma"),
        Filter(Operators.eq, key="experimental_strategy", "RNA-Seq"),
        Filter(Operators.eq, key="access", "open"))
      }

    CaseFileEntityBuilder()
      .withQuery(query)
      .withLimit(100)
      .build()
      .transform { CaseClinicalTransformer()
        .withCDEs(CommonDataElements())
        .transform
      }
  }
}
"Tumor Aggression" should "preview ClinicalOutcomes dataset" in {
  ClinicalOutcomes.loadOrBuild().show()
}
```
This object breaks into two main parts `CommonDataElements` and the `build` method. 

 The `object CommonDataElements` provides a namespace to generated dataset (see table below).  Common Data Elements are identifiers given to clinical and biological entities.  The [cde browser](https://cdebrowser.nci.nih.gov/CDEBrowser/) allows you to look up these common data elements.  We selected 7 common data elements as described in the [last section](README.md).

The `build` method works by providing a query to `CaseFileEntityBuilder` which is then transformed by `CaseClinicalTransformer`. The query selects open access colon adenocarcinoma rna-seq data. `CaseFileEntityBuilder` Finds all the cases, aliquots and files associated with the given query.  

 Case class `CaseClinicalTransformer` extracts clinical data for each case generated by the `CaseFileEntityBuilder`. It focuses on the common data elements provided by `CommonDataElements`.

The resulting table is below:
  
  
  | stage | lymphatic invasion | distant_metastasis | vascular_invasion | metastasis | percent_positive_lymph_nodes|
  |-------------|-------------|-------------|-------------|
  | IV | True | True | False | True | 58% |
  | III | False | True | False | True | 42% |
  | II | True | True | True | False | 0% |
  
## Per Gene Aggression
 The table generated by `ClinicalOutcomes` provides a base dataset. `PerGeneAggressionTF` generates a numeric aggression value for each gene from this base dataset.  

```scala

```

For each biomarker with a literature base larger than X (to be determined based on an assessment of the results of Aim 1), we will use a combination of existing pathways and text-mining to associate each biomarker with known pathways. While a great deal of research has gone into understanding the molecular chain of events underpinning cancer, this understanding is not yet reflected in the main databases used to understand genomic and transcriptomic data. For example, the most well-curated pathway database - KEGG Prostate Cancer Pathway – has 43 genes (in comparison to the breast cancer pathway, which has over 90) and neither metabolites, microRNA, nor drugs. The crowdsourced WikiPathway Prostate Cancer Pathway has over 130 nodes, consisting of 60 proteins as well as a genes and metabolites, but it has no drugs and only one RNA product, and is half the size of the breast cancer pathway. Therefore, we believe this project represents a substantial opportunity for improvement. [FIGURE LOOKING AT OVERLAP OF WIKIPATHWAY AND KEGG?]. 

We will use article text-mining and pathway-databases to relate biomarkers and pathways.

[INSERT BRIEF DESCRIPTION OF TEXT-MINING HERE].  Moreover, because neither of the main pathway datasets have drugs or radiation treatments, we will use both drug-target databank and STITCH to help connect drugs to pathways [IS THERE ANYTHING WE CAN DO WITH RADIATION DATA HERE?], which will help to identify biomarkers likely to be in proximity to clinically actionable pathways. In this Aim we will produce a global map that combines both the well-characterized pathways of prostate cancers along with putative connections between the existing biomarkers and drugs and radiation treatments. 