# Tumor Aggression
  In this example we will go through all the steps to generate per-sample aggression and per-gene aggression. These steps involve:

1. Build a dataset: We will need a dataset on which to define aggression
2. Per gene aggression: We will generate an aggresion value for each gene
3. Per sample aggression: We will generate an aggression value for each sample.

These tests use the following imports and exist within `co.insilica.booktests.TumorAggression.scala`: 
```scala

```
## Build a dataset
    In these examples we use a toy data set for cancer aggression.  TCGA clinical supplements define clinical outcomes.  The [Clinical Supplements](./1_gdc/clinical_supplements.md) section describes how co.insilica.gdcSpark converts TCGA clinical supplements into spark `Dataset`s. Clinical outcomes are derived below:
  
```scala
class Tumor_Aggression extends FlatSpec{

  import co.insilica.gdc.query.{Filter, Operators, Query}
  import org.apache.spark.sql.Dataset
  import co.insilica.gdcSpark.builders.CaseFileEntityBuilder

  object PrognosticOutcomes{

    //Common data elements are the features recorded in a clinical supplement.
    //read about naming conventions in the clinical supplements (1.4) section.  
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
    
    //columns 
    
    def build() : Dataset[_] = {
      val query = Query()  //The query is our entry point.  It tells us what to get from gdc
        .withFilter { Filter.and(
        Filter(Operators.eq, key="cases.project.disease_type", "Colon Adenocarcinoma"),
        Filter(Operators.eq, key="experimental_strategy", "RNA-Seq"),
        Filter(Operators.eq, key="access", "open"))
      }
      CaseFileEntityBuilder().withQuery(query).withLimit(100).build()
    }
    
  }
  ```
  TODO come back and finish all of this!
  
  
  | stage | lymphatic invasion | distant_metastasis | vascular_invasion | metastasis | percent_positive_lymph_nodes|
  |-------------|-------------|-------------|-------------|
  | IV | True | True | False | True | 58% |
  | III | False | True | False | True | 42% |
  | II | True | True | True | False | 0% |
  
### Per Case
```scala
```