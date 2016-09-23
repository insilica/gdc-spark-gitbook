# Tumor Aggression
  Cancer aggression describes cancer lethality, metastasis, growth velocity, and other prognostic features.  Aggression metrics include categorical, numeric and boolean features:
  
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

## Per case tumor aggression 
  Per-case univariate aggression is conceptually simpler than per-biological feature.  In this model algorithms combine different prognostic features into a single numeric feature.  For example, a **feature reduction** could combine tumor stage and lymphatic invasion. The resulting global metric loses information, but is easier to analyze. Ultimately per case tumor aggression enables us to label a tumor as 'very aggressive' or 'not aggressive.'  We show in the below examples how principal component analysis can derive a per-case tumor aggression metric.  
  
  | tumor stage | lymphatic invasion | distance metastasis | **tumor aggression** |
  |-------------|-------------|-------------|-------------|
  | IV | True | True | 1.0 |
  | III | True | True | 0.8 |
  | I | False | False | 0.0 | 
  <center>Per case tumor aggression derived from tumor stage, lymphatic invasion, and distance metastasis observations.  Note that aggression metrics are not necessarily normalized between 0 and 1.</center>  

## Per feature tumor aggression
  Per-feature tumor aggression determines whether a given biological observation (or feature) is indicative of cancer aggression.  