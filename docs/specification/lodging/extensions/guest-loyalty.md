# Lodging Guest Loyalty Extension

**Capability name:** `dev.ucp.lodging.guest_loyalty`
**Extends:** `dev.ucp.lodging.booking`
**Status:** Draft proposal (not accepted by the Lodging Technical Council)
**Schema:** `https://ucp.dev/draft/schemas/lodging/guest_loyalty.json`

> **Drafting notes (remove before submission).**
> - The structure follows the extension pattern used by `dev.ucp.lodging.policy.cancellation` and `dev.ucp.common.loyalty`: additions live in `$defs["dev.ucp.lodging.booking"]` and compose onto the base with `allOf`.
> - The base schema URL `https://ucp.dev/draft/schemas/lodging/booking.json` is **assumed**. I could not read the lodging booking schema, so I have not confirmed its file name, that `guests[]` items accept additional properties, or the exact operation names used by `ucp_request`.
> - The capability `version` value in the profile example is a placeholder.

## Overview

Hotel loyalty programs credit points to the person who stays, and usually only when the membership number is attached to the reservation before check-in. Identity Linking ([OAuth 2.0](https://ucp.dev/draft/specification/common/identity-linking/)) is the preferred way for a Business to learn a buyer's membership, but many properties do not support it. This extension gives those Businesses a narrow, consent-gated way to receive a membership number with the primary guest's details.

The extension is deliberately **non-gating and non-pricing**. A missing, wrong, or unverifiable number never changes booking `status`, never blocks `ready_for_complete`, and never changes the price. Price-impacting member rates remain the job of `dev.ucp.common.loyalty` claims.

### Design principles

1. **Business-initiated.** The Platform sends a number only after the Business asks for one.
2. **Request-only.** The number goes Platform → Business and is never returned, not even masked.
3. **Primary guest only.** The number belongs to the guest with `role: "primary"` on the stay, not to the `booker`.
4. **Consent-gated.** The buyer supplies the number and agrees to share it with the Business.
5. **Identity Linking wins.** If membership was resolved through Identity Linking, the Platform does not also send a number.

## Capability declaration

A Business declares the extension in its profile, extending the Lodging Booking capability:

<!-- validate: profile -->
```json
{
  "ucp": {
    "version": "2026-09-25",
    "capabilities": {
      "dev.ucp.lodging.booking": [
        {
          "version": "2026-09-25",
          "spec": "https://ucp.dev/draft/specification/lodging/booking/",
          "schema": "https://ucp.dev/draft/schemas/lodging/booking.json"
        }
      ],
      "dev.ucp.lodging.guest_loyalty": [
        {
          "version": "2026-09-25",
          "spec": "https://ucp.dev/draft/specification/lodging/extensions/guest-loyalty/",
          "schema": "https://ucp.dev/draft/schemas/lodging/guest_loyalty.json",
          "extends": ["dev.ucp.lodging.booking"]
        }
      ]
    }
  }
}
```

A Platform MUST NOT send `loyalty_memberships` to a Business that has not declared this capability.

## Schema

| Field | Where | Direction | Description |
|---|---|---|---|
| `guests[].loyalty_memberships[]` | Guest | Request, Update only | Membership numbers for the guest. Max 5. |
| `guests[].loyalty_memberships[].program` | | Request | Reverse-domain program key, one of `accepted_programs[].program`. |
| `guests[].loyalty_memberships[].member_id` | | Request | Number as entered by the buyer. 1–64 characters. |
| `guest_loyalty.accepted_programs[]` | Booking | Response | Programs the Business can credit (`program`, `name`). |
| `guest_loyalty.memberships[]` | Booking | Response | Outcome per submitted number (`guest_id`, `program`, `recorded`, `provisional`). |

`guest_loyalty.memberships[]` never contains `member_id` or a masked form of it. `display_id` is not defined by this extension, because `dev.ucp.common.loyalty` forbids it for unverified memberships and a lodging Business typically cannot verify a number until its property management system matches it at check-in.

`loyalty_memberships` is omitted from Create and Complete requests. Complete Booking Session accepts only `signals` and `payment`, and a completed booking is immutable, so the number has to arrive on an Update.

## Messages

All messages use the standard UCP message envelope. None of them uses `type: "error"`, because none may affect booking status.

| Code | Type | Emitted by | `path` | Meaning |
|---|---|---|---|---|
| `loyalty_member_id_requested` | `info` | Business | `$.stays[n]` | The Business can credit a loyalty number for this stay. |
| `loyalty_member_id_invalid_format` | `warning` (`presentation: notice`) | Business | `$.guest_loyalty.memberships[n]` | The number was malformed; it was **not** recorded. |
| `loyalty_program_not_accepted` | `warning` (`presentation: notice`) | Business | `$.guest_loyalty.memberships[n]` | The `program` is not in `accepted_programs`; the number was **not** recorded. |

Warnings point at `guest_loyalty.memberships[n]` rather than at `guests[n].loyalty_memberships[m]` because the Business never echoes the request field, so the latter path would not resolve in the response.

## Behavior

### Timing

1. **Request.** At any non-terminal status the Business MAY emit `loyalty_member_id_requested` with `path` `$.stays[n]` and populate `guest_loyalty.accepted_programs`. The path targets the stay because no guest object exists yet when the Business first asks. The Business MUST NOT re-emit the message after it has received a number for that stay, or after the Platform has indicated the buyer declined.
2. **Send.** The Platform MAY include `loyalty_memberships` in an Update only when all of the following hold:
   - the Business has requested a number for that stay;
   - the buyer supplied the number and agreed to share it with the Business;
   - the guest carrying the number has `role: "primary"` in `stays[n].guest_assignments`;
   - the `program` key appears in `accepted_programs`.
3. **Deadline.** The number MUST be in the last Update before Complete. After Complete there is no API path to add or correct it.
4. **Replay.** Update replaces the resource and the Business never returns the number. The Platform MUST retain the number for the life of the session and MUST resend it on every subsequent Update that should keep it. A Business MAY discard any number not present in the latest Update.
5. **Declined or absent.** If the buyer declines or has no number, the Platform sends nothing. This is not an error condition and requires no message.

### Business processing

- The Business MUST NOT change `status`, totals, or availability because of a membership number.
- The Business SHOULD validate syntax at Update time and, on failure, return `loyalty_member_id_invalid_format` with `recorded: false`. Early feedback is the only chance to fix a typo, because the booking cannot be amended after Complete.
- The Business MUST return `provisional: true` unless it has verified the membership out of band (for example through Identity Linking), and MUST NOT return `display_id`.
- If the same program is submitted twice, the Business SHOULD record the latest value only.

### Multiple memberships

A guest may hold numbers in several programs, for example a hotel brand program and an OTA program. The Platform MAY send up to 5 entries, one per `program`. The Business credits only programs it lists in `accepted_programs` and ignores the rest with `loyalty_program_not_accepted`. Which program earns when several are valid is a Business policy and is out of scope.

### Interaction with other capabilities

- **Identity Linking.** If the buyer has linked an account that resolves membership, the Platform SHOULD NOT send `loyalty_memberships`. If both arrive, the Business SHOULD prefer the linked identity.
- **`dev.ucp.common.loyalty`.** Claims in `context.eligibility` still govern price eligibility. A number sent through this extension MUST NOT be treated as an eligibility claim.

## Privacy and security

- The Platform MUST NOT send a number the buyer did not supply and agree to share, and MUST NOT source one from another context (for example a different merchant's profile).
- The Business MUST treat `member_id` as a sensitive identifier: no echo in responses, no clear-text logging, no use beyond crediting the stay.
- Platforms SHOULD show the buyer the program name from `accepted_programs[].name` when collecting the number, so the buyer knows who receives it.
- Loyalty programs typically match on the guest's legal name. The Platform SHOULD send the primary guest's name exactly as entered on the booking.

## Examples

### 1. Business requests a number (Create response)

The Business advertises one program and asks for a number. Status is untouched.

<!-- validate: response -->
```json
{
  "status": "incomplete",
  "guest_loyalty": {
    "accepted_programs": [
      { "program": "com.example.honors", "name": "Example Honors" }
    ]
  },
  "messages": [
    {
      "type": "info",
      "code": "loyalty_member_id_requested",
      "path": "$.stays[0]",
      "content": "Add your Example Honors number to earn points on this stay."
    }
  ]
}
```

### 2. Platform sends the number (Update request)

The buyer entered a number and agreed to share it. It rides on the primary guest.

<!-- validate: request -->
```json
{
  "guests": [
    {
      "id": "gst_01",
      "first_name": "Jane",
      "last_name": "Doe",
      "email": "jane.doe@example.com",
      "loyalty_memberships": [
        { "program": "com.example.honors", "member_id": "123456789" }
      ]
    }
  ],
  "stays": [
    {
      "guest_assignments": [
        { "guest_id": "gst_01", "role": "primary" }
      ]
    }
  ]
}
```

### 3. Business accepts it provisionally (Update response)

The booking is ready. The number is not returned, and nothing is marked verified.

<!-- validate: response -->
```json
{
  "status": "ready_for_complete",
  "guest_loyalty": {
    "accepted_programs": [
      { "program": "com.example.honors", "name": "Example Honors" }
    ],
    "memberships": [
      {
        "guest_id": "gst_01",
        "program": "com.example.honors",
        "recorded": true,
        "provisional": true
      }
    ]
  }
}
```

### 4. Later Update must resend the number

The buyer changes the guest's email. Because Update replaces the resource and the number was never echoed, the Platform resends it.

<!-- validate: request -->
```json
{
  "guests": [
    {
      "id": "gst_01",
      "first_name": "Jane",
      "last_name": "Doe",
      "email": "jane.d@example.com",
      "loyalty_memberships": [
        { "program": "com.example.honors", "member_id": "123456789" }
      ]
    }
  ],
  "stays": [
    {
      "guest_assignments": [
        { "guest_id": "gst_01", "role": "primary" }
      ]
    }
  ]
}
```

### 5. Malformed number: warn, do not block

The Business rejects the number but the booking still reaches `ready_for_complete`. The Platform can ask the buyer to correct it and send another Update before completing.

<!-- validate: response -->
```json
{
  "status": "ready_for_complete",
  "guest_loyalty": {
    "accepted_programs": [
      { "program": "com.example.honors", "name": "Example Honors" }
    ],
    "memberships": [
      {
        "guest_id": "gst_01",
        "program": "com.example.honors",
        "recorded": false,
        "provisional": true
      }
    ]
  },
  "messages": [
    {
      "type": "warning",
      "code": "loyalty_member_id_invalid_format",
      "path": "$.guest_loyalty.memberships[0]",
      "content": "That Example Honors number doesn't look right. Check it and try again, or continue without it.",
      "presentation": "notice"
    }
  ]
}
```

### 6. Program not accepted

The Platform sent a number for a program the property cannot credit, such as an OTA program.

<!-- validate: response -->
```json
{
  "status": "ready_for_complete",
  "guest_loyalty": {
    "accepted_programs": [
      { "program": "com.example.honors", "name": "Example Honors" }
    ],
    "memberships": [
      {
        "guest_id": "gst_01",
        "program": "com.example.otarewards",
        "recorded": false,
        "provisional": true
      }
    ]
  },
  "messages": [
    {
      "type": "warning",
      "code": "loyalty_program_not_accepted",
      "path": "$.guest_loyalty.memberships[0]",
      "content": "This property can't credit that program for this stay.",
      "presentation": "notice"
    }
  ]
}
```

### 7. Business never asks

A Business that does not emit `loyalty_member_id_requested` receives no numbers. The Platform's Update contains no `loyalty_memberships`, and the response has no `guest_loyalty` block. This is intentional: the Platform cannot volunteer a number.

### 8. Invalid: extra field on a membership

`tier` is not part of the request object. A schema-validating Business rejects this payload.

<!-- validate: invalid-request -->
```json
{
  "guests": [
    {
      "id": "gst_01",
      "loyalty_memberships": [
        { "program": "com.example.honors", "member_id": "123456789", "tier": "gold" }
      ]
    }
  ]
}
```

## Known limitations and open questions

- **No post-completion correction.** Completed bookings are immutable, so a mistyped number that passes format checks cannot be fixed through this API. A separate post-completion amendment capability would be needed.
- **Business-initiated only.** A Platform cannot offer the field on its own initiative. Businesses that want numbers must emit the request message.
- **Verification.** Verification at capture is out of scope; Lodging Technical Council members who own loyalty programs may prefer extending Identity Linking over carrying raw numbers.
- **Name matching.** Programs typically require the guest's name to match the member record. This extension does not define mismatch handling.
