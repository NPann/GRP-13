# GRP-13 Anonymized/De-identified Export
Template-based anonymization and export of DICOM files within a container
(project/subject/session). DICOM files within the source project will be
anonymized (according to a required template) and exported to a specified project.

_NOTE:_ A requirement for DICOM metadata generation is that the destination
project has a gear rule configured for GRP-3, as the gear itself does not
propagate/modify DICOM metadata.

# Inputs
### deid_template (required)
This is a YAML file that describes the protocol for de-identifying
input_file. This file covers all of the same functionality of Flywheel
CLI de-identifiction.
An example deid_template.yaml looks like this:

``` yaml
# Configuration for DICOM de-identification
dicom:
  # What date offset to use, in number of days
  date-increment: -17

  # Set patient age from date of birth
  patient-age-from-birthdate: true
  # Set patient age units as Years
  patient-age-units: Y
  # Remove private tags
  remove-private-tags: true

  fields:
    # Remove a dicom field (e.g.remove PatientID)
    - name: PatientID
      remove: true

    # Replace a dicom field value (e.g. replace “StationName” with "XXXX")
    - name: StationName
      replace-with: XXXX

    # Increment a date field by -17 days
    - name: StudyDate
      increment-date: true

    # One-Way hash a dicom field to a unique string
    - name: AccessionNumber
      hash: true

    # One-Way hash the ConcatenationUID,
    # keeping the prefix (4 nodes) and suffix (2 nodes)
    - name: ConcatenationUID
      hashuid: true
```

__Extended Template Functionality:__ Additional options have been introduced for this gear
(still unreleased for CLI):

