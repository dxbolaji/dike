# Diké Database Schema Specification v1

**Database:** PostgreSQL  
**Migrations:** Alembic  
**Primary key:** UUID  
**Time fields:** `timestamptz` stored in UTC  
**JSON:** Use `jsonb` only for flexible structured payloads  
**Ownership:** Every private record must be scoped to the authenticated user

This is the schema baseline. Jibola must implement it as SQLAlchemy models and Alembic migrations, then verify the final database structure against this document. Do not use this document as permission to collect extra personal data.

## 1. Core tables

### `users`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK, generated internally |
| `auth_subject` | text | NOT NULL, UNIQUE; verified identity-provider subject |
| `display_name` | text | NULL |
| `created_at` | timestamptz | NOT NULL |
| `updated_at` | timestamptz | NOT NULL |

Do not store passwords. Resolve `auth_subject` only from a validated bearer token.

### `cases`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `owner_id` | uuid | FK users.id, NOT NULL |
| `title` | varchar(160) | NOT NULL |
| `narrative` | text | NOT NULL |
| `issue_category` | text | NOT NULL, constrained enum/check |
| `country_code` | char(2) | NOT NULL; initially NG |
| `state` | text | NULL |
| `relevant_location` | text | NULL |
| `event_date` | date | NULL |
| `status` | text | NOT NULL, default draft |
| `created_at` | timestamptz | NOT NULL |
| `updated_at` | timestamptz | NOT NULL |

Indexes: `(owner_id, updated_at DESC)`, `(owner_id, status)`. Owner must be assigned server-side.

### `case_facts`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `case_id` | uuid | FK cases.id, NOT NULL |
| `fact_key` | text | NOT NULL |
| `fact_value` | jsonb | NOT NULL |
| `provenance` | text | user_supplied, extracted, user_approved |
| `confidence` | numeric(4,3) | NULL, range 0–1 |
| `review_status` | text | unreviewed, approved, rejected |
| `created_at` | timestamptz | NOT NULL |

Extracted facts are not treated as confirmed until the user approves them.

### `legal_sources`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `title` | text | NOT NULL |
| `issuing_authority` | text | NOT NULL |
| `jurisdiction_country` | char(2) | NOT NULL |
| `jurisdiction_state` | text | NULL |
| `source_url` | text | NOT NULL |
| `publication_date` | date | NULL |
| `effective_from` | date | NULL |
| `effective_to` | date | NULL |
| `verification_status` | text | NOT NULL |
| `last_verified_at` | timestamptz | NULL |
| `content_hash` | text | NULL |
| `created_at` | timestamptz | NOT NULL |
| `updated_at` | timestamptz | NOT NULL |

Do not mark a source verified without human/source verification. Track amendments or replacements as distinct source versions when necessary.

### `legal_chunks`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `source_id` | uuid | FK legal_sources.id, NOT NULL |
| `provision` | text | NULL |
| `chunk_text` | text | NOT NULL |
| `embedding` | vector | NULL until pgvector is configured |
| `metadata` | jsonb | NOT NULL, default `{}` |
| `created_at` | timestamptz | NOT NULL |

Every chunk must map to a parent source and preserve provision/page metadata where available. Do not enable vector search until the pgvector extension and embedding dimension are fixed.

### `analyses`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `case_id` | uuid | FK cases.id, NOT NULL |
| `status` | text | pending, completed, failed, needs_clarification |
| `issue_category` | text | NULL |
| `issue_confidence` | numeric(4,3) | NULL, 0–1 |
| `jurisdiction` | jsonb | NOT NULL |
| `summary` | text | NULL |
| `next_steps` | jsonb | NOT NULL, default `[]` |
| `uncertainties` | jsonb | NOT NULL, default `[]` |
| `urgent_help` | boolean | NOT NULL, default false |
| `model_provider` | text | NULL |
| `model_name` | text | NULL |
| `created_at` | timestamptz | NOT NULL |

Index `(case_id, created_at DESC)`. Persist enough metadata to reproduce which sources informed the analysis.

### `analysis_claims`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `analysis_id` | uuid | FK analyses.id, NOT NULL |
| `position` | integer | NOT NULL |
| `claim_text` | text | NOT NULL |
| `qualification` | text | NULL |

Unique `(analysis_id, position)`.

### `analysis_citations`
| Column | Type | Rules |
|---|---|---|
| `analysis_claim_id` | uuid | FK analysis_claims.id, NOT NULL |
| `legal_source_id` | uuid | FK legal_sources.id, NOT NULL |
| `provision` | text | NULL |

Composite primary key `(analysis_claim_id, legal_source_id)`. The API response assembles claims with their linked sources.

