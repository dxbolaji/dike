# Diké API Contract v1

**Status:** Contract baseline for implementation  
**Base path:** `/api/v1`  
**Format:** JSON, UTF-8, `snake_case` keys  
**IDs:** UUID strings  
**Timestamps:** ISO 8601 UTC  
**Authentication:** Bearer access token on all private endpoints

This document explains the OpenAPI contract in `openapi.yaml`. The YAML file is the machine-readable contract and must be treated as the source of truth for routes and wire formats. The examples here are illustrative only.

## 1. Shared conventions

### Authentication
Private endpoints require:

```http
Authorization: Bearer <access_token>
```

The backend validates the token server-side and resolves it to an internal user. The client must never send an authoritative `owner_id`; ownership is derived from the validated identity.

`GET /health` is public and returns only a minimal service status.

### Common error envelope

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request could not be processed.",
    "details": [
      {"field": "state", "issue": "Required"}
    ],
    "request_id": "f5e2d5b2-..."
  }
}
```

Use HTTP status codes consistently:
- `400` malformed request or invalid state transition
- `401` missing/invalid authentication
- `403` authenticated but disallowed action
- `404` not found or not owned; do not disclose another user's resource
- `409` conflicting state
- `413` request/file too large
- `415` unsupported media type
- `422` schema validation failure
- `429` rate limit
- `500` unexpected server error

Do not return stack traces, secrets, SQL details or provider error bodies to clients.

### Enumerations
- `issue_category`: `landlord_tenant`, `employment_unpaid_wages`, `contract_consumer`, `other`, `unknown`
- `case_status`: `draft`, `submitted`, `analysis_pending`, `analysis_ready`, `needs_clarification`, `archived`
- `verification_status`: `verified`, `unverified`, `stale`, `demo_only`, `rejected`
- `eligibility_result`: `potentially_eligible`, `criteria_not_met`, `insufficient_information`, `unavailable`
- `urgency`: `normal`, `high`, `urgent`

## 2. Endpoint map

| Method | Path | Auth | Purpose |
|---|---|---:|---|
| GET | `/health` | No | Health check |
| POST | `/api/v1/cases` | Yes | Create a case |
| GET | `/api/v1/cases` | Yes | List current user's cases |
| GET | `/api/v1/cases/{case_id}` | Yes | Retrieve one owned case |
| POST | `/api/v1/cases/{case_id}/analyze` | Yes | Run synchronous analysis for MVP |
| GET | `/api/v1/cases/{case_id}/analysis` | Yes | Retrieve latest analysis and citations |
| GET | `/api/v1/legal-aid` | Yes | Search assistance directory |
| POST | `/api/v1/eligibility/check` | Yes | Preliminary criteria check, only with verified criteria |
| POST | `/api/v1/cases/{case_id}/summary` | Yes | Generate factual case summary |
| PATCH | `/api/v1/cases/{case_id}/summary` | Yes | Save user's edited summary |
| POST | `/api/v1/documents` | Yes, P1 | Upload a private document |
| GET | `/api/v1/cases/{case_id}/documents` | Yes, P1 | List owned case documents |
| GET | `/api/v1/documents/{document_id}/download` | Yes, P1 | Obtain short-lived signed download URL |

## 3. Case creation

`POST /api/v1/cases`

Required:
- `title`: 1–160 characters
- `narrative`: 20–10,000 characters
- `country_code`: ISO-style two-letter code; initial supported value is `NG`

Optional:
- `issue_category`: one of the values above; use `unknown` when unsure
- `state`: state/province name; nullable when not known
- `relevant_location`: free-text city/LGA/neighbourhood; nullable
- `event_date`: `YYYY-MM-DD`; nullable

The endpoint persists the intake. It does not have to run full legal analysis. The backend sets `owner_id`, `status`, `created_at` and `updated_at`.

Illustrative response:

```json
{
  "id": "UUID",
  "title": "Unpaid salary dispute",
  "narrative": "My employer has not paid me for two months.",
  "issue_category": "employment_unpaid_wages",
  "country_code": "NG",
  "state": "Lagos",
  "relevant_location": "Ikeja",
  "event_date": null,
  "status": "draft",
  "created_at": "2026-11-09T12:00:00Z",
  "updated_at": "2026-11-09T12:00:00Z"
}
```

## 4. List and retrieve cases

`GET /api/v1/cases?page=1&page_size=20`

- `page` defaults to 1; minimum 1.
- `page_size` defaults to 20; maximum 100.
- Return only records owned by the authenticated user.
- List response omits the full narrative.

`GET /api/v1/cases/{case_id}` returns the full owned case record.

## 5. Run and retrieve analysis

`POST /api/v1/cases/{case_id}/analyze`

For the initial MVP this is a synchronous request-response operation. Do not introduce background jobs/polling without updating the contract first.

Request:

```json
{
  "additional_answers": [
    {"question_key": "work_location", "answer": "Ikeja, Lagos"}
  ]
}
```

`additional_answers` may be an empty array. Each answer is user-supplied. The backend must not silently turn model-generated guesses into facts.

Response includes:
- `analysis_id`, `case_id`, `status`
- provisional `issue_classification` and confidence
- `jurisdiction` with basis, confidence, clarification flag and questions
- plain-language `summary`
- `claims`, each linked to source IDs
- `sources`, with authority, jurisdiction, URL, provision and verification metadata
- `next_steps`, `uncertainties` and `urgent_help`

The backend must validate that every `source_id` in a claim exists in the sources retrieved for that analysis. If the jurisdiction needs clarification or no adequate source exists, disclose that and do not emit unsupported legal propositions.

`GET /api/v1/cases/{case_id}/analysis` returns the latest analysis for that owned case, or `404` if none exists.

## 6. Legal assistance directory

`GET /api/v1/legal-aid`

Optional query parameters: `state`, `issue_category`, `assistance_type`, `page`, `page_size`.

Each result includes:
- organisation name and description
- service areas and geographic coverage
- assistance types
- contact details, nullable if not verified
- `source_url`
- `verification_status`
- `last_verified_at`

Only records with valid provenance may be marked `verified`. `demo_only` records must be visibly labelled as demonstration data in the UI.

## 7. Eligibility

`POST /api/v1/eligibility/check`

Request supplies `organisation_id` and answers keyed to the organisation's published criteria.

The endpoint must return one of:
- `potentially_eligible`
- `criteria_not_met`
- `insufficient_information`
- `unavailable`

Every evaluated criterion must point to its source. If the organisation has no verified, current criteria, return `unavailable`; do not infer eligibility rules.

This is preliminary screening, not an eligibility decision or guarantee.

## 8. Factual case summary

`POST /api/v1/cases/{case_id}/summary`

Request: `{"format":"chronological"}`

The generated summary may use only user-supplied facts or facts explicitly approved by the user. It must preserve uncertainty and must not add legal conclusions. Return `requires_user_review: true`.

`PATCH /api/v1/cases/{case_id}/summary`

Request: `{"content":"user-edited summary"}`

The response returns the updated summary resource.

## 9. Documents (P1 only)

Do not implement uploads for real users until private object storage, file validation, ownership checks, access policies and deletion/retention behaviour are verified.

`POST /api/v1/documents` uses `multipart/form-data` with `case_id` and `file`.

The download endpoint returns a short-lived signed URL only after checking both document and case ownership. Never make case documents public.

## 10. Contract change process

1. Propose the change in a pull request before changing application code.
2. Update `openapi.yaml` and this document in the same pull request.
3. Include an example request/response and explain backward compatibility.
4. Get agreement from both DX and Jibola.
5. Regenerate frontend API types and update backend Pydantic models.
6. Add or update contract tests.
7. Merge the contract change before dependent application changes.

Do not merge a frontend or backend change that silently changes a shared request/response schema.
