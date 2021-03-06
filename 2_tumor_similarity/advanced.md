# Advanced Tumor Similarity
  The [first section](/README.md) of this chapter illustrates the naivety of the basic approach to tumor similarity.  The highest ranked pair of tumors are not the same kind of cancer or even the same tissue type.  The next sections describe a variety of methods to improve rna-seq based tumor similarity.
  
## Internal Normalization
  
  Spark uses cosine similarity to calculate vector - vector similarity:
  
<center>  $$sim(A,B) = \dfrac{\|A \cdot B||}{||A||\times||B||}$$ </center>
  
  This method of similarity will bias towards dimensions with large values. To correct for this we can normalize our rna-seq matrix.
  
```scala
"Advanced Tumor Similarity" should "normalize rna-seq matrix" in {

}
```

## Normal Samples Correction
  Tumor samples frequently have associated normal data.  
  
  Unsupervised machine learning methods do not fit models to a classification endpoint. Similarity methods fall under unsupervised machine learning. Similarity methods do not fit for a specific clinical or biological endpoint.  
  
  We might ask how we can improve our similarity methods to help answer specific questions.  