---
name: kufu-payroll-run
description: >-
  Push a payroll run into SmartHR and publish payslips to employees — create
  the payroll group, bulk-load payslips, fix (確定) the run, then publish
  (公開). Use when payroll is calculated in an external system and SmartHR is
  the delivery surface.
api: SmartHR API v1
generated: '2026-07-19'
method: generated
source: openapi/kufu-smarthr-openapi.json
operations:
  - getV1Crews
  - postV1Payrolls
  - postV1PayrollsPayrollIdPayslipsBulk
  - postV1PayrollsPayrollIdPayslips
  - getV1PayrollsPayrollIdPayslips
  - patchV1PayrollsIdFix
  - patchV1PayrollsIdPublish
  - patchV1PayrollsIdUnfix
  - getV1PayrollsId
---

# Deliver a payroll run through SmartHR

SmartHR does not calculate payroll here — this flow loads results computed elsewhere and delivers them to employees as payslips. The run moves through three states: draft, fixed (確定), published (公開).

## Before you start

- Auth and base URL as in `kufu-employee-onboarding`.
- Writes are not idempotent. A retried bulk load can double-post payslips. Always read back with `getV1PayrollsPayrollIdPayslips` before retrying.
- Publishing is employee-visible. Treat `patchV1PayrollsIdPublish` as the point of no return for communication purposes.

## 1. Resolve employees

Call `getV1Crews` to map your external payroll identifiers to SmartHR `crew_id` values. Filter to currently employed staff; paginate with `page`/`per_page` (max 100) and follow the `Link` header rather than guessing page counts.

## 2. Create the payroll group

Call `postV1Payrolls` to create the run (給与明細グループ). This is the container that payslips attach to.

## 3. Load payslips

Prefer `postV1PayrollsPayrollIdPayslipsBulk` for a whole run — it is one call rather than N, which matters against the 10 requests/second limit. Use `postV1PayrollsPayrollIdPayslips` only for single corrections.

Each `Payslip` carries `crew_id` plus arrays of `allowances` (支給), `deductions` (控除), `attendances` (勤怠) and `payroll_aggregates` (集計). Build these from your payroll engine's output.

## 4. Verify before fixing

Call `getV1PayrollsPayrollIdPayslips` and reconcile the count and totals against your source system. This is the last cheap point to catch a duplicate bulk load.

## 5. Fix the run

Call `patchV1PayrollsIdFix` to mark the run confirmed. If reconciliation fails after this, `patchV1PayrollsIdUnfix` reverses it — but only before publishing.

## 6. Publish

Call `patchV1PayrollsIdPublish` to make payslips visible to employees.

## 7. Confirm

Call `getV1PayrollsId` and check the run's state.

## Corrections

To remove an erroneous payslip use `deleteV1PayrollsPayrollIdPayslipsId`; to drop an entire draft run use `deleteV1PayrollsId`. Both are destructive against statutory records — require explicit human confirmation, never let an agent take these autonomously.

## Related

Withholding tax slips (源泉徴収票) follow a parallel shape: create the group with `postV1TaxWithholdings`, then attach slips with `postV1TaxWithholdingsTaxWithholdingIdTaxWithholdingSlips`.
