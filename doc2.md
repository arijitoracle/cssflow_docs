# CSSFlow Workflow Build Guide - SCM Inventory Issue Resolution

## 1. What this workflow must automate

The BRD is for SCM functional issue resolution under Inventory Inquiry & Issues. It has three main ticket subtypes:

1. On-Hand Balance Inquiries
2. Item can not be received in Subinventory
3. Item not populated in cycle count

The workflow should:

1. Take the ticket details from the CSS queue.
2. Classify the ticket into the correct subtype.
3. Confirm or collect the required item, organization, subinventory, locator, and cycle count details.
4. Call the relevant Fusion SCM REST APIs.
5. Check whether each API returned useful data.
6. Prepare a clear resolution report.
7. Ask the user for sign-off.
8. Close or hand off the ticket after sign-off.

## 2. Recommended CSSFlow design

Use a mostly deterministic workflow:

- Use Prompt Execution for classification, field extraction, API result interpretation, and final report writing.
- Use Tool Execution for each fixed REST API call from the BRD.
- Use Branching for routing between ticket subtypes and decisions such as FOUND or NOT_FOUND.
- Use Input Message (HITL) for analyst confirmation, missing information, and final user sign-off.
- Use Agent Execution only if you later want a flexible agent that decides which API to call by itself.

Reason in simple language: the BRD already tells us the exact checks and API endpoints. Tool Execution is safer because it calls exactly what we configure. A single free-running Agent Execution step may skip a check, call APIs in the wrong order, or make the result harder to audit.

### Mandatory descriptor rule

CSSFlow does not give you a separate optional/mandatory switch for each descriptor. So build this workflow with a simple rule:

- If a step defines an Input Descriptor, that input is mandatory for that step.
- If a step defines an Output Descriptor, the step must always populate it.
- Do not add "nice to have" descriptors to a step unless that step truly uses them.
- When the value is not known, pass `UNKNOWN`.
- When the value is not applicable for that ticket path, pass `NA`.
- Do not pass blank values, because blank values are hard to distinguish from binding mistakes.

Reason in simple language: this makes every CSSFlow step predictable. If the step has a variable, it must receive a value. If a prompt creates a variable, it must fill it every time.

## 3. Tools and LLM to use

### LLM

Use `openai.gpt-5.5` as the workflow default if available. If you need a cheaper model, `OpenAI GPT 5.2` should also work for classification and report drafting.

Reason: this workflow needs reliable extraction, classification, and summarization.

### Tools

Use these available CSSFlow tools:

- `call_api_tool` for Fusion SCM REST API calls.
- `api_tool` only if your CSSFlow admin says this is the preferred generic API wrapper.
- `OCI File Write` if you want to save the final investigation report as a file.
- `slack_tool` only if your support team wants Slack notifications.

You will also need a ticketing-system API tool if CSSFlow should close the ticket automatically. The BRD does not provide a ticket-close endpoint, so do not assume ticket closure is automatic unless that API is configured.

## 4. Create Workflow panel

Name:

`SCM Inventory Issue Resolution Agent`

Description:

`Classifies SCM inventory support tickets, validates item/org/subinventory details, calls Fusion SCM APIs, prepares an investigation report, requests sign-off, and supports ticket closure.`

LLM:

`openai.gpt-5.5`

## 5. Naming rules

Use exact branch values. Branching is case-sensitive in CSSFlow.

Ticket route branch values:

- `ON_HAND_BALANCE`
- `RECEIVE_SUBINVENTORY`
- `CYCLE_COUNT`
- `HOW_TO`
- `NAVIGATION`
- `ESCALATE`
- `default`

API result branch values:

- `FOUND`
- `NOT_FOUND`
- `ERROR`
- `default`

Sign-off branch values:

- `SIGNED_OFF`
- `MORE_HELP`
- `default`

Reason: CSSFlow Branching runs only the path whose label exactly matches the value passed into `next_step_name`.

## 6. Start Step

Step type: Start

Name:

`start_ticket`

Input descriptors:

| Name | Type | Description |
|---|---|---|
| `ticket_id` | String | Ticket number, for example `INC1579965`. |
| `issue_description` | String | Full issue description from the ticket. |
| `item_number` | String | Item number. Enter `UNKNOWN` if not known yet. |
| `organization_code` | String | Inventory organization code. Enter `UNKNOWN` if not known yet. |
| `organization_id` | String | Inventory organization ID. Enter `UNKNOWN` if not known yet. |
| `subinventory_code` | String | Subinventory code. Enter `UNKNOWN` if not known yet or `NA` if not applicable. |
| `locator_code` | String | Locator. Enter `UNKNOWN` if not known yet or `NA` if not applicable. |
| `cycle_count_name` | String | Cycle count name. Enter `UNKNOWN` if not known yet or `NA` if not applicable. |
| `compile_id` | String | ABC compile ID. Enter `UNKNOWN` if not known yet or `NA` if not applicable. |
| `requester_email` | String | User/requester email for sign-off. |
| `fusion_base_url` | String | Fusion host base URL, unless fixed inside the API tool. |

Output descriptors:

Do not manually add them. CSSFlow auto-creates outputs from Start inputs.

## 7. Main control flow

Build the high-level flow like this:

```text
start_ticket
  -> classify_ticket
  -> analyst_confirm_details
  -> finalize_ticket_context
  -> branch_ticket_route
       ON_HAND_BALANCE       -> onhand path
       RECEIVE_SUBINVENTORY -> receive-in-subinventory path
       CYCLE_COUNT           -> cycle-count path
       HOW_TO                -> how_to_or_navigation_response
       NAVIGATION            -> how_to_or_navigation_response
       ESCALATE/default      -> escalation_summary
```

Reason: the BRD starts with ticket intake, triage, and analyst review before the API analysis. This structure follows that order.

