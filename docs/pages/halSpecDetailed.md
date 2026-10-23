# GPON JSON HAL Detailed Reference

See the [overview](halSpec.md). RdkGponManager consumes `gpon_hal_schema.json` in the standard configuration or `gpon_wan_unify_hal_schema.json` in the WAN-unified configuration. Each selected schema maps to RdkGponManager; do not merge the variants. Locations below are JSON Pointers within the selected schema.

## Envelope and Transport
Both schemas declare JSON Schema draft-07. Required root fields:
| Field | Contract |
| --- | --- |
| `module` | String constant `gponhal`. |
| `version` | String constant `0.0.1`; does not distinguish variants. |
| `action` | One of the eleven case-sensitive action strings below. |
| `reqId` | String matching `^[0-9]+$`, not a JSON number. Preserve request correlation. |

The shared JSON HAL library supplies IPv4 TCP transport; configured port is `40100`. The server implementation uses loopback. This is not HTTP or JSON-RPC 2.0. The library does not establish reliable stream framing: serialized JSON has no explicit length prefix/delimiter, and receive boundaries are not JSON message boundaries. Confirm implementation compatibility before testing fragmentation or large payloads.

## Action API Reference
| Action | Direction, payload, and interpretation |
| --- | --- |
| `getSchema` | Manager to vendor. Definition describes schema retrieval without parameters. No action-specific root condition prohibits extra fields. |
| `getSchemaResponse` | Vendor to manager. Required `SchemaInfo` object containing required string `FilePath`, pattern `^(.+)/([^/]+)$`; no other properties inside `SchemaInfo`. A path is not schema contents or a downloadable URL. |
| `getParameters` | Manager to vendor. Required nonempty unique `params` array. Each item requires `name` and must match a supported/optional read definition. |
| `getParametersResponse` | Vendor to manager. Same read membership; each item requires `name`, `type`, and `value`. Validate the actual value constraint, not only the textual type. |
| `setParameters` | Manager to vendor. Required nonempty unique `params` array. Each item requires `name`, `type`, and `value` and must match a write-list definition. Multi-parameter atomicity is unspecified. |
| `subscribeEvent` | Manager to vendor. Required nonempty unique `params` array. Items require `name` and `notificationType` and must match the event list. Some listed definitions reject `notificationType`; see defects below. |
| `publishEvent` | Vendor to manager. Required nonempty unique `params` array. Items require `name`, `type`, and `value` and match the same event list. Event ordering, delivery reliability, and replay are unspecified. |
| `deleteObject` | Manager to vendor. Root requires a nonempty unique `params` array and items require `name`, but the definition contains invalid empty `anyOf`. No usable deletion target is specified. |
| `getActiveSubscriptions` | Enumerated action only; no action-conditioned payload definition. Do not infer support, request shape, or response semantics. |
| `getActiveSubscriptionsResponse` | Enumerated action only; no action-conditioned payload definition. |
| `result` | Vendor result message. Required `Result` object containing required `Status`; no extra properties inside `Result`. |

`Result.Status` is one of `Success`, `Failed`, `Invalid Argument`, or `Not Supported`. The schema default does not synthesize a missing required field. These strings are distinct from C library `RETURN_OK`/`RETURN_ERR`.

`uniqueItems` compares complete array elements, not merely parameter names. Do not assume it prohibits duplicate names with different values. The root does not impose a universal `additionalProperties: false`; many parameter definitions do. Apply restrictions at their actual schema location.

## Complete Parameter-Contract Lookup
Every parameter contract is defined under `/definitions/{definition-key}` in the selected schema. The following action indexes enumerate the authoritative entries and are the entry points for every supported/optional API parameter:

| Action index | Standard alternatives | WAN-unified alternatives |
| --- | ---: | ---: |
| `/definitions/getParameterSupportedList/anyOf` | 95 | 100 |
| `/definitions/getParameterOptionalList/anyOf` | 24 | 24 |
| `/definitions/setParameterSupportedList/anyOf` | 8 | 11 |
| `/definitions/setParameterOptionalList/anyOf` | 0 | 0 |
| `/definitions/subscribeEventSupportedList/anyOf` | 18 | 18 |

