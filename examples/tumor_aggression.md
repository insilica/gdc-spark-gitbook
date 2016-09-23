# Tumor Aggression
  Cancer aggression describes cancer lethality, metastasis, growth velocity, and other prognostic endpoints.  Aggression metrics include categorical, numeric and boolean endpoints:
  
  1. **Tumor Stage:**  
  Categorical with least aggressive (1) to most aggressive (4).  Based on tumor location, size, lymph node involvement and distant metastasis (see [cancerstaging.org](https://cancerstaging.org/references-tools/Pages/What-is-Cancer-Staging.aspx)).
  2. **Lympatic invasion:**  
  True/False. Occurs when cancer cells break into lymphatic vessels.
  3. **Vascular Invasion:**  
  True/False. Occurs when cancer cells break into blood vessels. 
  4. **Histological Type:**  
  Categorical. Non-Mucinous vs Mucinous. 
  5. **Percent Positive Lymph Nodes**  
  Numeric 0-100. Histological sections contain lymph nodes positive/negative for cancer.  
  6. **Distant Metastasis**
  True/False

Analysis of cancer data frequently targets cancer aggression.  Researchers seek to find genes, drugs, or other factors capable of changing cancer outcomes.  These analyses simplify when we reduce cancer aggression metrics to a single numeric metric. 

There are two high level approaches to measuring univariate cancer aggression.  Univariate aggression is either measured on a per-case or on a per-feature basis. 

## Per Case tumor aggression 
  Per-case univariate aggression is conceptually simpler than per-biological feature.  Algorithms combine different prognostic endpoints into a single numeric endpoint.  For example, a **feature reduction** algorithm could combine tumor stage and lymphatic invasion. The resulting global metric loses information, but is easier to analyze. 
  
  Ultimately per case tumor aggression enables us to label a tumor as 'very aggressive' or 'not aggressive.'  We show in the below examples how principal component analysis can derive a per-case tumor aggression metric.  
  
  | tumor stage | lymphatic invasion | distance metastasis | **tumor aggression** |
  |-------------|-------------|-------------|-------------|
  | IV | True | True | 1.0 |
  | III | True | True | 0.8 |
  | I | False | False | 0.0 | 
  <center>Per case tumor aggression derived from tumor stage, lymphatic invasion, and distance metastasis observations.  Note that aggression metrics are not necessarily normalized between 0 and 1.</center>  

## Per Feature tumor aggression
  Per-feature tumor aggression determines whether a given biological observation (or feature) is indicative of cancer aggression.  For example, the lack of expression of tumor suppressor gene P53 would  indicate a more aggressive tumor.  
  
  Per feature tumor aggression correlates a biological observation with each clinical endpoint.  These correlations are then combined into a single confidence value.  A biological feature that highly correlates with many clinical endpoints is concered an 'aggressive' feature.  In our P53 example we:

1. Identify a biological feature type (expression of P53)
2. Identify prognostic endpoints (eg. tumor stage and lymphatic invasion)
3. Correlate the biological feature with each prognostic endpoint
4. Fuse the correlation values in 3 to form a single per feature tumor aggression metric

The [cancer regulome](explorer.cancerregulome.org) uses per feature tumor aggression.  The cancer regulome correlates genetic features with 6 prognostic outcomes. These correlations are then "fused" via [Fishers Combined Statistic](https://en.wikipedia.org/wiki/Fisher%27s_method). Note that prognostic outcomes are highly correlated. The cancer regulome accounts for non-independence with more statistical methods.

## Pros and Cons

## Mathematical Examples
## Coding examples
  In these examples we use a toy data set for cancer aggression.  TCGA clinical supplements define prognostic outcomes.  The [Clinical Supplements](./1_gdc/clinical_supplements.md) section describes how co.insilica.gdcSpark converts TCGA clinical supplements into spark `Dataset`s. Prognostic outcomes are derived below:
  
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