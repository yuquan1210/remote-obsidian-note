- remove 'not null' validation in plugin
	- originally, 'not null' check only for 非空指针
	- actually not null rely on zendesk (two enquiry type set field required, has *)


enquiry api:
- QualifiedExternalReferralLead: 
	- ticket status = SOLVED 
	- Enquiry type = `Broker Lead Captured - Lending Solutions"` **or** `"Broker Lead Captured - Franchisee`
	- Source = EXTERNAL_REFERRAL
- QualifiedExternalReferralLead: 
	- ticket status = SOLVED 
	- Enquiry type = `Broker Lead Captured - Lending Solutions"` **or** `"Broker Lead Captured - Franchisee
	- Source != EXTERNAL_REFERRAL
- pending/unqualified/junk leads
	- ticket status != SOLVED or  Enquiry type not either above

what validation logic to use for those fields


why validate at plugin, and then validate again in enquiry api? redundant?


先讨论在哪加校验
再讨论加什么校验
- 对于junk pending unqualified lead是否允许 不符合规则的值？

how do lead engine handle zendesk leads?
- what happens to qualified leads, what happens to pending/unqualified/junk leads
- lead-store if has zendesk ticket id, then DB Action is UPDATE
- update to db
- pending/unqualified/junk leads (isNotSolvedTicketStatus || isSolvedAsUnqualified = true) -> PublishAction.None
	- those tickets still on zendesk, can be updated again become qualified?
- qualified leads can be allocated

2026-03-30
phil:
- we other types of leads coming in, may need more validations on enquiry-api, pause this implementaton, wait for that

jim:
- current validation in zendesk plugin do not include pending/unqualified/junk lead is becasue we didn't have zendesk leads into lead-engine
- now we have zendesk leads into lead-engine and its not matching our lead-engine behavior, we simply extend the validation to include those zendesk leads
- adding more validations to enquiry-api will be messy
- consider consistency of validation logics between zendesk and enquiry-api (no need to be consistent, but need to be clear what validations happen in enquiry-api, what validation in zendesk)
- reuse exisiting ones, rather than creating new ones