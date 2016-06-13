# neuropointillist
Flexible modeling of neuroimaging data in R

##Usage
`npointillist --set1 listoffiles1.txt --setlabels1 file1.csv --set2 listoffiles2.txt  --setlabels2 file2.csv`
`--covariates covariatefile.csv  --mask mask.nii.gz --model code.R  [ -p N | --sgeN N] --output output`
`--debugfile outputfile `

_Right now the code is unpackaged. However, I am working on packaging it and when that is complete it will be possible to use standalone R scripts OR to write your own R package and call the R routines_

File inputs that are supported are nifti files. _Cifti, and mgz files will be supported in the future.  Alternatively, the user should also be able to supply a CSV file with the data in it, for other types of neuroimaging analysis that might not conform to this model._ The file type is determined simply by the extension (.nii = cifti, .nii.gz = nifti, .mgz is vertex surface). 

`--set1`, `--set2`, ..`--set5`
 By default, the command line variant of the program supports up to 5 input sets. If more than five files are necessary one should use the programmatic interface. Each setfile (`listoffiles1.text ... listoffiles2.txt`) is a text file that contains the filenames in each set, one per row. These sets can correspond to longitudinal data points. They do not have to have the same subjects in each set (i.e., there can be missing data if the models you intend to use support that). **At least one set and setlabel must be provided **
 
 `--setlables1`, `--setlables2`, ..`setlables5`
 
The setlabel files are csv files that specify variables that correspond to the files in the sets provided above. There must be exactly the same number of setlabel files as sets. If the files are three dimensional, the list of files in the set  should have exactly the same number of entries as the corresponding label csv file.  If the files are four dimensional (fMRI data), the corresponding label csv files should include one entry per volume (e.g., the TR). The data from the setlabels must be provided in the same order as the data in the set files. Normally, set label files will include (at the least) an id number and a time point for each 3D file that is specified. For 4D files, set label files will probably include an id number, time point, TR and elements of the fMRI design matrix. The headers of the setlabel files must be consistent across sets, and consistent with headers in the covariate file, if specified, below. **At least one set and setlabel must be provided **

`--covariates`  Subject identifiers can be associated with any number of covariates as specified in the covariate file, which are csv files. These files can contain additional information about subjects, for example, age at a particular time or IQ. All information in covariate files can also be specified in the setlablels; this option is largely a convenience option. If a covariate file is specified, it will be merged with the content of the setlabel files based on all the header fields that are common to both. An error will occur if there are no header fields in common. 

|Subjectid, |time, |age, |IQ |
|----------|-----|----|---|
|1,         | 1,   |13,  |110|
|2,         |1,    |13.2,|115|
|3,         |1,    |12.9,|98 |


`--mask` The mask must be a file of the same type and size of the first three dimensions  of all the set inputs. The mask must contain 1s and zeros only and computation will be limited to those voxels/vertices which are set to 1 in the mask. (**required**)

`--model` The model specifies the R template code to run the model and return results.  This can also include any initialization code (e.g., libraries that must be included. The model must define the function `processVoxel(v)` that takes as an argument a voxel number `v`. (**required**).

`-p x` The `-p` argument specifies that multicore parallelism will be implemented using `x` processors. An warning is given if the number of processors specified exceeds number of cores. **See notes below on running a model using multicore parallelism.**

`--sgeN N` Alternatively, the `--sge` argument specifies to read the data and divide it into `N` jobs that can be submitted to  the SGE (using a script that is generated called, suggestively, `runme`) or divided among machines by hand and run using GNU make. If SGE parallelism is used, we assume that the directory that the program is called from is read/writeable from all cluster nodes. **See notes below on running a model using SGE parallelism.**

`--output` Specify an output prefix that is prepended to output files. This is useful for organizing output for SGE runs; you can specify something like `--output model-stressXtime/mod1` to organize all the output files and execution scripts into a subdirectory. 

`--debug debugfile` Write out external representations of the design matrix and the fMRI data to the file `debugfile`. This may be useful for development and testing of your model, or troubleshooting problems with the setfiles or covariate files.

## General considerations for running in parallel
You will be doing the same thing over a lot of voxels in some kind of loop. If your model generates an error, you will lose the entire job. Therefore, you want to be proactive about trapping errors of all types. Specifically, multicore parallelism is a little touchier with respect to error handling than running in a loop, so it is entirely possible that a code that runs correctly for SGE parallelism does not run on a multicore.

Practically, I recommend running using SGE parallelism to give yourself the widest range of opportunities to complete your job using limited resources. 

## Running a model using SGE parallelism 

The `readargs.R` file in `example.rawfmri` is configured so that it will create a directory called `sgetest` with the assembled design matrix file (in rds format), the split up fMRI data (also in rds format), and files to run the job. These files are:

`Makefile` This file contains the rules for running each subjob and assembling the results. Note that the executables `npointrun` and `npointmerge` must be in your path environment. You can run your job by typing `make -j <ncores>` at the command line in the `sgetest` directory. You can also type `make clean` to remove all the intermediate files once your job has completed and you have reassembled your output (by any method).


`sgejob.bash` This is the job submission script for processing the data using SGE. Note that `npointrun` needs to be in your path. 

`runme` This script will submit the job to the SGE and call Make to merge the resulting files when the job has completed. It is an SGE/Make hybrid.

## Running a model using multicore parallelism

The `readargs.R` file in the `example.flournoy` directory is configured so that it will use 24 cores to compare two models. Make sure that you change this number to be lower if your  machine does not have 24 cores.  

Note that the syntax for trapping errors is a little bit different. We check to see whether the error inherits from `try-error` as in this example. 

``` R
processVoxel <-function(v) {
    Y <- voxeldat[,v]
    e <- try(mod1 <- lme(Y ~ AllCue+AllCueDerivative+AllTarget+AllTargetDerivat
ive+conf1+conf2+conf3+conf4+conf5+conf6+conf7+conf8+conf9+conf10 +conf11+conf12
+conf13+conf14, random=~1|idnum, method=c("ML"), na.action=na.omit, control=lme
Control(returnObject=TRUE,singular.ok=TRUE)))
    if(inherits(e, "try-error")) {
        mod1 <- NULL
    }
    if(!is.null(mod1)) {
        retvals <- list(summary(mod1)$tTable["AllCue", "t-value"],
                        summary(mod1)$tTable["AllTarget", "t-value"])
    } else {
        retvals <- list(999,999)
    }
    names(retvals) <- c("tstat-AllCue", "tstat-AllTarget")
    retvals
}
```
