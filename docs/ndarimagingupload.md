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
all_niftis=[] #filter oaut jsons, as we only need niftis

for file in all_files:
    if '.nii.gz' in file: # and 'dwi' in file - add one more if statement if you want only a specific type of nifti
        all_niftis.append(file)

print(len(all_niftis))

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
EFProj = fw.projects.find_first("label=EFR01")
sessions = EFProj.sessions()

versions = []  # create a list of tuples containing session ID + dcm_version
count = 0
for r in sessions:
    acq = r.acquisitions()
    for a in acq:
        a = a.reload()
        files = a.files
        for fi in files:
            if ( # for T1: if 'T1w' in a.label and 'setter' not in a.label and 'nifti' in fi.type:
                (
                    "dwi" in a.label
                    or "dwi" in fi.name
                    or "Diffusion" in fi.classification.get("Measurement", [])
                )
                and "setter" not in a.label
                and "nifti" in fi.type
            ):  # for DWI, this would be: # if 'dwi' in a.label and 'nifti' in fi.type:
                print(fi.name)
                print(f"{count=}")
                count = count + 1
                dcm_info = fi["info"].get("ConversionSoftwareVersion", None)
                versions.append((r.label, "dcm2niix", dcm_info))

print(len(versions))
df = pd.DataFrame(versions)  # convert to dataframe and save as a .CSV
df.columns = ["a", "b", "c"]
df.to_csv("EF_dcm_version.csv", sep=",")
df.columns = ['a', 'b', 'c']
df.to_csv('EF_dcm_version.csv', sep=',')
```

> Warning
> This code has created issues before - sometimes it does not catch all the files. I recommend adding any missing files in manually. You can copy paste the sessions generated from the first code > > block into a column `A` in a .csv called `test`, and then the sessions generated from this code into the column `B`. The code below will help you pull out the missing items.

```
import pandas as pd

def compare_columns(csv1_path, column1_name, column2_name):
    # Read CSV files
    df1 = pd.read_csv(csv1_path,dtype='string')
    df2 = pd.read_csv(csv1_path,dtype='string')
    # Extract unique items from each column
    items1 = set(df1[column1_name])
    items2 = set(df2[column2_name])
    print(len(items1))
    print(len(items2))
    # Find items in the first column not in the second column
    difference = items1 - items2

    return list(difference)

diff = (compare_columns('~/test.csv',"A","B"))
print(len(diff)) # how many items differ?
df = pd.DataFrame(diff)
df.to_csv('~/Desktop/differences.csv') # this csv will contain the differences
```

Now, let's create our `scan_data.csv`. The original `scan_data.csv` is obtained from Tableau with these columns: `bblid	protocol	guid	doscan	scagemonths	scanid	sex `. We can download timepoint data from Oracle itself, going to `procedures`, then `img`, then exporting the data to Excel. Make sure to save this excel spreadsheet later as a .csv, with only the columns `VISITNUM` and `SCANID`. Then use python: 

```
import pandas as pd
df1=pd.read_csv('oracle.csv')
df1=df1.rename(columns={"VISITNUM":"timepoint","SCANID":"scanid"})
df1=df1.dropna()astype(int)
df2 = pd.read_csv('scan_data.csv',encoding="utf-16",sep="\t") #from tableau
df3 = pd.merge(df1,df2,how='right')
df3.to_csv('scan_data.csv') # overwrite old scan data
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
setwd('~/Desktop')
nda_scans<-read.csv('nda_scans.csv')
scan_data<-read.csv('scan_data.csv',header =T, skipNul = T,fileEncoding="latin2") 
dcm<- read.csv('EF_dcm_version.csv')
dcm<- dcm %>% unite('dcm_ver', b:c, remove=FALSE)
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
image03$image_thumbnail_file<-"" 
image03$image_description<- "MRI"
image03$experiment_id<- ""
image03$scan_type<-data$suffix
image03$scan_type[ image03$scan_type == 'T1w' ]<-'MR structural (T1)'
image03$scan_type[ image03$scan_type == 'T2w' ]<-'MR structural (T2)'
image03$scan_object<- "Live"
image03$image_file_format<- "NIFTI"
image03$data_file2<-""
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
image03$visit<-data$timepoint
image03$visit[ image03$visit == '1' ]<-'baseline'
image03$visit[ image03$visit == '2' ]<-'followup'
image03$visit[ image03$visit == '3' ]<-'followup'
image03$slice_timing<-''
image03$bvek_bval_files<-''
image03$bvecfile<-''
image03$bvalfile<-''
image03$deviceserialnumber<-''
image03$procdate<-''
image03$visnum<-data$timepoint
​
write.csv(image03, file='image03_NDA.csv')
# there are more fields that need to be calculated, but they are empty (conditional on other modalities)

```
You will need to add the image03.csv template in after this file is generated.

Once you write out the .csv, copy/paste the data into the original template from NIH, and the rest of the fields will generate blank. It is recommended to copy this data into the template downloaded directly from NDA, as the validator is extremely particular, and the headings need to match identically.
