# LongitudinalThickness
Development of longitudinal cortical thickness pipelines.

## FTDC-PICSL tools for longitudinal thickness

* [T1wPreprocessing](https://github.com/ftdc-picsl/T1wPreprocessing) conforms data orientation, does GPU-accelerated brain extraction using HD-BET, and outputs a QC image. Optionally, it will also perform neck trimming.

* [antsnetct](https://github.com/ftdc-picsl/antsnetct) performs neck trimming, brain extraction, segmentation, cortical thickness estimation, and normalization to a template. Both cross-sectional and longitudinal pipelines are supported.

* [synthseg](https://github.com/ftdc-picsl/SynthSeg) runs synthseg on the GPU and can generate priors for antsnetct.

All of these have wrapper scripts for PMACS users under

```
/project/ftdc_pipeline/ftdc-picsl
```
