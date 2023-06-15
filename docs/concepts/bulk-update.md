# Bulk update overview
Bulk update is a feature that allows to perform updates of DICOM attributes/metadata without needing to delete/add. Currently only patient attributes can be updated using the bulk update API.  Supported attributes include those included in the [Patient Identification Module](https://dicom.nema.org/dicom/2013/output/chtml/part03/sect_C.2.html#table_C.2-2) and the [Patient Demographic Module](https://dicom.nema.org/dicom/2013/output/chtml/part03/sect_C.2.html#table_C.2-3) that are not sequences. Only the first and last version of the DICOM instances are stored in the server.

## Feature Enablement
The bulk update feature can be enabled by setting the configuration key `DicomServer:Features:EnableUpdate` to `true` through your local [appsettings.json](../../src/Microsoft.Health.Dicom.Web/appsettings.json) file or host-specific options.

## API Design
Following URIs assume an implicit DICOM service base URI. For example, the base URI of a DICOM server running locally would be `https://localhost:63838/`.
Example requests can be sent in the [Postman collection](../resources/Conformance-as-Postman.postman_collection.json).

### Bulk update study
Bulk update endpoint starts a long running operation that update all the instances in the study with the specified attributes.

```http
POST ...v1/studies/$bulkUpdate
POST ...v1/partitions/{PartitionName}/studies/$bulkUpdate
```

#### Request Header

| Name         | Required  | Type   | Description                     |
| ------------ | --------- | ------ | ------------------------------- |
| Content-Type | False     | string | `application/json` is supported |

#### Request Body

```json
{
    "studyInstanceUids": ["12.3.4.5"], 
    "changeDataset": {
        "00100010": {
            "vr": "LO",
            "Value": ["New patient name"]
        }
    }
}
```

#### Responses

```json
{
    "id": "1323c079a1b64efcb8943ef7707b5438",
    "href": "../v1/operations/1323c079a1b64efcb8943ef7707b5438"
}
```

| Name              | Type                                        | Description                                                  |
| ----------------- | ------------------------------------------- | ------------------------------------------------------------ |
| 202 (Accepted)    | [Operation Reference](#operation-reference) | Extended query tag(s) have been added, and a long-running operation has been started to re-index existing DICOM instances |
| 400 (Bad Request) |                                             | Request body has invalid data                                |

#### Limitations

> Only Patient identificaton and demographic attributes are supported for bulk update.

**Patient Identification Module**
| Attribute Name   | Tag           | Description           |
| ---------------- | --------------| --------------------- |
| Patient's Name   | (0010,0010)   | Patient's full name   |
| Patient ID       | (0010,0020)   | Primary hospital identification number or code for the patient. |
| Other Patient IDs| (0010,1000) | Other identification numbers or codes used to identify the patient. 
| Type of Patient ID| (0010,0022) |  The type of identifier in this item. Enumerated Values: TEXT RFID BARCODE Note The identifier is coded as a string regardless of the type, not as a binary value. 
| Other Patient Names| (0010,1001) | Other names used to identify the patient. 
| Patient's Birth Name| (0010,1005) | Patient's birth name. 
| Patient's Mother's Birth Name| (0010,1060) | Birth name of patient's mother. 
| Medical Record Locator | (0010,1090)| An identifier used to find the patient's existing medical record (e.g., film jacket). 

**Patient Demographic Module**
| Attribute Name   | Tag           | Description           |
| ---------------- | --------------| --------------------- |
| Patient's Age | (0010,1010) | Age of the Patient.  |
| Occupation | (0010,2180) | Occupation of the Patient.  |
| Confidentiality Constraint on Patient Data Description | (0040,3001) | Special indication to the modality operator about confidentiality of patient information (e.g., that he should not use the patients name where other patients are present).  |
| Patient's Birth Date | (0010,0030) | Date of birth of the named patient  |
| Patient's Birth Time | (0010,0032) | Time of birth of the named patient  |
| Patient's Sex | (0010,0040) | Sex of the named patient.  |
| Quality Control Subject |(0010,0200) | Indicates whether or not the subject is a quality control phantom.  |
| Patient's Size | (0010,1020) | Patient's height or length in meters  |
| Patient's Weight | (0010,1030) | Weight of the patient in kilograms  |
| Patient's Address | (0010,1040) | Legal address of the named patient  |
| Military Rank | (0010,1080) | Military rank of patient  |
| Branch of Service | (0010,1081) | Branch of the military. The country allegiance may also be included (e.g., U.S. Army).  |
| Country of Residence | (0010,2150) | Country in which patient currently resides  |
| Region of Residence | (0010,2152) | Region within patient's country of residence  |
| Patient's Telephone Numbers | (0010,2154) | Telephone numbers at which the patient can be reached  |
| Ethnic Group | (0010,2160) | Ethnic group or race of patient  |
| Patient's Religious Preference | (0010,21F0) | The religious preference of the patient  |
| Patient Comments | (0010,4000) | User-defined comments about the patient | 
| Responsible Person | (0010,2297) | Name of person with medical decision making authority for the patient.  |
| Responsible Person Role | (0010,2298) | Relationship of Responsible Person to the patient.  |
| Responsible Organization | (0010,2299) | Name of organization with medical decision making authority for the patient.  |
| Patient Species Description | (0010,2201) | The species of the patient.  |
| Patient Breed Description | (0010,2292) | The breed of the patient.See Section C.7.1.1.1.1.  |
| Breed Registration Number | (0010,2295) | Identification number of a veterinary patient within the registry.  |

> Maximum of 50 studies can be updated at once.

> Only one update operation can be performed at a time.

### Get Operation
Get a long-running operation.

```http
GET .../operations/{operationId}
```

#### URI Parameters

| Name        | In   | Required | Type   | Description      |
| ----------- | ---- | -------- | ------ | ---------------- |
| operationId | path | True     | string | The operation id |

#### Responses

```json
{
    "operationId": "1323c079a1b64efcb8943ef7707b5438",
    "type": "update",
    "createdTime": "2023-05-08T05:01:30.1441374Z",
    "lastUpdatedTime": "2023-05-08T05:01:42.9067335Z",
    "status": "completed",
    "percentComplete": 100,
    "results": {
        "studyUpdated": 1,
        "instanceUpdated": 16,
        // Errors will go here
    }
}
```

| Name            | Type                    | Description                                  |
| --------------- | ----------------------- | -------------------------------------------- |
| 200 (OK)        | [Operation](#operation) | The completed operation for the specified ID |
| 202 (Accepted)  | [Operation](#operation) | The running operation for the specified ID   |
| 404 (Not Found) |                         | The operation is not found                   |

## Retrieve (WADO-RS)

The Retrieve Transaction feature provides the ability to retrieve stored studies, series, and instances, including both the original and latest versions.

| Method | Path                                                                    | Description |
| :----- | :---------------------------------------------------------------------- | :---------- |
| GET    | ../studies/{study}                                                      | Retrieves all instances within a study. |
| GET    | ../studies/{study}/metadata                                             | Retrieves the metadata for all instances within a study. |
| GET    | ../studies/{study}/series/{series}                                      | Retrieves all instances within a series. |
| GET    | ../studies/{study}/series/{series}/metadata                             | Retrieves the metadata for all instances within a series. |
| GET    | ../studies/{study}/series/{series}/instances/{instance}                 | Retrieves a single instance. |
| GET    | ../studies/{study}/series/{series}/instances/{instance}/metadata        | Retrieves the metadata for a single instance. |

In order to retrieve original version, `msdicom-request-original` header should be set to `true`.

```
- `msdicom-request-original=true
```

## Delete

This transaction will delete both original and latest version of instances.

### Other APIs

There is no change in other APIs.