## 8. Step: classify_ticket

Step type: Prompt Execution

Purpose: classify the ticket and extract likely fields from the ticket text.

Input descriptors:

| Name | Type |
|---|---|
| `ticket_id` | String |
| `issue_description` | String |
| `item_number` | String |
| `organization_code` | String |
| `organization_id` | String |
| `subinventory_code` | String |
| `locator_code` | String |
| `cycle_count_name` | String |
| `compile_id` | String |

Output descriptors:

| Name | Type | Description |
|---|---|---|
| `initial_route` | String | One exact route value. |
| `request_type` | String | `ISSUE`, `HOW_TO`, `NAVIGATION`, or `UNKNOWN`. |
| `extracted_item_number` | String | Item from text or input. |
| `extracted_organization_code` | String | Org code from text or input. |
| `extracted_organization_id` | String | Org ID from text or input. |
| `extracted_subinventory_code` | String | Subinventory from text or input. |
| `extracted_locator_code` | String | Locator from text or input. |
| `extracted_cycle_count_name` | String | Cycle count name from text or input. |
| `extracted_compile_id` | String | ABC compile ID from text or input. |
| `missing_fields` | String | Comma-separated list of missing fields. |
| `classification_reason` | String | Short reason for the selected route. |

Prompt template:

```text
You are classifying an Oracle Fusion SCM inventory support ticket.

Ticket ID: {{ ticket_id }}
Issue description: {{ issue_description }}
Known item_number: {{ item_number }}
Known organization_code: {{ organization_code }}
Known organization_id: {{ organization_id }}
Known subinventory_code: {{ subinventory_code }}
Known locator_code: {{ locator_code }}
Known cycle_count_name: {{ cycle_count_name }}
Known compile_id: {{ compile_id }}

Classify the ticket.

Rules:
- If the user asks for general instructions, set request_type = HOW_TO and initial_route = HOW_TO.
- If the user asks where to navigate in the application, set request_type = NAVIGATION and initial_route = NAVIGATION.
- If the issue is missing on-hand, available-to-transact, available-to-reserve, shipment interface on-hand error, or reservation availability, set initial_route = ON_HAND_BALANCE.
- If the issue says item cannot be received in a subinventory, receiving is blocked, subinventory is restricted, or material status blocks receiving, set initial_route = RECEIVE_SUBINVENTORY.
- If the issue says item is not populated in cycle count, missing from cycle count, ABC classification, cycle count enabled, stockable, or transactable, set initial_route = CYCLE_COUNT.
- If you cannot classify it, set initial_route = ESCALATE.

Required fields:
- ON_HAND_BALANCE needs item_number and organization_code or organization_id.
- RECEIVE_SUBINVENTORY needs item_number, organization_code or organization_id, and subinventory_code.
- CYCLE_COUNT needs item_number and organization_code or organization_id. Use `UNKNOWN` for subinventory_code, cycle_count_name, and compile_id if the ticket does not provide them.

Populate every output descriptor by name.
Use exact route values only: ON_HAND_BALANCE, RECEIVE_SUBINVENTORY, CYCLE_COUNT, HOW_TO, NAVIGATION, ESCALATE.
Use UNKNOWN for any extracted field you cannot determine.
```

Data Flow Binding:

Bind every matching Start output into this step's input descriptors.

## 9. Step: analyst_confirm_details

Step type: Input Message (HITL)

Purpose: implement the BRD's analyst review and make sure the automation does not call APIs with wrong item/org data.

Input descriptors:

| Name | Type |
|---|---|
| `ticket_id` | String |
| `issue_description` | String |
| `initial_route` | String |
| `extracted_item_number` | String |
| `extracted_organization_code` | String |
| `extracted_organization_id` | String |
| `extracted_subinventory_code` | String |
| `extracted_locator_code` | String |
| `extracted_cycle_count_name` | String |
| `extracted_compile_id` | String |
| `missing_fields` | String |
| `classification_reason` | String |

Output descriptors:

| Name | Type | Description |
|---|---|---|
| `user_provided_input` | String | Analyst confirmation or corrections. |

Message template:

```text
Please confirm the ticket details before API checks start.

Ticket: {{ ticket_id }}
Issue: {{ issue_description }}
Suggested route: {{ initial_route }}
Reason: {{ classification_reason }}

Extracted values:
- item_number: {{ extracted_item_number }}
- organization_code: {{ extracted_organization_code }}
- organization_id: {{ extracted_organization_id }}
- subinventory_code: {{ extracted_subinventory_code }}
- locator_code: {{ extracted_locator_code }}
- cycle_count_name: {{ extracted_cycle_count_name }}
- compile_id: {{ extracted_compile_id }}

Missing or weak fields: {{ missing_fields }}

Reply with exactly one of these:
APPROVE
or corrections in this format:
route=ON_HAND_BALANCE; item_number=...; organization_code=...; organization_id=...; subinventory_code=...; locator_code=...; cycle_count_name=...; compile_id=...
or ESCALATE if this ticket is not covered by the automation.
```

Data Flow Binding:

Use the exact `analyst_confirm_details` bindings listed in Section 12A.

Reason: this gives one clean human checkpoint before any REST call. It also handles the BRD's "jump to Step 2 for re-verifying information" in a controlled way.

## 10. Step: finalize_ticket_context

Step type: Prompt Execution

Purpose: produce the single final set of clean variables used by all downstream API steps.

Input descriptors:

| Name | Type |
|---|---|
| `ticket_id` | String |
| `issue_description` | String |
| `initial_route` | String |
| `request_type` | String |
| `extracted_item_number` | String |
| `extracted_organization_code` | String |
| `extracted_organization_id` | String |
| `extracted_subinventory_code` | String |
| `extracted_locator_code` | String |
| `extracted_cycle_count_name` | String |
| `extracted_compile_id` | String |
| `user_provided_input` | String |
| `requester_email` | String |
| `fusion_base_url` | String |

