# Guide to the Genomic Data Commons
  The Genomic Data Commons is a large data repository for cancer related patient data.  This book is an introduction to simple analyses made possible with access to GDC (Genomic Data Commons).  We review some of the biology, statistics and computer science behind every step of the examples.  These reviews include explanations of the biology behind GDC experimental files. We describe how to extract and organize GDC data.  We also propose methods for building computational workflows on top of this data.
  
  ## Book Breakdown
  {{come to this after the book is coming together more | todo}}

# Basic Concepts
  The Genomic Data Commons provides an API for accessing many aspects of the underlying cancer data files.  The GDC API documentation is at https://gdc-docs.nci.nih.gov/API/Users_Guide/Getting_Started/. 
  
# A Simple API
  In this book we will be making use of the Insilica scala client for the Genomic Data Commons REST API.  To include the Insilica in scala api `co.insilica.gdc-core.0.1.3-SNAPSHOT` in a scala project via:

```scala
libraryDependencies ++= "co.insilica" %% "gdc-core" % "0.1.3-SNAPSHOT"
```
{{include in build.sbt | caption}}

## Search and Retrieval
  GDC-API documentation goes into more detailed discussion of all the API capabilities. We will only discuss Four 
  
# Files, Cases and Samples oh my!
  The Genomic Data Commons is an enormous repository of cancer data.  Exploring this data can overwhelm even well-seasoned bioinformaticians.  We will focus on a subset of the available functionality and thus provide a more gentle introduction.  
  
## Files
  To search for files contained in GDC we can search for them via the GDC-API. The search functionality br
## Cases
## Samples

# GDC Biospecimen Supplement Parsing
Biospecimen samples are one of the major entities within the Genomic Data Commons.  Biospecimens represent samples and **Biospecimen Supplements** (a GDC **data_type** {{cite GDC concepts|todo}}) provide structured descriptions.  

Like **Clinical Supplements** {{should have a chapter on parsing these|todo}} (another GDC **data_type**), **Biospecimen Supplements** are xml files.   

Both Biospecimen supplements and Clinical supplements identify with a single **case_id**.  These supplements catalogue administrative data (such as time of submission or project codes) and . 

Below we diagram the anatomy of a Biospecimen supplement. Ellipses show skipped fields.   

```xml
    <admin:admin>...</admin:admin>
    <bio:patient>
      <bio:bcr_canonical_check>...</bio:bcr_canonical_check>
      <bio:samples>
        <bio:sample>
          <bio:portions>
            <bio:portion>
              <bio:analytes>
                <bio:analyte>
                  <bio:aliquots>...</bio:aliquots>
                  <bio:aliquots>...</bio:aliquots>
                </bio:analyte>
                <bio:analyte>...</bio:analyte>
              </bio:analytes>
              <bio:diagnostic_slides>
                <bio:diagnostic_slide>...</bio:diagnostic_slide>
                <bio:diagnostic_slide>...</bio:diagnostic_slide>
              </bio:slides>
            </bio:portion>
            <bio:portion>...</bio:portion>
          </bio:portions>
        </bio:sample>
        <bio:sample>...</bio:sample>
      </bio:samples>
    </bio:patient>
```
{{example taken from https://gdc-portal.nci.nih.gov/files/f5f83382-19e5-486d-b0f5-19bc9f679839 | caption}}

As seen in {{create labeling | todo}}, a nested structure composes biospecimen supplements:
1. **admin data**: file_uuid, upload_date ...
2. **patient**: patient identification (uuid, gender, ...)
  3. **canonical_check**: completion status
  4. **samples**: sample_type, dimension, weight, preservation method, sample uuid
    5. **portions**
      6. **analytes**
        7. **aliquots**
      8. **slides**: Human characterization (cell counting, percent_necrosis, ...)

Biospecimen supplement data enables diverse analyses:
 * Image analysis models from sample-portion-slide data.
 * Models predicting tumor features such as weight or dimension
 * Differential genetics from tumor vs normal samples

These ideas just touch the surface of what is possible.  

##Aliquots
Aliquots form the smallest catologued unit in biospecimen supplements.   Samples consist of biological portions and microscopy slides.  Portions  which are made up of aliquots.  Aliquots are used directly in 

# RNA-Sequencing and Sample Similarity
  Tumor similarity is an important method to enable data exploration and model creation. Similarity models rely on two components **fingerprinting** and **similarity metric**
  
  ## Tumor Fingerprinting
  We define **tumor fingerprinting** to be the creation of numeric vectors from tumor samples. Biological assays exist to measure diverse tumor and normal tissue features.  Here we will focus on rna-sequencing as a method to estimate protein expression.
  
  In this example we represent a tumor as a vector of gene expression values. Each  Genes are identified via ensembl identifiers and expression given as FPKM
  
| Tumor_Sample | ENSG | 2:0 | 3:0 | 4:0 |
 | -- | -- | -- | -- | -- |
 | Tumor_1 | 1:2 | 2:2 | 3:2 | 4:2 |
 | Tumor_2 | 1:3 | 2:3 | 3:3 | 4:3 |
 
 ###RNA-Sequencing
 The central dogma of molecular biology states 
 ####method
 ####fpkm

  
  ## Similarity Metrics
  Similarity metrics compare vectors. Specifically a similarity metric is a function $$f:R^n \times R^n \rightarrow R$$.  Similarity metrics generate large values for dissimilar vectors and small values for similar vectors.  There is a rich literature around similarity metrics {{citation for similarity literature | todo}} for this example we will use cosine similarity:
  
  <center> $$\text{cosineSimilarity}(\vec{A},\vec{B}) = cos(\theta) = \dfrac{\vec{A} \cdot \vec{B}}{\lVert \vec{A} \rVert \lVert \vec{B} \rVert}$$ </center>
  
  Cosine similarity has a value of 1 for identical vectors and 0 for perpendicular vectors. Similarity metrics are a special case of distance metrics which all hold common traits:

  1. **Identity**  
  $$f(A,B) = 0 \iff A = B$$
  2. **symmetry**  
  $$f(A,B) = f(B,A)$$
  3. **triangle inequality**  
  $$f(A,C) \le f(A,B) + f(B,C)$$
  4. **non-negativity**  
  $$\forall A,B : f(A,B) \ge 0 $$

We end our discussion of similarity metrics here but encourage the reader to read further. Wikipedia provides good introductory material and references to more detailed research.{{add more similarity references | todo}}

# gene ranking
