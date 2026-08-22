# OpenAPI Completeness Audit — SBA Loan Origination API

Audit date: 2026-08-20
Artifacts reviewed and corrected in this repository:

* `openapi.json`
* `REDOC Loan Origination Documentation.html`

Result: `openapi.json` now describes **101 operations across 69 paths**, is valid
OpenAPI 3.0.3, is fully self-contained (no external `$ref`s), and every operation has a unique
`operationId`, a summary, a description, and documented responses. The ReDoc HTML embeds that
specification inline and renders all 101 operations.

---

## 1. Source-of-truth access — please read first

The brief asked for the specification to be diffed against the implementation in
`USSBA/ocio-gh-lending-mono` (path `lender_experience/backend/lx_src/`, branch `staging`).
**That repository could not be read from this environment in either run.** Both attempts returned
`404 Not Found`, and direct network access to `api.github.com` is blocked by the environment
proxy, so there was no alternative route to the code.

### Run 1 (PR #5 initial pass)

| Attempt | Target | Result |
| --- | --- | --- |
| Read file | `USSBA/ocio-gh-lending-mono` → `lender_experience/backend/lx_src/urls.py` (ref `refs/heads/staging`) | `404 Not Found` — reference could not be resolved |
| Read repository root | `USSBA/ocio-gh-lending-mono` → `/` | `404 Not Found` — repository metadata not available |
| List branches | `USSBA/ocio-gh-lending-mono` | `404 Not Found` |
| Code search | `org:USSBA path:lender_experience/backend/lx_src` | `0` results |
| Repository search | `org:USSBA ocio-gh`, `ocio-gh-lending in:name` | Repository not returned; only `USSBA/ocio-gh-lending-components` is visible to these credentials |
| Control test (read access works at all) | `octocat/Hello-World` → `README` | Success |
| Control test (private cross-repository read) | `USSBA/ocio-gh-lending-components` → `/` | `404 Not Found` |
| Direct HTTPS | `https://api.github.com/repos/USSBA/ocio-gh-lending-mono` | `403 Blocked by DNS monitoring proxy` |

### Run 2 (PR #5 follow-up re-verification attempt — 2026-08-20)

| Attempt | Target | Result |
| --- | --- | --- |
| Read directory | `USSBA/ocio-gh-lending-mono` → `lender_experience/backend/lx_src/views` (ref `staging`) | `404 Not Found` — `failed to resolve git reference: could not resolve ref "staging" as a branch or a tag` |
| Read repository metadata | `USSBA/ocio-gh-lending-mono` → `/` (default ref) | `404 Not Found` — `failed to get repository info: GET https://api.github.com/repos/USSBA/ocio-gh-lending-mono: 404 Not Found []` |

**Both runs confirm:** the GitHub API returns `404 Not Found` for every request to
`USSBA/ocio-gh-lending-mono`. The repository is either private and the agent's credentials have
not been granted collaborator access, or the repository name/organization is incorrect. This is an
**access/permissions issue that must be escalated**.

**Consequence:** this pass could not verify the specification field-by-field against the code, and
therefore did **not** add or remove endpoints, fields, or behaviour. What it did do in run 1 is
audit and correct the specification itself against the evidence that was available:

1. The previous `openapi.json`, whose own metadata recorded that it was generated from the
   `staging` source at commit `ca267f4d7dd3d8a66b3bc23393ca3f8a26f284c6`, including content that had
   been parked in a non-standard `x-disabled-paths` block and a non-standard
   `components/pathOperations` block.
2. The behaviour of the state-transition actions, the decline payload, and the error shapes as set
   out in the task brief.

Everything that still requires the code to confirm is listed in sections 6 and 8. **All corrections
in this PR remain unverified against the Django source code and should be treated as best-effort
pending source access.** A follow-up run with confirmed read access to `USSBA/ocio-gh-lending-mono`
is required to close those items. This PR should **not** be treated as fully verified.

---

## 2. Corrections made

### 2.1 Specification was invalid before this change

`openapi.json` failed OpenAPI 3.0.3 validation. The failures have been fixed:

| Problem | Detail | Fix |
| --- | --- | --- |
| Invalid components section | `components.pathOperations` is not a permitted field of the Components Object | Removed; its nine shared operation bodies were written out explicitly on the operations that used them |
| `$ref` used in operation position | 43 operations consisted of `{"$ref": "#/components/pathOperations/…", "tags": […], "security": […]}`; the Operation Object does not support `$ref`, so the operation had no `responses` and the sibling `tags`/`security` were ignored | All 43 operations written out in full |
| Missing `operationId` | 43 operations had none, and the shared bodies they pointed at collapsed onto 9 duplicate ids | Every operation now has a unique, resource-specific `operationId` |
| Unresolved path parameters | `{id}` was undeclared on 8 paths (`submit`, `approve`, `decline`, `recommend-approval`, `recommend-rejection`, `send-lender-requests`, `associated-documents`, `refresh-esri`) | Path-level parameters added |
| Broken internal `$ref`s | References to `components/parameters/CsrfTokenHeader` and `components/schemas/LoanAppRequest` after de-duplication | All references now resolve; validated automatically |

### 2.2 Content that was hidden from consumers

| Problem | Fix |
| --- | --- |
| Four loan-application actions (`approve`, `decline`, `recommend-approval`, `recommend-rejection`) were parked in a vendor extension `x-disabled-paths` and therefore did not appear in the documentation | Restored into `paths` with request bodies, responses, and examples |
| 43 supporting-resource operations were tagged `loans_other` ("other") and rendered without names or descriptions | Given proper tags (`Loan Application Supporting Resources`, `Loan Tasks and Comments`, `Affiliated Loans`), summaries, descriptions, and error responses |
| The decline/rejection payload schema was defined but unreachable | Reworked into `LoanAppStateActionRequest` and attached to `POST …/decline/` and `POST …/recommend-rejection/` |
| Validation failures were documented with a generic error body | `ValidationError` now documents the field-keyed message document actually returned, and is used by the `400` responses |
| State-transition failures were documented as ordinary validation errors | New `StateTransitionError` (`{"error": "…"}`) and `StateTransitionNotAllowed` response, used by all six action endpoints |

### 2.3 Consistency and usability

* Component names no longer carry the `1.0.0_` prefix that the previous generator emitted
  (`#/components/schemas/1.0.0_LoanApp` → `#/components/schemas/LoanApp`).
* Four identical session security schemes (`sessionCookie`, `authenticatedSession`, `userSession`,
  `webSessionCookie`) merged into one `sessionCookie` scheme, applied document-wide via a top-level
  `security` entry and still stated on each operation.
* Per-path `servers` blocks — ten different placeholder URLs and descriptions across the document —
  replaced with a single document-level `servers` entry that is clearly marked as a placeholder.
* `X-CSRFToken` documented on all 62 state-changing operations (18 were missing it); the duplicate
  `CsrfTokenHeader` parameter was removed.
* Separate request documents for create and partial update: `LoanAppCreateRequest`,
  `LoanAppPatchRequest`, `CompanyPatchRequest`, `CompanyEmployeePatchRequest`. Partial-update
  documents no longer inherit the create-time `required` list.
* Read-only fields are excluded from request examples and write-only fields from response examples.
* Fictional examples added to 38 request bodies and 89 JSON responses.
* All implementation vocabulary removed from consumer-facing text (framework names, serializers,
  view classes, models, and internal service names no longer appear anywhere in `openapi.json`).
* Rewritten `info.description` covering pagination, media types, money and timestamp formats,
  authentication, and a statement that all examples are fictional.

---

## 3. POST / PUT / PATCH operation inventory

Security for every operation below: session cookie (`sessionid`) plus the `X-CSRFToken` header on
state-changing requests. `401` is returned without a valid session and `403` when the signed-in user
may not act on the record.

**Notes legend**

| Code | Meaning |
| --- | --- |
| C1 | Operation was absent from the previous document and is now published |
| C2 | Operation body was an invalid `$ref` to `components/pathOperations`; written out in full |
| C3 | `operationId` was missing; a unique one was assigned |
| C4 | Operation was parked in `x-disabled-paths` and is now restored |
| C5 | Request example added |
| C6 | `X-CSRFToken` header added |
| C7 | Dedicated create/partial-update request document introduced |
| C8 | Description added |
| U1 | Request and response fields for this collection are not enumerated — see §6.1 |
| U2 | Body documented as free-form because the accepted field list is unverified — see §6.1 |
| U3 | Full-replacement vs partial-update semantics unverified — see §6.3 |
| U4 | Response body of this action is documented as empty — see §6.2 |