Output descriptors:

| Name | Type |
|---|---|
| `route` | String |
| `item_number` | String |
| `organization_code` | String |
| `organization_id` | String |
| `subinventory_code` | String |
| `locator_code` | String |
| `cycle_count_name` | String |
| `compile_id` | String |
| `requester_email` | String |
| `fusion_base_url` | String |
| `ready_status` | String |
| `context_summary` | String |

Prompt template:

```text
Create the final ticket context for API execution.

Ticket: {{ ticket_id }}
Issue: {{ issue_description }}
Initial route: {{ initial_route }}
Initial request_type: {{ request_type }}
Initial extracted values:
- item_number: {{ extracted_item_number }}
- organization_code: {{ extracted_organization_code }}
- organization_id: {{ extracted_organization_id }}
- subinventory_code: {{ extracted_subinventory_code }}
- locator_code: {{ extracted_locator_code }}
- cycle_count_name: {{ extracted_cycle_count_name }}
- compile_id: {{ extracted_compile_id }}

Analyst response:
{{ user_provided_input }}

Rules:
- If analyst response is APPROVE, use the extracted values.
- If analyst response contains corrections, use the corrected values.
- If analyst response is ESCALATE, set route = ESCALATE.
- If route is ON_HAND_BALANCE, item_number and organization_code or organization_id are required.
- If route is RECEIVE_SUBINVENTORY, item_number, organization_code or organization_id, and subinventory_code are required.
- If route is CYCLE_COUNT, item_number and organization_code or organization_id are required.
- If required fields are still missing, set route = ESCALATE and ready_status = MISSING_REQUIRED_FIELDS.
- Otherwise set ready_status = READY.

Populate every output descriptor by name.
Use exact route values only.
Use UNKNOWN for unavailable fields and NA for not-applicable fields.
```

Data Flow Binding:

Bind:

- `classify_ticket` outputs into matching inputs.
- `analyst_confirm_details.user_provided_input` into `user_provided_input`.
- `start_ticket.requester_email` into `requester_email`.
- `start_ticket.fusion_base_url` into `fusion_base_url`.

## 11. Step: branch_ticket_route

Step type: Branching

Branch configuration:

| Input variable | Branch name |
|---|---|
| `ON_HAND_BALANCE` | `ON_HAND_BALANCE` |
| `RECEIVE_SUBINVENTORY` | `RECEIVE_SUBINVENTORY` |
| `CYCLE_COUNT` | `CYCLE_COUNT` |
| `HOW_TO` | `HOW_TO` |
| `NAVIGATION` | `NAVIGATION` |
| `ESCALATE` | `ESCALATE` |
| `default` | `default` |

Data Flow Binding:

`finalize_ticket_context.route` -> `next_step_name`

Canvas edges:

- `ON_HAND_BALANCE` -> `get_onhand_stocking`
- `RECEIVE_SUBINVENTORY` -> `verify_subinventory_exists`
- `CYCLE_COUNT` -> `verify_item_org_assignment`
- `HOW_TO` -> `draft_guidance_response`
- `NAVIGATION` -> `draft_guidance_response`
- `ESCALATE` -> `escalation_summary`
- `default` -> `escalation_summary`

## 12. Common Tool Execution setup

For every Fusion REST API call:

Step type: Tool Execution

Tool:

`call_api_tool`

Common fixed tool fields:

- method: `GET`
- auth type: use your configured Fusion auth
- username/password/token: fixed in the tool or secure configuration
- headers: include `Accept: application/json`

Important CSSFlow binding rule:

- In Prompt, Agent Prompt, and HITL messages, use double braces: `{{ item_number }}`.
- In Tool argument fields, do not use double braces.
- If your CSSFlow tool fields support placeholders, use single braces such as `{item_number}`.
- If your CSSFlow setup expects runtime values through tool arguments, add Input Descriptors with the same names as the tool arguments and bind them in Data Flow Binding.

Recommended API step input descriptors:

| Name | Type |
|---|---|
| `fusion_base_url` | String |
| `item_number` | String |
| `organization_code` | String |
| `organization_id` | String |
| `subinventory_code` | String |
| `locator_code` | String |
| `compile_id` | String |

Because every descriptor you create is mandatory, only add the exact descriptors needed for that API step. Do not add extra descriptors "just in case".

Important REST API note:

The BRD lists several endpoints with placeholders like `{itemStockingInquiriesUniqID}` and `{MaterialStatusId}`. Those IDs may not be known at ticket intake. If you do not already have the unique ID, configure the tool call as a collection/search call first, using query parameters for item/org/subinventory, then extract the returned ID for a detail call only if needed.

## 12A. Complete Data Flow Binding Map

Use this section while configuring the CSSFlow Data Flow Binding panels. Each row means: click `+ Add` in Data Flow Binding and map the source step output into the target step input.

Important:

- Create the target step Input Descriptor before adding the binding.
- Every input listed here is mandatory because CSSFlow has no optional/mandatory selector.
- Do not add any extra input descriptors unless you also add a binding for them.
- For Branching steps, the target input is the built-in `next_step_name`.

### Core intake and routing bindings

`classify_ticket`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `start_ticket` | `issue_description` | `issue_description` |
| `start_ticket` | `item_number` | `item_number` |
| `start_ticket` | `organization_code` | `organization_code` |
| `start_ticket` | `organization_id` | `organization_id` |
| `start_ticket` | `subinventory_code` | `subinventory_code` |
| `start_ticket` | `locator_code` | `locator_code` |
| `start_ticket` | `cycle_count_name` | `cycle_count_name` |
| `start_ticket` | `compile_id` | `compile_id` |

