# IR and GSTP Methylation

## Create a dataset
  We need a dataset to show a link between interventional radiation and GSTP methylation. This dataset needs a 2 critical values:
  
  1. IR response information
  2. GSTP methylation information

## IR Response Data
  To collect IR response information we look for all Genomic Data Commons cases where the patient received IR. We use `gdc-spark` to compile methylation data with response data.  The `DatasetBuilder` called `IRCases` builds this dataset and its code is given below in **Appendix - IRCases**

```scala

```

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
  ```