| Operation | operationId | Query parameters | Headers | Request body | Required fields | Optional fields | Success responses | Error responses | Security | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `POST /loan-apps/` | `createLoanApp` | none | `X-CSRFToken` | `LoanAppCreateRequest` (`application/json`), body required | none | `program`, `processing_method_code`, `lender_location_id`, `calendar_basis`, `is_app_structured_as_epc_oc`, `is_funded_by_robs` (+26 more) | `201` LoanApp | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 C7 |
| `PATCH /loan-apps/{id}/` | `partiallyUpdateLoanApp` | none | `X-CSRFToken` | `LoanAppPatchRequest` (`application/json`), body required | none | `program`, `processing_method_code`, `lender_location_id`, `calendar_basis`, `is_app_structured_as_epc_oc`, `is_funded_by_robs` (+26 more) | `200` LoanApp | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C5 C6 C7 C8 |
| `POST /loan-apps/{id}/submit/` | `submitLoanApp` | none | `X-CSRFToken` | no request body | n/a | n/a | `200` no body | `400` StateTransitionError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C6 C8 U4 |
| `POST /loan-apps/{id}/approve/` | `approveLoanApp` | none | `X-CSRFToken` | no request body | n/a | n/a | `200` no body | `400` StateTransitionError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C4 C6 C8 U4 |
| `POST /loan-apps/{id}/decline/` | `declineLoanApp` | none | `X-CSRFToken` | `LoanAppStateActionRequest` (`application/json`), body optional | none | `decline_reasons`, `decline_comment` | `200` no body | `400` StateTransitionError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C4 C5 C6 C8 U4 |
| `POST /loan-apps/{id}/recommend-approval/` | `recommendLoanAppApproval` | none | `X-CSRFToken` | no request body | n/a | n/a | `200` no body | `400` RecommendationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C4 C6 C8 U4 |
| `POST /loan-apps/{id}/recommend-rejection/` | `recommendLoanAppRejection` | none | `X-CSRFToken` | `LoanAppStateActionRequest` (`application/json`), body optional | none | `decline_reasons`, `decline_comment` | `200` no body | `400` RecommendationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C4 C5 C6 C8 U4 |
| `POST /loan-apps/{id}/send-lender-requests/` | `sendLoanAppLenderRequests` | none | `X-CSRFToken` | no request body | n/a | n/a | `200` no body | `400` StateTransitionError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C6 C8 U4 |
| `POST /loan-app-project-details/` | `createLoanAppProjectDetail` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-project-details/{id}/` | `partiallyUpdateLoanAppProjectDetail` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-project-details/{id}/refresh-esri/` | `refreshLoanAppProjectDetailLocationData` | none | `X-CSRFToken` | no request body | n/a | n/a | `200` no body | `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C6 C8 U1 |
| `POST /loan-app-agents/` | `createLoanAppAgent` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-agents/{id}/` | `partiallyUpdateLoanAppAgent` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-agent-services/` | `createLoanAppAgentService` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-agent-services/{id}/` | `partiallyUpdateLoanAppAgentService` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-credit-unavailable-elsewhere-explanations/` | `createCreditUnavailableElsewhereExplanation` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-credit-unavailable-elsewhere-explanations/{id}/` | `partiallyUpdateCreditUnavailableElsewhereExplanation` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-associated-users/` | `createLoanAppAssociatedUser` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-terms-periods/` | `createLoanAppTermsPeriod` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-terms-periods/{id}/` | `partiallyUpdateLoanAppTermsPeriod` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-use-of-proceeds/` | `createLoanAppUseOfProceeds` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-use-of-proceeds/{id}/` | `partiallyUpdateLoanAppUseOfProceeds` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-task/{id}/` | `partiallyUpdateLoanAppTask` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-task-comments/` | `createLoanAppTaskComment` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /loan-app-comments/` | `createLoanAppComment` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `PATCH /loan-app-comments/{id}/` | `partiallyUpdateLoanAppComment` | none | `X-CSRFToken` | `SupportingResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | C2 C3 C6 C8 U1 U2 |
| `POST /collateral-assets/` | `createCollateralAsset` | none | `X-CSRFToken` | `CollateralAssetRequest` (`application/json`), body required | `loan_app_id`, `collateral_type`, `address_id`, `gross_value_amount`, `liquidation_rate_percent`, `lender_lien_position`, `valuation_source` | `description`, `valuation_date`, `property_type`, `is_acquired_with_loan_or_project_funds`, `junior_lien_position_acquisition_exception_granted_at`, `property_improvement_status` (+15 more) | `201` CollateralAsset | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `415` ErrorResponse | session cookie | C5 C6 |
| `POST /collateral-titled-assets/` | `createCollateralTitledAsset` | none | `X-CSRFToken` | `CollateralTitledAssetRequest` (`application/json`), body required | `collateral_asset_id`, `titled_asset_type`, `id_str` | `make`, `model`, `year`, `owner`, `name` | `201` CollateralTitledAsset | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 |
| `POST /collateral-lienholders/` | `createCollateralLienholder` | none | `X-CSRFToken` | `CollateralLienholderRequest` (`application/json`), body required | `loan_app_collateral_id`, `lien_position` | `name`, `amount` | `201` CollateralLienholder | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 |
| `POST /collateral-pledgers/` | `createCollateralPledger` | none | `X-CSRFToken` | `CollateralPledgerRequest` (`application/json`), body required | `loan_app_collateral_id` | `loan_app_company_id`, `loan_app_person_id` | `201` CollateralPledger | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 |
| `POST /companies/` | `createCompany` | none | `X-CSRFToken` | `CompanyRequest` (`application/json`), body required | none | `tax_id_type`, `tax_id`, `name`, `legal_entity_type`, `special_ownership_type`, `trade_name` (+6 more) | `201` CompanyResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 |
| `PUT /companies/{id}/` | `updateCompany` | none | `X-CSRFToken` | `CompanyRequest` (`application/json`), body required | none | `tax_id_type`, `tax_id`, `name`, `legal_entity_type`, `special_ownership_type`, `trade_name` (+6 more) | `200` CompanyResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 U3 |
| `PATCH /companies/{id}/` | `partiallyUpdateCompany` | none | `X-CSRFToken` | `CompanyPatchRequest` (`application/json`), body required | none | `tax_id_type`, `tax_id`, `name`, `legal_entity_type`, `special_ownership_type`, `trade_name` (+6 more) | `200` CompanyResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 C7 |
| `POST /company-snapshots/` | `createCompanySnapshot` | none | `X-CSRFToken` | `CompanySnapshotRequest` (`application/json`), body required | none | `tax_id_type`, `tax_id`, `name`, `legal_entity_type`, `special_ownership_type`, `trade_name` (+7 more) | `201` CompanySnapshotResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 |
| `POST /company-employees/` | `createCompanyEmployee` | none | `X-CSRFToken` | `CompanyEmployeeRequest` (`application/json`), body required | `company_id` | `first_name`, `last_name`, `title`, `tax_id`, `citizenship_status`, `email` (+8 more) | `201` CompanyEmployeeResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 |
| `PUT /company-employees/{id}/` | `updateCompanyEmployee` | none | `X-CSRFToken` | `CompanyEmployeeRequest` (`application/json`), body required | `company_id` | `first_name`, `last_name`, `title`, `tax_id`, `citizenship_status`, `email` (+8 more) | `200` CompanyEmployeeResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 U3 |
| `PATCH /company-employees/{id}/` | `partiallyUpdateCompanyEmployee` | none | `X-CSRFToken` | `CompanyEmployeePatchRequest` (`application/json`), body required | none | `first_name`, `last_name`, `title`, `tax_id`, `citizenship_status`, `email` (+9 more) | `200` CompanyEmployeeResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 C7 |
| `POST /company-employee-snapshots/` | `createCompanyEmployeeSnapshot` | none | `X-CSRFToken` | `CompanyEmployeeSnapshotRequest` (`application/json`), body required | none | `first_name`, `last_name`, `title`, `tax_id`, `citizenship_status`, `email` (+9 more) | `201` CompanyEmployeeSnapshotResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 |
| `POST /loan-app-doc-files/` | `uploadLoanApplicationDocumentFile` | none | `X-CSRFToken` | `DocumentFileUploadRequest` (`multipart/form-data`), body required | `loan_app_doc_id`, `file` | none | `201` DocumentFile | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 C6 |
| `POST /loan-app-doc-files/validate/` | `validateLoanApplicationDocumentFile` | none | `X-CSRFToken` | `FileValidationRequest` (`multipart/form-data`), body required | `file` | none | `200` ValidationSuccess | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `405` Error; `415` ErrorResponse | session cookie | C5 C6 |
| `POST /api/v1/loan-app-issues/{id}/address/` | `markLoanApplicationIssueAddressed` | none | `X-CSRFToken` | no request body | n/a | n/a | `200` no body | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | — |
| `POST /api/v1/loan-app-issues/{id}/unaddress/` | `markLoanApplicationIssueUnaddressed` | none | `X-CSRFToken` | no request body | n/a | n/a | `200` no body | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse | session cookie | — |
| `POST /loan-app-companies/` | `createBusiness` | none | `X-CSRFToken` | `FlexibleResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` Error; `403` Error | session cookie | C5 C6 C8 U2 |
| `POST /loan-app-persons/` | `createIndividual` | none | `X-CSRFToken` | `FlexibleResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError; `401` Error; `403` Error | session cookie | C5 C6 C8 U2 |
| `POST /company-to-person-ownership-snapshots/` | `createIndividualBusinessOwnershipRelationship` | none | `X-CSRFToken` | `FlexibleResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError | session cookie | C5 C6 C8 U2 |
| `POST /company-to-company-ownership-snapshots/` | `createBusinessOwnershipRelationship` | none | `X-CSRFToken` | `FlexibleResourceRequest` (`application/json`), body required | none | `_free-form_` | `201` SupportingResource | `400` ValidationError | session cookie | C5 C6 C8 U2 |
| `POST /loan-app-company-eligibilities/` | `createOrUpdateBusinessEligibility` | none | `X-CSRFToken` | `FlexibleResourceRequest` (`application/json`), body required | none | `_free-form_` | `200` SupportingResource | `400` ValidationError; `404` ErrorResponse | session cookie | C5 C6 C8 U2 |
| `POST /applicant-equity-injection-sources/` | `createApplicantEquityInjectionSource` | none | `X-CSRFToken` | `ApplicantEquityInjectionSourceRequest` (`application/json`), body required | `loan_app_id`, `source_type`, `source_amount` | `source_loan_app_company_id`, `source_loan_app_person_id`, `source_other_equity_holder`, `source_type_other_description`, `standby_debt_creditor_name`, `standby_debt_borrower_name` (+4 more) | `201` ApplicantEquityInjectionSourceResponse | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 |
| `POST /other-financing-sources/` | `createOtherFinancingSource` | none | `X-CSRFToken` | `OtherFinancingSourceRequest` (`application/json`), body required | `loan_app_id`, `source`, `source_amount` | none | `201` OtherFinancingSourceResponse | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 |
| `POST /loan-app-uses/` | `createLoanApplicationUse` | none | `X-CSRFToken` | `LoanApplicationUseRequest` (`application/json`), body required | `loan_app_id`, `use_type`, `use_amount`, `applicant_equity_injection_amount`, `other_financing_amount` | `will_applicant_occupy_51_percent_of_location`, `will_there_be_subleases`, `lease_term_years`, `is_land_leased`, `will_applicant_occupy_60_percent_of_location`, `is_construction_diy` (+28 more) | `201` LoanApplicationUseResponse | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 |
| `POST /loan-app-use-addresses/` | `createLoanApplicationUseAddress` | none | `X-CSRFToken` | `LoanApplicationUseAddressRequest` (`application/json`), body required | `address_id`, `loan_app_use_id` | none | `201` LoanApplicationUseAddressResponse | `400` ValidationError; `401` ErrorResponse; `403` ErrorResponse | session cookie | C5 C6 |
| `PATCH /compliance-alerts/{id}/` | `updateComplianceAlertStatus` | none | `X-CSRFToken` | `ComplianceAlertStatusUpdate` (`application/json`), body required | none | `is_resolved`, `is_declined` | `200` ComplianceAlert | `400` ValidationError; `401` Error; `403` Error; `404` ErrorResponse | session cookie | C5 C6 |
| `POST /api/v1/validation/validate/` | `validateParentObject` | `page` (optional) | `X-CSRFToken` | `ValidateRequest` (`application/json`), body required | `parent_object_id`, `parent_object_model` | `changed` | `200` ValidationFailureFieldPage | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse; `415` ErrorResponse | session cookie | — |
| `PUT /api/v1/validation/validation-failures/{id}/` | `updateValidationFailure` | none | `X-CSRFToken` | `ValidationFailureUpdateRequest` (`application/json`), body required | none | `dismissed_at` | `200` ValidationFailureResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse; `415` ErrorResponse | session cookie | U3 |
| `PATCH /api/v1/validation/validation-failures/{id}/` | `partiallyUpdateValidationFailure` | none | `X-CSRFToken` | `ValidationFailureUpdateRequest` (`application/json`), body required | none | `dismissed_at` | `200` ValidationFailureResponse | `400` ErrorResponse; `401` ErrorResponse; `403` ErrorResponse; `404` ErrorResponse; `415` ErrorResponse | session cookie | — |

---

## 4. Files reviewed in `USSBA/ocio-gh-lending-mono`

**None.** As set out in section 1, no file in that repository could be read. The following files
were the intended review set and remain outstanding:

* `lender_experience/backend/lx_src/urls.py` (and the project-level URL configuration)
* `lender_experience/backend/lx_src/views/` — including `base.py` and the loan application,
  project details, agent, agent service, task, task comment, comment, affiliated loan, and decline
  reason view modules
* `lender_experience/backend/lx_src/serializers/`
* `lender_experience/backend/lx_src/filters/`
* `lender_experience/backend/lx_src/models/loans.py` and `models/loan_parties.py`
* Permission classes and `utils.py`
* Tests and fixtures

## 5. Files reviewed in this repository

* `openapi.json` (218 KB before, 310 KB after)
* `REDOC Loan Origination Documentation.html`

---

## 6. Missing information, conflicts, and assumptions

### 6.1 Collections documented without a field list

Twenty-two write operations across sixteen collections accept a free-form object because the
previous document did not record their fields:

`/loan-app-project-details/`, `/loan-app-agents/`, `/loan-app-agent-services/`,
`/loan-app-credit-unavailable-elsewhere-explanations/`, `/loan-app-associated-users/`,
`/loan-app-terms-periods/`, `/loan-app-use-of-proceeds/`, `/loan-app-task/`,
`/loan-app-task-comments/`, `/loan-app-comments/`, `/affiliated-loans/`, plus
`/loan-app-companies/`, `/loan-app-persons/`, `/company-to-person-ownership-snapshots/`,
`/company-to-company-ownership-snapshots/`, and `/loan-app-company-eligibilities/`.

Nothing was invented for them. Their query filters are also unknown: only `limit`, `offset`, and
`ordering` are documented, although the implementation is described as having per-collection
filters. **Action: supply the field lists and filter names.**

### 6.2 Action endpoints

`submit`, `approve`, `decline`, `recommend-approval`, `recommend-rejection`,
`send-lender-requests`, and `refresh-esri` are documented as returning `200` with no response body,
and `400` with `{"error": "…"}` when the action is not allowed from the current state. This
follows the behaviour described in the brief. `decline` and `recommend-rejection` accept an optional
body of `decline_reasons` and `decline_comment`. **Action: confirm the response bodies, whether the
body is genuinely optional, and whether `decline_reasons` values come from a fixed list.**

### 6.3 `PUT` semantics

`PUT /companies/{id}/` and `PUT /company-employees/{id}/` are documented as full replacements
that require `name` / the create-time required fields. The brief notes that some update handlers
apply partial semantics even for `PUT`. **Action: confirm; if these are partial, the `PUT` bodies
should use the same documents as `PATCH`.**

### 6.4 Code lists are not enumerated

String fields such as `program`, `processing_method_code`, `calendar_basis`,
`interest_rate_type`, `state`, and `decline_reasons` have no `enum`. Examples deliberately omit
these fields rather than show an invented code. **Action: supply the permitted values** — the brief
refers to defined choice sets for application state, recommendation, underwriting authority, loan
processing method, and transfer status.

### 6.5 Request documents without matching operations

`CreateLoanApplicationIssueRequest`, `UpdateLoanApplicationIssueRequest`, and
`PartialUpdateLoanApplicationIssueRequest` are defined but no operation uses them: only
`GET /api/v1/loan-app-issues/`, `POST …/{id}/address/`, and `POST …/{id}/unaddress/` are
documented. This strongly suggests create/update operations for issues exist but were dropped.
The schemas were retained rather than deleted, and no operations were invented.
**Action: confirm whether `POST /api/v1/loan-app-issues/` and `PUT`/`PATCH
/api/v1/loan-app-issues/{id}/` exist and should be published.**

### 6.6 Endpoints referenced by the brief but absent from the document

The brief names a decline-reason collection among the exposed collections; no such path exists in
the document, and one was not added. **Action: confirm the route and payload if it is public.**

### 6.7 Server URL

The document previously contained ten different placeholder base URLs. A single placeholder is now
published: `https://api.example.gov/lender-experience/api/v1`. Two path prefixes coexist in the
document — most paths are relative to the base URL, while issue and validation paths carry an
explicit `/api/v1/` prefix. **Action: supply the approved base URL and confirm whether the `/api/v1`
prefix on those paths is correct.**