`analyst_confirm_details`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `start_ticket` | `issue_description` | `issue_description` |
| `classify_ticket` | `initial_route` | `initial_route` |
| `classify_ticket` | `extracted_item_number` | `extracted_item_number` |
| `classify_ticket` | `extracted_organization_code` | `extracted_organization_code` |
| `classify_ticket` | `extracted_organization_id` | `extracted_organization_id` |
| `classify_ticket` | `extracted_subinventory_code` | `extracted_subinventory_code` |
| `classify_ticket` | `extracted_locator_code` | `extracted_locator_code` |
| `classify_ticket` | `extracted_cycle_count_name` | `extracted_cycle_count_name` |
| `classify_ticket` | `extracted_compile_id` | `extracted_compile_id` |
| `classify_ticket` | `missing_fields` | `missing_fields` |
| `classify_ticket` | `classification_reason` | `classification_reason` |

`finalize_ticket_context`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `start_ticket` | `issue_description` | `issue_description` |
| `classify_ticket` | `initial_route` | `initial_route` |
| `classify_ticket` | `request_type` | `request_type` |
| `classify_ticket` | `extracted_item_number` | `extracted_item_number` |
| `classify_ticket` | `extracted_organization_code` | `extracted_organization_code` |
| `classify_ticket` | `extracted_organization_id` | `extracted_organization_id` |
| `classify_ticket` | `extracted_subinventory_code` | `extracted_subinventory_code` |
| `classify_ticket` | `extracted_locator_code` | `extracted_locator_code` |
| `classify_ticket` | `extracted_cycle_count_name` | `extracted_cycle_count_name` |
| `classify_ticket` | `extracted_compile_id` | `extracted_compile_id` |
| `analyst_confirm_details` | `user_provided_input` | `user_provided_input` |
| `start_ticket` | `requester_email` | `requester_email` |
| `start_ticket` | `fusion_base_url` | `fusion_base_url` |

`branch_ticket_route`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `route` | `next_step_name` |

### On-hand path bindings

`get_onhand_stocking`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |
| `finalize_ticket_context` | `locator_code` | `locator_code` |

`check_onhand_found`

| Source step | Source output | Target input |
|---|---|---|
| `get_onhand_stocking` | `onhand_response` | `onhand_response` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`branch_onhand_found`

| Source step | Source output | Target input |
|---|---|---|
| `check_onhand_found` | `onhand_result_route` | `next_step_name` |

`reverify_onhand_details`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `check_onhand_found` | `onhand_summary` | `onhand_summary` |

`get_pending_transactions`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`get_inventory_picks`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`get_inventory_reservations`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`get_subinventory_reservable_status`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |

`get_completed_transactions`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |

`collate_onhand_report`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `context_summary` | `context_summary` |
| `check_onhand_found` | `onhand_summary` | `onhand_summary` |
| `get_onhand_stocking` | `onhand_response` | `onhand_response` |
| `get_pending_transactions` | `pending_transactions_response` | `pending_transactions_response` |
| `get_inventory_picks` | `inventory_picks_response` | `inventory_picks_response` |
| `get_inventory_reservations` | `reservations_response` | `reservations_response` |
| `get_subinventory_reservable_status` | `subinventory_response` | `subinventory_response` |
| `get_completed_transactions` | `completed_transactions_response` | `completed_transactions_response` |

### Receive-in-subinventory path bindings

`verify_subinventory_exists`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |

`check_subinventory_found`

| Source step | Source output | Target input |
|---|---|---|
| `verify_subinventory_exists` | `subinventory_response` | `subinventory_response` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |

`branch_subinventory_found`

| Source step | Source output | Target input |
|---|---|---|
| `check_subinventory_found` | `subinventory_result_route` | `next_step_name` |

`reverify_receive_details`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |
| `check_subinventory_found` | `subinventory_check_summary` | `subinventory_check_summary` |

`get_subinventory_item_setup`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |

`get_item_attributes_receive`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`get_material_status_controls`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `verify_subinventory_exists` | `subinventory_response` | `subinventory_response` |
| `get_subinventory_item_setup` | `subinventory_item_response` | `subinventory_item_response` |
| `get_item_attributes_receive` | `item_attributes_response` | `item_attributes_response` |

`collate_receive_report`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `context_summary` | `context_summary` |
| `verify_subinventory_exists` | `subinventory_response` | `subinventory_response` |
| `get_subinventory_item_setup` | `subinventory_item_response` | `subinventory_item_response` |
| `get_item_attributes_receive` | `item_attributes_response` | `item_attributes_response` |
| `get_material_status_controls` | `material_status_controls_response` | `material_status_controls_response` |

### Cycle count path bindings

`verify_item_org_assignment`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`check_item_org_assignment`

| Source step | Source output | Target input |
|---|---|---|
| `verify_item_org_assignment` | `item_org_assignment_response` | `item_org_assignment_response` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`branch_item_org_assignment`

| Source step | Source output | Target input |
|---|---|---|
| `check_item_org_assignment` | `item_org_result_route` | `next_step_name` |

`reverify_cycle_count_details`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `check_item_org_assignment` | `item_org_summary` | `item_org_summary` |

`get_item_cycle_count_attributes`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |

`get_abc_classification`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `compile_id` | `compile_id` |

`get_cycle_count_onhand`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `fusion_base_url` | `fusion_base_url` |
| `finalize_ticket_context` | `item_number` | `item_number` |
| `finalize_ticket_context` | `organization_code` | `organization_code` |
| `finalize_ticket_context` | `organization_id` | `organization_id` |
| `finalize_ticket_context` | `subinventory_code` | `subinventory_code` |

`collate_cycle_count_report`

