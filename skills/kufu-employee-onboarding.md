---
name: kufu-employee-onboarding
description: >-
  Onboard a new employee into SmartHR — create the employee record, attach
  department and employment type, register family/dependent records, and send
  the self-service invitation. Use when a new hire must be provisioned into
  SmartHR from an ATS, HRIS, or spreadsheet.
api: SmartHR API v1
generated: '2026-07-19'
method: generated
source: openapi/kufu-smarthr-openapi.json
operations:
  - getV1EmploymentTypes
  - getV1Departments
  - getV1BizEstablishments
  - postV1Crews
  - postV1CrewsCrewIdDependents
  - putV1CrewsIdInvite
  - getV1CrewsId
---

# Onboard an employee into SmartHR

## Before you start

- Base URL is `https://{subdomain}.smarthr.jp/api` — the subdomain identifies the tenant.
- Send `Authorization: Bearer ACCESS_TOKEN`. The token is tenant-scoped; a token from another subdomain will not work.
- **There is no idempotency key.** If a `postV1Crews` call fails ambiguously, do not retry blindly — search with `getV1Crews` first, or you will create a duplicate employee.
- Keep under 10 requests/second per token. A loop over a hire list must be paced.

## 1. Resolve reference data

Employment type, department, position, grade, job category and recruitment type are tenant-configurable resources, not fixed enums. Resolve the ids you need before creating anything.

- `getV1EmploymentTypes` — 雇用形態 (employment type)
- `getV1Departments` — 部署 (department); note that departments form a tree, so a child carries its parent through the `children` relation
- `getV1BizEstablishments` — 事業所 (business establishment), needed for social and labour insurance handling

If the target department does not exist, create it with `postV1Departments`.

## 2. Create the employee record

Call `postV1Crews`. The `Crew` schema is large — over 130 fields — because it carries Japanese statutory data. At minimum supply name fields (`last_name`, `first_name` and their `_yomi` readings), `birth_at`, `gender`, `entered_at`, and `emp_type`.

Attach the ids resolved in step 1: `biz_establishment_id`, `employment_type`, `departments`.

Tenant-specific fields go in `custom_fields`, keyed by templates from `getV1CrewCustomFieldTemplates`.

Validation is strict. On failure the response carries `{code, type, message, errors}`, where `errors` names the offending fields — read it rather than retrying.

## 3. Register dependents

For each family member, call `postV1CrewsCrewIdDependents` with the `crew_id` from step 2. Resolve the relationship code first with `getV1DependentRelations` (続柄). Dependents drive social-insurance eligibility, so get these right before insurance qualification dates are set.

## 4. Invite the employee

Call `putV1CrewsIdInvite` to send the self-service invitation, so the employee completes their own remaining details in SmartHR. Do this last — inviting before the record is complete sends the employee into a half-configured form.

## 5. Verify

Call `getV1CrewsId` and confirm the record reads back as expected.

## Watch out for

- **Bitemporality.** SmartHR records are effective-dated. A future `entered_at` means the record exists now but takes effect later, and the `crew_updated` webhook will fire on the effective date, not today. Do not treat webhook silence as failure.
- **My Number.** Individual numbers are regulated under Japan's My Number Act and are handled through a separate restricted surface. Do not route them through general integration code or an agent context.
- **Webhook loops.** If your integration also consumes SmartHR webhooks, append `skip_sending_webhook=true` to these writes so your own changes do not trigger your own consumer.

## Bulk alternative

For more than a handful of hires, use the async path instead: `postV1BatchJobsJobTypesCrewImport`, then poll `getV1BatchJobsId` until complete, then read results via `getV1BatchJobAttachmentsIdFile`.