### 6.8 Assumptions applied

1. The four session security schemes describe the same cookie and were merged.
2. `X-CSRFToken` applies to every state-changing request, not only the ones that previously listed it.
3. Supporting-resource operations return the same `401`/`403`/`404` errors already documented on
   comparable endpoints.
4. Every example value is fictional. Tax identifiers are omitted from examples entirely, and
   demographic fields use "Not Disclosed".

---

## 7. Validation results

| Check | Tool | Before | After |
| --- | --- | --- | --- |
| JSON syntax | `python -m json.tool` | pass | pass |
| OpenAPI 3.0.3 structure | `openapi-spec-validator` 0.7.x (installed from PyPI for this audit) | **fail** — `responses` missing on 43 operations; unresolved path parameters | **pass** |
| Duplicate `operationId` | script | 43 operations with no id; 9 shared ids reused across paths | 0 missing, 0 duplicates (101 unique) |
| Internal `$ref` resolution | script | 2 broken after de-duplication | 0 broken |
| External `$ref`s | script | 0 | 0 |
| Response descriptions | script | pass | pass |
| Parameter placement | script | 8 paths with undeclared `{id}` | all path parameters declared |
| OpenAPI 3.1-only keywords | script | none | none |
| ReDoc rendering | ReDoc standalone bundle executed in a headless DOM | not re-verified | all 101 operation summaries and all tags present in the rendered output |

