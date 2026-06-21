# CSSFlow workflow implementation: Item not populated in Cycle Count

This guide converts the pseudocode in `flow.txt` into a CSSFlow workflow using the step types shown in the screenshots and the REST details from `rest api and extra.xlsx`.

The workflow should be built mostly with Tool Execution, Prompt Execution, Branching, Input Message, and Loop steps. Do not build this as one big Agent Execution step. The reason is simple: your pseudocode is a fixed decision tree. Fixed decision trees are easier to test and safer when each API call, decision, approval, and update is visible as its own step.

## Important rules to follow

1. Use only mandatory descriptors.
   - CSSFlow does not give you an optional/mandatory switch.
   - Treat every Input Descriptor and Output Descriptor listed below as mandatory.
   - Do not create extra descriptors that are not bound or used.

2. Use double braces only inside text fields.
   - Prompt Template, Agent Prompt, and HITL Message Template use `{{ variable_name }}`.

3. Use single braces only inside Tool Execution argument fields.
   - Tool URL/body fields can use `{variable_name}`.
   - Example: `...CycleCountName="{cycle_count_name}"; OrganizationCode="{organization_code}"`

4. Always bind data separately.
   - A canvas arrow controls execution order.
   - Data Flow Binding passes values from one step output into another step input.
   - You need both.

5. Use exact branch labels.
   - Branch values are case-sensitive.
   - If a prompt says it will output `ITEM_IN_CC`, the branch name must be exactly `ITEM_IN_CC`.

6. The Excel sheet has a typo in the visible text for count subinventories.
   - Do not use `countsubinventoryentories`.
   - Use the hyperlink-style path:
     `/cycleCountDefinitions/{cycle_count_definition_id}/child/countSubinventories`

7. Two ESS jobs are shown as UI screenshots, not complete REST payloads.
   - `Create ABC Classification Set`
   - `Perform ABC Assignments and Synchronize Cycle Counts`
   - The screenshots give the process names and parameters, but not the Oracle internal job package/job definition values.
   - Before building the ESS Tool steps, get the exact `JobPackageName` and `JobDefName` from your Fusion admin or existing ESS REST examples in your environment.

## Workflow create screen

Use these values:

| Field | Value |
|---|---|
| Name | `Cycle Count Item Diagnosis and Repair` |
| Description | `Checks why an item is not populated in a cycle count definition, fixes safe setup gaps after approval, and runs ABC/cycle count synchronization jobs when required.` |
| LLM | `openai.gpt-5.5` if available. Otherwise use the strongest configured OpenAI model in your CSSFlow LLM list. |

Reason: the LLM is only used for controlled JSON reading and branch labels. A stronger model reduces wrong branch labels.

## Start step

Step name: `start_cycle_count_request`

Create these Input Descriptors:

| Name | Type | Description |
|---|---|---|
| `base_url` | String | Fusion host URL, for example `https://fa-esec-dev40-saasfademo1.ds-fa.oraclepdemos.com` |
| `cycle_count_name` | String | Cycle count name, example `CC5` |
| `organization_code` | String | Inventory organization code, example `NEW_PLANT` |
| `item_number` | String | Item number, example `NEW_ASSET1` |
| `target_subinventory` | String | Subinventory mentioned by the user/ticket, example `P1` |

Output Descriptors are auto-created from these Start inputs.

Reason: these five values are the minimum needed by your pseudocode and the provided REST examples.

## Tool convention

For every Tool Execution step below:

| Field | Value |
|---|---|
| Select tool | `call_api_tool` or the available API tool of type `api_request` |
| Auth | Use the same fixed Fusion authentication settings used by your existing CSSFlow API tool |
| Headers | `Content-Type: application/json` and `Accept: application/json` when the tool exposes headers |

If your tool has fixed fields for method, URL, headers, and body, enter the values there. If the field value changes at runtime, create an Input Descriptor with the same variable used in the field and bind it.

## High-level workflow map

```mermaid
flowchart TD
  A["Start"] --> B["GET cycle count definition"]
  B --> C["Extract cycle count fields"]
  C --> D{"Definition found?"}
  D -->|"DEFINITION_NOT_FOUND"| Z1["Finish: manual check"]
  D -->|"DEFINITION_FOUND"| E["GET count item"]
  E --> F["Decide item in CC"]
  F --> G{"Item in CC?"}
  G -->|"ITEM_IN_CC"| H["GET onhand"]
  G -->|"ITEM_NOT_IN_CC"| N["GET item attributes"]
  H --> I["Decide onhand"]
  I --> J{"Has onhand?"}
  J -->|"ONHAND_EXISTS"| K["GET CC subinventory"]
  J -->|"NO_ONHAND"| L["Check Count Zero flag"]
  K --> M{"Subinventory included?"}
  M -->|"SUBINV_INCLUDED"| Z2["Finish: raise Product SR"]
  M -->|"SUBINV_NOT_INCLUDED"| U1["PATCH CC subinventory include in schedule"]
  M -->|"SUBINV_NOT_FOUND"| Z3["Finish: manual check"]
  L --> O{"Count zero enabled?"}
  O -->|"COUNT_ZERO_ENABLED"| Z2
  O -->|"COUNT_ZERO_DISABLED"| P["HITL approve Count Zero"]
  P --> Q{"Approval?"}
  Q -->|"APPROVE"| U2["PATCH Count Zero flag"]
  Q -->|"REJECT"| Z4["Finish: user rejected"]
  N --> R["Decide item CC enabled"]
  R --> S{"Item enabled?"}
  S -->|"ITEM_CC_ENABLED"| ESS1["Submit Create ABC Classification Set"]
  S -->|"ITEM_CC_DISABLED"| T["HITL approve item flag + jobs"]
  T --> V{"Approval?"}
  V -->|"APPROVE"| U3["PATCH item CycleCountEnabledFlag"]
  V -->|"REJECT"| Z4
  U3 --> ESS1
  ESS1 --> W["Wait for Create ABC job success"]
  W --> X{"Create ABC succeeded?"}
  X -->|"CREATE_ABC_SUCCEEDED"| ESS2["Submit Perform ABC Assignments and Synchronize Cycle Counts"]
  X -->|"CREATE_ABC_FAILED"| Z5["Finish: ESS failed"]
  U1 --> Z6["Finish: subinventory updated"]
  U2 --> Z7["Finish: count zero updated"]
  ESS2 --> Z8["Finish: jobs submitted"]
```

