# Comparing National and EU Base Profiles

**Track lead: Mikael Rinnetmaki**

## Overview

This track focuses on comparing national FHIR base profiles across Nordic and European countries with the emerging EU base profiles. The goal is to understand the similarities, differences, and potential alignment opportunities.

Denmark, Finland, Norway, and Sweden have all published national FHIR base profiles through their HL7 affiliates. There is an ongoing initiative to harmonize the base profiles and to create common Nordic FHIR Base profile specification. Given the similarity of the healthcare systems of all the Nordic countries, we can learn from each other in developing and using national FHIR base profiles and reach a higher level of base-profile harmonization between Nordic countries.  

Also, due to the EHDS regulation, standardization efforts in Europe are now of central importance and HL7 Europe is a key organization in those efforts. With collaboration and harmonization between Nordic countries we can also in a better way contribute and influence European work, including European HL7 FHIR base profiles.

## Goals

- Get clarity on what kind of differences there are between each of the Nordic base profile specifications and the EU/EHDS ones.
- Establish a vision and guidelines for harmonising Nordic base profiles with the EU/EHDS ones, where necessary.
- get the [nordic-base](https://github.com/fhir-fi/nordic-base) FHIR implementation guide repo in sync with the [HL7 EU](https://github.com/hl7-eu/coalesced) one.

## Prerequisites

- None

## Tasks

- Let's discuss potential alternatives for checking profile compatibility.

  - The IG publisher can do some comparisons. It is not trivial to see which changes matter.

  - Repos [https://github.com/hl7-eu/coalesced](https://github.com/hl7-eu/coalesced),
    [https://github.com/fhir-fi/nordic-base](https://github.com/fhir-fi/nordic-base),
    and[multi-profile-validation](https://github.com/Sensotrend/multi-profile-validation)
    show approaches for comparing profiles.

  - It could perhaps be possible to do the comparison fully programmatically. Read all profile
    definitions, generate sample resources that fulfil each requirement, validate against
    other profiles... This may be too ambitious for the hackathon?

- Let's explore how Gazelle and any other EHDS online testing tools could help.

  - Gazelle and the European Digital Test Environment (DTE) are available for hackathon 
    participants.
  - See [the presentation](Xt-EHR_Testing_Hackathon_Vitalis_20260508.pdf) for instructions on how to get started!

## Expected Outcomes

- Understanding of known and potential conflicts between the base profile specifications
- Some guidance on further development of Nordic base profiles. Should the EU/EHDS profiles
  be acknowleged at all? If so, when and how?

## Resources

- [FHIR Base Profiles in the Nordics](https://invitepeople.com/public/events/cceaf0ab1a/seminars/99491)
  session in Vitalis conference programme
- The [CoalescedImplementationGuide](https://github.com/hl7-eu/coalesced) repo for comparing
  EU base profiles
- The [nordic-base](https://github.com/fhir-fi/nordic-base) FHIR implementation guide repo
- An example [multi-profile-validation](https://github.com/Sensotrend/
  multi-profile-validation) repo demonstrating how profiles can be compared
- [EHDS Gazelle Test Bed](https://ehds.gazelle-platform.net/)

## Results

Participants discussed the problem and agreed that the problem of comparing profiles and evaluating
whether they are compatible is foreseen to be a real issue and something the current FHIR tooling
does not address properly.

The group drew inspiration from Gino Canessa's
[Cross-Version Extensions project](https://github.com/GinoCanessa/fhir-cross-version-source) (see 
also the DevDays
[presentation](https://www.devdays.com/wp-content/uploads/2025/06/250605_GinoCanessa_CrossVersionExtensions.pdf)
and [video](https://www.youtube.com/watch?v=AFTnGTd-yWs)) for FHIR versions.

### FHIR Schema Approach

Aleksandr worked on an implementation based on
[FHIR Schema](https://github.com/fhir-schema/fhir-schema) to compare profiles.

Identifying "hard differences" like conflicting cardinalities or bindings is important and trivial
with this kind of a tool. However, there 

Aleksandr's implementation also includes Claude to provide AI-based analysis of the compatibility
of profiles. It was surprisingly eye-opening.

### Automated Creation of Example Instances

Each FHIR implementation guide should have examples that demonstrate the use of profiles. However,
it is rare that national base profile specifications have an extensive set of examples covering
each constraint and extension.

If we could get the implementation guides to include extensive example resources, we could validate
those example resources against profiles from other implementation guides and thereby evaluate
compatibility of the implementation guides.

The participants made some attempts to create resource instances automatically from profile
definitions. The work did not proceed to a demonstratable level.