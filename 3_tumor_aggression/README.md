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
  
  | tumor stage | lymphatic invasion | distant metastasis | **tumor aggression** |
  |-------------|-------------|-------------|-------------|
  | IV | True | True | 1.0 |
  | III | True | True | 0.8 |
  | I | False | False | 0.0 | 
  <center>Per case tumor aggression derived from tumor stage, lymphatic invasion, and distance metastasis observations.  Note that aggression metrics are not necessarily normalized between 0 and 1.</center>  
  
  

## Per Feature tumor aggression
  Per-feature tumor aggression determines whether a given biological observation (or feature) is indicative of cancer aggression.  For example, the lack of expression of tumor suppressor gene P53 would  indicate a more aggressive tumor.  
  
  Per feature tumor aggression correlates a biological observation with each clinical endpoint.  These correlations are then combined into a single confidence value.  A biological feature that highly correlates with many clinical endpoints is concered an 'aggressive' feature.  In our P53 example we:

1. Identify a biological feature type (expression of P53)
2. Identify clinical endpoints (eg. tumor stage and lymphatic invasion)
3. Correlate the biological feature with each clinical endpoint
4. Fuse the correlation values in 3 to form a single per feature tumor aggression metric

The [cancer regulome](http://explorer.cancerregulome.org/) uses per feature tumor aggression. The [regulome supplemental document pg 39](http://www.nature.com/nature/journal/v487/n7407/extref/nature11252-s1.pdf) describes their approach, which we attempt to replicate.  The cancer regulome correlates genetic features with 6 clinical outcomes. These correlations are then "fused" via [Fishers Combined Statistic](https://en.wikipedia.org/wiki/Fisher%27s_method). Note that clinical outcomes are highly correlated. The cancer regulome accounts for non-independence with more statistical methods.

## Pros and Cons

## Mathematical Examples