| Source step | Source output | Target input |
|---|---|---|
| `finalize_ticket_context` | `context_summary` | `context_summary` |
| `verify_item_org_assignment` | `item_org_assignment_response` | `item_org_assignment_response` |
| `get_item_cycle_count_attributes` | `item_cycle_count_attributes_response` | `item_cycle_count_attributes_response` |
| `get_abc_classification` | `abc_classification_response` | `abc_classification_response` |
| `get_cycle_count_onhand` | `cycle_count_onhand_response` | `cycle_count_onhand_response` |

### Guidance, escalation, sign-off, and close bindings

`draft_guidance_response`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `issue_description` | `issue_description` |
| `finalize_ticket_context` | `context_summary` | `context_summary` |
| `finalize_ticket_context` | `route` | `route` |

`escalation_summary`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `issue_description` | `issue_description` |
| `finalize_ticket_context` | `context_summary` | `context_summary` |

Create separate sign-off steps for each report path. This avoids one step trying to bind `user_message` from several different upstream report steps.

`request_onhand_signoff`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `collate_onhand_report` | `user_message` | `user_message` |
| `collate_onhand_report` | `resolution_report` | `resolution_report` |

`request_receive_signoff`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `collate_receive_report` | `user_message` | `user_message` |
| `collate_receive_report` | `resolution_report` | `resolution_report` |

`request_cycle_count_signoff`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `collate_cycle_count_report` | `user_message` | `user_message` |
| `collate_cycle_count_report` | `resolution_report` | `resolution_report` |

`request_guidance_signoff`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `draft_guidance_response` | `user_message` | `user_message` |
| `draft_guidance_response` | `resolution_report` | `resolution_report` |

`request_escalation_ack`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `escalation_summary` | `user_message` | `user_message` |
| `escalation_summary` | `resolution_report` | `resolution_report` |

Create one sign-off Branching step per sign-off HITL step.

| Branch step | Source step | Source output | Target input |
|---|---|---|---|
| `branch_onhand_signoff` | `request_onhand_signoff` | `user_provided_input` | `next_step_name` |
| `branch_receive_signoff` | `request_receive_signoff` | `user_provided_input` | `next_step_name` |
| `branch_cycle_count_signoff` | `request_cycle_count_signoff` | `user_provided_input` | `next_step_name` |
| `branch_guidance_signoff` | `request_guidance_signoff` | `user_provided_input` | `next_step_name` |
| `branch_escalation_ack` | `request_escalation_ack` | `user_provided_input` | `next_step_name` |

If you have a ticket-close API, create one close step per resolved path.

`close_onhand_ticket`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `collate_onhand_report` | `resolution_report` | `resolution_report` |
| `collate_onhand_report` | `recommended_ticket_status` | `recommended_ticket_status` |

`close_receive_ticket`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `collate_receive_report` | `resolution_report` | `resolution_report` |
| `collate_receive_report` | `recommended_ticket_status` | `recommended_ticket_status` |

`close_cycle_count_ticket`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `collate_cycle_count_report` | `resolution_report` | `resolution_report` |
| `collate_cycle_count_report` | `recommended_ticket_status` | `recommended_ticket_status` |

`close_guidance_ticket`

| Source step | Source output | Target input |
|---|---|---|
| `start_ticket` | `ticket_id` | `ticket_id` |
| `draft_guidance_response` | `resolution_report` | `resolution_report` |
| `draft_guidance_response` | `recommended_ticket_status` | `recommended_ticket_status` |

## 13. On-Hand Balance path

### 13.1 get_onhand_stocking

Step type: Tool Execution

Purpose: check available-to-transact and available-to-reserve on-hand.

API from BRD:

`/fscmRestApi/resources/11.13.18.05/itemStockingInquiries/{itemStockingInquiriesUniqID}`

Practical configuration:

- If unique ID is known, call the detail endpoint.
- If unique ID is not known, call/search item stocking inquiries by item and organization.

Input descriptors:

`fusion_base_url`, `item_number`, `organization_code`, `organization_id`, `subinventory_code`, `locator_code`

Output descriptor:

`onhand_response` String

Data Flow Binding:

Use the exact `get_onhand_stocking` bindings listed in Section 12A.

### 13.2 check_onhand_found

Step type: Prompt Execution

Input descriptors:

`onhand_response`, `item_number`, `organization_code`, `organization_id`

Output descriptors:

| Name | Type |
|---|---|
| `onhand_result_route` | String |
| `onhand_summary` | String |

Prompt:

```text
Review this on-hand API response for item {{ item_number }} and org {{ organization_code }} / {{ organization_id }}.

API response:
{{ onhand_response }}

If the response contains at least one valid on-hand or item stocking record, set onhand_result_route = FOUND.
If the response is empty, has no matching records, or says the item/org was not found, set onhand_result_route = NOT_FOUND.
If the response is an API error, set onhand_result_route = ERROR.

Also write a short onhand_summary.
Use exact route values: FOUND, NOT_FOUND, ERROR.
```

### 13.3 branch_onhand_found

Step type: Branching

Branches:

- `FOUND` -> continue to `get_pending_transactions`
- `NOT_FOUND` -> `reverify_onhand_details`
- `ERROR` -> `escalation_summary`
- `default` -> `escalation_summary`

Data Flow Binding:

`check_onhand_found.onhand_result_route` -> `next_step_name`

### 13.4 reverify_onhand_details

Step type: Input Message (HITL)

Purpose: BRD Step 5 says if no information is returned, re-verify with the user.

Input descriptors:

| Name | Type |
|---|---|
| `item_number` | String |
| `organization_code` | String |
| `organization_id` | String |
| `onhand_summary` | String |

Message:

```text
No on-hand/item stocking records were found for:
- item_number: {{ item_number }}
- organization_code: {{ organization_code }}
- organization_id: {{ organization_id }}

Finding:
{{ onhand_summary }}

Please confirm corrected values in this format:
item_number=...; organization_code=...; organization_id=...

Reply ESCALATE if the values are correct but no data exists.
```

Output descriptor:

`user_provided_input` String

Next step:

