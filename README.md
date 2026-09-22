
# GenesysTestArchitectFlow

## Overview

This repository contains a **Genesys Cloud Architect** flow definition for an **inbound voice** call flow used by a German (DE) collections/customer service operation (branded **Myntro**, formerly **Alektum**).

The flow is exported as an Architect Flow YAML file:

- [`DE_InboundMain 4.2_v1-0.yaml`](./DE_InboundMain%204.2_v1-0.yaml)

## Flow Details

| Property | Value |
|---|---|
| Flow type | `inboundCall` |
| Flow name | `DE_InboundMain 4.2` |
| Division | Germany |
| Default language | `de-de` (German) |
| Supported languages | `de-de`, `en-us` |
| Text-to-Speech | Genesys Enhanced TTS (`de-DE-ConradNeural`, `en-US-AmberNeural`) |

### Description (from flow metadata)

> DK Main Inbound, Dynamic Schedules & dynamic SelfService from DataTable.
> Includes iterative updates such as validating mobile numbers before sending SMS, and changes to how self-service works.

## What the Flow Does

At a high level, this Architect flow handles inbound customer calls and performs the following:

1. **Call setup & debugging**
   - Sets participant data for debugging/telemetry (call duration, ANI, etc.).
   - Initializes flow variables (country code, queue response, identification flags).

2. **Routing lookup**
   - Performs a **Data Table lookup** (`DE_RoutingVariables 3.0`) keyed on the dialed number (`Call.CalledAddressOriginal`) to dynamically resolve routing/queue configuration:
     - Queue ID/Name, Skill, Site, Country
     - Schedule Group and associated open/closed/holiday prompts
     - Avalanche (overflow) queue and prompts
     - Callback offer flags for closed/meeting states
   - Handles `found`, `notFound`, and `failure` outcomes, playing an error prompt (`Prompt.DE_GeneralError`) if the lookup fails or the DNIS is unrecognized.

3. **Schedule evaluation**
   - Evaluates the resolved **Schedule Group** (with nested sub-schedule evaluation for `DE_InboundMain`) to determine whether the business is **open**, **closed**, **holiday**, or in an **emergency** state.
   - Plays the appropriate greeting/closure prompt for each state (open, closed, holiday, emergency) and records the corresponding **Flow Milestone**.

4. **Emergency / Avalanche (overflow) handling**
   - When open, checks for an active **Emergency Group** (`<Country>_EmergencyAlektum`).
   - If activated, plays an avalanche/overflow prompt and routes to a dedicated task (`T. Play Avalanche & Meeting`) before continuing to self-service.

5. **Callback offer**
   - If the queue is closed/in a meeting and callback is enabled (`Flow.CallBackOfferClosed`), collects caller info (ANI, country code, dialing prefix, act number, etc.) and invokes a reusable **Callback Module 2.0** common module to offer a scheduled callback.

6. **Self-service hand-off**
   - Jumps to a reusable task, **`2. Self Service Offer`**, to continue the self-service/IVR experience (e.g., account/debtor lookup, self-service menu) when self-service is enabled (`Flow.OfferSelfService`).

## Key Tasks

| Task | Purpose |
|---|---|
| `1. Main Start` | Entry point: initializes variables, performs routing/data table lookup, evaluates schedules and emergency/avalanche state, and branches to callback or self-service. |
| `2. Self Service Offer` | Reusable task offering self-service options to the caller. |
| `T. Play Avalanche & Meeting` | Reusable task for playing avalanche/overflow and meeting-related prompts. |

## Notable Flow Variables

The flow declares a large set of `Flow.*` variables used throughout call handling, including (non-exhaustive):

- **Identification / Account:** `ActNoInput`, `ActNoSaved`, `actNoIdentified`, `DebtorNo`, `isIdentified`, `alreadyIdentified`, `foundInNova_Act`, `foundInNova_Debtor`, `multipleDebtorFoundInNova`
- **Caller / Contact:** `call_phoneNumber`, `call_countryCode`, `call_privacyId`, `ExternalContactId`, `ExternalContactResult`, `MobileNumber`
- **Routing / Queue:** `QueueID`, `QueueName`, `SkillVoice`, `Site`, `ScheduleGroup`, `AvalancheQueue`, `ClosedMeetingQueue`
- **Prompts / Schedule state:** `SchedulePromptOpen`, `SchedulePromptClosed`, `SchedulePromptHoliday`, `ScheduleOpen`
- **Call recording / consent:** `CallRecording`, `CallRecordingConsentQuestion`, `CallRecordingStartWithSecurePause`, `RecordingOptOut`
- **Self-service:** `OfferSelfService`, `SelfServiceAnswer`
- **Nova (CRM) integration:** `novaDebtorNo`, `novaEmail`, `novaFirstName`, `novaLastName`, `novaPhonePriv`, `novaPhoneWork`, `NovaQueueResponce`

## External Integrations

- **Data Table:** `DE_RoutingVariables 3.0` — drives dynamic routing, queues, skills, and schedule/prompt selection based on the dialed number.
- **Common Module:** `Callback Module 2.0` — reusable callback-offer logic shared across flows.
- **Nova CRM** — customer/account/debtor lookups (act number, debtor number, contact details).

## Settings Summary

- **Error handling:** Errors disconnect the call by default; a brief pre-handling audio blank is played.
- **Menu settings:** 10s selection timeout, 3 repeats, extension dialing enabled.
- **Speech recognition:** ASR disabled at the flow level; recognition timeouts configured for completeness/incomplete match.
- **Silence detection:** 1s silence duration, 40s timeout, up to 5 repeat prompts on silence.

## Repository Structure

```
.
└── DE_InboundMain 4.2_v1-0.yaml   # Genesys Cloud Architect inbound call flow definition
```

## Usage

This YAML file is a **Genesys Cloud Architect** flow export. It is intended to be imported into Genesys Cloud Architect (Admin > Architect > Flows > Import) rather than executed directly. Ensure that referenced dependencies exist in the target org before importing:

- Data table: `DE_RoutingVariables 3.0`
- Common module: `Callback Module 2.0`
- Schedules/Schedule Groups: `DE_InboundMain`, `DE_InboundMain_SubSchedule`
- Emergency Groups: `DE_Alektum_Emergency_Nova_Closed`, `<Country>_EmergencyAlektum`
- Prompts: `Prompt.DE_GeneralError`, `Prompt.Alektum_Emergency_Nova_Closed`, and various dynamically-resolved user prompts

## Disclaimer

This README was generated from an automated analysis of the flow's YAML structure and may not capture every branch or edge case in the flow. Review the full flow in Architect for complete details before making changes.
