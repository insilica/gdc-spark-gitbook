# RNA-Sequencing and Sample Similarity
  Tumor similarity is an important method to enable data exploration and model creation. Similarity models rely on two components **fingerprinting** and **similarity metric**
  
  ## Tumor Fingerprinting
 **Tumor fingerprinting** is the creation of numeric vectors from tumor samples. Biological assays exist to measure diverse tumor and normal tissue features.  Here we will focus on rna-sequencing as a method to estimate protein expression.
  
  In this example we represent a tumor as a vector of gene expression values. Ensembl identifiers identify genes and FPKM {cite this | todo} encodes numeric expression values.
  
| Tumor_Sample | ENSG | 2:0 | 3:0 | 4:0 |
| -- | -- | -- | -- | -- |
| Tumor_1 | 1:2 | 2:2 | 3:2 | 4:2 |
| Tumor_2 | 1:3 | 2:3 | 3:3 | 4:3 |

The below code shows how to build an RNA Dataset like the above:
```scala

```
 ###RNA-Sequencing
 The central dogma of molecular biology states 
 ####method
 ####fpkm

  
  ## Similarity Metrics
  Similarity metrics compare vectors. Specifically a similarity metric is a function $$f:R^n \times R^n \rightarrow R$$.  Similarity metrics generate large values for dissimilar vectors and small values for similar vectors.  There is rich literature around similarity metrics {citation for similarity literature | todo}. We use cosine similarity in this example:
  
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

We end our discussion of similarity metrics here but encourage the reader to read further. Wikipedia provides good introductory material and references to more detailed research.{add more similarity references | todo}