## Step-by-step build

### 1. GET cycle count definition

Step type: Tool Execution  
Step name: `get_cycle_count_definition`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `cycle_count_name` | String |
| `organization_code` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `cycle_count_definition_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `GET` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/cycleCountDefinitions?q=CycleCountName="{cycle_count_name}"; OrganizationCode="{organization_code}"` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `start_cycle_count_request` | `cycle_count_name` | `cycle_count_name` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |

Reason: this returns the cycle count definition and the important IDs/flags needed by later calls.

### 2. Extract cycle count fields

Step type: Prompt Execution  
Step name: `extract_cycle_count_definition`

Input Descriptors:

| Name | Type |
|---|---|
| `cycle_count_definition_response` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `definition_route` | String |
| `cycle_count_definition_id` | String |
| `organization_id` | String |
| `abc_assignment_group_name` | String |
| `abc_classification_set_name` | String |
| `count_zero_quantities_flag` | String |

Prompt Template:

```text
Read this Oracle Fusion cycleCountDefinitions API response:
{{ cycle_count_definition_response }}

Populate the CSSFlow output descriptors exactly as follows.

definition_route:
- Use DEFINITION_FOUND if one cycle count definition item exists.
- Use DEFINITION_NOT_FOUND if no item exists or the response is empty.

cycle_count_definition_id:
- Return the unique ID/key used in child URLs for this cycle count definition.
- If missing, return NOT_FOUND.

organization_id:
- Return the organization id from the response.
- If missing, return NOT_FOUND.

abc_assignment_group_name:
- Return the ABC assignment group name from the response.
- If missing, return NOT_FOUND.

abc_classification_set_name:
- Return the ABC classification set name from the response.
- If missing, return NOT_FOUND.

count_zero_quantities_flag:
- Return TRUE if Count Zero Quantities is enabled.
- Return FALSE if it is disabled.
- If unknown, return UNKNOWN.

Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `get_cycle_count_definition` | `cycle_count_definition_response` | `cycle_count_definition_response` |

Reason: Branching can only route using a clean string. This prompt converts raw JSON into exact values.

### 3. Branch: definition found?

Step type: Branching  
Step name: `branch_definition_found`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `DEFINITION_FOUND` | `DEFINITION_FOUND` |
| `DEFINITION_NOT_FOUND` | `DEFINITION_NOT_FOUND` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `extract_cycle_count_definition` | `definition_route` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `DEFINITION_FOUND` | `get_cycle_count_item` |
| `DEFINITION_NOT_FOUND` | `finish_definition_not_found` |

Reason: if the cycle count definition itself is missing, the rest of the workflow cannot safely update anything.

### 4. Finish: definition not found

Step type: Prompt Execution  
Step name: `finish_definition_not_found`

Input Descriptors:

| Name | Type |
|---|---|
| `cycle_count_name` | String |
| `organization_code` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `final_message` | String |

Prompt Template:

```text
Write this exact result in final_message:
Cycle count definition {{ cycle_count_name }} was not found for organization {{ organization_code }}. Please verify the cycle count name and organization before running the workflow again.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `cycle_count_name` | `cycle_count_name` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |

Connect this step to End.

### 5. GET cycle count item

Step type: Tool Execution  
Step name: `get_cycle_count_item`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `cycle_count_definition_id` | String |
| `item_number` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `cycle_count_item_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `GET` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/cycleCountDefinitions/{cycle_count_definition_id}/child/countItems?q=ItemNumber="{item_number}"` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `extract_cycle_count_definition` | `cycle_count_definition_id` | `cycle_count_definition_id` |
| `start_cycle_count_request` | `item_number` | `item_number` |

Reason: this validates whether the item is already part of the cycle count definition.

### 6. Decide item in cycle count

Step type: Prompt Execution  
Step name: `decide_item_in_cycle_count`

Input Descriptors:

| Name | Type |
|---|---|
| `cycle_count_item_response` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `item_in_cc_route` | String |
| `cycle_count_item_id` | String |

Prompt Template:

