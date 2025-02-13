# MatLab

Interactive MatLab sessions run as a [GUI App](https://docs.bear.bham.ac.uk/portal/gui_apps/) accessible from the [Bear Portal](https://docs.bear.bham.ac.uk/portal/accessing/). Please follow the information on the Bear Technical Docs to start up an interactive MatLab session.

Some parallelisation is available through [parfor](https://www.mathworks.com/help/matlab/ref/parfor.html) loops within single MatLab but users looking for to run many individual matlab scripts in parallel are likely to want to use the [Slurm job submissions](https://docs.bear.bham.ac.uk/bluebear/jobs/). Examples of both are included below.

## Neuroimaging toolboxes

Neuroimaging toolboxes can be added to the MatLab path on BlueBEAR in the normal way. Toolboxes can be downloaded from the developer and stored on an [RDS](https://docs.bear.bham.ac.uk/rds/accessing/) space. These folders can be added to the path within a MatLab session using `addpath`.

```
addpath(genpath('/rds/q/quinna-example-project/code/fieldtrip'))
```

These pages include some specific examples using popular MatLab toolboxes

 - [Fieldtrip](fieldtrip.md)
 - EEGLab
 - SPM

## Parallel for-loop

Simple parallelisation of a for-loop can be performed using [parfor](https://www.mathworks.com/help/matlab/ref/parfor.html). This functionality is provided by MatLab and enables faster processing of `for` loops simply by changing the syntax at the start to say `parfor` rather than `for`.

Here is an example function which makes use of `parfor` whilst computing GLMs using SPM.

```Matlab
function glm_level1(model)
% This function takes a model structure as input and performs first-level
% estimations in a General Linear Model (GLM) analysis for a set of subjects.

subjects = model.Subj;

% FIRST LEVEL (individual) estimations
% Get the number of subjects to be processed.
N = size(subjects,2);

% Iterate over each subject in "subjects" using parallel processing
parfor i = 1:N

        % Get the current subject ID from "subjects"
        id = subjects(i);

        % Get the corresponding BIDS (Brain Imaging Data Structure) ID and
        % session information for the current subject.
        BIDS_id = model.ids{id};
        BIDS_sess = model.sess{id};

        % Construct the path to the GLM folder for the current subject.
        path = [model.glmfolder BIDS_id];

        % Construct the path to the SPM.mat file for the current subject.
        modelfile = [path '/SPM.mat'];

        % Delete the existing SPM.mat file for the current subject (clean
        % up previously done models)
        delete(modelfile);

        % Create a job structure for the current subject.
        job = analysis_job_func(BIDS_id, BIDS_sess, model);

        % Create an empty cell array to be used as inputs for the "spm_jobman" function.
        inputs = cell(0,1);

        % Set the SPM defaults to 'FMRI'.
        spm('defaults', 'FMRI');

        % Run the current job using the "spm_jobman" function.
        spm_jobman('run', job, inputs{:});

end

end
```
*Example contributed by Arkady Konovalov*

> **_NOTE:_**  Make sure you specify the appropriate number of cores when starting the MatLab GUI App, you may not notice a substantial speed-up if you run MatLab using the default of 4 cores. Do try to avoid asking for substantially more than you might need however - BlueBEAR is a shared resource.

## Submitting Matlab jobs with parfor to Bear
*Example contributed by Dagmar Fraser*

The following Matlab code performs some matrix calculations on simulated data. The inclusion of a `parfor` loop means that the code can take advantage of computers with multiple CPUs to accelerate processing.

```Matlab
tic
n = 200;
A= 500;
a = zeros(1,n);
parfor i = 1:n
    a(i) = max(abs(eig(rand(A))));
end
toc
```

You can run this code in an interactive Matlab session, or save it as a script that can be executed on the big cluster. If we save this file as `parforDemo.m`, we can write a second 'submission' script to execute it on the cluster.

```
#!/bin/bash
#SBATCH --ntasks 8
#SBATCH --time 5:0
#SBATCH --qos bbshort
#SBATCH --mail-type ALL

set -e

module purge
module load bluebear
module load MATLAB/2020a

matlab -nodisplay -r parforDemo
```

If we save that second script as `RunMyCode.sh` it can be run using `sbatch RunMyCode.sh` on a terminal to send the job to the cluster.

The `ntasks` line specifies we are looking to use 8 cores. The last line contains the filename we are sending to MATLAB to execute

## Submitting multiple MatLab jobs

The previous example submits a single Matlab job that uses `parfor` BlueBEAR, for larger analyses we may want to parallelise jobs across entire matlab instances. This can be done by submitting MatLab jobs to BEAR using Slurm. The BEAR Technical Docs contain a simple example on [submitting a matlab job to bear](https://docs.bear.bham.ac.uk/bluebear/jobs/#an-example-job-script).

For neuroimaging analyses, you'll generally need to organise your scripts so that each part that you want to parallelise runs from a single function that takes a single ID as an argument. Here is a specific example that runs a function `e1_fun_ICA` on each of 48 datasets.

```
#!/bin/bash
#SBATCH --ntasks 1
#SBATCH --time 30:0
#SBATCH --mem 50G
#SBATCH --qos bbdefault
#SBATCH --array=1-48

set -eu

module purge; module load bluebear

# load the MATLAB version you need
module load MATLAB/2019b

# apply matlab script to each index in the array
# (the MATLAB script is programmed such that the input ID is used as the subject ID)
matlab -nodisplay -r "run /rds/homes/d/dueckerk/startup.m, e1_fun_ICA(${SLURM_ARRAY_TASK_ID}), quit"
```
*Example contributed by Katharina Deucker*