Counts are references/alternatives, not unique hardware capabilities. Standard/WAN-unified schemas contain 141/146 definitions respectively, including envelope, list, enum, and parameter definitions.

For **each** referenced parameter:
1. Follow its `$ref` into `/definitions`.
2. Read `description` for operation semantics, units, and descriptive access annotations.
3. Use `properties.name.pattern` as the exact path grammar, including instance indices. Do not replace a regex with an invented fixed path.
4. Use `properties.type.const` as the textual HAL type and independently enforce `properties.value`'s actual JSON type, enum, bounds, pattern, and length. Resolve all nested `$ref` values.
5. Check membership in the relevant action list. A read description or type does not grant write permission.
6. Apply action-level `required` together with leaf-level `additionalProperties` and constraints. A leaf's property declaration alone does not make the field required.
7. Use only contract-backed units, completion criteria, persistence, and failure behavior; do not infer guarantees from TR-181 conventions.

This lookup is intentionally schema-qualified: the two variants are not merged, and referenced constraints remain authoritative rather than being weakened by a prose summary.

## Event Parameter Inventory
Both variants reference these exact definitions:
- `ontPhysicalMediaStatus`: optical-interface operational status.
- `ontPhysicalMediaAlarmRdi`: remote defect indication.
- `ontPhysicalMediaAlarmPee`, `ontPhysicalMediaAlarmLos`, `ontPhysicalMediaAlarmLof`.
- `ontPhysicalMediaAlarmDact`, `ontPhysicalMediaAlarmDis`, `ontPhysicalMediaAlarmMis`.
- `ontPhysicalMediaAlarmMem`, `ontPhysicalMediaAlarmSuf`, `ontPhysicalMediaAlarmSd`.
- `ontPhysicalMediaAlarmSf`, `ontPhysicalMediaAlarmLcdg`, `ontPhysicalMediaAlarmTf`, `ontPhysicalMediaAlarmRogue`.
- `ontPloamRegistrationState`: PLOAM registration.
- `ontVeipAdministrativeState`, `ontVeipOperationalState`: VEIP administration and observation.

Use each definition's description and enum reference for the alarm meaning and value set. Similar names do not imply identical alarm semantics or a normative state machine.

## Minimal Envelope Example
This demonstrates message shape, not proof of server action support:
```json
{"module":"gponhal","version":"0.0.1","action":"getSchema","reqId":"1"}
```

## Verified Schema Defects and Test Boundaries
- Both variants contain `/definitions/deleteObject/anyOf: []`.
- Both contain `/definitions/setParameterOptionalList/anyOf: []`. Draft-07 requires nonempty combinator arrays; strict schema checking can reject the entire document before instance validation.
- Some event alternatives, including `ontPhysicalMediaAlarmRdi`, disallow additional properties but do not declare `notificationType`, although `subscribeEvent` requires it. Such alternatives cannot satisfy a subscription as written.
- Active-subscription action names lack conditioned payload contracts.
- No unresolved internal definition references were found in the inspected variants.

These are documentation findings, not silent schema repairs. Obtain approved contract corrections before treating affected cases as normative vendor tests. Schema validity, runtime handler registration, optical-link state, request transport, and hardware completion must be evaluated separately.

## Runtime and Recovery Limits
Synchronous library timeout does not cancel a remote operation. Restore altered provisioning state using an approved product procedure.

## Source References
- [Standard schema](https://github.com/rdkcentral/gpon-manager/blob/main/hal_schema/gpon_hal_schema.json)
- [WAN-unified schema](https://github.com/rdkcentral/gpon-manager/blob/main/hal_schema/gpon_wan_unify_hal_schema.json)
- [Standard configuration](https://github.com/rdkcentral/gpon-manager/blob/main/config/gpon_manager_conf.json)
- [WAN-unified configuration](https://github.com/rdkcentral/gpon-manager/blob/main/config/gpon_manager_wan_unify_conf.json)
