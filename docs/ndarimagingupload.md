---
layout: default
title: NDAR Imaging Upload
has_children: false
parent: Documentation
has_toc: false
nav_order: 10
---
# NDAR

Uploading images to NDA requires a formatted spreadsheet of including information from the JSON and the file-path to NIFTI images. Using an exported BIDS dataset and PyBIDS tools, you can download and format all images from a Flywheel project for NDA upload. For periodic uploads, only T1 data is required.
​
In addition to a BIDS-valid dataset, either on disk or on Flywheel, you will need Demographics information pulled from Tableau, including including BBLID, SCANID, NDAR-GUID, Date of Scan, Sex, Age at Scan (in months), and Visit Number.
​
If using data hosted on Flywheel, start by exporting to a cluster or external hard drive using the [/Flywheel CLI](https://docs.flywheel.io/hc/en-us/articles/360008224733-Exporting-a-BIDS-Project-using-the-CLI) and the command `fw export bids -p "PROJECT_NAME" ~/path/to/directory/BIDS`. If only sharing anatomical data  (ie:  T1w, T2w) use the `--data-type anat` call to specify. Otherwise download all data-types that are required to share.
​
Next, using PyBIDS/NiBABLE parse to parse file names into a CSV file of relevant information (ie: subject and session labels, data type, file name, and other imaging parameters) as follows:
​
Formatting might be off - it is recommended to copy this code into a code block in Slack, then copy it from there and execute it.

``` R
pip install bids
from bids import BIDSLayout
from bids.layout import parse_file_entities
import glob
import pandas as pd
import nibabel as nb
import json
​
root_dir = '/path/to/directory' #The folder containing the BIDS directory 
bids_dir = '/BIDS' #The folder that contains all the subjects 
all_files=glob.glob(root_dir + bids_dir + '**/**/**/**/*') # make a list of all images in the BIDS directory
all_niftis=[] #filter out jsons, as we only need niftis

for file in all_files:
    if '.nii.gz' in file: # and 'dwi' in file - add one more if statement if you want only a specific type of nifti
        all_niftis.append(file)

print(len(all_niftis)

scans = [] #for each file in the list, parse the information into a dictionary, add the filename + path key value pair, as well as other fields from the NIFTIS and add it to the list we just initialized
​
for file in all_niftis:
    img = img = nb.load(file)
    voxel_sizes = img.header.get_zooms()
    matrix_dims = img.shape
    result = parse_file_entities(file)
    result['filename']=file
    result["VoxelSizeDim1"] = float(voxel_sizes[0])
    result["VoxelSizeDim3"] = float(voxel_sizes[2])
    result["VoxelSizeDim2"] = float(voxel_sizes[1])
    result['acquisitionmatrix']=matrix_dims
    result["Dim1Size"] = matrix_dims[0]
    result["Dim2Size"] = matrix_dims[1]
    result["Dim3Size"] = matrix_dims[2]
    scans.append(result)

df = pd.DataFrame(scans) #and convert to a dataframe and save
df.to_csv('nda_scans.csv', sep=',')
```
You will also need to get information on the version of dcm2nii used to convert each scan. To do this, we can use an SDK script.
​
```R
import pandas as pd
import flywheel
fw = flywheel.Client()
EFProj=fw.projects.find_first('label=EFR01') #select the Flywheel project you are working on and iterate through all subjects and sessions
subs= EFProj.subjects()
sessions=[]
for s in subs :
    tempsessions=s.sessions()
    sessions.extend(tempsessions)
​

versions=[]  #create a list of tuples containing session ID + dcm_version
for r in sessions:
  acq=r.acquisitions()
  for a in acq:
      a = fw.get(a.id)
      files=a.files
      types=[x.type for x in files]
      for fi in files:
          if 'T1w' in a.label and 'setter' not in a.label and 'nifti' in fi.type: #for DWI, this would be: # if 'dwi' in a.label and 'nifti' in fi.type
              dcm_info=fi['info'].get('ConversionSoftwareVersion',None)
              versions.append((r.label, 'dcm2niix', dcm_info))



df=pd.DataFrame(versions) #convert to dataframe and save as a .CSV
df.columns = ['a', 'b', 'c']
df.to_csv('EF_dcm_version.csv', sep=',')
```

Finally, use R to read in the image03 template and format the information from Flywheel along with demographics from tableau to NDA specifications. Step through code, paying close attention to values that may need to be changed for specific protocols. Currently, the example included in the code is for BBL ABCD protocols (EF, Motive, etc) but may need to be changed based on specifications included in CAMRIS protocol (ie: TE in line 54, TR in line 51, or lines that assign different values for different scan types (ie: T1 vs T2 scans).  
​
In R: 

```R

#begin filling data into the template. most of this information can be found in the CAMRIS protocol or in the json files.
#note sections where columns are separated into 2 categories for T1 and T2 scans, and whether this will
#be necessary for the data submitted for your specific protocol.

library(dplyr)
library(tidyr)
library(naniar)
library(car)
​
setwd('~/Desktop/informatics/ndar')
nda_scans<-read.csv('nda_scans.csv')
scan_data<-read.csv('scan_data.csv',header =T, na.strings=c(""))
image03_headers<-read.csv('image03_template.csv', header= TRUE, sep=',') #import the image_03 template
dcm<- read.csv('EF_dcm_version.csv')
dcm<- dcm %>% unite('dcm_ver', b:c, remove=FALSE)
​
​
data<-merge(nda_scans, scan_data, by.x='session', by.y='scanid')
data<-merge(data, dcm, by.x='session', by.y='a')

image03<-data.frame(data$guid)
image03<-image03%>% rename(subjectkey = data.guid)
image03$src_subject_id<-data$bblid
image03$interview_date<- data$doscan
image03$interview_age<- data$scanagemonths
image03$sex<- data$sex

image03$comments_misc<- data$session #keep Scan ID saved as comments
image03$image_file<- data$filename


image03$image_thumbnail_file<-"" #using this as a place holder at the moment
image03$image_description<- "MRI"
image03$experiment_id<- ""
image03$scan_type<-data$suffix


image03$scan_type[ image03$scan_type == 'T1w' ]<-'MR structural (T1)'
image03$scan_type[ image03$scan_type == 'T2w' ]<-'MR structural (T2)'
image03$scan_object<- "Live"
image03$image_file_format<- "NIFTI"
image03$data_file2<- ""
image03$data_file2_type<-""
image03$image_modality<- "MRI"
image03$scanner_manufacturer_pd<- "SIEMENS"
image03$scanner_type_pd<- "MAGNETOM Prisma_fit"
image03$scanner_software_versions_pd<-'VE11C'
image03$magnetic_field_strength<-"3T"

image03$mri_repetition_time_pd[ image03$scan_type == 'MR structural (T1)' ]<-2.5 #mri_repetition_time_pd for EF: T1= 2.5, for T2=3.2
image03$mri_repetition_time_pd[ image03$scan_type == 'MR structural (T2)' ]<-3.2

image03$mri_echo_time_pd[ image03$scan_type == 'MR structural (T1)' ]<-0.0029 #mri_echo_time_pd for T1=2.9 ms , for T2= 565 ms
image03$mri_echo_time_pd[ image03$scan_type == 'MR structural (T2)' ]<-0.565

image03$flip_angle[ image03$scan_type == 'MR structural (T1)' ]<-'8.0 deg' #flip_angle for T1=8.0 deg, T2=missing?
image03$flip_angle[ image03$scan_type == 'MR structural (T2)' ]<-'120.0 deg'


image03$acquisition_matrix<-data$acquisitionmatrix
image03$mri_field_of_view_pd<-data$acquisitionmatrix #mri_field_of_view_pd	(dim1 x voxelsize1, dim2 x voxelsize2, dim3 x voxelsize3) or the same as acquisition matrix
image03$patient_position<-'L0.0 A20.0 F30.0 mm' #patient_position	is L0.0 A20.0 F30.0 mm for both


image03$photomet_interpret<- 'grayscale'
image03$receive_coil<- ''
image03$transmit_coil<- ''
image03$transformation_performed<-'No'
image03$transformation_type<-''
image03$image_history<-''
image03$image_num_dimensions<-3
image03$image_extent1<-data$Dim1Size
image03$image_extent2<-data$Dim2Size
image03$image_extent3<-data$Dim3Size
image03$image_extent4<-''
image03$extent4_type<-''
image03$image_extent5<-''
image03$extent5_type<-''
image03$image_unit1<-'Millimeters'
image03$image_unit2<-'Millimeters'
image03$image_unit3<-'Millimeters'
image03$image_unit4<-''
image03$image_unit5<-''
image03$image_resolution1<-data$VoxelSizeDim1
image03$image_resolution2<-data$VoxelSizeDim2
image03$image_resolution3<-data$VoxelSizeDim3
image03$image_resolution4<-''
image03$image_resolution5<-''
image03$image_slice_thickness<-'1.00'
image03$image_orientation<-'Sagittal'
image03$qc_outcome<-''
image03$qc_description<-''
image03$qc_fail_quest_reason<-''
image03$decay_correction<-''
image03$frame_end_times<-''
image03$frame_end_unit<-''
image03$frame_start_times<-''
image03$frame_start_unit<-''
image03$pet_isotope<-''
image03$pet_tracer<-''
image03$time_diff_inject_to_image<-''
image03$time_diff_units<-''
image03$pulse_seq<-''
image03$slice_acquisition<-''
image03$software_preproc<-data$dcm_ver
image03$study<-''
image03$week<-''
image03$experiment_description<-''
image03$visit<-data$timepoints


image03$visit[ image03$visit == '1' ]<-'baseline'
image03$visit[ image03$visit == '2' ]<-'followup'
image03$slice_timing<-''
image03$bvek_bval_files<-''
image03$bvecfile<-''
image03$bvalfile<-''
image03$deviceserialnumber<-''
image03$procdate<-''
image03$visnum<-data$timepoints
​
write.csv(image03, file='image03_NDA.csv')
# there are more fields that need to be calculated, but they are empty (conditional on other modalities)

```

Once you write out the .csv, copy/paste the data into the original template from NIH, and the rest of the fields will generate blank. It is recommended to copy this data into the template downloaded directly from NDA, as the validator is extremely particular, and the headings need to match identically.
