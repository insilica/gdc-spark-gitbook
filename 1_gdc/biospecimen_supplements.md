# GDC Biospecimen Supplement Parsing
Biospecimen samples are one of the major entities within the Genomic Data Commons.  Biospecimens are physical samples derived from patient tissue.  **Biospecimen Supplements** (a GDC **data_type**) provide structured descriptions of these samples.  

Like **Clinical Supplements** {should have a chapter on parsing these|todo} (another GDC **data_type**), **Biospecimen Supplements** are xml files.   

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
{example taken from https://gdc-portal.nci.nih.gov/files/f5f83382-19e5-486d-b0f5-19bc9f679839 | caption}

As seen in {create labeling | todo}, a nested structure composes biospecimen supplements:
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
