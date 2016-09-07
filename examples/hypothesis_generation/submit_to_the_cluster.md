# Submit to the cluster

##Recap
  In the last section we reviewed all the information necessary for `Bioinformatics Orphan Gene Rescue Graphical Model` (BORG model) generation. Recall that the **BORG** pipeline identifies genes of importance to a clinical endpoint.  These genes are further filtered to those with a small number of publications.  The information necessary for BORG includes three main feature categories:
  
  1. **Clinical:** We want to identify genetic importance relative to a clinical endpoint.  In this example we pick **tumor_stage** and make it into a numeric feature.

  2. **Genetic Behavior:** Genes with behavior determined by clinical contexts are clinically important genes. We measure that behavior via **RNA-SEQ**.
  
  3. **Genetic Orphan:** Hypothesis generation should identify genes for experiment.  Genes with a large number of publications may already be well understood.  We focus on genes with a small number of publications.

In this section we derive spark `Dataset`s for each of these categories and then join them together.  We build and save these datasets on a spark cluster with a hadoop file system.  The final product is one large spark `Dataset` that combines all feature categories. 

##DatasetBuilder
Insilica uses a simple trait to 

### Clinical

### Genetic Behavior

### Genetic Orphan

##Deriving Importance
  The easiest way to derive importance is by univariate correlation.  This means we will look for the pearson correlation of each genetic variant count with the clinical target.  Of course clinical target is not a numeric value, so lets fix that.
  
  1. Look up tumor_stage values
  2. 