```text
Read this countItems API response:
{{ cycle_count_item_response }}

Set item_in_cc_route:
- ITEM_IN_CC if the response contains one matching item.
- ITEM_NOT_IN_CC if the response has no matching item.

Set cycle_count_item_id:
- Return the count item unique id if present.
- Otherwise return NOT_FOUND.

Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `get_cycle_count_item` | `cycle_count_item_response` | `cycle_count_item_response` |

### 7. Branch: item in cycle count?

Step type: Branching  
Step name: `branch_item_in_cycle_count`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `ITEM_IN_CC` | `ITEM_IN_CC` |
| `ITEM_NOT_IN_CC` | `ITEM_NOT_IN_CC` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `decide_item_in_cycle_count` | `item_in_cc_route` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `ITEM_IN_CC` | `get_item_onhand` |
| `ITEM_NOT_IN_CC` | `get_item_attributes` |

Reason: this is the first main IF/ELSE in your pseudocode.

## Path A: item is already part of the cycle count

### 8. GET item onhand

Step type: Tool Execution  
Step name: `get_item_onhand`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `item_number` | String |
| `organization_code` | String |
| `target_subinventory` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `onhand_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `GET` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/itemStockingInquiries?q=ItemNumber="{item_number}";OrganizationCode="{organization_code}";Subinventory="{target_subinventory}"` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `start_cycle_count_request` | `item_number` | `item_number` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |
| `start_cycle_count_request` | `target_subinventory` | `target_subinventory` |

Reason: your flow says that if the item is in the CC definition, the next question is whether it has onhand.

Note: `itemStockingInquiries` comes from the BRD sheet. If your Fusion environment uses a different onhand endpoint/query format, keep the step and bindings the same but replace only the URL.

### 9. Decide onhand exists

Step type: Prompt Execution  
Step name: `decide_onhand_exists`

Input Descriptors:

| Name | Type |
|---|---|
| `onhand_response` | String |
| `target_subinventory` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `onhand_route` | String |
| `onhand_subinventory` | String |

Prompt Template:

```text
Read this onhand/stocking inquiry response:
{{ onhand_response }}

The user's target subinventory is:
{{ target_subinventory }}

Set onhand_route:
- ONHAND_EXISTS if available onhand quantity is greater than zero.
- NO_ONHAND if there is no onhand quantity.

Set onhand_subinventory:
- If the target subinventory has onhand, return the target subinventory.
- If another subinventory has onhand, return that subinventory name.
- If no onhand exists, return NOT_FOUND.

Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `get_item_onhand` | `onhand_response` | `onhand_response` |
| `start_cycle_count_request` | `target_subinventory` | `target_subinventory` |

### 10. Branch: has onhand?

Step type: Branching  
Step name: `branch_onhand_exists`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `ONHAND_EXISTS` | `ONHAND_EXISTS` |
| `NO_ONHAND` | `NO_ONHAND` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `decide_onhand_exists` | `onhand_route` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `ONHAND_EXISTS` | `get_cycle_count_subinventory` |
| `NO_ONHAND` | `route_count_zero_flag` |

### 11. GET cycle count subinventory

Step type: Tool Execution  
Step name: `get_cycle_count_subinventory`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `cycle_count_definition_id` | String |
| `onhand_subinventory` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `cycle_count_subinventory_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `GET` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/cycleCountDefinitions/{cycle_count_definition_id}/child/countSubinventories?q=Subinventory="{onhand_subinventory}"` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `extract_cycle_count_definition` | `cycle_count_definition_id` | `cycle_count_definition_id` |
| `decide_onhand_exists` | `onhand_subinventory` | `onhand_subinventory` |

Reason: this checks whether the onhand subinventory is included in the cycle count schedule.

### 12. Extract subinventory setup

Step type: Prompt Execution  
Step name: `extract_subinventory_setup`

Input Descriptors:

| Name | Type |
|---|---|
| `cycle_count_subinventory_response` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `subinventory_route` | String |
| `subinventory_unique_id` | String |
| `subinventory_update_payload` | String |

Prompt Template:

```text
Read this cycle count countSubinventories API response:
{{ cycle_count_subinventory_response }}

Set subinventory_route:
- SUBINV_INCLUDED if a matching subinventory record exists and Include In Schedule is true.
- SUBINV_NOT_INCLUDED if a matching subinventory record exists and Include In Schedule is false.
- SUBINV_NOT_FOUND if no matching subinventory record exists.

Set subinventory_unique_id:
- Return the child subinventory unique ID needed in the PATCH URL.
- If missing, return NOT_FOUND.

Set subinventory_update_payload:
- Return a JSON object that updates the same Include In Schedule boolean field to true.
- Use the exact REST attribute name found in the response.
- If you cannot find the field name, return {"IncludeInScheduleFlag": true}.

Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `get_cycle_count_subinventory` | `cycle_count_subinventory_response` | `cycle_count_subinventory_response` |

### 13. Branch: subinventory included?

Step type: Branching  
Step name: `branch_subinventory_in_schedule`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `SUBINV_INCLUDED` | `SUBINV_INCLUDED` |
| `SUBINV_NOT_INCLUDED` | `SUBINV_NOT_INCLUDED` |
| `SUBINV_NOT_FOUND` | `SUBINV_NOT_FOUND` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `extract_subinventory_setup` | `subinventory_route` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `SUBINV_INCLUDED` | `finish_raise_product_sr` |
| `SUBINV_NOT_INCLUDED` | `update_cycle_count_subinventory` |
| `SUBINV_NOT_FOUND` | `finish_subinventory_not_found` |

Reason: this implements the pseudocode line: if CC subinventory and onhand subinventory match, root cause cannot be found; otherwise update Include In Schedule.

### 14. PATCH cycle count subinventory Include In Schedule

Step type: Tool Execution  
Step name: `update_cycle_count_subinventory`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `cycle_count_definition_id` | String |
| `subinventory_unique_id` | String |
| `subinventory_update_payload` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `update_subinventory_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `PATCH` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/cycleCountDefinitions/{cycle_count_definition_id}/child/countSubinventories/{subinventory_unique_id}` |
| data/body | `{subinventory_update_payload}` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `extract_cycle_count_definition` | `cycle_count_definition_id` | `cycle_count_definition_id` |
| `extract_subinventory_setup` | `subinventory_unique_id` | `subinventory_unique_id` |
| `extract_subinventory_setup` | `subinventory_update_payload` | `subinventory_update_payload` |

Connect to `finish_subinventory_updated`.

### 15. Route Count Zero flag

Step type: Prompt Execution  
Step name: `route_count_zero_flag`

Input Descriptors:

| Name | Type |
|---|---|
| `count_zero_quantities_flag` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `count_zero_route` | String |

Prompt Template:

```text
The Count Zero Quantities flag value is:
{{ count_zero_quantities_flag }}

