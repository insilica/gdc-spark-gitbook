# Advanced Tumor Similarity
  The [first section](/README.md) of this chapter illustrates the naivety of the basic approach to tumor similarity.  The highest ranked pair of tumors are not the same kind of cancer or even the same tissue type.  
  
  Spark uses cosine similarity to calculate vector - vector similarity:
  
  $$sim(A,B) = \dfrac{\|A \cdot B||}{||A||\times||B||}$$
  
  This method of similarity will bias towards dimensions with large values. To correct for this we can normalize our rna-seq matrix.
  