The ReDoc HTML embeds the specification inline and includes the ReDoc bundle, so it renders with no
network access and no separate spec file.

---

## 8. Items needing confirmation from the implementation team

1. **Grant read access to `USSBA/ocio-gh-lending-mono`** so the specification can be diffed against
   `lender_experience/backend/lx_src/`. Both automated runs returned `404 Not Found` — the
   repository must be made accessible to the agent's GitHub credentials (e.g. by adding the Copilot
   app as a collaborator or changing visibility). Until access is confirmed, the items below cannot
   be closed and all corrections in this PR remain best-effort and unverified.
2. Field lists and validation rules for the sixteen collections listed in §6.1.
3. Query parameters for every list endpoint: filter names and types, ordering fields, and search
   fields. Only `limit`, `offset`, `ordering`, `optional_fields`, and `state` are documented today.
4. Response bodies and status codes for the six loan-application actions and `refresh-esri` (§6.2).
5. Whether `decline_reasons` is a fixed code list, and whether `decline_comment` is required when
   reasons are supplied.
6. `PUT` semantics for companies and company employees (§6.3).
7. Permitted values for the coded string fields in §6.4.
8. Whether loan-application issue create/update operations should be published (§6.5).
9. Whether a decline-reason collection is part of the public contract (§6.6).
10. The approved base URL, and whether the `/api/v1` prefix on the issue, reference-data, and
    validation paths is correct (§6.7).
11. Which endpoints are restricted to particular roles. The document states that `403` is returned
    when the user may not act on a record, but does not describe the roles involved.