1. Per GRP-13-03, the header tag can be referenced in a by tag location
as below:

    ```yaml
    dicom:
      fields:
        - name: PatientBirthDate
          remove: true
        - name: (0010, 0010)
          remove: true
        - name: '00100020'
          remove: true
    ```

    [migration_toolkit changes](https://gitlab.com/flywheel-io/public/migration-toolkit/merge_requests/39)


2. Per GRP-13-02, nested elements/sequences can be specified as below:

    ``` yaml
    name: nametag-profile
    description: Replace nested data element and test a few indexing variants
    dicom:
      fields:
        - name: AnatomicRegionSequence.0.CodeValue
          replace-with: 'new SH value'
        - name: '00082218.0.00080102'
          replace-with: 'new SH value'
        - name: AnatomicRegionSequence.0.00080104
          remove: true
        - name: AnatomicRegionSequence.*.CodeValue
          replace-with: 'new SH value'

    ```
    [migration_toolkit changes](https://gitlab.com/flywheel-io/public/migration-toolkit/merge_requests/40/diffs)  
    In the last field example, the wildcard character will be expanded to 
    all matching indices.

3. Per GRP-13-13, filename can be modified given a user defined pattern
captured in the filenames namespace of the yml template. Example:

    ```yaml
    dicom:
      date-increment: -17
      filenames:
        - output: '{SOPInstanceUID}_{regdate}.dcm'
          input-regex: '^(?P<notused>\w+)-(?P<regdate>\d{4}-\d{2}-\d{2}).dcm$'
          groups:
            - name: regdate
              increment-date: true
        - output: '{filenameuid}_{regdatetime}.dcm'
          input-regex: '^(?P<filenameuid>[\w.]+)-(?P<regdatetime>[\d\s:-]+).dcm$'
          groups:
            - name: filenameuid
              hashuid: true
            - name: regdatetime
              increment-datetime: true
      fields:
        ...
    ```
   In this example, a file matching the first `input-regex` (e.g. acquisition-2020-02-20.dcm) 
   will be saved as `1.2.840.113619.2.408.5282380.5220731_2020-02-03.dcm`, matching the
   `output` specification:
    * `SOPInstanceUID` is replaced by the corresponding Dicom keyword
    * `regdate` is replaced by the `regdate` group extracted from regex match defined
    by `input-regex` and processed by the action listed under `groups` 
    (e.g. incremented by `date-increment`).
   If multiple `input-regex` match the filename, the first match in the `filenames` list 
   gets precedence.
   
4. Per GRP-13-08, UID can be hashed. UID will be generated with a defined root IOD as prefix if
   `uid_numeric_name` is specified. If no root IOD (`uid_numeric_name`) is provided, the 
   original uid prefix is preserved. Number of prefix groups to preserved in the original UID is defined by 
   `uid_prefix_fields` (default=4). For using Flywheel ANSI registered IOD, you can use 
   `uid_numeric_name=2.16.840.1.114570.2.2`.
   
   Example:
   ```yaml
   dicom:
     uid_prefix_fields: 7
     uid_numeric_name: 2.16.840.1.114570.2.2 
     fields:
        - name: ConcatenationUID
          hashuid: true       
   ``` 
    Note: when `uid_numeric_name` is provided, it must match the number of blocks defined
     by `uid_prefix_fields`.

6. The `export` namespace can be used to whitelist metadata fields for
propagation where `info` is specifically for container.info fields and
`metadata` is a list for metadata that appear flat on the container
for example, subject.sex. This is demonstrated below:

    ``` yaml
    dicom:
      fields:
        - name: PatientBirthDate
          remove: true
    export:
      session:
        whitelist:
          info:
            - cats
          metadata:
            - operator
            - weight
      subject:
        whitelist:
          info:
            - cats
          metadata:
            - sex
            - strain
      acquisition: 
        whitelist:
          info: all
          metadata: all 
      file:
        whitelist:
          info: 
            - cats
          metadata:
            - classification
            - modality
    ```
    With the above template subject.info.cats, subject.sex, subject.strain,
    session.info.cats, session.operator, session.weight, all 
    acquisition.info fields, all editable acquisition metadata, file.info.cats, file.classification and file.modality would 
    be propagated to the exported containers. File info.header values cannot be 
    exported.
   
    
    _NOTE:_ metadata fields that can be propagated are defined by the dictionary
    below which reflects container metadata outside of info that can be
    updated via the SDK:

    ``` python
    META_WHITELIST_DICT = {
        'file': ['classification', 'info', 'modality', 'type'],
        'acquisition': ['timestamp', 'timezone', 'uid'],
        'subject': ['firstname', 'lastname', 'sex', 'cohort', 'ethnicity', 'race', 'species', 'strain'],
        'session': ['age', 'operator', 'timestamp', 'timezone', 'uid', 'weight']
    }
    ```
    
7. De-identification of JPG is supported with same action on fields as Dicom. 
Field name must match keyword names defined by piexif (full list available 
[here](https://github.com/hMatoba/Piexif/blob/master/piexif/_exif.py)). A few additional
attributes can be specified: `remove-gps: true` to remove all GPS metadata and
`remove-exif` to remove the whole EXIF ImageFileDirectory block. Example:
    ```yaml
   jpg: 
      date-increment: -17
      remove-gps: true
      fields:
        - name: DateTime
          increment-datetime: true
        - name: Artist
          remove: true
        - name: DateTimeOriginal
          increment-datetime: true
        - name: PreviewDateTime
          remove: true
        - name: DateTimeDigitized
          increment-datetime: true
        - name: CameraOwnerName
          replace-with: 'REDACTED'
        - name: ImageUniqueID
          hash: true
   ```
   
8. De-identification of TIFF is supported with same action on fields as Dicom.
Field name must match keyword names as defined Pillow (full list of keyword can be seen 
[here](https://github.com/python-pillow/Pillow/blob/4.1.x/PIL/TiffTags.py#L67)). A few additional
attributes can be specified: `remove-private-tags: true` to remove all private tags (i.e.
tag index >= 32768) . Example:
    ```yaml
    tiff:
      date-increment: -17
      remove-private-tags: True
      fields:
        - name: DateTime
          increment-datetime: true
        - name: Software
          remove: true
        - name: Model
          replace-with: 'REDACTED'
    ``` 

9. De-identification of XML is supported with same action on fields as Dicom.
Field name must use [XPath](https://en.wikipedia.org/wiki/XPath) to specify the DOM element 
in the tree. If XPath return multiple elements, each element will be processed with the
specified action. Example:
    ```yaml
    xml:
      date-increment: -17
      fields:
        - name: /Patient/Patient_Date_Of_Birth
          replace-with: '1900-01-01'
        - name: /Patient/Patient_Name
          remove: true
        - name: /Patient/SUBJECT_ID
          hash: true
        - name: /Patient/Visit/Scan/ScanTime
          increment-datetime: true
    ```
    In the above example, if the last field name XPath (`/Patient/Visit/Scan/ScanTime`) 
    matches multiple elements (i.e /Patient/Visit[1]/Scan[1]/ScanTime, 
    /Patient/Visit[2]/Scan[1]/ScanTime  
    and /Patient/Visit[1]/Scan[2]/ScanTime), each
    element will be processed with the specified action. 
    
10. De-identification of PNG is supported with remove action only.
Field name can match any metadata chunks. To remove all private chunks the following
attributes can be specified: `remove-private-chunks: true` (more on public and private
chunks [here](https://en.wikipedia.org/wiki/Portable_Network_Graphics)). Example:
    ```yaml
    png:
      remove-private-chunks: True
      fields:
        - name: tEXt
          remove: true
        - name: eXIf
          remove: true
    ``` 

11. De-identification of ZIP archives (including DICOM zip archives) is now supported (INCLUDING DICOM ARCHIVES).
comment is currently the only specific field to archive. Filenames can use attributes of their 
member files such as DICOM. The attributes of the first successfully de-identified file of each type will
be used. The `hash-subdirectories` option will apply an md5 hash to any subdirectories within the archive.
By default, the export of an zip archive will fail if any member files cannot be de-identified. If you would like
to export partial zip archives, then `validate-zip-members: false` can be provided. 
```yaml
zip:
  filenames:
    - output: '{SeriesDescription}_{SeriesNumber}.dcm.zip'
      input-regex: '^.*\.zip$'
  hash-subdirectories: true
  validate-zip-members: false
  fields:
    - name: comment
      replace-with: 'FLYWHEEL'
```

12. As of version 2, file types to be exported are now selected on the basis of the templates provided (ie zip, dicom, jpg)
Every file type has default extension pattens as demonstrated below:
```
dicom: ['*.dcm', '*.DCM', '*.ima', '*.IMA']
jpg: ['*.jpg', '*.jpeg', '*.JPG', '*.JPEG']
png: ['*.png', '*.PNG']
tiff: ['*.tif', '*.tiff', '*.TIF', '*.TIFF']
xml: ['*.XML', '*.xml']
zip: ['*.zip', '*.ZIP']
``` 

If you would like to use different matching patterns, you can specify them
for a given file type by defining a list for `file-filter`, for example, 
files with extensions of .dicom or .DICOM would be added as follows:

```yaml
dicom:
  file-filter: 
    - '*.DICOM'
    - '*.dicom'
    - '*.dcm'
    - '*.DCM'
    - '*.ima'
    - '*.IMA'
```


### subject_csv (optional)
The subject_csv facilitates subject-specific configuration of 
de-identification templates. This is a csv file that contains the column
`subject.code` with unique values correspond to the subject.code 
values in the project to be exported. If a subject in the project to be 
exported is not listed in `subject.code` in the provided subject_csv 
this subject will not be exported. 

#### Subject-level customization with subject_csv and deid_template

Requirements:
* To update subject fields, the fields must both be represented in the 
subject_csv and deid_template files. 
* If a field is represented in both the deid_template and the 
subject_csv, the value in the deid_template will be replaced with the 
value listed in the corresponding column of the subject_csv for each
subject that has a code listed in subject.code.
* Fields represented in the deid_template but not the subject_csv will 
be the same for all subjects. 
* subject.code can be modified by including an export.subject.code
column in subject_csv that contains unique values to be applied to 
subjects in the destination project.
* **NOTE: If `dicom.fields.PatientID.replace-with` is provided without 
`export.subject.code`, `dicom.fields.PatientID.replace-with` will be 
used to set subject.code for destination subjects. If 
`export.subject.code` is provided, then it `export.subject.code` will be 
used to set subject.code for destination subjects. 
`dicom.fields.PatientID.replace-with` is required to set PatientID in
DICOM files on a subject-to-subject basis** 

Let's walk through an example pairing of subject_csv and deid_template
to illustrate. 

The following table represents subject_csv (../tests/data/example-csv-mapping.csv):

|subject.code|dicom.date-increment|dicom.fields.PatientID.replace-with|dicom.fields.PatientBirthDate.remove|
|------------|--------------------|-----------------------------------|------------------------------------|
|001         |-15                 |Patient_IDA                        |false                               |
|002         |-20                 |Patient_IDB                        |true                                |
|003         |-30                 |Patient_IDC                        |true                                |

The deid_template:
``` yaml
dicom:
  # date-increment can be any integer value since dicom.date-increment is defined in example-csv-mapping.csv
  date-increment: -10
  # # since example-csv-mapping.csv doesn't define dicom.remove-private-tags, all subjects will have private tags removed
  remove-private-tags: true
  fields:
    - name: PatientBirthDate
      # remove can be any boolean since dicom.fields.PatientBirthDate.remove is defined in example-csv-mapping.csv
      remove: true
    - name: PatientID
      # replace-with can be any string value since dicom.fields.PatientID.replace-with is defined in example-csv-mapping.csv
      replace-with: FLYWHEEL
export:
  session:
    whitelist:
      info:
        - cats
      metadata:
        - operator
        - weight
  subject:
    # code can be any string value since dicom.fields.PatientID.replace-with is defined in example-csv-mapping.csv
    code: FLYWHEEL
    whitelist:
      info:
        - cats
      metadata:
        - sex
        - strain
```
The resulting template for subject 003 given the above would be: 
The deid_template:
``` yaml
dicom:
  # date-increment can be any integer value since dicom.date-increment is defined in example-csv-mapping.csv
  date-increment: -30
  remove_private_tags: true
  fields:
    - name: PatientBirthDate
      remove: true
    - name: PatientID
      replace-with: Patient_IDC 
export:
  session:
    whitelist:
      info:
        - cats
      metadata:
        - operator
        - weight
  subject:
    # code can be any string value since export.subject.code is defined in example-csv-mapping.csv
    code: Patient_IDC
    whitelist:
      info:
        - cats
      metadata:
        - sex
        - strain
```

### Manifest JSON for inputs
``` json
"inputs": {
    "api-key": {
      "base": "api-key"
    },
    "deid_template": {
      "base": "file",
      "description": "A Flywheel de-identification template specifying the de-identification actions to perform.",
      "optional": false,
      "type": {
        "enum": [
          "source code"
        ]
      }
    },
    "subject_csv": {
      "base": "file",
      "description": "A CSV file that contains mapping values to apply for subjects during de-identification.",
      "optional": true,
      "type": {
        "enum": [
          "source code"
        ]
      }
    }
}
```

## Configuration Options
### project_path (required)
The resolver path (<group>/<project>) to the destination project.
This project must exist AND the user running the gear must have
read/write access on this project.

### overwrite_files (default = true)
If true, any existing files in the destination project that have been
exported previously will be overwritten so long as their parent container
has `info.export.origin_id` defined.

### Manifest JSON for configuration options
```json
"config": {
    "project_path": {
        "optional": false,
        "description": "The resolver path of the destination project, for example, flywheel/test",
        "type": "string"
    },
    "overwrite_files": {
        "default": true,
        "description": "If true, existing files in destination containers will be overwritten if a file to  be exported shares their filename",
        "type": "boolean"
    }
}
```

# Workflow

1. User creates a destination project and enables gear rules. Importantly,
GRP-3 should be enabled to parse the de-identified DICOM headers. Further,
the permissions on this project should be restricted until the user
exporting the project has reviewed this project (after gear execution).
1. Within the source project, GRP-7 is used to modify container metadata in
preparation for export.
1. User runs GRP-13 (this analysis gear) at the project, subject, or session
level and provides the following:
    * Files:
        * A de-identification template specifying how to
        de-identify/anonymize the each file type
        * an optional csv that contains a column that maps to a
        Flywheel session or subject metadata field and columns that
        specify values with which to replace DICOM header tags
    * Configuration options:
        * The group_id/project name for the project to which to export
        anonymized files
        * Whether to overwrite files if they already exist in the target
        project

1. The gear will find/create all subject/session/acquisition containers
associated with the destination analysis parent container.
    * <container>.info.export.origin_id is used to find containers in
    the export project
1. The gear will attempt to download all files that match the `file-filter` list for any of the file profiles within the template, de-identify them per the template with matching file-filter, and upload them to the
destination container

1. The  gear will then write an output file reporting the status files
that were exported. The gear will then exit.
