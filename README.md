# MRI Processing Scripts — Manual
## Introduction
Before starting, make sure that you have sufficient knowledge about the Terminal, which you will use frequently throughout this process in both Linux and Mac. If you do not know how to use the Terminal, watch a few quick tutorials on YouTube to orient yourself. Some recommended tutorials are listed below (note that the commands are the same on Linux and Mac).

[Linux Terminal Tutorial Episode 1: Back to Basics](https://www.youtube.com/watch?v=2FiQSLdnBqA)  
[Linux Terminal Tutorial Episode 2: File Operations](https://www.youtube.com/watch?v=_JWj6u8mI7k)

MRI data is organized first by Medical Record Numbers (MRN) and then by date. Most of the scripts will be run from the parent directory `/MRI_data` and use a variable to specify which MRN and date is being processed.

It is important to read through each section before copying any lines to the command line, as it might explain necessary steps to complete prior to running the scripts. Pay attention to semicolons when copy/pasting! Text wrapping may create multiple lines, but semicolons will denote the end of a single line of script.

## Overview
When a fetal MRI is acquired, one major problem is the high movement of the fetus. The blurring of images caused by the extra motion is undesirable for brain analysis in the clinical setting as well as in the research setting. To try to solve this problem, radiologists need to acquire the brain images in the three directions (axial, coronal, and sagittal) individually. By doing this, each set of MRI data is going to show one direction clearly and the two other directions will appear blurry. In the clinical setting, this is adequate, but for research, we need to have good data in all three directions for proper segmentation. Thus, we need to transform our three sets of data with blurred motion into one good set of data.

We begin by masking the region of interest, which can be performed by a neural network much faster than manual methods. The function of the mask is to delimit the region of the MRI that will be used in motion correction.

The next step is non uniformity correction. In an MRI image, the intensity measured from a tissue is supposed to be the same value, but actually it varies smoothly across the image. We usually call it “intensity inhomogeneity” which is attributed to poor radio frequency coil and biased magnetic field. Correcting intensity non uniformity will allow us to get reliable results in the following steps.

Now the image is ready for reconstruction.

After a successful motion correction and reconstruction is achieved, you will have to align the data so that all images are oriented in roughly the same way (this allows for easier segmentation and comparison between cases). Alignment is also performed automatically, but requires manual correction afterwards in order to ensure correct orientation of the fetal brain.

Now that the data has been masked, corrected, and reconstructed, we can label the structures in the fetal brain. This process is the segmentation, and is performed by another neural network that assigns each voxel of the brain volume a value that corresponds to a structure name. In our case, we are interested in only segmenting four main structures: 

* The right cortical plate
* The rest of the right cerebral tissue (including white matter, ventricles, etc.)
* The left cortical plate
* The rest of the left cerebral tissue (including white matter, ventricles, etc.)

## SSH
`ssh -Y [your network ID]@[machine name]`

All fetal MRI data processing must happen on an FNNDSC machine.

We will get errors running some scripts (Freeview) if we don’t use the `-Y` flag! 

## Environment Setup
`ss;`  
`. neuro-fs stable 6.0;`  
`FSLDIR=/neuro/users/henry.pehr/arch/Linux64/packages/fsl/6.0;`  
`. ${FSLDIR}/etc/fslconf/fsl.sh;`  
`PATH=${FSLDIR}/bin:${PATH};`  
`export FSLDIR PATH;`  

These lines must be run every time we `ssh` into a FNNDSC machine in order to run FreeSurfer applications (Freeview) in your terminal. 

## Setting Variables in Terminal
Using variables in Terminal should feel very familiar. Setting a variable to a file path can save us a lot of time when we are running scripts. Here is an example of something we might do:

> `file=1234567/2008.04.06-031Y-MR_Fetal_Body_Indications-12345/;`

Now, when we run processing scripts on the MRI data in this directory we can simply type `file` instead of a long file path.
Keep in mind that variables must be reassigned each time we `ssh` or open a new terminal tab.
We can print the value of a variable like this:

> ` echo $[variable]`

As we can see, most of the processing scripts use the `file` variable to access the specific working directory. It is important to set this variable before we start processing any data.

## Brain Mask
`/neuro/labs/grantlab/research/HyukJin_MRI/code/brain_mask2 --target-dir ./${file}/ ;`  
`python /neuro/labs/grantlab/research/HyukJin_MRI/code/brain_masking_auto_v02.py ./${file}/;`  

This code masks out the region of interest (brain) in MRI scans. 

The first script uses a 2D fully-connected CNN to extract the fetal brain and generates `[filename]_mask.nii` files. These are simply a black and white mask that denote where the brain region exists in the MRI scans. 

The second script combines the mask and original scan to generate a masked scan of the full brain region. It also enhances the color map of the brain scans and organizes the data into these directories:

* `/brain`	contains the masked brain scans with enhanced color map
* `/masks`	contains the black and white mask images
* `/raw`	contains the original, full infant MRI scans

## Non Uniformity Correction
`mkdir ./${file}/nuc/;`  
`singularity exec docker://fnndsc/pl-ants_n4biasfieldcorrection:0.2.7.1 bfc ${file}/brain/ ${file}/nuc/;`  

First, a new directory must be created for the corrected MRI scans. The script completes a process called ‘non-uniformity correction’. 
Read more on this process [here](https://doi.org/10.1016/j.media.2005.09.004) .

The corrected scans are placed in the `/nuc` directory and have identical filenames as the scans in `/brain`.

## Quality Assessment
`singularity exec --nv docker://fnndsc/pl-fetal-brain-assessment:1.3.0 fetal_brain_assessment ${file}/nuc/ ${file}/;`  

This step checks the quality of the MRI data thus far and chooses the highest quality images to crop and move to a directory named `/Best_Images_Crop`. These images will be used for the next step: brain age estimation.

The evaluation of the quality is exported to `$file/quality_assessment.csv` and contains the filename, quality score, and slice thickness for each scan.

## Brain Age Estimation
`cp -r ./${file}/Best_Images_crop/ ./${file}/estimated_brain_age/;`  
`python /neuro/labs/grantlab/research/HyukJin_MRI/code/make_verify_single.py ./${file}/estimated_brain_age/;`  
`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/resample_inplane.py "./${file}/estimated_brain_age/";`  
`/neuro/labs/grantlab/research/HyukJin_MRI/code/fetal_brainage_predic -input "./${file}/estimated_brain_age/*_brain_crop_dn_rsl.nii.gz" -output ./${file}/estimated_brain_age/brain_age_rsl.txt -detail_output ./${file}/estimated_brain_age/slice_age_list_rsl.txt;`  
`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/mode_brain_age.py ./${file}/estimated_brain_age/slice_age_list_rsl.txt;`  

These five scripts perform brain age estimation. It begins by copying the contents of `/Best_Images_crop` to a new directory `/estimated_brain_age`.

Four files are generated for each scan file:

* `[filename]_brain.nii` 			identical data from /Best_Images_crop
* `[filename]_brain_crop.nii.gz`     		data cropped differently
* `[filename]_brain_crop_dn.nii.gz`  		data cropped differently and smoothed
* `[filename]_brain_crop_dn_verify.png`		.png file with a few brain slices for manual verification

Next, the smoothed scan is resampled for each MRI and is saved as `[filename]_brain_crop_dn_rsl.nii.gz`. 

The final two scripts calculate the Estimated brain age and generate two directories:

* `Brain_age_rsl.txt`			contains the mode estimated brain age
* `Slice_age_list_rsl.txt`		lists all MRI scans and their respective estimated brain ages

## Reconstruction
`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/reconstruction_auto.py ${file} 0.4 1;`  
`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/reconstruction_auto.py ${file} 0.4 2;`  
`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/reconstruction_auto.py ${file} 0.4 3;`  

Reconstruction takes roughly ~4 hours to run, and must be performed on one of the three computing cluster nodes: `rc-drno`, `rc-twice`, `rc-golden`.

We can access a computing cluster the same way we access a FNNDSC machine; `ssh` into the node using your network ID and the machine name. Remember to rebuild the environment and reinitialize the `file` variable again, since a new secure shell has been started. The three reconstructions should be run in parallel on the cluster by running each one in a new terminal window.

Note the numbers at the end of each script line. The scripts process the best three reconstructions for the MRI data so that if a reconstruction fails, processing can still proceed using the other reconstructions. Each script creates a numerically ranked directory `/recon_#` where the reconstruction files are stored. 

Once reconstruction has successfully completed, each `/recon_#` should have these files inside them:

* `/recon_1/recon_Best.nii`  
* `/recon_2/recon_Second.nii`  
* `/recon_3/recon_Third.nii`  

Continue processing with the successful `/recon_#` directories. 

## Alignment
`mkdir ${file}/recon_1/recon/;`  
`cp ${file}/recon_1/recon_Best.nii ${file}/recon_1/recon/recon.nii;`  
`mkdir ${file}/recon_2/recon/;`  
`cp ${file}/recon_2/recon_Second.nii ${file}/recon_2/recon/recon.nii;`  
`mkdir ${file}/recon_3/recon/;`  
`cp ${file}/recon_3/recon_Third.nii ${file}/recon_3/recon/recon.nii;`  

`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/alignment_updated_HJ.py ${file}/recon_1/;`  
`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/alignment_updated_HJ.py ${file}/recon_2/;`  
`python3 /neuro/labs/grantlab/research/HyukJin_MRI/code/alignment_updated_HJ.py ${file}/recon_3/;`  

`~/arch/Linux64/packages/ANTs/current/bin/N4BiasFieldCorrection -d 3 -o ${file}/recon_1/segmentation/recon_to31_nuc.nii -i ${file}/recon_1/segmentation/recon_to31.nii;`  
`~/arch/Linux64/packages/ANTs/current/bin/N4BiasFieldCorrection -d 3 -o ${file}/recon_2/segmentation/recon_to31_nuc.nii -i ${file}/recon_2/segmentation/recon_to31.nii;`  
`~/arch/Linux64/packages/ANTs/current/bin/N4BiasFieldCorrection -d 3 -o ${file}/recon_3/segmentation/recon_to31_nuc.nii -i ${file}/recon_3/segmentation/recon_to31.nii;`  

Be sure to `ssh` back into the FNNDSC machine for the rest of the MRI processing. Again, remember to rebuild the environment and reinitialize the `file` variable.

The first set of scripts is self-explanatory.

The second set of scripts process the alignment of the MRI images and creates three new directories within `/recon_#`:

* `alignment/`			contains templates that we will check later
* `recon/`			contains a copy of the recon file in the parent directory
* `segmentation/`		contains segmentation files that we will process later

The third set of scripts perform non-uniformity correction in `/segmentation` and create a new file named `recon_to31_nuc.nii`.

## Alignment Correction
`convert_xfm -omat InvAligned-${template}.xfm -inverse Temp-Recon-7dof-${template}.xfm;`  
`convert_xfm -omat recon_to31.xfm -concat`  
`/neuro/labs/grantlab/research/HyukJin_MRI/templates_for_alginment/template-${template}/template-${template}to31.xfm InvAligned-${template}.xfm;`  
`flirt -in recon.nii -ref /neuro/labs/grantlab/research/HyukJin_MRI/templates_for_alginment/template-31/template-31.nii -out recon_to31.nii.gz -init recon_to31.xfm -applyxfm;`  
`rm recon_to31.nii;`  
`gunzip recon_to31.nii.gz;`  
`cp recon_to31.nii ../segmentation;`  
`cp recon_to31.xfm ../segmentation;`  

Before running these scripts, cd into each `/recon_#/alignment` directory and check the images in `/recon_#/alignment/Temp-Recon-7dof-*.nii`. Find a well-aligned template and set a new variable `template` to the number of the template (i.e. `template=27`).

Run the alignment correction scripts from the `/alignment` directory. Do this for each `/recon_#` directory. 

## Segmentation
`/neuro/labs/grantlab/research/HyukJin_MRI/code/fetal_cp_seg2 -input ${file}/recon_1/segmentation/recon_to31_nuc.nii -output ${file}/recon_1/segmentation/;`  
`/neuro/labs/grantlab/research/HyukJin_MRI/code/fetal_cp_seg2 -input ${file}/recon_2/segmentation/recon_to31_nuc.nii -output ${file}/recon_2/segmentation/;`  
`/neuro/labs/grantlab/research/HyukJin_MRI/code/fetal_cp_seg2 -input ${file}/recon_3/segmentation/recon_to31_nuc.nii -output ${file}/recon_3/segmentation/;`  

Make sure that the working directory is `/MRI_data` before running these scripts.

This script uses deep learning to generate a label map `recon_to31_nuc_deep_agg.nii.gz` for the processed MRI scan in the `/recon_#/segmentation` directories.

Select “Lookup Table” as the Color Map when viewing this volume in Freeview.


