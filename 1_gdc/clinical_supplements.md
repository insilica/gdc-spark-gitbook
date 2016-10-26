# Clinical Supplements
  The Genomic Data Commons hosts clinical supplements.  These supplements do not all have the same file type or encoding.  The different projects supported by GDC have different ways of encoding clinical supplements.  As of 10/25/2016 these projects had the below encoding for the 'clinical supplement' `data_type`:
  
  | Project | Clinical supplement encoding | # Files | # Cases Covered |
  | --------|------------------------------|-------  | --------------|
  | TCGA    | BCR XML                      | 11160   | 11,160
  | TARGET  | XLSX                         | 8       | 1820
  
  Clinical supplements are specific to a patient and contain information on clinical outcomes.  TCGA identifies clinical outcome variables with numeric identifiers called [common data elements](https://www.nlm.nih.gov/cde/).  GDC contains data from the [TARGET](https://ocg.cancer.gov/programs/target) and [TCGA](https://cancergenome.nih.gov/) cancer projects.  
  
  In the case of TCGA, each clinical supplement is an xml file and references an individual patient. The below is an excerpt from one of these files:
  
  TARGET stores clinical data on groups of patients in each excel XLSX file. //TODO need to add more information on TARGET.  The below is an excerpt from one of these files:
  
  ##TCGA Clinical Supplements
  
  ##TARGET Clinical Supplements