Route this to `escalation_summary` for the first mandatory build. The analyst can still see the HITL correction in the run history. If you later need the correction inside the escalation text, create a path-specific escalation prompt and bind `reverify_onhand_details.user_provided_input` into that prompt.

### 13.5 get_pending_transactions

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/inventoryStagedTransactions`

Purpose: check if pending/staged inventory transactions are blocking availability.

Output descriptor:

`pending_transactions_response` String

### 13.6 get_inventory_picks

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/inventoryPicks`

Purpose: check if pick slips/picks exist for the item and org.

Output descriptor:

`inventory_picks_response` String

### 13.7 get_inventory_reservations

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/inventoryReservations`

Purpose: check if reservations are consuming available quantity.

Output descriptor:

`reservations_response` String

### 13.8 get_subinventory_reservable_status

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/subinventories`

Purpose: check whether the subinventory where on-hand exists is reservable.

Output descriptor:

`subinventory_response` String

Note:

If the on-hand response contains multiple subinventories, add a For-each Loop:

- Loop name: `loop_onhand_subinventories`
- Input descriptor: `onhand_locations` List
- Items Input: `onhand_locations`
- Item Alias: `current_location`
- Body steps: a prompt/tool pair that builds and calls the subinventory and completed-transaction APIs for each location.
- Keep parallel execution off for first build so logs are easy to read.

### 13.9 get_completed_transactions

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/inventoryCompletedTransactions`

Purpose: retrieve completed transactions for item, org, and subinventory.

Output descriptor:

`completed_transactions_response` String

### 13.10 collate_onhand_report

Step type: Prompt Execution

Inputs:

- ticket context from `finalize_ticket_context`
- `onhand_summary`
- `onhand_response`
- `pending_transactions_response`
- `inventory_picks_response`
- `reservations_response`
- `subinventory_response`
- `completed_transactions_response`

Outputs:

| Name | Type |
|---|---|
| `resolution_report` | String |
| `user_message` | String |
| `recommended_ticket_status` | String |

Prompt:

```text
Prepare an SCM L1 investigation report for an On-Hand Balance Inquiry.

Use these checks:
1. Available-to-transact and available-to-reserve on-hand.
2. Pending/staged inventory transactions.
3. Pick slips or picks.
4. Reservations.
5. Whether the subinventory is reservable.
6. Completed transactions.

Ticket context:
{{ context_summary }}

On-hand summary:
{{ onhand_summary }}

Raw API responses:
On-hand: {{ onhand_response }}
Pending transactions: {{ pending_transactions_response }}
Picks: {{ inventory_picks_response }}
Reservations: {{ reservations_response }}
Subinventory: {{ subinventory_response }}
Completed transactions: {{ completed_transactions_response }}

Write:
- What was checked.
- What was found.
- Likely cause of the on-hand or availability issue.
- Recommended next action.
- Clear user-facing message.

Populate resolution_report, user_message, and recommended_ticket_status.
```

## 14. Item cannot be received in Subinventory path

### 14.1 verify_subinventory_exists

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/subinventories`

Purpose: verify item, org, and subinventory information exists.

Output descriptor:

`subinventory_response` String

### 14.2 check_subinventory_found

Step type: Prompt Execution

Input descriptors:

- `subinventory_response` String
- `item_number` String
- `organization_code` String
- `organization_id` String
- `subinventory_code` String

Output descriptors:

- `subinventory_result_route` String
- `subinventory_check_summary` String

Route values:

- `FOUND`
- `NOT_FOUND`
- `ERROR`

Branch:

- `FOUND` -> `get_subinventory_item_setup`
- `NOT_FOUND` -> `reverify_receive_details`
- `ERROR/default` -> `escalation_summary`

Add a Branching step named `branch_subinventory_found` after this prompt. Bind `check_subinventory_found.subinventory_result_route` to `next_step_name`.

### 14.2A reverify_receive_details

Step type: Input Message (HITL)

Input descriptors:

- `item_number` String
- `organization_code` String
- `organization_id` String
- `subinventory_code` String
- `subinventory_check_summary` String

Output descriptor:

- `user_provided_input` String

Message:

```text
The subinventory check did not return a valid match.

Current values:
- item_number: {{ item_number }}
- organization_code: {{ organization_code }}
- organization_id: {{ organization_id }}
- subinventory_code: {{ subinventory_code }}

Finding:
{{ subinventory_check_summary }}

Please reply with corrected values in this format:
item_number=...; organization_code=...; organization_id=...; subinventory_code=...

Reply ESCALATE if the values are correct but the setup is still not found.
```

Next step:

Route this to `escalation_summary` for the first mandatory build. The analyst can still see the HITL correction in the run history. If you later need the correction inside the escalation text, create a path-specific escalation prompt and bind `reverify_receive_details.user_provided_input` into that prompt.

### 14.3 get_subinventory_item_setup

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/subinventoryItems`

Purpose: check item-subinventory setup for the item and subinventory combination.

Output descriptor:

`subinventory_item_response` String

### 14.4 get_item_attributes_receive

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/itemsV2`

Purpose: check restricted subinventory field and item-level setup.

Output descriptor:

`item_attributes_response` String

### 14.5 get_material_status_controls

Step type: Tool Execution

API from BRD:

`/fscmRestApi/resources/11.13.18.05/materialStatuses/{MaterialStatusId}/child/transactionControls/{transactionControlsUniqID}`

Purpose: check whether material status disallows receiving or other transactions.

Implementation note:

If `MaterialStatusId` or `transactionControlsUniqID` is not already known:

1. Extract `MaterialStatusId` from subinventory or item setup response.
2. Call/search the material status transaction controls child collection.
3. Only call the detail endpoint if CSSFlow needs a specific child record.

Output descriptor:

`material_status_controls_response` String

### 14.6 collate_receive_report