### `organisations`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `name` | text | NOT NULL |
| `description` | text | NULL |
| `service_areas` | jsonb | NOT NULL, default `[]` |
| `coverage` | jsonb | NOT NULL, default `[]` |
| `assistance_types` | jsonb | NOT NULL, default `[]` |
| `website` | text | NULL |
| `email` | text | NULL |
| `phone` | text | NULL |
| `address` | text | NULL |
| `source_url` | text | NOT NULL |
| `verification_status` | text | NOT NULL |
| `last_verified_at` | timestamptz | NULL |
| `created_at` | timestamptz | NOT NULL |
| `updated_at` | timestamptz | NOT NULL |

Null contact fields mean not verified/not available. Do not fabricate contact details.

### `eligibility_programmes`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `organisation_id` | uuid | FK organisations.id, NOT NULL |
| `name` | text | NOT NULL |
| `source_url` | text | NOT NULL |
| `verification_status` | text | NOT NULL |
| `last_verified_at` | timestamptz | NULL |

### `eligibility_criteria`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `programme_id` | uuid | FK eligibility_programmes.id, NOT NULL |
| `criterion_key` | text | NOT NULL |
| `criterion_text` | text | NOT NULL |
| `criterion_type` | text | NOT NULL |
| `comparator` | jsonb | NULL |
| `source_excerpt` | text | NOT NULL |
| `active_from` | date | NULL |
| `active_to` | date | NULL |

Only encode criteria found in a verified source. Version changes rather than silently overwriting history.

### `eligibility_checks`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `case_id` | uuid | FK cases.id, NOT NULL |
| `programme_id` | uuid | FK eligibility_programmes.id, NOT NULL |
| `owner_id` | uuid | FK users.id, NOT NULL |
| `result` | text | constrained eligibility result |
| `answers` | jsonb | NOT NULL |
| `explanation` | text | NOT NULL |
| `created_at` | timestamptz | NOT NULL |

### `case_summaries`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `case_id` | uuid | FK cases.id, NOT NULL |
| `content` | text | NOT NULL |
| `facts_used` | jsonb | NOT NULL, default `[]` |
| `requires_user_review` | boolean | NOT NULL, default true |
| `created_at` | timestamptz | NOT NULL |
| `updated_at` | timestamptz | NOT NULL |

### `documents` (P1)
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `case_id` | uuid | FK cases.id, NOT NULL |
| `owner_id` | uuid | FK users.id, NOT NULL |
| `storage_key` | text | NOT NULL, UNIQUE |
| `filename` | text | NOT NULL |
| `mime_type` | text | NOT NULL |
| `size_bytes` | bigint | NOT NULL, positive |
| `checksum` | text | NULL |
| `processing_status` | text | NOT NULL |
| `created_at` | timestamptz | NOT NULL |

Never persist a public file URL. Store the private object key only.

### `activity_events`
| Column | Type | Rules |
|---|---|---|
| `id` | uuid | PK |
| `owner_id` | uuid | FK users.id, NOT NULL |
| `case_id` | uuid | FK cases.id, NULL |
| `event_type` | text | NOT NULL |
| `safe_metadata` | jsonb | NOT NULL, default `{}` |
| `created_at` | timestamptz | NOT NULL |

Never store case narratives, document contents, access tokens or secrets in activity metadata.

## 2. Controlled values

- Issue categories: `landlord_tenant`, `employment_unpaid_wages`, `contract_consumer`, `other`, `unknown`.
- Case status: `draft`, `submitted`, `analysis_pending`, `analysis_ready`, `needs_clarification`, `archived`.
- Verification status: `verified`, `unverified`, `stale`, `demo_only`, `rejected`.
- Fact provenance: `user_supplied`, `extracted`, `user_approved`.
- Fact review status: `unreviewed`, `approved`, `rejected`.
- Eligibility result: `potentially_eligible`, `criteria_not_met`, `insufficient_information`, `unavailable`.
- Document status: `uploaded`, `scanning`, `processed`, `failed`, `rejected`.

Use database constraints/enums or validated check constraints. Keep API values and database values identical.

## 3. Migration order

1. `users`
2. `cases`
3. `case_facts`
4. `legal_sources`
5. `legal_chunks`
6. `analyses`
7. `analysis_claims`
8. `analysis_citations`
9. `organisations`
10. `eligibility_programmes`
11. `eligibility_criteria`
12. `eligibility_checks`
13. `case_summaries`
14. `activity_events`
15. `documents` only when P1 storage is ready

## 4. Ownership and deletion rules

- Resolve internal `users.id` from validated `auth_subject`.
- Derive case and document owner on the server; never trust client-supplied owner IDs.
- Every case read/write must filter by `owner_id`.
- Every document download must verify document owner and parent case owner before issuing a short-lived signed URL.
- Prefer restrictive deletion rules. Deleting a user/case must not accidentally erase shared legal sources or organisations.
- Define retention and deletion behaviour before accepting real legal documents.
- Do not collect extra personal information merely because a column could hold it.
