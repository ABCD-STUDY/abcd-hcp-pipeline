# dcan bids fmri-pipeline

This software takes a bids folder as input and determines parameters
for the dcan lab's modified hcp pipeline, calling upon the proper code
to run the subject(s).

### Installation

If you wish to use this package without docker, you will need to meet the 
current version requirements of each software package contained in the 
version of the dcan pipeline code which you are using.  Note that the version 
of this software is independent of pipeline code, but should be compatible 
with pipeline-code version 2.0 and onward.


```{bash}
# clone the repository
# SSH:
git clone git@gitlab.com:Fair_lab/bidsapp.git
# HTTPS:
git clone https://gitlab.com/Fair_lab/bidsapp.git

# change directory into the repository's folder
cd bidsapp

# install the requirements from within the cloned repository
pip3 install --user -r requirements.txt
```

### Usage:

To call using docker:

```{bash}
docker run --rm \
    -v /path/to/bids_dataset:/bids_input:ro \
    -v /path/to/outputs:/output \
    dcan/fmri-pipeline /bids_input /output [OPTIONS]
```

Usage:

```{bash}
usage: run.py [-h] [--participant-label ID [ID ...]] [--all-sessions]
              [--ncpus NCPUS] [--stage STAGE] [--bandstop LOWER UPPER]
              [--check-only] [--abcd-task] [--study-template HEAD BRAIN]
              bids_dir output_dir

The Developmental Cognition and Neuroimaging (DCAN) lab fMRI Pipeline. This
BIDS application initiates a functional MRI processing pipeline built upon the
Human Connectome Project's minimal processing pipelines. The application
requires only a dataset conformed to the BIDS specification, and little-to-no
additional configuration on the part of the user. BIDS format and applications
are explained in detail at http://bids.neuroimaging.io/

positional arguments:
  bids_dir              path to the input bids dataset root directory. Read
                        more about bids format in the link above. It is
                        recommended to use the dcan bids gui or dcm2bids to
                        convert from participant dicoms.
  output_dir            path to the output directory for all intermediate and
                        output files from the pipeline, also path in which
                        logs are stored.

optional arguments:
  -h, --help            show this help message and exit
  --participant-label ID [ID ...]
                        optional list of participant ids to run. Default is
                        all ids found under the bids input directory. A
                        participant label does not include "sub-"
  --all-sessions        collapses all sessions into one when running a
                        subject.
  --ncpus NCPUS         number of cores to use for concurrent processing and
                        algorithmic speedups. Warning: causes ANTs and
                        FreeSurfer to produce non-deterministic results.
  --stage STAGE         begin from a given stage, continuing through. Options:
                        PreFreeSurfer, FreeSurfer, PostFreeSurfer, FMRIVolume,
                        FMRISurface, DCANBOLDProcessing, ExecutiveSummary
  --bandstop LOWER UPPER
                        parameters for motion regressor band-stop filter. It
                        is recommended for the boundaries to match the inter-
                        quartile range for participant group heart rate (bpm),
                        or to match bids physio data directly. These
                        parameters are only recommended for data acquired with
                        a frequency of approx. 1 Hz or more. Default is no
                        filter
  --check-only          checks for the existence of outputs for each stage.
                        Useful for debugging.

special pipeline options:
  options which pertain to an alternative pipeline or an extra stage which
  is not inferred from the bids data.

  --abcd-task           runs abcd task data through task fmri analysis, adding
                        this stage to the end
  --study-template HEAD BRAIN
                        template head and brain images for intermediate
                        registration, effective where population differs
                        greatly from average adult, e.g. in elderly
                        populations with large ventricles.
```

### Additional Information:

#### Outputs

The outputs are organized in the following structure:

output_dir/sub-id/ses-session/
- files/
- logs/

##### files

- T1w:  contains native space anatomical data as well as intermediate 
preprocessing files. 
- T1w/participantID: The participant ID folder within T1w is the FreeSurfer 
subject folder. 
- MNINonLinear: contains the final space results of anatomy in 164k 
resolution. 
- MNINonLinear/Results: final space functional data.
- MNINonLinear/fsaverage_32K: final space anatomy in 32k resolution, where 
functional data is ultimately projected.
- task-taskname: these folders contain intermediate functional preprocessing 
files.
- executive_summary: the .html file within can be opened for quality 
inspection of pipeline results.

##### logs

logs contains the log files for each stage. In the case of an error, consult 
these files in addition to the standard error/out of the app itself.


#### Rerunning

The --stage option exists so you can restart the pipeline in the case that 
it terminated prematurely

#### Special Pipelines

The special pipeline options are designed for use with specific data sets. 

If you are using an elderly or neurodegenerative population, adding a 
"study template" tends to improve results. This is generally constructed 
using ANTs to build an average template of your subjects. This template is 
then used as an intermediate warp stage, assisting in nonlinear registration 
of subjects with large ventricles.

It should be noted that "abcd-task" is not yet compatible with a bids folder 
structure, and an actual task module will be added in a future version.

#### Misc.

The pipeline may take over 24 hours if run on a single core.

Temporary/Scratch space:  If you need to make use of a particular mount
for fast file io, you can provide the docker command with an additional
volume mount: `docker run -v /my/fast/file/mnt:/tmp` on most systems, 
you can add an argument to docker: `docker run --tmpfs /tmp`

software will use spin echo field maps if they are present, then
gradient field maps, then None, consistent with best observed
performances.

For specified use of spin echo field maps, i.e. mapping a pair to each
individual functional run, it is necessary to insert the "IntendedFor"
field into the bids input sidecar jsons, which specifies a functional
run for each field map.  This field is explained in greater detail
within the bids specification.

In the case of multiband (fast TR) data, it is recommended to employ a
band-stop filter to mitigate artifactually high motion numbers.  The
band-stop filter used on motion regressors prior to frame-wise
displacement calculation has parameters which must be chosen based on
subject respiratory rate.

#### Some current limitations

diffusion field maps are still a work in progress, as this data differs
significantly between scanner makes. We will happily add new formats to 
the pipeline, so please post an issue if you run into fieldmap trouble.

The ideal motion filtering parameters have not been robustly tested
across repetition times or populations outside of adolescents.
Additionally, automatic reading of physio data from bids format has not
yet been implemented.

software does not currently support dynamic acquisition parameters for
a single modality (e.g. different phase encoding direction for 2 fmri).
Other parameters would have to be processed by creating separate bids
datasets for sessions with varied fmri parameters.