Set count_zero_route:
- COUNT_ZERO_ENABLED if the value is TRUE.
- COUNT_ZERO_DISABLED if the value is FALSE.
- COUNT_ZERO_UNKNOWN if the value is anything else.

Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `extract_cycle_count_definition` | `count_zero_quantities_flag` | `count_zero_quantities_flag` |

### 16. Branch: Count Zero enabled?

Step type: Branching  
Step name: `branch_count_zero_enabled`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `COUNT_ZERO_ENABLED` | `COUNT_ZERO_ENABLED` |
| `COUNT_ZERO_DISABLED` | `COUNT_ZERO_DISABLED` |
| `COUNT_ZERO_UNKNOWN` | `COUNT_ZERO_UNKNOWN` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `route_count_zero_flag` | `count_zero_route` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `COUNT_ZERO_ENABLED` | `finish_raise_product_sr` |
| `COUNT_ZERO_DISABLED` | `confirm_enable_count_zero` |
| `COUNT_ZERO_UNKNOWN` | `finish_manual_count_zero_check` |

### 17. HITL: confirm Count Zero update

Step type: Input Message (HITL)  
Step name: `confirm_enable_count_zero`

Input Descriptors:

| Name | Type |
|---|---|
| `cycle_count_name` | String |
| `organization_code` | String |
| `item_number` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `user_count_zero_decision` | String |

Message Template:

```text
Item {{ item_number }} is already in cycle count {{ cycle_count_name }} for organization {{ organization_code }}, but it has no onhand quantity and Count Zero Quantities is disabled.

Do you approve enabling Count Zero Quantities for this cycle count?

Reply with exactly one word:
APPROVE
or
REJECT
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `cycle_count_name` | `cycle_count_name` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |
| `start_cycle_count_request` | `item_number` | `item_number` |

### 18. Branch: Count Zero approval

Step type: Branching  
Step name: `branch_count_zero_approval`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `APPROVE` | `APPROVE` |
| `REJECT` | `REJECT` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `confirm_enable_count_zero` | `user_count_zero_decision` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `APPROVE` | `build_count_zero_update_payload` |
| `REJECT` | `finish_user_rejected` |

### 19. Build Count Zero update payload

Step type: Prompt Execution  
Step name: `build_count_zero_update_payload`

Input Descriptors:

| Name | Type |
|---|---|
| `cycle_count_definition_response` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `count_zero_update_payload` | String |

Prompt Template:

```text
Read this cycle count definition response:
{{ cycle_count_definition_response }}

Set count_zero_update_payload to a JSON object that enables Count Zero Quantities.
Use the exact REST field name from the response.
If the response does not clearly show the field name, use:
{"CountZeroQuantityFlag": true}

Return only the JSON object.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `get_cycle_count_definition` | `cycle_count_definition_response` | `cycle_count_definition_response` |

### 20. PATCH Count Zero Quantities flag

Step type: Tool Execution  
Step name: `update_count_zero_quantities`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `cycle_count_definition_id` | String |
| `count_zero_update_payload` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `update_count_zero_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `PATCH` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/cycleCountDefinitions/{cycle_count_definition_id}` |
| data/body | `{count_zero_update_payload}` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `extract_cycle_count_definition` | `cycle_count_definition_id` | `cycle_count_definition_id` |
| `build_count_zero_update_payload` | `count_zero_update_payload` | `count_zero_update_payload` |

Connect to `finish_count_zero_updated`.

## Path B: item is not part of the cycle count

### 21. GET item attributes

Step type: Tool Execution  
Step name: `get_item_attributes`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `item_number` | String |
| `organization_code` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `item_attributes_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `GET` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/itemsV2?q=ItemNumber="{item_number}";OrganizationCode="{organization_code}"` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `start_cycle_count_request` | `item_number` | `item_number` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |

Reason: your pseudocode next checks whether the item is cycle-count enabled.

### 22. Decide item Cycle Count enabled

Step type: Prompt Execution  
Step name: `decide_item_cycle_count_enabled`

Input Descriptors:

| Name | Type |
|---|---|
| `item_attributes_response` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `item_cc_enabled_route` | String |
| `item_unique_id` | String |
| `item_update_payload` | String |

Prompt Template:

```text
Read this itemsV2 API response:
{{ item_attributes_response }}

Set item_cc_enabled_route:
- ITEM_CC_ENABLED if CycleCountEnabledFlag is true.
- ITEM_CC_DISABLED if CycleCountEnabledFlag is false.
- ITEM_NOT_FOUND if no item is returned.

Set item_unique_id:
- Return the unique ID/key needed for the itemsV2 PATCH URL.
- If missing, return NOT_FOUND.

Set item_update_payload:
- Return this JSON if the item exists:
{"CycleCountEnabledFlag": true}
- If the item does not exist, return {}.

Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `get_item_attributes` | `item_attributes_response` | `item_attributes_response` |

### 23. Branch: item Cycle Count enabled?

Step type: Branching  
Step name: `branch_item_cycle_count_enabled`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `ITEM_CC_ENABLED` | `ITEM_CC_ENABLED` |
| `ITEM_CC_DISABLED` | `ITEM_CC_DISABLED` |
| `ITEM_NOT_FOUND` | `ITEM_NOT_FOUND` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `decide_item_cycle_count_enabled` | `item_cc_enabled_route` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `ITEM_CC_ENABLED` | `submit_create_abc_classification_set` |
| `ITEM_CC_DISABLED` | `confirm_enable_item_and_run_jobs` |
| `ITEM_NOT_FOUND` | `finish_item_not_found` |

### 24. HITL: confirm item flag update and ESS jobs

Step type: Input Message (HITL)  
Step name: `confirm_enable_item_and_run_jobs`

Input Descriptors:

| Name | Type |
|---|---|
| `item_number` | String |
| `organization_code` | String |
| `cycle_count_name` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `user_item_update_decision` | String |

Message Template:

```text
Item {{ item_number }} is not currently cycle-count enabled in organization {{ organization_code }}.

Do you approve enabling CycleCountEnabledFlag for the item and then running the ABC/cycle count synchronization jobs for cycle count {{ cycle_count_name }}?

Reply with exactly one word:
APPROVE
or
REJECT
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `item_number` | `item_number` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |
| `start_cycle_count_request` | `cycle_count_name` | `cycle_count_name` |

### 25. Branch: item update approval

Step type: Branching  
Step name: `branch_item_update_approval`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `APPROVE` | `APPROVE` |
| `REJECT` | `REJECT` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `confirm_enable_item_and_run_jobs` | `user_item_update_decision` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `APPROVE` | `update_item_cycle_count_flag` |
| `REJECT` | `finish_user_rejected` |

### 26. PATCH item CycleCountEnabledFlag

Step type: Tool Execution  
Step name: `update_item_cycle_count_flag`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `item_unique_id` | String |
| `item_update_payload` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `update_item_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `PATCH` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/itemsV2/{item_unique_id}` |
| data/body | `{item_update_payload}` |

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `decide_item_cycle_count_enabled` | `item_unique_id` | `item_unique_id` |
| `decide_item_cycle_count_enabled` | `item_update_payload` | `item_update_payload` |

Connection: go next to `submit_create_abc_classification_set`.

## ESS job path

The workflow reaches this path in two situations:

1. The item was not in the cycle count, but the item was already cycle-count enabled.
2. The item was not in the cycle count, the item was not cycle-count enabled, the user approved, and the workflow updated the item flag.

### ESS setup values you must confirm first

The Excel screenshots show:

Create ABC Classification Set parameters:

| Parameter | Example |
|---|---|
| Organization | `NEW_PLANT` |
| ABC Classification Set | `ABC_CLASS_SET` |

Perform ABC Assignments and Synchronize Cycle Counts parameters:

| Parameter | Example |
|---|---|
| Organization | `NEW_PLANT` |
| Assignment Group Name | `G1` |
| Synchronize Cycle Counts | `Yes` |

Before configuring the tool payloads, confirm the exact REST submission format used by your Fusion instance. Common Fusion REST implementations submit ESS jobs through:

```text
POST {base_url}/fscmRestApi/resources/11.13.18.05/erpintegrations
```

with an operation like `submitESSJobRequest`. Your environment must provide the exact `JobPackageName`, `JobDefName`, and `ESSParameters` format.

### 27. Submit Create ABC Classification Set

Step type: Tool Execution  
Step name: `submit_create_abc_classification_set`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `organization_code` | String |
| `abc_classification_set_name` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `create_abc_submit_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `POST` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/erpintegrations` |
| data/body | Use your confirmed ESS payload. It must submit `Create ABC Classification Set` with parameters Organization = `{organization_code}` and ABC Classification Set = `{abc_classification_set_name}`. |

Example body shape, to adjust with real job metadata:

```json
{
  "OperationName": "submitESSJobRequest",
  "JobPackageName": "<CONFIRM_FROM_FUSION>",
  "JobDefName": "<CONFIRM_FROM_FUSION>",
  "ESSParameters": "{organization_code},{abc_classification_set_name}"
}
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |
| `extract_cycle_count_definition` | `abc_classification_set_name` | `abc_classification_set_name` |

### 28. Extract Create ABC request id

Step type: Prompt Execution  
Step name: `extract_create_abc_request_id`

Input Descriptors:

| Name | Type |
|---|---|
| `create_abc_submit_response` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `create_abc_request_id` | String |

Prompt Template:

```text
Read this ESS submit response:
{{ create_abc_submit_response }}