Step type: Prompt Execution

Inputs:

- `subinventory_response`
- `subinventory_item_response`
- `item_attributes_response`
- `material_status_controls_response`
- `context_summary`

Outputs:

- `resolution_report`
- `user_message`
- `recommended_ticket_status`

Prompt:

```text
Prepare an SCM L1 investigation report for "Item cannot be received in Subinventory".

Use these checks:
1. Whether the subinventory exists for the organization.
2. Whether item-subinventory setup exists.
3. Whether the item has restricted subinventory setup.
4. Whether material status transaction controls disallow receiving or inventory transactions.

Ticket context:
{{ context_summary }}

API responses:
Subinventory: {{ subinventory_response }}
Subinventory item setup: {{ subinventory_item_response }}
Item attributes: {{ item_attributes_response }}
Material status controls: {{ material_status_controls_response }}

Explain the likely reason the item cannot be received in the subinventory.
Include the exact setup area that should be corrected if a setup issue is found.
Populate resolution_report, user_message, and recommended_ticket_status.
```

## 15. Item not populated in cycle count path

### 15.1 verify_item_org_assignment

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/itemSourceOrganizationsLOV/{OrganizationId}`

Purpose: verify item is assigned to the required organization.

Output descriptor:

`item_org_assignment_response` String

### 15.2 check_item_org_assignment

Step type: Prompt Execution

Input descriptors:

- `item_org_assignment_response` String
- `item_number` String
- `organization_code` String
- `organization_id` String

Output descriptors:

- `item_org_result_route` String
- `item_org_summary` String

Route values:

- `FOUND`
- `NOT_FOUND`
- `ERROR`

Branch:

- `FOUND` -> `get_item_cycle_count_attributes`
- `NOT_FOUND` -> `reverify_cycle_count_details`
- `ERROR/default` -> `escalation_summary`

Add a Branching step named `branch_item_org_assignment` after this prompt. Bind `check_item_org_assignment.item_org_result_route` to `next_step_name`.

### 15.2A reverify_cycle_count_details

Step type: Input Message (HITL)

Input descriptors:

- `item_number` String
- `organization_code` String
- `organization_id` String
- `item_org_summary` String

Output descriptor:

- `user_provided_input` String

Message:

```text
The item organization assignment check did not return a valid match.

Current values:
- item_number: {{ item_number }}
- organization_code: {{ organization_code }}
- organization_id: {{ organization_id }}

Finding:
{{ item_org_summary }}

Please reply with corrected values in this format:
item_number=...; organization_code=...; organization_id=...

Reply ESCALATE if the values are correct but the item assignment is still not found.
```

Next step:

Route this to `escalation_summary` for the first mandatory build. The analyst can still see the HITL correction in the run history. If you later need the correction inside the escalation text, create a path-specific escalation prompt and bind `reverify_cycle_count_details.user_provided_input` into that prompt.

### 15.3 get_item_cycle_count_attributes

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/itemsV2`

Purpose: check item is cycle count enabled, stockable, and transactable.

BRD lists these as two checks, but one `itemsV2` API response may contain all three fields. Keep it as one tool call unless your Fusion response requires separate finders.

Output descriptor:

`item_cycle_count_attributes_response` String

### 15.4 get_abc_classification

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/abcClassifications/{CompileId}/child/abcClassificationSetItems/{abcClassificationSetItemsUniqID}`

Purpose: verify whether the item is part of ABC classification.

Implementation note:

If `CompileId` or `abcClassificationSetItemsUniqID` is not known, first search/list ABC classification sets for the org/cycle count context, then call detail if required.

Output descriptor:

`abc_classification_response` String

### 15.5 get_cycle_count_onhand

Step type: Tool Execution

API:

`/fscmRestApi/resources/11.13.18.05/itemStockingInquiries/{itemStockingInquiriesUniqID}`

Purpose: verify if the item has on-hand quantity in subinventories mentioned in the cycle count definition.

Output descriptor:

`cycle_count_onhand_response` String

### 15.6 collate_cycle_count_report

Step type: Prompt Execution

Inputs:

- `item_org_assignment_response`
- `item_cycle_count_attributes_response`
- `abc_classification_response`
- `cycle_count_onhand_response`
- `context_summary`

Outputs:

- `resolution_report`
- `user_message`
- `recommended_ticket_status`

Prompt:

```text
Prepare an SCM L1 investigation report for "Item not populated in cycle count".

Use these checks:
1. Item is assigned to the organization.
2. Item is cycle count enabled.
3. Item is stockable.
4. Item is transactable.
5. Item is part of ABC classification.
6. Item has on-hand in the subinventories included in cycle count definition.

Ticket context:
{{ context_summary }}

API responses:
Item org assignment: {{ item_org_assignment_response }}
Item attributes: {{ item_cycle_count_attributes_response }}
ABC classification: {{ abc_classification_response }}
On-hand: {{ cycle_count_onhand_response }}

Explain the likely reason the item is not appearing in cycle count.
Include the exact failing check and suggested correction.
Populate resolution_report, user_message, and recommended_ticket_status.
```

## 16. HOW_TO and NAVIGATION path

Step name:

`draft_guidance_response`

Step type: Prompt Execution

Purpose: if the ticket is not a break/fix issue but a how-to or navigation request, do not call all diagnostic APIs. Draft a helpful guidance response instead.

Inputs:

- `issue_description`
- `context_summary`
- `route`

Outputs:

- `resolution_report`
- `user_message`
- `recommended_ticket_status`

Prompt:

```text
The ticket was classified as {{ route }}, not a break/fix API investigation.

Ticket context:
{{ context_summary }}

Issue:
{{ issue_description }}

Draft a concise support response with:
- The likely navigation or how-to guidance.
- Any information the user must confirm.
- A recommendation to convert to investigation if they are seeing an error.

