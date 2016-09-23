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