Set create_abc_request_id to the submitted ESS request id.
If no request id is present, set it to NOT_FOUND.
Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `submit_create_abc_classification_set` | `create_abc_submit_response` | `create_abc_submit_response` |

### 29. GET Create ABC job status

Step type: Tool Execution  
Step name: `get_create_abc_job_status`

This step will be selected as a body step inside the While loop.

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `create_abc_request_id` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `create_abc_status_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `POST` or `GET`, based on your Fusion ESS status API |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/erpintegrations` |
| data/body | Use your confirmed ESS status payload for request `{create_abc_request_id}` |

Example body shape, to adjust with real job metadata:

```json
{
  "OperationName": "getESSJobStatus",
  "ReqstId": "{create_abc_request_id}"
}
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `extract_create_abc_request_id` | `create_abc_request_id` | `create_abc_request_id` |

Step Timing:

| Timing | Value |
|---|---|
| Pause before | `30` seconds |
| Pause after | `0` seconds |

Reason: this prevents the loop from hammering the ESS status API.

### 30. Decide Create ABC job status

Step type: Prompt Execution  
Step name: `decide_create_abc_job_status`

This step will also be selected as a body step inside the While loop.

Input Descriptors:

| Name | Type |
|---|---|
| `create_abc_status_response` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `create_abc_done` | Boolean |
| `create_abc_status_route` | String |
| `create_abc_status_text` | String |

Prompt Template:

```text
Read this ESS job status response:
{{ create_abc_status_response }}

Set create_abc_done:
- true if the job is in a terminal status such as SUCCEEDED, SUCCESS, ERROR, FAILED, WARNING, or CANCELLED.
- false if the job is still running, waiting, pending, or processing.

Set create_abc_status_route:
- CREATE_ABC_SUCCEEDED if the terminal status is success/succeeded.
- CREATE_ABC_FAILED if the terminal status is failed/error/cancelled/warning.
- CREATE_ABC_RUNNING if the job is not terminal.

Set create_abc_status_text to the raw status text.
Do not add explanations.
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `get_create_abc_job_status` | `create_abc_status_response` | `create_abc_status_response` |

### 31. While loop: wait for Create ABC success

Step type: Loop  
Step name: `wait_for_create_abc_completion`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `create_abc_request_id` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `create_abc_status_route` | String |
| `create_abc_status_text` | String |

Loop Step Configuration:

| Field | Value |
|---|---|
| Loop Type | `While` |
| Max Iterations | `20` or less if your timeout policy is stricter |
| Body Steps | Select `get_create_abc_job_status` and `decide_create_abc_job_status` |
| Exit Condition Kind | `Boolean output` |
| Source Body Step | `decide_create_abc_job_status` |
| Source Output Descriptor Name | `create_abc_done` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `extract_create_abc_request_id` | `create_abc_request_id` | `create_abc_request_id` |

Reason: your pseudocode explicitly says to wait for the Create ABC Classification Set program to complete successfully.

Important: the boolean `create_abc_done` is used internally by the loop. To use the final status outside the loop, keep the separate outputs `create_abc_status_route` and `create_abc_status_text`.

### 32. Branch: Create ABC succeeded?

Step type: Branching  
Step name: `branch_create_abc_success`

Branch Configuration:

| Input variable value | Branch name |
|---|---|
| `CREATE_ABC_SUCCEEDED` | `CREATE_ABC_SUCCEEDED` |
| `CREATE_ABC_FAILED` | `CREATE_ABC_FAILED` |
| `CREATE_ABC_RUNNING` | `CREATE_ABC_RUNNING` |

Data Flow Binding:

| From step | Output | To input |
|---|---|---|
| `wait_for_create_abc_completion` | `create_abc_status_route` | `next_step_name` |

Connections:

| Branch | Next step |
|---|---|
| `CREATE_ABC_SUCCEEDED` | `submit_perform_abc_assignments_sync` |
| `CREATE_ABC_FAILED` | `finish_create_abc_failed` |
| `CREATE_ABC_RUNNING` | `finish_create_abc_timeout` |

Reason: if the first ESS job fails or times out, the second job should not run.

### 33. Submit Perform ABC Assignments and Synchronize Cycle Counts

Step type: Tool Execution  
Step name: `submit_perform_abc_assignments_sync`

Input Descriptors:

| Name | Type |
|---|---|
| `base_url` | String |
| `organization_code` | String |
| `abc_assignment_group_name` | String |

Output Descriptors:

| Name | Type |
|---|---|
| `perform_abc_submit_response` | String |

Tool arguments:

| Argument | Value |
|---|---|
| method | `POST` |
| url | `{base_url}/fscmRestApi/resources/11.13.18.05/erpintegrations` |
| data/body | Use your confirmed ESS payload. It must submit `Perform ABC Assignments and Synchronize Cycle Counts` with Organization = `{organization_code}`, Assignment Group Name = `{abc_assignment_group_name}`, Synchronize Cycle Counts = `Yes`. |

Example body shape, to adjust with real job metadata:

```json
{
  "OperationName": "submitESSJobRequest",
  "JobPackageName": "<CONFIRM_FROM_FUSION>",
  "JobDefName": "<CONFIRM_FROM_FUSION>",
  "ESSParameters": "{organization_code},{abc_assignment_group_name},Yes"
}
```

Bindings:

| From step | Output | To input |
|---|---|---|
| `start_cycle_count_request` | `base_url` | `base_url` |
| `start_cycle_count_request` | `organization_code` | `organization_code` |
| `extract_cycle_count_definition` | `abc_assignment_group_name` | `abc_assignment_group_name` |

Connect to `finish_jobs_submitted`.

## Finish steps

Create these Prompt Execution finish steps. Each one should have only the inputs it needs, one output descriptor named `final_message`, and then connect to End.

### finish_raise_product_sr

Use when the item is already in the CC definition, onhand exists, and the CC subinventory setup already matches/includes the onhand subinventory; or when no onhand exists and Count Zero Quantities is already enabled.

Inputs:

| Name | Type | Binding |
|---|---|---|
| `item_number` | String | `start_cycle_count_request.item_number` |
| `cycle_count_name` | String | `start_cycle_count_request.cycle_count_name` |
| `organization_code` | String | `start_cycle_count_request.organization_code` |

Output:

| Name | Type |
|---|---|
| `final_message` | String |

Prompt:

```text
Write this exact result in final_message:
No setup root cause was found for item {{ item_number }} in cycle count {{ cycle_count_name }} for organization {{ organization_code }}. Suggest the user raise a Product SR.
```

### finish_subinventory_not_found

Inputs:

| Name | Type | Binding |
|---|---|---|
| `onhand_subinventory` | String | `decide_onhand_exists.onhand_subinventory` |
| `cycle_count_name` | String | `start_cycle_count_request.cycle_count_name` |

Prompt:

```text
Write this exact result in final_message:
The onhand subinventory {{ onhand_subinventory }} was not found in the cycle count subinventory setup for {{ cycle_count_name }}. Manual review is required because the provided API can update an existing subinventory record but did not provide a create-subinventory step.
```

### finish_subinventory_updated

Inputs:

| Name | Type | Binding |
|---|---|---|
| `item_number` | String | `start_cycle_count_request.item_number` |
| `onhand_subinventory` | String | `decide_onhand_exists.onhand_subinventory` |

Prompt:

```text
Write this exact result in final_message:
Updated cycle count subinventory {{ onhand_subinventory }} to Include In Schedule for item {{ item_number }}.
```

### finish_manual_count_zero_check

Inputs:

| Name | Type | Binding |
|---|---|---|
| `cycle_count_name` | String | `start_cycle_count_request.cycle_count_name` |

Prompt:

```text
Write this exact result in final_message:
Count Zero Quantities could not be read clearly for cycle count {{ cycle_count_name }}. Manual review is required before making changes.
```

### finish_count_zero_updated

Inputs:

| Name | Type | Binding |
|---|---|---|
| `cycle_count_name` | String | `start_cycle_count_request.cycle_count_name` |

Prompt:

```text
Write this exact result in final_message:
Enabled Count Zero Quantities for cycle count {{ cycle_count_name }} after user approval.
```

### finish_user_rejected

Inputs:

| Name | Type | Binding |
|---|---|---|
| `item_number` | String | `start_cycle_count_request.item_number` |

Prompt:

```text
Write this exact result in final_message:
User rejected the proposed update for item {{ item_number }}. No setup changes were made after the approval step.
```

### finish_item_not_found

Inputs:

| Name | Type | Binding |
|---|---|---|
| `item_number` | String | `start_cycle_count_request.item_number` |
| `organization_code` | String | `start_cycle_count_request.organization_code` |

Prompt:

```text
Write this exact result in final_message:
Item {{ item_number }} was not found in organization {{ organization_code }}. Please verify the item and organization before running again.
```

### finish_create_abc_failed

Inputs:

| Name | Type | Binding |
|---|---|---|
| `create_abc_status_text` | String | `wait_for_create_abc_completion.create_abc_status_text` |

Prompt:

```text
Write this exact result in final_message:
Create ABC Classification Set did not complete successfully. Final ESS status: {{ create_abc_status_text }}. The synchronization job was not submitted.
```

### finish_create_abc_timeout

Inputs:

| Name | Type | Binding |
|---|---|---|
| `create_abc_status_text` | String | `wait_for_create_abc_completion.create_abc_status_text` |

Prompt:

```text
Write this exact result in final_message:
Create ABC Classification Set did not reach a terminal success status before the workflow polling limit. Last known status: {{ create_abc_status_text }}. Check the ESS request manually before running synchronization.
```

### finish_jobs_submitted

Inputs:

| Name | Type | Binding |
|---|---|---|
| `item_number` | String | `start_cycle_count_request.item_number` |
| `cycle_count_name` | String | `start_cycle_count_request.cycle_count_name` |
| `perform_abc_submit_response` | String | `submit_perform_abc_assignments_sync.perform_abc_submit_response` |

Prompt:

```text
Write this exact result in final_message:
Submitted Perform ABC Assignments and Synchronize Cycle Counts for item {{ item_number }} and cycle count {{ cycle_count_name }}. Submit response: {{ perform_abc_submit_response }}
```

## Control connections to draw on the canvas

Draw these arrows:

| From | To |
|---|---|
| Start | `get_cycle_count_definition` |
| `get_cycle_count_definition` | `extract_cycle_count_definition` |
| `extract_cycle_count_definition` | `branch_definition_found` |
| `branch_definition_found` `DEFINITION_FOUND` | `get_cycle_count_item` |
| `branch_definition_found` `DEFINITION_NOT_FOUND` | `finish_definition_not_found` |
| `get_cycle_count_item` | `decide_item_in_cycle_count` |
| `decide_item_in_cycle_count` | `branch_item_in_cycle_count` |
| `branch_item_in_cycle_count` `ITEM_IN_CC` | `get_item_onhand` |
| `branch_item_in_cycle_count` `ITEM_NOT_IN_CC` | `get_item_attributes` |
| `get_item_onhand` | `decide_onhand_exists` |
| `decide_onhand_exists` | `branch_onhand_exists` |
| `branch_onhand_exists` `ONHAND_EXISTS` | `get_cycle_count_subinventory` |
| `branch_onhand_exists` `NO_ONHAND` | `route_count_zero_flag` |
| `get_cycle_count_subinventory` | `extract_subinventory_setup` |
| `extract_subinventory_setup` | `branch_subinventory_in_schedule` |
| `branch_subinventory_in_schedule` `SUBINV_INCLUDED` | `finish_raise_product_sr` |
| `branch_subinventory_in_schedule` `SUBINV_NOT_INCLUDED` | `update_cycle_count_subinventory` |
| `branch_subinventory_in_schedule` `SUBINV_NOT_FOUND` | `finish_subinventory_not_found` |
| `update_cycle_count_subinventory` | `finish_subinventory_updated` |
| `route_count_zero_flag` | `branch_count_zero_enabled` |
| `branch_count_zero_enabled` `COUNT_ZERO_ENABLED` | `finish_raise_product_sr` |
| `branch_count_zero_enabled` `COUNT_ZERO_DISABLED` | `confirm_enable_count_zero` |
| `branch_count_zero_enabled` `COUNT_ZERO_UNKNOWN` | `finish_manual_count_zero_check` |
| `confirm_enable_count_zero` | `branch_count_zero_approval` |
| `branch_count_zero_approval` `APPROVE` | `build_count_zero_update_payload` |
| `branch_count_zero_approval` `REJECT` | `finish_user_rejected` |
| `build_count_zero_update_payload` | `update_count_zero_quantities` |
| `update_count_zero_quantities` | `finish_count_zero_updated` |
| `get_item_attributes` | `decide_item_cycle_count_enabled` |
| `decide_item_cycle_count_enabled` | `branch_item_cycle_count_enabled` |
| `branch_item_cycle_count_enabled` `ITEM_CC_ENABLED` | `submit_create_abc_classification_set` |
| `branch_item_cycle_count_enabled` `ITEM_CC_DISABLED` | `confirm_enable_item_and_run_jobs` |
| `branch_item_cycle_count_enabled` `ITEM_NOT_FOUND` | `finish_item_not_found` |
| `confirm_enable_item_and_run_jobs` | `branch_item_update_approval` |
| `branch_item_update_approval` `APPROVE` | `update_item_cycle_count_flag` |
| `branch_item_update_approval` `REJECT` | `finish_user_rejected` |
| `update_item_cycle_count_flag` | `submit_create_abc_classification_set` |
| `submit_create_abc_classification_set` | `extract_create_abc_request_id` |
| `extract_create_abc_request_id` | `wait_for_create_abc_completion` |
| `wait_for_create_abc_completion` | `branch_create_abc_success` |
| `branch_create_abc_success` `CREATE_ABC_SUCCEEDED` | `submit_perform_abc_assignments_sync` |
| `branch_create_abc_success` `CREATE_ABC_FAILED` | `finish_create_abc_failed` |
| `branch_create_abc_success` `CREATE_ABC_RUNNING` | `finish_create_abc_timeout` |
| `submit_perform_abc_assignments_sync` | `finish_jobs_submitted` |
| every finish step | End |

## Testing checklist

Run tests in this order:

1. Happy path: item not in CC, item already CC enabled.
   - Expected: submits Create ABC job, waits for success, submits Perform ABC Assignments and Synchronize Cycle Counts.

2. Item not in CC, item not CC enabled, user rejects.
   - Expected: no PATCH, no ESS job.

3. Item not in CC, item not CC enabled, user approves.
   - Expected: PATCH itemsV2 with `{"CycleCountEnabledFlag": true}`, then ESS path.

4. Item in CC, has onhand, subinventory already included.
   - Expected: final message says raise Product SR.

5. Item in CC, has onhand, subinventory not included.
   - Expected: PATCH countSubinventories Include In Schedule true.

6. Item in CC, no onhand, Count Zero enabled.
   - Expected: final message says raise Product SR.

7. Item in CC, no onhand, Count Zero disabled, user approves.
   - Expected: PATCH cycleCountDefinitions Count Zero Quantities true.

8. Create ABC job fails.
   - Expected: second ESS job is not submitted.

## Common mistakes to avoid

1. Do not paste `{{ item_number }}` into a Tool URL.
   - Use `{item_number}` in Tool argument fields.

2. Do not forget Data Flow Binding.
   - The canvas arrow is not enough.

3. Do not branch directly on raw API JSON.
   - Always use a Prompt Execution step to produce a clean branch label.

4. Do not let a HITL response be free-form.
   - Tell the user to reply exactly `APPROVE` or `REJECT`.

5. Do not run the second ESS job before the first job completes successfully.

6. Do not hard-code the example unique IDs from Excel.
   - IDs such as `300000336573683` and the long item unique ID are examples.
   - Always extract IDs from the GET responses in the workflow.

7. Do not use the visible typo `countsubinventoryentories`.
   - Use `countSubinventories`.

