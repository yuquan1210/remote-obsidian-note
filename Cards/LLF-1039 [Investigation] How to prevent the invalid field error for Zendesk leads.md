a bunch of errors on this breeze task. A quick look at the logs shows the error Could not convert value 'string_value: 	 "14:06"' to integer. Field: enquiry_postcode,

Why not add validation to all fields for zendesk leads
- zendesk leads not in 'final state', some fields are allowed to be blank
- more specific? 

Option 1
in enquiry-api:
- add validation to all zendesk lead fields
- src/validation/unqualifiedOrPendingLeadValidator.ts
- if provided, value must be valid 
- cron: if invalid zendesk field need manual fix & replay

plug in 
only when 
finx concierge enquiry type is two
- plug in will run, will validate all fields

Option 2
in zendesk plugin
- we let all enquiry type to have input field validation
- only validated can create ticket
- but impact broker, sometimes they want 'pending' ticket, leave fields invalid, then complete later
- pro: eliminate invalid zendesk field value case at enquiry-api, so no manual fix & replay needed


enquiry-api & most lead-engine systems
- check dead letter queue to trigger alert
- only mc-ala use log error to trigger alert

enquiry-api zendesk/rea lead validation check failed (upstream responsibility)
- return 400
- upstream (zendesk adapter) receive error, save to dead letter queue

enquiry-api save file failed / push to topic failed (enquiry-api responsibility)
- handle error
- save to dead letter queue

enquiry-api 
rawLeadPayload -> build zendeak lead enquiry detail, postcode is string
![[Pasted image 20260325172501.png]]

why lead store make postcode to integer? what about other fields?
is postcode a special case?


Rollback card, can't pick up, wait til problem fixed, then rollbakc


more for option2:
- in our codes (shouldValidateFields), the lead must match these three things + Concierge Enquiry Type to enable validation
- webhook setting, when the three things match, will send to lead engine
![[Pasted image 20260326111544.png]]
- we only need to remove Concierge Enquiry Type condition in shouldValidateFields to enable validation for all zendesk lead coming to lead engine

problem with zendesk plugin
- simply let enquiry type go thru validation, 
- current validations are for the first two enquiry-types (some fields are required, some are not)
- we can't just use those validation logics to enquiry types
- for non-required fields to "must be valid if provided"

for fields that are not required (don't have '\*')
- To my understanding, it can be blank
- but if a value is filled in, can it be a invalid value?


zendesk fields that is free-text