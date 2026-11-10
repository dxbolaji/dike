# Diké shared-repository and integration workflow

## 1. Repository setup

Create one GitHub repository named `dike`. Keep it private while development is underway unless the hackathon requires public visibility.

Commit the following before feature development:
- `docs/MASTER_SPEC.md` — Markdown copy of the approved product specification
- `docs/UI_GUIDE.md`
- `docs/API_CONTRACTS.md`
- `docs/openapi.yaml`
- `docs/DATABASE_SCHEMA.md`
- `docs/FRONTEND_PLAYBOOK.md`
- `docs/BACKEND_PLAYBOOK.md`
- `docs/INTEGRATION_PLAYBOOK.md`
- `.env.example`

Keep the DOCX as a human-readable reference, but do not make it the only source agents can inspect.

## 2. Branches

- `main`: stable shared baseline
- `frontend/dx`: frontend feature branch
- `backend/jibola`: backend feature branch

Both feature branches start from the same `main` commit. Each developer opens pull requests back to `main`. Do not push directly to `main` for feature work.

## 3. Ownership boundaries

DX owns `apps/web/**` and frontend tests/docs.
Jibola owns `apps/api/**`, backend tests and migrations.
Shared API/schema/spec changes under `docs/**` require agreement from both owners before merge.

Avoid simultaneous edits to the same files. Use pull requests for shared changes.

## 4. Contract-first implementation

1. Review `docs/openapi.yaml` and `docs/API_CONTRACTS.md` together.
2. DX confirms the frontend can consume the request/response shapes.
3. Jibola confirms the backend can implement the endpoints and validation.
4. Resolve disagreements in the contract before coding dependent features.
5. Commit the accepted contract to `main`.
6. Jibola implements FastAPI/Pydantic models and endpoints that conform to the contract.
7. DX generates or maintains TypeScript types from the committed OpenAPI file and uses those types in the API client.
8. As backend endpoints become available, DX replaces only the matching mock calls with real API calls.

The OpenAPI file is the wire-contract source of truth. FastAPI's generated OpenAPI output must be checked against it. If it differs, reconcile the difference deliberately; do not let two undocumented versions drift.

## 5. Suggested frontend type generation

Use a TypeScript OpenAPI type generator such as `openapi-typescript` as a development dependency. Generate types from `docs/openapi.yaml` into a frontend-owned generated file. Do not hand-edit generated output. Keep the API client typed against those generated schemas.

Example command (run from repository root after installing the dev dependency):

```bash
npx openapi-typescript docs/openapi.yaml -o apps/web/src/lib/generated/api-schema.d.ts
```

The exact output path may be adjusted to match the repository's existing layout.

## 6. Integration sequence

Integrate one vertical slice at a time:
1. Authentication token reaches backend and is validated.
2. Create case: frontend form → `POST /cases` → persisted response.
3. List/open case: `GET /cases` and `GET /cases/{case_id}`.
4. Analyse case: `POST /cases/{case_id}/analyze`.
5. Display analysis and citations: `GET /cases/{case_id}/analysis`.
6. Find assistance: `GET /legal-aid`.
7. Generate and edit case summary.

Do not wait until every screen and endpoint is complete before attempting integration.

## 7. Contract change procedure

Any shared contract change must:
- update `openapi.yaml` and `API_CONTRACTS.md` in the same PR;
- explain why the change is required;
- include request/response examples;
- state whether the change is backwards compatible;
- update backend Pydantic models and frontend generated types;
- add/update contract tests;
- receive approval from both DX and Jibola.

## 8. Secrets and agent safety

- Commit `.env.example`, never `.env`.
- Do not paste service-role keys, database passwords, LLM keys or signing secrets into chat prompts or GitHub issues.
- Configure secrets in each developer's local environment and deployment secret store.
- Use synthetic case data during development and demos.
