# Tutorial

## Data

Test-retest T1w data for a single participant, JP01, one session acquired on HUP6 and the
other on SC3T. The data has been defaced and downsampled from its original resolution. The
data is provided only for the purpose of this tutorial.


# Tutorial steps

We will go through the following steps:

* [Preprocessing](#preprocessing)
* [Synthseg](#synthseg)
* [Cross-sectional analysis](#cross-sectional-analysis)
* [Longitudinal analysis](#longitudinal-analysis)

All output will be to the `workdir` directory under the current working directory.


## Wrapper details

We will use wrapper scripts to run the individual steps. The wrapper scripts are under
`/project/ftdc_pipeline/ftdc-picsl`. Specifically, we will use:
```bash
/project/ftdc_pipeline/ftdc-picsl/
                                 pmacsT1wPreprocessing-0.4.1/bin/submit_preproc.sh
                                 pmacsSynthSeg-0.3.0/bin/submit_synthseg_session.sh
                                 pmacsAntsnetct-0.1.0/bin/submit_antsnetct.sh
```
These scripts handle bsub submission, expect BIDS input, and produce BIDS derivative
datasets.


## wrapper output datasets

Wrapper output directories contain a `dataset_description.json` file that describes the
dataset, the input data, and the container version used.

They also contain a directory `code/logs`, where bsub logs are stored.


## Preprocessing

The only preprocessing we do at this stage is brain extraction and conforming the image to
LPI orientation. The batch input to `pmacsT1wPreprocessing` is a text file with a
participant on each line.

```bash

echo "JP01" > workdir/preproc.txt

/project/ftdc_pipeline/ftdc-picsl/pmacsT1wPreprocessing-0.4.1/bin/submit_preproc.sh \
  -i $PWD/input_bids \
  -o $PWD/workdir/brain_masks \
  workdir/preproc.txt \
  participant
```

The T1w preprocessing package is capable of doing several preprocessing steps including
neck trimming and setting the origin to the brain mask centroid. However, to avoid a GPU
bottleneck, we will only do LPI orientation followed by brain extraction.

This creates a BIDS derivative dataset `brain_masks` in the `workdir` directory.


### Preprocessing output

The output of the preprocessing step is a BIDS derivative dataset, in this example
```bash
brain_masks/
├── code
│   └── logs
│       └── ftdc-t1w-preproc_20240830_161117_31958425.txt
├── dataset_description.json
└── sub-JP01
    ├── ses-HUP6x20240509x1351
    │   └── anat
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-brain_mask.json
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-brain_mask.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-preproc_T1w.json
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-preproc_T1w.nii.gz
    │       └── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-qcslice_rgb.png
    └── ses-SC3Tx20240605x1610
        └── anat
            ├── sub-JP01_ses-SC3Tx20240605x1610_acq-sag_rec-gradwarp_desc-brain_mask.json
            ├── sub-JP01_ses-SC3Tx20240605x1610_acq-sag_rec-gradwarp_desc-brain_mask.nii.gz
            ├── sub-JP01_ses-SC3Tx20240605x1610_acq-sag_rec-gradwarp_desc-preproc_T1w.json
            ├── sub-JP01_ses-SC3Tx20240605x1610_acq-sag_rec-gradwarp_desc-preproc_T1w.nii.gz
            └── sub-JP01_ses-SC3Tx20240605x1610_acq-sag_rec-gradwarp_desc-qcslice_rgb.png
```

If there were multiple T1w images for a participant, this output structure would be
replicated for each image.

The QC images are in the `anat` directory for each session. The `qcslice_rgb.png` image is
sliced through the middle of the brain, and shows the brain image with the brain mask
overlaid in three planes.


## SynthSeg

Synthseg is a deep learning-based segmentation tool that segments the brain into cortical
and subcortical regions. Our wrapper tool has a BIDS interface and uses an existing brain
mask to center the synthseg ROI on the brain. It also combines the output probability maps
into six classes expected by `antsCortcialThickness.sh`.

There are two interfaces to `pmacsSynthSeg`, one to process a single session, another to
process a list of selected T1w images in a BIDS dataset. We will demonstrate the
single-session interface here.
```bash
for sess in HUP6x20240509x1351 SC3Tx20240605x1610; do
    /project/ftdc_pipeline/ftdc-picsl/pmacsSynthSeg-0.3.0/bin/submit_synthseg_session.sh \
        -a 1 \
        -p 1 \
        -m ${PWD}/workdir/brain_masks \
        -i ${PWD}/input_bids \
        -o ${PWD}/workdir/synthseg \
        JP01 $sess
done
```

The output, for each session, is a BIDS derivative dataset in the `workdir` directory. FOr
example, for the HUP6 session:

```bash
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-qc.tsv
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-volumes.tsv
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_dseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_dseg.tsv
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_probseg.json
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_dseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_dseg.tsv
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_label-BS_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_label-CBM_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_label-CGM_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_label-CSF_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_label-SGM_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-orig_seg-antsct_label-WM_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-SynthSeg_dseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-SynthSeg_dseg.tsv
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-SynthSeg_probseg.json
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-SynthSeg_probseg.nii.gz
sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-SynthSeg_T1w.nii.gz
```

Synthseg resamples data to 1mm isotropic resolution, this is the `space-SynthSeg`. The
output in the original session space is the `space-orig` files.


## The pmacsAntsnetct wrapper

The general usage is:

```bash
/project/ftdc_pipeline/ftdc-picsl/pmacsAntsnetct-0.1.0/bin/submit_antsnetct.sh -B bind_list \
  -i input_data -m mem_mb -n nslots -o output_data -v antsnetct_version -- \
  [antsnetct_options]
```

The options in detail:

### `-B` bind_list

A comma-separated list of bind mounts for the container. Note that the bound paths may be
written into output metadata, so if you bind things like `/path/to/masks:/data/masks`, the
output may contain references to a dataset at `/data/masks`.

### `-i` input_data

The input BIDS dataset, on the local file system. For longitudinal analysis, this should
be the output dataset from the cross-sectional analysis.

### `-m` mem_mb

Memory to request for the job in MB. This isn't a limit, but prevents the job from
starting if the requested memory is not available.

### `-n` nslots

How many slots to request. This also sets the number of threads to use in ITK processes.

### `-o` output_data

The output BIDS dataset, on the local filesystem. If this does not exist, it will be
created. A log directory `code/logs` will be created under this directory.

### `-v` antsnetct_version

The version of the ANTsNetCT container to use.


## Cross-sectional analysis

The cross-sectional pipeline is similar to `antsConcticalThickness.sh`. Its processing
steps are

1. Conform the T1w image to LPI orientation
2. Neck trim the image
3. Brain extraction
4. Segmentation
5. Cortical thickness estimation
6. Registration to the template

The pipeline is designed to be highly customizable. The LPI orientation and neck trimming
can optionally be done in the preprocessing stage. Neck trimming can be skipped if needed.


### Brain extraction

A pre-defined brain mask can be defined in the input BIDS dataset or a separate mask
dataset. If a mask is not provided, a mask will be computed using ANTsPyNet.


### Segmentation

Like in `antsCorticalThickness.sh`, the brain is segmented into six classes:
```
1. CSF
2. GM
3. WM
4. Deep GM
5. Brainstem
6. Cerebellum
```

The default method for this is ANTsPyNet `deep_atropos`.

Optionally, the segmentation can be performed using `antsAtroposN4.sh`. This requires prior
probabilities, which can be generated in three ways:
    1. External priors provided by the user (in this tutorial, we use SynthSeg output).
    2. Compute priors on the fly using deep_atropos in ANTsPyNet.
    3. Warp priors from a template - as in `antsCorticalThickness.sh`.

The output of the segmentation step is a segmentation image and a probability map for each
class.

To summarize, the options are:

* default: use deep_atropos for segmentation
* `--do-ants-atropos-n4`: use deep_atropos to get priors for `antsAtroposN4.sh`
* `--segmentation-dataset`: use a precomputed segmentation dataset
* `--segmentation-dataset` and `--do-ants-atropos-n4`: use priors from the dataset for
  `antsAtroposN4.sh`.
* `--segmentation-template`: use a template for priors in `antsAtroposN4.sh`.


### Cortical thickness estimation

Cortical thickness is estimated using ANTs `KellyKapowski`, which implements the DiReCT
method. The parameters are set to the default values in `antsCorticalThickness.sh`. The
user may reduce the number of iterations to speed up processing, this is mostly to enable
rapid testing, as thickness estimation is one of the most time-consuming steps.


### Registration to the template

Registration is to a templateflow template. The default is `MNI152NLin2009cAsym`. The user
can disable this step with the option `--template-name none`. A default templateflow
installation is provided in the `pmacsAntsnetct` wrapper, if running outside the wrapper,
the environment variable `TEMPLATEFLOW_HOME` must be defined. Any template can be used, as
long as it has both a T1w image and a brain mask.

Cortical thickness and other derivatives are resampled to the template space.


## Cross-sectional example command

```bash
/project/ftdc_pipeline/ftdc-picsl/pmacsAntsnetct-0.2.0/bin/submit_antsnetct.sh \
    -B /path/to/input_bids:/data/input_bids,/path/to/workdir:/data/workdir \
    -i /data/input_bids \
    -m 16000 \
    -n 4 \
    -o /data/workdir/antsnetct \
    -v 0.1.0 \
    -- \
    --participant JP01 \
    --session SC3Tx20240605x1610 \
    --brain-mask-dataset /data/masks \
    --segmentation-dataset /data/synthseg \
    --thickness-iterations 5 \
    --template-name ADNINormalAgingANTs \
    --template-reg-quick
```

Note that we use only 5 iterations for thickness estimation, and we use the
`--template-reg-quick` option to speed up registration. This is useful for testing, but
not recommended for final results.

Run the same command for the `HUP6x20240509x1351` session, to be ready for the
longitudinal processing.


## Longitudinal analysis

The longitudinal pipeline uses the cross-sectional output to perform a longitudinal
analysis. The steps are:

* Compute a subject-specific template (SST)
* Define a brain mask for the SST
* Register the SST to a group template
* Segment the SST
* Use the SST brain mask and segmentations to segment each session
* Compute cortical thickness for each session


### Longitudinal input

By default, all T1w images for a participant are used to compute the SST. The user can
override this by providing a list of images to use. In this example, there are two
sessions with one T1w image each, so we will use those.


### SST construction

The SST is computed using `antsMultivariateTemplateConstruction2.sh`. Both the head and
brain-extracted session images are used to compute the SST. The user can control the
relative weighting of each image with the `--sst-brain-extracted-weight` option.

### SST brain mask and segmentation

After SST construction, its brain mask is computed by averaging and thresholding the brain
masks from each of the session images. A low threshold is used so that the mask is close
to the union of the session masks resampled into the SST space.

Segmentation of the SST is done with one of four options defined by the argument
`--sst-segmentation-method`:

* 'antspynet_atropos': deep_atropos is used to generate priors, which are then used as
  priors for Atropos segmentation.

* 'antspynet': deep_atropos is used for segmentation.

* 'cx_atropos': average the session posteriors, then use as priors for Atropos.

* 'cx': average the session posteriors.

## Session segmentation

Session segmentation uses the SST brain mask and priors warped to each session space. The
segmentation is done with `antsAtroposN4.sh`.

## Cortical thickness

Cortical thickness computation is identical to the cross-sectional pipeline.


## Longitudinal example command

```bash
/project/ftdc_misc/pcook/code/pmacsAntsnetct/bin/submit_antsnetct.sh \
    -B ${PWD}/workdir/brain_masks:/data/masks,${PWD}/workdir/synthseg:/data/synthseg \
    -i ${PWD}/workdir/cx_output \
    -o ${PWD}/workdir/long_output \
    -n 4 \
    -m 16000 \
    -v 0.1.4 \
    -- \
    --longitudinal \
    --sst-segmentation-method cx \
    --participant JP01 \
    --sst-reg-quick \
    --thickness-iterations 5 \
    --template-reg-quick \
    --template-name ADNINormalAgingANTs
```

## Longitudinal output

Output for the longitudinal pipeline is at both the session and participant levels. The
tree below does not include BIDS sidecar files, which are also generated.

```bash
└── sub-JP01
    ├── anat
    │   ├── sub-JP01_desc-brain_mask.nii.gz
    │   ├── sub-JP01_desc-brain_T1w.nii.gz
    │   ├── sub-JP01_from-ADNINormalAgingANTs_to-T1w_mode-image_xfm.h5
    │   ├── sub-JP01_from-T1w_to-ADNINormalAgingANTs_mode-image_xfm.h5
    │   ├── sub-JP01_seg-antsnetct_dseg.nii.gz
    │   ├── sub-JP01_seg-antsnetct_label-BS_probseg.nii.gz
    │   ├── sub-JP01_seg-antsnetct_label-CBM_probseg.nii.gz
    │   ├── sub-JP01_seg-antsnetct_label-CGM_probseg.nii.gz
    │   ├── sub-JP01_seg-antsnetct_label-CSF_probseg.nii.gz
    │   ├── sub-JP01_seg-antsnetct_label-SGM_probseg.nii.gz
    │   ├── sub-JP01_seg-antsnetct_label-WM_probseg.nii.gz
    │   └── sub-JP01_T1w.nii.gz
    ├── ses-HUP6x20240509x1351
    │   └── anat
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-biascorrbrain_T1w.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-biascorr_T1w.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-brain_mask.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_desc-preproc_T1w.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_from-sst_to-T1w_mode-image_xfm.h5
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_from-T1w_to-sst_mode-image_xfm.h5
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_desc-thickness.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_dseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_label-BS_probseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_label-CBM_probseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_label-CGM_probseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_label-CSF_probseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_label-SGM_probseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_seg-antsnetct_label-WM_probseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-ADNINormalAgingANTs_res-01_desc-biascorrbrain_T1w.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-ADNINormalAgingANTs_res-01_desc-thickness.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-ADNINormalAgingANTs_res-01_label-GM_probseg.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-sst_desc-biascorrbrain_T1w.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-sst_desc-biascorr_T1w.nii.gz
    │       ├── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-sst_desc-thickness.nii.gz
    │       └── sub-JP01_ses-HUP6x20240509x1351_acq-sag_rec-gradwarp_space-sst_label-GM_probseg.nii.gz
    └── ses-SC3Tx20240605x1610
        └── anat
        [similar to HUP6 session]
```

The participant-level `anat/` directory contains the SST and its brain mask, its
segmentation, and a transform to the group template. The session-level `anat/` directories
contain warps to the SST. These transforms must be combined to link the session space to
the group template.


## Tutorial citations

For the T1w preprocessing:
```
Isensee F, Schell M, Pflueger I, Brugnara G, Bonekamp D, Neuberger U, Wick A, Schlemmer HP, Heiland S, Wick W, Bendszus M,
Maier-Hein KH, Kickingereder P.
Automated brain extraction of multisequence MRI using artificial neural networks.
Human Brain Mapping, 2019 Dec 1;40(17):4952-4964
```

If using SynthSeg output or priors in research, please cite its paper:
```
Billot B, Greve DN, Puonti O, Thielscher A, Van Leemput K, Fischl B, Dalca AV, Iglesias JE.
SynthSeg: Segmentation of brain MRI scans of any contrast and resolution without retraining.
Medical Image Analysis, 2023;83:102789.
```

For antsnetct:
```
Tustison NJ, Cook PA, Holbrook AJ, Johnson HJ, Muschelli J, Devenyi GA, Duda JT, Das SR, Cullen NC, Gillen DL, Yassa MA, Stone JR, Gee JC, Avants BB.
The ANTsX ecosystem for quantitative biological and medical imaging.
Scientific Reports, 2021;11:9068.
```

