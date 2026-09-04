# Outcome

to have an API that can handle ingress of patient and visit data from different hospitals and maintain 
 - a single central repository of patient data
 - a per hospital location repository. i.e. wards, clinics, beds

to be able to surface data via API that shows
 - current patient data
 - current location data

## Technical requirements

must be able to handle the following message formats
 - hl7
 - fhir

should be a fifo system, i.e. messages are processed sequentially and latest message is taken as the source of truth for its data

must be able to handle faulty messages in terms of 
 - incorrect format
 - does not meet business rule requirements for processing i.e. a patient is admitted to a bed that is already occupied, a deceased patient is booked an appointment in the future etc.

- should be able to reprocess messages
- should be able to freeze a particular patient or location that needs some manual inteference to unblock where there is bad business rule data coming in as described above
- should be able to provide a summary of messages processed related to a patient, visit or location

## business requirements

needs to be able to provide observability for patients, locations and visits. i.e. as a clinician i should be able to access a patients history in reasonable time
should be interoperable with other healthcare systems and devices


## Proposed tech stack / guidance

I'd like this to be written so that it is scalable and can be deployed either on premise or in the cloud

I'd like the business rules processing to be accessible so that some psuedo code like IF THEN ELSE can be used to define the business rules and those rules can be managed as artefacts

I'd like a test harness that can generate sample data based on real data sets to use for pumping into the system
