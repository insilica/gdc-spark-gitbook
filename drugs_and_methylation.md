# Drugs and Methylation

## Prerequisites
  To complete this tutorial you will need knowledge of:
  1. [gdc-core](1_gdc/2_a_client.md): a client for connecting to the [GDC-API](https://gdc-docs.nci.nih.gov/API/Users_Guide/Getting_Started/)
  2. [clinical-supplements](1_gdc/clinical_supplements.md): an explanation of TCGA clinical supplements

##Getting Started
Epigenetics affect drug toxicity and efficacy.  In some cases, specific epigenetic marks  make a good prognostic biomarker for cancer treatment. {Need a citation |todo}. In this example we will:

1. Use gdc-core to downlad TCGA methylation data
2. Find patient treatments
3. Find patient adverse events
4. Search for relationships between methylation and adverse events in the context of drug treatment.

###Downloading TCGA methylation data
  At the time of writing, the GDC had not completed harmonizing methylation data. When the GDC incorporates a new data type it undergoes a harmonization procedure.  Different  projects must conform to the same standards for harmonized data.
  
  The `legacy` gdc-api provides access to unharmonized data. An example of this kind of legacy data is available at https://gdc-api.nci.nih.gov/legacy/files?pretty=true. To perform a legacy query one can prepend `/legacy/` the GDC-API endpoint. GDC-Core provides a client for the legacy api as well:
  
  ```scala
  asdf
  ```
  <center> todo </center>
  
###Finding Patient Treatments