Populate resolution_report, user_message, and recommended_ticket_status.
```

Then connect to the sign-off step.

## 17. Escalation path

Step name:

`escalation_summary`

Step type: Prompt Execution

Purpose: prepare a clean handoff when the workflow cannot safely resolve the ticket.

Outputs:

- `resolution_report`
- `user_message`
- `recommended_ticket_status`

Prompt:

```text
Prepare an escalation summary for this SCM inventory ticket.

Ticket context:
{{ context_summary }}
Issue:
{{ issue_description }}

Explain why automation could not complete the ticket. Mention missing data, unsupported subtype, API error, or repeated no-data condition if applicable.
List what the human analyst should check next.
Populate resolution_report, user_message, and recommended_ticket_status.
```

Then connect to sign-off or manual close depending on your support process.

## 18. Final sign-off steps

Do not use one shared sign-off step for all paths. Create one sign-off step per report path so every input has one clear source binding.

Create these Input Message (HITL) steps:

- `request_onhand_signoff`
- `request_receive_signoff`
- `request_cycle_count_signoff`
- `request_guidance_signoff`
- `request_escalation_ack`

Step type: Input Message (HITL)

Purpose: implement BRD "L1 Agent shares the document with User and user signs off."

Input descriptors:

- `ticket_id`
- `user_message`
- `resolution_report`

Output descriptor:

- `user_provided_input` String

Message:

```text
Investigation completed for ticket {{ ticket_id }}.

Summary for you:
{{ user_message }}

Please reply with exactly one:
SIGNED_OFF
MORE_HELP
```

Data Flow Binding:

Use the path-specific bindings listed in Section 12A.

## 19. Branch after sign-off

Create one Branching step per sign-off step:

- `branch_onhand_signoff`
- `branch_receive_signoff`
- `branch_cycle_count_signoff`
- `branch_guidance_signoff`
- `branch_escalation_ack`

Step type: Branching

Branches:

- `SIGNED_OFF` -> matching close-ticket step, if a close-ticket API exists
- `MORE_HELP` -> `more_help_summary`
- `default` -> `more_help_summary`

Data Flow Binding:

Use the path-specific sign-off branch bindings listed in Section 12A.

## 20. Close or update ticket

Create one close-ticket Tool Execution step per resolved path:

- `close_onhand_ticket`
- `close_receive_ticket`
- `close_cycle_count_ticket`
- `close_guidance_ticket`

Step type: Tool Execution

Purpose: close or update the source ticket.

Tool:

Use `call_api_tool` only if your ticket system endpoint is available.

Inputs:

- `ticket_id`
- `resolution_report`
- `recommended_ticket_status`

Data Flow Binding:

Use the path-specific close-ticket bindings listed in Section 12A.

If no ticketing API is configured:

- Replace this step with a Slack notification or an HITL/manual action.
- Do not pretend CSSFlow closed the ticket.

## 21. Data Flow Binding checklist

For every Prompt Execution step:

- Add Input Descriptors first.
- Use `{{ variable_name }}` inside the prompt.
- Bind prior outputs into those inputs.

For every Tool Execution step:

- Add Input Descriptors for runtime values.
- Use fixed tool fields for fixed values like method/auth.
- Use single-brace placeholders in tool fields only if your CSSFlow setup supports them.
- Never use `{{ variable_name }}` inside URL/query/header/body tool fields.

For every Branching step:

- Bind the upstream decision output into `next_step_name`.
- Ensure branch names exactly match upstream output values.
- Add `default` as a safety branch.

For every HITL step:

- Tell the user the exact accepted replies.
- Keep the question simple.
- Use the output descriptor `user_provided_input`.

## 22. Testing plan

Test 1: Sample on-hand issue

- ticket_id: `INC1579965`
- issue_description: `Missing onhand creating shipment interface errors`
- expected route: `ON_HAND_BALANCE`
- expected behavior: on-hand API, pending transactions, picks, reservations, subinventory, and completed transaction checks run.

Test 2: Missing item number

- omit `item_number`
- expected behavior: analyst confirmation asks for missing item number before API calls.

Test 3: Cannot receive in subinventory

- issue_description: `Item cannot be received in subinventory`
- expected route: `RECEIVE_SUBINVENTORY`
- expected behavior: subinventory, subinventoryItems, itemsV2, and material status controls checks run.

Test 4: Cycle count issue

- issue_description: `Item not populated in cycle count`
- expected route: `CYCLE_COUNT`
- expected behavior: org assignment, itemsV2, ABC classification, and on-hand checks run.

Test 5: How-to request

- issue_description: `How do I check on-hand balance for an item?`
- expected route: `HOW_TO`
- expected behavior: guidance response, no diagnostic API chain.

## 23. Common mistakes to avoid

1. Do not use `{{ item_number }}` inside API URL fields. Use data binding and, if supported, `{item_number}`.
2. Do not skip the analyst confirmation step. Wrong item/org data will make the API results misleading.
3. Do not create branch values like `Found` if your prompt returns `FOUND`. Branch values are case-sensitive.
4. Do not wire a branch to the wrong output descriptor. Always wire the decision field, not the explanation field.
5. Do not close the ticket unless the user has signed off or your process allows auto-closure.
6. Do not assume `{UniqID}` path parameters are known. Add lookup/search calls when those IDs are missing.

## 24. Simplest first version

If you want the fastest reliable build, implement this first:

1. Start
2. `classify_ticket`
3. `analyst_confirm_details`
4. `finalize_ticket_context`
5. `branch_ticket_route`
6. Three deterministic API paths
7. Three report prompts
8. Three sign-off HITL steps
9. Manual close or close-ticket API

After that works, add:

- retry copies for NOT_FOUND cases,
- For-each loop for multiple on-hand subinventories,
- OCI File Write for report storage,
- Slack notification,
- automatic ticket close.
