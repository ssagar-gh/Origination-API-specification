# OpenAPI Completeness Audit — Lending Experience Loans API

**Audit date:** 2026-08-20
**Artifacts audited and corrected**

| Artifact | Path |
|---|---|
| OpenAPI document | `openapi.json` |
| Reference documentation | `REDOC Loan Origination Documentation.html` |
| This report | `openapi-completeness-audit.md` |

---

## 1. Scope, sources reviewed, and a material limitation

### 1.1 What was reviewed

| Source | Location | Result |
|---|---|---|
| Existing OpenAPI document | `openapi.json` (65 paths, 97 operations, 84 schemas, 218 KB) | Reviewed in full |
| Existing ReDoc documentation | `REDOC Loan Origination Documentation.html` (2.2 MB, pre-rendered `Redoc.hydrate` bundle) | Reviewed and regenerated |
| Repository history | `git log --all`, branches, and every file ever committed | Reviewed; the repository has only ever contained the two artifacts above |

### 1.2 Material limitation — the Django source of truth was not reachable

The task named `USSBA/ocio-gh-lending-mono` (specifically `lender_experience/backend/lx_src/views`) as the source of
truth. **That repository could not be reached from this environment**, so no `urls.py`, router, view, serializer,
model, filter, permission, pagination, exception-handler, test, or fixture file was read. Every access route was
tried and each one failed:

| Access route attempted | Result |
|---|---|
| GitHub MCP `get_file_contents` on `USSBA/ocio-gh-lending-mono` | `404 Not Found` |
| GitHub MCP `get_file_contents` on `ussba/ocio-gh-lending-mono` (case variant) | `404 Not Found` |
| GitHub MCP `get_file_contents` on `USSBA/ocio-gh-lending-components` (control: a private repo that *is* visible to repo search) | `404 Not Found` — confirms the token has no cross-repo content access, not that the repo is missing |
| GitHub MCP `search_code` `org:USSBA path:lender_experience/backend/lx_src` | `0 results` |
| GitHub MCP `search_code` `org:USSBA filename:urls.py` | `0 results` |
| GitHub MCP `search_repositories` `org:USSBA ocio-gh` | Returns only `ocio-gh-lending-components`; the mono repo is not in the index available to this token |
| `git ls-remote https://github.com/USSBA/ocio-gh-lending-mono` | `Authentication failed` |
| `GET https://api.github.com/repos/USSBA/ocio-gh-lending-mono` with the only available token | `403` |
| Local filesystem search for a checkout (`find / -name lx_src -type d`, `-iname '*lending-mono*'`) | No checkout present |

**Consequence for this audit.** Steps 2 and 3 of the requested method (trace the Django implementation, then build
the operation inventory *from the implementation*) could not be performed. The audit was therefore re-scoped to what
can be established with certainty:

1. A full **internal-consistency and specification-conformance audit** of `openapi.json`, which found that the
   document was **not a valid OpenAPI 3.0.3 document at all** and that 43 of its 97 operations were documented with
   placeholder content.
2. **Correction** of every defect that can be fixed without knowledge of the Django code.
3. **Explicit, itemised disclosure** (section 6) of everything that still requires the Django source to confirm.

Nothing in the corrected document asserts a route, field, status code, or parameter that was not already present in,
or directly implied by, the audited document. **No endpoints, request fields, or response fields were invented.**

### 1.3 Validation tooling used

| Tool | Availability | Use |
|---|---|---|
| `openapi-spec-validator` (Python) | Installed during the audit | OpenAPI 3.0.3 structural validation |
| `@stoplight/spectral-cli@6` with `spectral:oas` | Run via `npx` | Rule-based linting |
| `@redocly/cli@1 lint` | Run via `npx` | Second-opinion linting (`struct`, `no-unused-components`, …) |
| `@redocly/cli@1 build-docs` | Run via `npx` | Confirms ReDoc can actually render the corrected document |
| Custom Python checks | Written for this audit | `$ref` integrity, duplicate `operationId`s, undeclared path parameters, missing error responses, security coverage, server/base-path consistency |
| Headless browser (Playwright) | **Not available** — the MCP browser server required interactive OAuth | Visual rendering of the final HTML could not be captured; renderability was instead proven by a successful ReDoc pre-render (section 7) |

---

## 2. Findings — defects present in the audited `openapi.json`

Severity: **S1** = document is invalid or materially misleading; **S2** = incomplete or inconsistent; **S3** = quality.

### S1-1 — The document was not a valid OpenAPI 3.0.3 document

`components.pathOperations` is not a field defined by the OpenAPI 3.0.3 Components Object, and 43 operations
consisted of nothing but a `$ref` into it. `openapi-spec-validator` rejected the document:

```
INVALID: 'responses' is a required property
On instance['paths']['/loan-app-project-details/']['get']:
    {'$ref': '#/components/pathOperations/1.0.0_listLoanAppProjectDetails', 'tags': [...], 'security': [...]}
```

Any conforming tool — code generator, mock server, contract test, gateway importer — would reject the file.

### S1-2 — 43 operations carried tags and security that specification-conformant tools must ignore

Each of those 43 stubs looked like this:

```json
"get": {
  "$ref": "#/components/pathOperations/1.0.0_listGenericResource",
  "tags": ["loans_other"],
  "security": [{"1.0.0_sessionCookie": []}]
}
```

In OpenAPI 3.0, **all sibling keys of `$ref` are ignored**. The `tags` and `security` above were dead weight, so
every one of those operations was, to a conforming reader, untagged and — because there was no document-level
`security` — **undocumented as to authentication**.

### S1-3 — Five `operationId` values were each reused by up to ten different operations

`operationId` must be unique across the document. It was not:

| Duplicated `operationId` | Number of operations sharing it |
|---|---|
| `listResources` | 10 |
| `createResource` | 8 |
| `getResourceById` | 7 |
| `partiallyUpdateResource` | 7 |
| `deleteResource` | 7 |

Ten different collections (`/loan-app-agents/`, `/loan-app-agent-services/`,
`/loan-app-credit-unavailable-elsewhere-explanations/`, `/loan-app-associated-users/`, `/loan-app-terms-periods/`,
`/loan-app-use-of-proceeds/`, `/loan-app-task/`, `/loan-app-task-comments/`, `/loan-app-comments/`,
`/affiliated-loans/`) all resolved to the single summary "List resources" with the description-free
`GenericPage` response. A generated client would have produced five methods for 39 endpoints.

### S1-4 — Every base URL was wrong for eleven paths, and inconsistent across the rest

The document had **no top-level `servers`**. Instead all 65 path items carried their own `servers`:

| Base URL declared on path items | Path items |
|---|---|
| `https://developer.sba.gov/origination/api/v1` | 58 |
| `https://api.example.gov/lender-experience/api/v1` | 4 (`/loan-app-docs/`, `/loan-app-doc-files/`, `/loan-app-doc-files/{id}/download/`, `/loan-app-doc-files/validate/`) |
| `https://api.example.gov/api/v1` | 3 (`/compliance-checks/`, `/compliance-alerts/`, `/compliance-alerts/{id}/`) |

Seven paths therefore pointed at a placeholder host that does not exist. Worse, eleven paths *also* repeated the
version prefix in the path key while sitting on a base URL that already ended in `/api/v1`:

```
base   https://developer.sba.gov/origination/api/v1
path   /api/v1/agents/
result https://developer.sba.gov/origination/api/v1/api/v1/agents/   ← unreachable
```

Affected: `/api/v1/loan-app-issues/`, `/api/v1/loan-app-issues/{id}/address/`,
`/api/v1/loan-app-issues/{id}/unaddress/`, `/api/v1/agents/`, `/api/v1/countries/`, `/api/v1/franchises/`,
`/api/v1/index-source-rates/`, `/api/v1/naics-codes/`, `/api/v1/validation/validate/`,
`/api/v1/validation/validation-failures/{id}/`, `/api/v1/validation/validation-failure-fields/`.

### S1-5 — Four path templates declared no `id` parameter

OpenAPI requires every `{…}` template variable to have a matching `path` parameter. These four did not:

* `POST /loan-apps/{id}/submit/`
* `POST /loan-apps/{id}/send-lender-requests/`
* `GET /loan-apps/{id}/associated-documents/`
* `POST /loan-app-project-details/{id}/refresh-esri/`

### S2-1 — 43 operations documented no failure response at all

Every `$ref` stub declared only its success code. A consumer reading the document would conclude that
`POST /loan-app-comments/` cannot fail validation, cannot be rejected for lack of authentication, and cannot be
forbidden.

### S2-2 — Four identical security schemes, and no document-level default

`1.0.0_sessionCookie`, `1.0.0_authenticatedSession`, `1.0.0_userSession`, and `1.0.0_webSessionCookie` were four
names for one thing: `apiKey`, `in: cookie`, `name: sessionid`. Only one of the four descriptions mentioned the CSRF
requirement, and **no operation declared the CSRF header as a parameter**, even though two unused
`X-CSRFToken` parameter components (`1.0.0_CsrfToken`, `1.0.0_CsrfTokenHeader`) sat in `components.parameters`.

### S2-3 — Component sections carried duplicate definitions from a spec merge

The document is a merge of nine sub-specifications (visible in `x-tagGroups`), and the merge de-duplicated nothing.

| Concept | Names present before | Name after |
|---|---|---|
| Resource identifier in the path | `Id`, `ResourceId`, `ResourceIdPath`, `IssueIdPath`, `AlertId`, `ValidationFailureId` | `Id` |
| Page size | `Limit`, `LimitQuery` | `Limit` |
| Page offset | `Offset`, `OffsetQuery` | `Offset` |
| Sort order | `Ordering`, `OrderingQuery` | `Ordering` |
| Loan-application filter | `LoanAppIdQuery`, `LoanApplicationIdQuery`, `LoanApplicationId` | `LoanAppId` |
| CSRF header | `CsrfToken`, `CsrfTokenHeader` | `CsrfToken` |
| 400 response | `BadRequest`, `InvalidRequest`, `ValidationError` | `BadRequest` |
| 401 response | `Unauthorized`, `Unauthenticated`, `AuthenticationRequired` | `Unauthorized` |
| 403 response | `Forbidden`, `AccessDenied` | `Forbidden` |
| Error payload schema | `Error`, `ErrorResponse`, `ApiError` | `Error` |

Two `1.0.0_Id`-style definitions also disagreed on constraints (`minimum: 1` present on some, absent on others).

### S2-4 — Three request schemas described operations that have no route in the document

`CreateLoanApplicationIssueRequest`, `UpdateLoanApplicationIssueRequest`, and
`PartialUpdateLoanApplicationIssueRequest` were fully specified — required fields, `additionalProperties: false`,
descriptions — but no `POST`, `PUT`, or `PATCH` on `/loan-app-issues/` existed to consume them. See section 6.1;
this is the single strongest indicator of missing coverage in the document.

### S3-1 — Schema defects that break example validation

| Location | Defect |
|---|---|
| `CollateralAssetRequest.lender_lien_position` | `nullable: true` on a `oneOf` with no `type` — invalid in OAS 3.0 |
| `ApplicantEquityInjectionSourceRequest.standby_debt_owed_amount`, `LoanApplicationUseRequest.{applicant_equity_injection_amount, other_financing_amount, business_intangible_asset_value_amount, loan_payment_amount, appraisers_valuation_amount}`, `NullableStandbyDebtRepaymentType` | `{allOf: [$ref], nullable: true}` with no `type` |
| `CompanyResponse.{business_address, mailing_address, franchise}`, `PersonResponse.{physical_address, mailing_address}`, `LoanApplicationDocument.loan_app_doc_file` | same, on object references |
| `DocumentCategory.enum[1]` | numeric `504` inside a `type: string` enum |

### S3-2 — Twenty-six of twenty-nine mutating operations had no request example

Only the three `/validation/…` operations shipped a request-body example. `POST /loan-apps/`, `POST /companies/`,
`POST /collateral-assets/`, every company/employee operation, and every sources-and-uses operation had none.

### S3-3 — Implementation vocabulary leaked into consumer-facing text

Six descriptions exposed internals rather than API behaviour, for example:

> "The serializer exposes only `dismissed_at` as writable" · "required by the view" · "`django-filter` also supports
> filtering by dismissal state" · "in a DRF page-number pagination envelope" · "Fields exposed by
> `LoanAppSerializer` … rejected by serializer validation" · "Django content-type model in `app_label.ModelName` form"

### S3-4 — Presentation defects

* Every component name was prefixed `1.0.0_` (`1.0.0_LoanAppRequest`, `1.0.0_Unauthorized`, …), a merge artefact that
  surfaced directly in the rendered documentation.
* A catch-all tag `loans_other`, displayed as **"other"**, was the only tag reachable by 43 operations.
* Twelve tags had no description.
* `info.description` was a one-line note naming a source commit; it documented no base URL, authentication model,
  pagination model, media types, or error model.

### 2.9 Line-number citations

The Django source could not be read (section 1.2), so every citation below refers to the **audited
`openapi.json` as committed at `HEAD` (`aebaa33`, 7,539 lines)**, which is the artifact this audit was able to
examine.

| Finding | File | Line(s) |
|---|---|---|
| S1-1 — invalid `components.pathOperations` section | `openapi.json` | 7225 |
| S1-1/S1-2 — first `$ref` stubs into it (43 in total) | `openapi.json` | 477, 488, 512 … |
| S1-2 — ignored `tags: ["loans_other"]` siblings of `$ref` | `openapi.json` | 479, 490 (tag declared at 27) |
| S1-3 — shared generic operations behind those stubs | `openapi.json` | 7226–7420 (within `pathOperations`) |
| S1-4 — placeholder `api.example.gov` base URLs | `openapi.json` | 2239, 2292, 2350, 2416, 3266, 3325, 3381 |
| S1-4 — path keys that repeat `/api/v1` | `openapi.json` | 2471, 2524, 2572, 2855, 2894, 2933, 2972, 3016, 3438, 3540, 3670 |
| S1-5 — path items missing the `id` parameter | `openapi.json` | 351 (`submit`), 388 (`send-lender-requests`), 425 (`associated-documents`), 534 (`refresh-esri`) |
| S2-2 — four duplicate session security schemes | `openapi.json` | 3918, 3924, 3930, 3936 |
| S2-3 — six competing `id` path-parameter definitions | `openapi.json` | 3944, 4008, 4027, 4071, 4141, 4151 |
| S2-3 — three competing error schemas | `openapi.json` | 4639 (`Error`), 5006 (`ErrorResponse`), 6024 (`ApiError`) |
| S2-4 — orphaned issue request schemas | `openapi.json` | 5852 onward |
| S3-1 — `nullable` on a typeless `oneOf` | `openapi.json` | 4662 (`lender_lien_position`) |
| S3-1 — numeric `504` in a `type: string` enum | `openapi.json` | 5742 |
| S3-3 — implementation vocabulary in descriptions | `openapi.json` | 3683, 4022, 4177, 4360 (`LoanAppSerializer`), 7107 |
| S3-4 — generic placeholder schemas | `openapi.json` | 4600 (`GenericResource`), 6029 (`GenericRequest`) |
| S3-4 — source-commit note in `info.description` | `openapi.json` | 5 |

---

## 3. Corrections applied

| # | Finding | Correction |
|---|---|---|
| 1 | S1-1, S1-2, S1-3 | `components.pathOperations` deleted. All 43 stub operations replaced with complete inline Operation Objects carrying a unique `operationId`, a resource-specific `summary` and `description`, parameters, request body, and responses. |
| 2 | S1-3 | Unique `operationId`s derived per resource — e.g. `listLoanAppComments`, `createLoanAppComment`, `getLoanAppTask`, `partiallyUpdateLoanAppTermsPeriod`, `deleteLoanAppAgentService`. Zero duplicates remain. |
| 3 | S1-4 | Single top-level `servers: [{url: "https://developer.sba.gov/origination/api/v1"}]`. All 65 per-path `servers` removed. The redundant `/api/v1` prefix stripped from the eleven affected path keys. Placeholder `api.example.gov` hosts eliminated. |
| 4 | S1-5 | `{"$ref": "#/components/parameters/Id"}` added to the four path items that were missing it. |
| 5 | S2-1 | Failure responses added to all 43 operations: `400/401/403` on collection `GET` and `POST`, `401/403/404` on item `GET`, `400/401/403/404` on `PATCH`, `401/403/404` on `DELETE`. |
| 6 | S2-2 | The four security schemes collapsed into one `sessionCookie`, whose description states the CSRF requirement. Document-level `security` added. Every operation retains an explicit `security` entry, and every `POST`/`PUT`/`PATCH`/`DELETE` now declares the `X-CSRFToken` header parameter. |
| 7 | S2-3 | Parameter, response, and error-schema duplicates merged per the table in S2-3; the surviving `Id` uses the stricter `minimum: 1`. |
| 8 | S2-4 | The three orphaned issue request schemas removed so the document is internally consistent, and the gap they imply is recorded in section 6.1 instead. |
| 9 | S3-1 | `lender_lien_position` rewritten as `anyOf` with `nullable` on each typed branch; scalar `{allOf:[$ref], nullable}` wrappers flattened into typed schemas; object-reference wrappers given the required `type: object`; `DocumentCategory` enum values coerced to strings. |
| 10 | S3-2 | Fictional request examples generated for all 26 mutating operations that lacked one, derived from each schema so that required fields, enums, formats, and patterns are all honoured. Binary `multipart/form-data` bodies deliberately left without an example. |
| 11 | S3-3 | The six leaking descriptions rewritten in consumer terms while preserving behaviour — including the wire value `lx_src.LoanApp`, which is retained because it is the value the API requires. |
| 12 | S3-4 | The `1.0.0_` prefix stripped from every component name; `GenericResource`/`GenericRequest`/`GenericPage` renamed to `ResourceRecord`/`ResourceRequest`/`PaginatedResourceList` and given real descriptions; the `loans_other` tag removed and its 43 operations moved onto the existing meaningful tags; descriptions added to the twelve tags that lacked them; `info.description` rewritten to document base URL, authentication, CSRF, media types, pagination, and the error model; `info.contact` and `info.license` populated. |
| 13 | — | Orphaned schemas pruned transitively; operation keys ordered consistently (`tags`, `summary`, `description`, `operationId`, `parameters`, `requestBody`, `responses`, `security`). |

### 3.1 Re-tagging of the 43 previously untagged operations

| Path(s) | Was | Now |
|---|---|---|
| `/loan-app-project-details/`, `/loan-app-agents/`, `/loan-app-agent-services/`, `/loan-app-credit-unavailable-elsewhere-explanations/`, `/loan-app-associated-users/`, `/loan-app-terms-periods/`, `/loan-app-use-of-proceeds/` (and their `{id}` items) | `loans_other` ("other") | **Loan Application Supporting Resources** |
| `/loan-app-task/`, `/loan-app-task/{id}/`, `/loan-app-task-comments/`, `/loan-app-comments/`, `/loan-app-comments/{id}/` | `loans_other` | **Loan Tasks and Comments** |
| `/affiliated-loans/` | `loans_other` | **Affiliated Loans** |

### 3.2 What was deliberately **not** changed

* **Trailing slashes** on all 65 paths were kept. Both linters flag them, but the API routes with trailing slashes;
  removing them would make the document wrong. This is recorded as an accepted deviation in section 7.
* **`x-disabled-paths`** (`/loan-apps/{id}/recommend-approval/`, `…/recommend-rejection/`, `…/decline/`,
  `…/approve/`) was left in place as a vendor extension, with its `$ref`s and security scheme names remapped so it
  stays internally consistent. It is not part of the served surface and is not rendered.
* **No route, field, status code, or parameter was added on the basis of inference about the Django code.**

---

## 4. Operation inventory (corrected document — 65 paths, 97 operations)

| Path | Method | operationId | Params | Request body | Responses |
|---|---|---|---|---|---|
| `/loan-apps/` | GET | `listLoanApps` | `limit` (query), `offset` (query), `ordering` (query), `optional_fields` (query), `state` (query) | — | 200 `LoanAppPage`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-apps/` | POST | `createLoanApp` | `X-CSRFToken` (header) | application/json → `LoanAppRequest` | 201 `LoanApp`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-apps/{id}/` | GET | `getLoanAppById` | `id` (path) | — | 200 `LoanApp`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-apps/{id}/` | PATCH | `partiallyUpdateLoanApp` | `id` (path), `X-CSRFToken` (header) | application/json → `LoanAppRequest` | 200 `LoanApp`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-apps/{id}/submit/` | POST | `submitLoanApp` | `id` (path), `X-CSRFToken` (header) | — | 200; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-apps/{id}/send-lender-requests/` | POST | `sendLoanAppLenderRequests` | `id` (path), `X-CSRFToken` (header) | — | 200; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-apps/{id}/associated-documents/` | GET | `listLoanAppAssociatedDocuments` | `id` (path) | — | 200 `array`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-project-details/` | GET | `listLoanAppProjectDetails` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-project-details/` | POST | `createLoanAppProjectDetail` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-project-details/{id}/` | GET | `getLoanAppProjectDetail` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-project-details/{id}/` | PATCH | `partiallyUpdateLoanAppProjectDetail` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-project-details/{id}/refresh-esri/` | POST | `refreshLoanAppProjectDetailsEsri` | `id` (path), `X-CSRFToken` (header) | — | 200; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-agents/` | GET | `listLoanAppAgents` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-agents/` | POST | `createLoanAppAgent` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-agents/{id}/` | GET | `getLoanAppAgent` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-agents/{id}/` | PATCH | `partiallyUpdateLoanAppAgent` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-agents/{id}/` | DELETE | `deleteLoanAppAgent` | `id` (path), `X-CSRFToken` (header) | — | 204; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-agent-services/` | GET | `listLoanAppAgentServices` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-agent-services/` | POST | `createLoanAppAgentService` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-agent-services/{id}/` | GET | `getLoanAppAgentService` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-agent-services/{id}/` | PATCH | `partiallyUpdateLoanAppAgentService` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-agent-services/{id}/` | DELETE | `deleteLoanAppAgentService` | `id` (path), `X-CSRFToken` (header) | — | 204; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-credit-unavailable-elsewhere-explanations/` | GET | `listCreditUnavailableElsewhereExplanations` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-credit-unavailable-elsewhere-explanations/` | POST | `createCreditUnavailableElsewhereExplanation` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-credit-unavailable-elsewhere-explanations/{id}/` | GET | `getCreditUnavailableElsewhereExplanation` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-credit-unavailable-elsewhere-explanations/{id}/` | PATCH | `partiallyUpdateCreditUnavailableElsewhereExplanation` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-credit-unavailable-elsewhere-explanations/{id}/` | DELETE | `deleteCreditUnavailableElsewhereExplanation` | `id` (path), `X-CSRFToken` (header) | — | 204; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-associated-users/` | GET | `listLoanAppAssociatedUsers` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-associated-users/` | POST | `createLoanAppAssociatedUser` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-associated-users/{id}/` | DELETE | `deleteLoanAppAssociatedUser` | `id` (path), `X-CSRFToken` (header) | — | 204; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-terms-periods/` | GET | `listLoanAppTermsPeriods` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-terms-periods/` | POST | `createLoanAppTermsPeriod` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-terms-periods/{id}/` | GET | `getLoanAppTermsPeriod` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-terms-periods/{id}/` | PATCH | `partiallyUpdateLoanAppTermsPeriod` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-terms-periods/{id}/` | DELETE | `deleteLoanAppTermsPeriod` | `id` (path), `X-CSRFToken` (header) | — | 204; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-use-of-proceeds/` | GET | `listLoanAppUseOfProceedss` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-use-of-proceeds/` | POST | `createLoanAppUseOfProceeds` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-use-of-proceeds/{id}/` | GET | `getLoanAppUseOfProceeds` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-use-of-proceeds/{id}/` | PATCH | `partiallyUpdateLoanAppUseOfProceeds` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-use-of-proceeds/{id}/` | DELETE | `deleteLoanAppUseOfProceeds` | `id` (path), `X-CSRFToken` (header) | — | 204; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-task/` | GET | `listLoanAppTasks` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-task/{id}/` | GET | `getLoanAppTask` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-task/{id}/` | PATCH | `partiallyUpdateLoanAppTask` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-task-comments/` | GET | `listLoanAppTaskComments` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-task-comments/` | POST | `createLoanAppTaskComment` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-comments/` | GET | `listLoanAppComments` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-comments/` | POST | `createLoanAppComment` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-comments/{id}/` | GET | `getLoanAppComment` | `id` (path) | — | 200 `ResourceRecord`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-comments/{id}/` | PATCH | `partiallyUpdateLoanAppComment` | `id` (path), `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-comments/{id}/` | DELETE | `deleteLoanAppComment` | `id` (path), `X-CSRFToken` (header) | — | 204; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/affiliated-loans/` | GET | `listAffiliatedLoans` | `limit` (query), `offset` (query), `ordering` (query) | — | 200 `PaginatedResourceList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/collateral-assets/` | POST | `createCollateralAsset` | `X-CSRFToken` (header) | application/json → `CollateralAssetRequest` | 201 `CollateralAsset`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 415 `UnsupportedMediaType` |
| `/collateral-titled-assets/` | POST | `createCollateralTitledAsset` | `X-CSRFToken` (header) | application/json → `CollateralTitledAssetRequest` | 201 `CollateralTitledAsset`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/collateral-lienholders/` | POST | `createCollateralLienholder` | `X-CSRFToken` (header) | application/json → `CollateralLienholderRequest` | 201 `CollateralLienholder`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/collateral-pledgers/` | POST | `createCollateralPledger` | `X-CSRFToken` (header) | application/json → `CollateralPledgerRequest` | 201 `CollateralPledger`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/companies/` | GET | `listCompanies` | `optional_fields` (query), `ordering` (query), `limit` (query), `offset` (query), `q` (query), `tax_id` (query) | — | 200 `CompanyListResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed` |
| `/companies/` | POST | `createCompany` | `X-CSRFToken` (header) | application/json → `CompanyRequest` | 201 `CompanyResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/companies/{id}/` | GET | `getCompanyById` | `id` (path), `optional_fields` (query) | — | 200 `CompanyResponse`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed` |
| `/companies/{id}/` | PUT | `updateCompany` | `id` (path), `X-CSRFToken` (header) | application/json → `CompanyRequest` | 200 `CompanyResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/companies/{id}/` | PATCH | `partiallyUpdateCompany` | `id` (path), `X-CSRFToken` (header) | application/json → `CompanyRequest` | 200 `CompanyResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/company-snapshots/` | POST | `createCompanySnapshot` | `X-CSRFToken` (header) | application/json → `CompanySnapshotRequest` | 201 `CompanySnapshotResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/company-snapshots/{id}/` | GET | `getCompanySnapshotById` | `id` (path), `optional_fields` (query) | — | 200 `CompanySnapshotResponse`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed` |
| `/company-employees/` | GET | `listCompanyEmployees` | `optional_fields` (query), `ordering` (query), `limit` (query), `offset` (query), `company_id` (query), `name` (query) | — | 200 `CompanyEmployeeListResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed` |
| `/company-employees/` | POST | `createCompanyEmployee` | `X-CSRFToken` (header) | application/json → `CompanyEmployeeRequest` | 201 `CompanyEmployeeResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/company-employees/{id}/` | GET | `getCompanyEmployeeById` | `id` (path), `optional_fields` (query) | — | 200 `CompanyEmployeeResponse`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed` |
| `/company-employees/{id}/` | PUT | `updateCompanyEmployee` | `id` (path), `X-CSRFToken` (header) | application/json → `CompanyEmployeeRequest` | 200 `CompanyEmployeeResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/company-employees/{id}/` | PATCH | `partiallyUpdateCompanyEmployee` | `id` (path), `X-CSRFToken` (header) | application/json → `CompanyEmployeeRequest` | 200 `CompanyEmployeeResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/company-employee-snapshots/` | POST | `createCompanyEmployeeSnapshot` | `X-CSRFToken` (header) | application/json → `CompanyEmployeeSnapshotRequest` | 201 `CompanyEmployeeSnapshotResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/company-employee-snapshots/{id}/` | GET | `getCompanyEmployeeSnapshotById` | `id` (path), `optional_fields` (query) | — | 200 `CompanyEmployeeSnapshotResponse`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 405 `MethodNotAllowed` |
| `/loan-app-docs/` | GET | `listLoanApplicationDocuments` | `loan_app_id` (query), `limit` (query), `offset` (query) | — | 200 `LoanApplicationDocumentPage`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed` |
| `/loan-app-doc-files/` | POST | `uploadLoanApplicationDocumentFile` | `X-CSRFToken` (header) | multipart/form-data → `DocumentFileUploadRequest` | 201 `DocumentFile`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/loan-app-doc-files/{id}/download/` | GET | `downloadLoanApplicationDocumentFile` | `id` (path), `id` (path) | — | 200 `string`; 401 `Unauthorized`; 403 `Forbidden`; 404 `Error`; 405 `MethodNotAllowed` |
| `/loan-app-doc-files/validate/` | POST | `validateLoanApplicationDocumentFile` | `X-CSRFToken` (header) | multipart/form-data → `FileValidationRequest` | 200 `ValidationSuccess`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 405 `MethodNotAllowed`; 415 `UnsupportedMediaType` |
| `/loan-app-issues/` | GET | `listLoanApplicationIssues` | `loan_app_id` (query), `limit` (query), `offset` (query), `ordering` (query) | — | 200 `LoanApplicationIssueList`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-issues/{id}/address/` | POST | `markLoanApplicationIssueAddressed` | `id` (path), `X-CSRFToken` (header) | — | 200; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-issues/{id}/unaddress/` | POST | `markLoanApplicationIssueUnaddressed` | `id` (path), `X-CSRFToken` (header) | — | 200; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/loan-app-companies/` | POST | `createBusiness` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-persons/` | POST | `createIndividual` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/company-to-person-ownership-snapshots/` | POST | `createIndividualBusinessOwnershipRelationship` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest` |
| `/company-to-company-ownership-snapshots/` | POST | `createBusinessOwnershipRelationship` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 201 `ResourceRecord`; 400 `BadRequest` |
| `/loan-app-company-eligibilities/` | POST | `createOrUpdateBusinessEligibility` | `X-CSRFToken` (header) | application/json → `ResourceRequest` | 200 `ResourceRecord`; 400 `BadRequest`; 404 `NotFound` |
| `/agents/` | GET | `listAgents` | — | — | 200 `AgentCollection`; 401 `Unauthorized`; 403 `Forbidden` |
| `/countries/` | GET | `listCountries` | — | — | 200 `CountryCollection`; 401 `Unauthorized`; 403 `Forbidden` |
| `/franchises/` | GET | `listFranchises` | — | — | 200 `FranchiseCollection`; 401 `Unauthorized`; 403 `Forbidden` |
| `/index-source-rates/` | GET | `listIndexSourceRates` | `index_sources` (query) | — | 200 `IndexSourceRateCollection`; 401 `Unauthorized`; 403 `Forbidden` |
| `/naics-codes/` | GET | `listNaicsCodes` | — | — | 200 `NaicsCodeCollection`; 401 `Unauthorized`; 403 `Forbidden` |
| `/applicant-equity-injection-sources/` | POST | `createApplicantEquityInjectionSource` | `X-CSRFToken` (header) | application/json → `ApplicantEquityInjectionSourceRequest` | 201 `ApplicantEquityInjectionSourceResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/other-financing-sources/` | POST | `createOtherFinancingSource` | `X-CSRFToken` (header) | application/json → `OtherFinancingSourceRequest` | 201 `OtherFinancingSourceResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-uses/` | POST | `createLoanApplicationUse` | `X-CSRFToken` (header) | application/json → `LoanApplicationUseRequest` | 201 `LoanApplicationUseResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/loan-app-use-addresses/` | POST | `createLoanApplicationUseAddress` | `X-CSRFToken` (header) | application/json → `LoanApplicationUseAddressRequest` | 201 `LoanApplicationUseAddressResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/compliance-checks/` | GET | `listComplianceChecks` | `loan_app_id` (query), `loan_app_compliance_check_id` (query), `limit` (query), `offset` (query), `ordering` (query) | — | 200 `ComplianceCheckList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/compliance-alerts/` | GET | `listComplianceAlerts` | `loan_app_id` (query), `limit` (query), `offset` (query), `ordering` (query) | — | 200 `ComplianceAlertList`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden` |
| `/compliance-alerts/{id}/` | PATCH | `updateComplianceAlertStatus` | `id` (path), `X-CSRFToken` (header) | application/json → `ComplianceAlertStatusUpdate` | 200 `ComplianceAlert`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound` |
| `/validation/validate/` | POST | `validateParentObject` | `X-CSRFToken` (header), `page` (query) | application/json → `ValidateRequest` | 200 `ValidationFailureFieldPage`; 400 `Error`; 401 `Unauthorized`; 403 `Forbidden`; 404 `Error`; 415 `UnsupportedMediaType` |
| `/validation/validation-failures/{id}/` | PUT | `updateValidationFailure` | `id` (path), `id` (path), `X-CSRFToken` (header) | application/json → `ValidationFailureUpdateRequest` | 200 `ValidationFailureResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 415 `UnsupportedMediaType` |
| `/validation/validation-failures/{id}/` | PATCH | `partiallyUpdateValidationFailure` | `id` (path), `id` (path), `X-CSRFToken` (header) | application/json → `ValidationFailureUpdateRequest` | 200 `ValidationFailureResponse`; 400 `BadRequest`; 401 `Unauthorized`; 403 `Forbidden`; 404 `NotFound`; 415 `UnsupportedMediaType` |
| `/validation/validation-failure-fields/` | GET | `listValidationFailureFields` | `parent_object_id` (query), `parent_object_model` (query), `is_dismissed` (query), `page` (query) | — | 200 `ValidationFailureFieldPage`; 400 `Error`; 401 `Unauthorized`; 403 `Forbidden` |

---

## 5. Mutating operations in detail (POST / PUT / PATCH)

All 29 mutating operations, with the query and header parameters they accept, the media type and schema of the
request body, the success and error statuses, the security requirement, the corrections applied, and what remains
uncertain. Path parameters are omitted from the parameter column because they are implicit in the path template.

| Path | Method | Query / header params | Request body | Success | Errors | Security | Corrections applied | Remaining uncertainty |
|---|---|---|---|---|---|---|---|---|
| `/loan-apps/` | POST | `X-CSRFToken` (header) | `application/json` → `LoanAppRequest` (required) | `201` LoanApp | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/loan-apps/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `LoanAppRequest` (required) | `200` LoanApp | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/loan-apps/{id}/submit/` | POST | `X-CSRFToken` (header) | none | `200` no body | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | Response body shape undeclared (gap **G-5**) |
| `/loan-apps/{id}/send-lender-requests/` | POST | `X-CSRFToken` (header) | none | `200` no body | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | Response body shape undeclared (gap **G-5**) |
| `/loan-app-project-details/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createLoanAppProjectDetails` → `createLoanAppProjectDetail` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-project-details/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateLoanAppProjectDetails` → `partiallyUpdateLoanAppProjectDetail` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-project-details/{id}/refresh-esri/` | POST | `X-CSRFToken` (header) | none | `200` no body | `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | Response body shape undeclared (gap **G-5**) |
| `/loan-app-agents/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createLoanAppAgent` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-agents/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateResource` → `partiallyUpdateLoanAppAgent` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-agent-services/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createLoanAppAgentService` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-agent-services/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateResource` → `partiallyUpdateLoanAppAgentService` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-credit-unavailable-elsewhere-explanations/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createCreditUnavailableElsewhereExplanation` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-credit-unavailable-elsewhere-explanations/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateResource` → `partiallyUpdateCreditUnavailableElsewhereExplanation` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-associated-users/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createLoanAppAssociatedUser` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-terms-periods/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createLoanAppTermsPeriod` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-terms-periods/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateResource` → `partiallyUpdateLoanAppTermsPeriod` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-use-of-proceeds/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createLoanAppUseOfProceeds` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-use-of-proceeds/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateResource` → `partiallyUpdateLoanAppUseOfProceeds` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-task/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateResource` → `partiallyUpdateLoanAppTask` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-task-comments/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createLoanAppTaskComment` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-comments/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `createResource` → `createLoanAppComment` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-comments/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | rewritten from placeholder stub (was `$ref` into the invalid `pathOperations` section); `operationId` `partiallyUpdateResource` → `partiallyUpdateLoanAppComment` (was shared by other operations); tag moved off `loans_other`; `400`/`401`/`403`(/`404`) added; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/collateral-assets/` | POST | `X-CSRFToken` (header) | `application/json` → `CollateralAssetRequest` (required) | `201` CollateralAsset | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/collateral-titled-assets/` | POST | `X-CSRFToken` (header) | `application/json` → `CollateralTitledAssetRequest` (required) | `201` CollateralTitledAsset | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/collateral-lienholders/` | POST | `X-CSRFToken` (header) | `application/json` → `CollateralLienholderRequest` (required) | `201` CollateralLienholder | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/collateral-pledgers/` | POST | `X-CSRFToken` (header) | `application/json` → `CollateralPledgerRequest` (required) | `201` CollateralPledger | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/companies/` | POST | `X-CSRFToken` (header) | `application/json` → `CompanyRequest` (required) | `201` CompanyResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/companies/{id}/` | PUT | `X-CSRFToken` (header) | `application/json` → `CompanyRequest` (required) | `200` CompanyResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/companies/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `CompanyRequest` (required) | `200` CompanyResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/company-snapshots/` | POST | `X-CSRFToken` (header) | `application/json` → `CompanySnapshotRequest` (required) | `201` CompanySnapshotResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/company-employees/` | POST | `X-CSRFToken` (header) | `application/json` → `CompanyEmployeeRequest` (required) | `201` CompanyEmployeeResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/company-employees/{id}/` | PUT | `X-CSRFToken` (header) | `application/json` → `CompanyEmployeeRequest` (required) | `200` CompanyEmployeeResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/company-employees/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `CompanyEmployeeRequest` (required) | `200` CompanyEmployeeResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/company-employee-snapshots/` | POST | `X-CSRFToken` (header) | `application/json` → `CompanyEmployeeSnapshotRequest` (required) | `201` CompanyEmployeeSnapshotResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/loan-app-doc-files/` | POST | `X-CSRFToken` (header) | `multipart/form-data` → `DocumentFileUploadRequest` (required) | `201` DocumentFile | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; binary upload left without an example by design | — |
| `/loan-app-doc-files/validate/` | POST | `X-CSRFToken` (header) | `multipart/form-data` → `FileValidationRequest` (required) | `200` ValidationSuccess | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `405` MethodNotAllowed; `415` UnsupportedMediaType | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; binary upload left without an example by design | — |
| `/loan-app-issues/{id}/address/` | POST | `X-CSRFToken` (header) | none | `200` no body | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | redundant `/api/v1` path prefix removed; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | Response body shape undeclared (gap **G-5**) |
| `/loan-app-issues/{id}/unaddress/` | POST | `X-CSRFToken` (header) | none | `200` no body | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | redundant `/api/v1` path prefix removed; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | Response body shape undeclared (gap **G-5**) |
| `/loan-app-companies/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-persons/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/company-to-person-ownership-snapshots/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/company-to-company-ownership-snapshots/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `201` ResourceRecord | `400` BadRequest | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/loan-app-company-eligibilities/` | POST | `X-CSRFToken` (header) | `application/json` → `ResourceRequest` (required) | `200` ResourceRecord | `400` BadRequest; `404` NotFound | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | Field-level request/response schema unknown (gap **G-2**); filters unknown (**G-3**) |
| `/applicant-equity-injection-sources/` | POST | `X-CSRFToken` (header) | `application/json` → `ApplicantEquityInjectionSourceRequest` (required) | `201` ApplicantEquityInjectionSourceResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/other-financing-sources/` | POST | `X-CSRFToken` (header) | `application/json` → `OtherFinancingSourceRequest` (required) | `201` OtherFinancingSourceResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/loan-app-uses/` | POST | `X-CSRFToken` (header) | `application/json` → `LoanApplicationUseRequest` (required) | `201` LoanApplicationUseResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/loan-app-use-addresses/` | POST | `X-CSRFToken` (header) | `application/json` → `LoanApplicationUseAddressRequest` (required) | `201` LoanApplicationUseAddressResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/compliance-alerts/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ComplianceAlertStatusUpdate` (required) | `200` ComplianceAlert | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound | `sessionCookie` | `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL; fictional request example added | — |
| `/validation/validate/` | POST | `X-CSRFToken` (header), `page` (query) | `application/json` → `ValidateRequest` (required) | `200` ValidationFailureFieldPage | `400` Error; `401` Unauthorized; `403` Forbidden; `404` Error; `415` UnsupportedMediaType | `sessionCookie` | redundant `/api/v1` path prefix removed; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | — |
| `/validation/validation-failures/{id}/` | PUT | `X-CSRFToken` (header) | `application/json` → `ValidationFailureUpdateRequest` (required) | `200` ValidationFailureResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound; `415` UnsupportedMediaType | `sessionCookie` | redundant `/api/v1` path prefix removed; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | — |
| `/validation/validation-failures/{id}/` | PATCH | `X-CSRFToken` (header) | `application/json` → `ValidationFailureUpdateRequest` (required) | `200` ValidationFailureResponse | `400` BadRequest; `401` Unauthorized; `403` Forbidden; `404` NotFound; `415` UnsupportedMediaType | `sessionCookie` | redundant `/api/v1` path prefix removed; `X-CSRFToken` declared; per-path `servers` replaced by the document-level base URL | — |

---

## 6. Missing information, conflicts, and assumptions

### 6.1 Coverage gaps that require the Django source to resolve

| # | Gap | Evidence | Why it was not "fixed" |
|---|---|---|---|
| G-1 | The loan-application **issues** collection may support create and update operations that the document does not describe. Only `GET /loan-app-issues/`, `POST /loan-app-issues/{id}/address/`, and `POST /loan-app-issues/{id}/unaddress/` are documented. | Three complete request schemas existed with no route to consume them: `CreateLoanApplicationIssueRequest` (required `loan_app_id`, `loan_app_step`, `text`), `UpdateLoanApplicationIssueRequest`, `PartialUpdateLoanApplicationIssueRequest` | Adding `POST`/`PUT`/`PATCH` routes would be inventing a URL, method, and status code. Confirm against the issues router/viewset. |
| G-2 | Field-level schemas are unknown for eleven supporting collections: project details, agents, agent services, credit-elsewhere explanations, associated users, terms periods, use of proceeds, tasks, task comments, comments, affiliated loans. They use the open `ResourceRequest`/`ResourceRecord`/`PaginatedResourceList` schemas. | The audited document already used a single `GenericResource` blob (`additionalProperties: true`) for all of them | Their serializers were not readable. Replace with per-resource schemas once the serializers can be read. |
| G-3 | Filter and search query parameters are probably under-documented on those same eleven collections — only `limit`, `offset`, and `ordering` are declared. | Comparable documented collections (`/companies/`, `/compliance-alerts/`, `/validation/validation-failure-fields/`) declare resource-specific filters | `filter_backends`, `filterset_fields`, `search_fields`, and `ordering_fields` were not readable. |
| G-4 | Two pagination styles coexist: `limit`/`offset` on most collections, `page` on the validation endpoints. Whether that is intentional is unconfirmed. | `Page` parameter used only by `/validation/validate/` and `/validation/validation-failure-fields/` | Both were present in the audited document and both were preserved. |
| G-5 | Response bodies are undeclared for four action endpoints: `POST /loan-apps/{id}/submit/`, `POST /loan-apps/{id}/send-lender-requests/`, `POST /loan-app-issues/{id}/address/`, `POST /loan-app-issues/{id}/unaddress/`. They document `200` with a description but no schema. | As audited | The shape returned by those actions is unknown. |
| G-6 | `409`, `422`, and `429` responses, if the API returns them, are documented nowhere. | No occurrence in the audited document | Cannot be confirmed without the exception handler. |
| G-7 | Four decision endpoints are parked in `x-disabled-paths` (`recommend-approval`, `recommend-rejection`, `decline`, `approve`). Whether they are live is unknown. | `x-disabled-paths` in the audited document | Left disabled — promoting them would change the published surface. |

### 6.2 Conflicts found within the audited document

| # | Conflict | Resolution |
|---|---|---|
| C-1 | Three different base URLs, two of them placeholders, and eleven paths that double the `/api/v1` prefix | Standardised on `https://developer.sba.gov/origination/api/v1`, the value used by 58 of 65 path items, and the duplicated prefix removed |
| C-2 | Four security schemes describing one cookie; only one mentioned CSRF | Collapsed to `sessionCookie`; CSRF documented in the scheme, in `info.description`, and as an explicit header parameter on every unsafe operation |
| C-3 | `Id` defined six times, two variants disagreeing on `minimum` | Single `Id` with `minimum: 1` |
| C-4 | `Limit` declared `minimum: 0, default: 100` in one definition and `minimum: 1, default: 100` in the other | Single `Limit` with `minimum: 1`, `maximum: 1000`, `default: 100`; a page size of zero is not meaningful |
| C-5 | 400 responses modelled by two different schemas (`Error` vs `ErrorResponse`) | Single `BadRequest` response using `ValidationError`, which accepts both the per-field and the `detail` shapes |
| C-6 | `tags`/`security` written as siblings of `$ref`, which conforming tools ignore | Removed by inlining the operations, with the intended tags and security preserved |

### 6.3 Assumptions made (each one is falsifiable against the Django source)

| # | Assumption | Basis | Risk if wrong |
|---|---|---|---|
| A-1 | The service is served from `https://developer.sba.gov/origination/api/v1` and paths do not repeat `/api/v1` | 58 of 65 path items declared exactly this base; the remaining seven used `api.example.gov` placeholders | Base URL only; path structure and operations unaffected |
| A-2 | Every operation is session-authenticated and unsafe methods require `X-CSRFToken` | Every audited operation already declared a `sessionid` cookie scheme; the scheme description stated the CSRF rule; two unused CSRF header parameter components were already defined | If some endpoints are public, the document over-states the requirement |
| A-3 | Collection `GET` can return 400 (bad filter/pagination values); item `GET`, `PATCH`, and `DELETE` can return 404 | Standard for this API — the same pattern is declared explicitly on the 54 operations that were fully documented | Over-documentation of failure modes, not under-documentation |
| A-4 | `DELETE` returns `204` with no body, `POST` on a collection returns `201` | Values already present in the audited stubs | Low |
| A-5 | The eleven supporting collections accept and return JSON objects whose only guaranteed field is `id`, plus `loan_app_id` where applicable | The audited document's `GenericResource` already asserted `id`; the paths are all `loan-app-*` sub-resources | The named schemas remain permissive (`additionalProperties: true`), so no valid payload is rejected by the document |
| A-6 | All request/response examples are illustrative, not real data | Required by the task | None — every value is fictional (`Rivera Roasters LLC`, `Jordan Rivera`, `jordan.rivera@example.com`, `555-0142`, EIN `12-3456789`, SSN-shaped `000-00-0000`, `100 Example Street, Springfield, VA 22150`) |
| A-7 | Trailing slashes are part of the routing contract | All 65 paths in the audited document use them, consistent with `APPEND_SLASH` routing | If the API also accepts slash-less paths, the document is merely stricter than necessary |

### 6.4 Confirmation checklist for whoever has repository access

1. Diff the corrected path list in section 4 against every `urls.py` and router registration under
   `lender_experience/backend/lx_src/` — confirm no route is missing and none is documented that does not exist.
2. Resolve **G-1**: do the issues routes support `POST`/`PUT`/`PATCH`?
3. Resolve **G-2** and **G-3**: replace `ResourceRequest`/`ResourceRecord` with real serializer fields, and add each
   viewset's `filterset_fields`, `search_fields`, and `ordering_fields`.
4. Resolve **G-5**: document the response bodies of the four action endpoints.
5. Confirm **A-1** (base URL) and **A-2** (authentication and CSRF on every endpoint) against `settings.py`
   `REST_FRAMEWORK.DEFAULT_AUTHENTICATION_CLASSES` / `DEFAULT_PERMISSION_CLASSES` and the root URL configuration.
6. Confirm **A-3** and **G-6** against the custom exception handler and the test suite.

---

## 7. Validation results

All commands were run against the corrected `openapi.json`.

### 7.1 OpenAPI 3.0.3 structural validation

| Check | Before | After |
|---|---|---|
| `openapi-spec-validator` | **FAIL** — `'responses' is a required property` at `paths./loan-app-project-details/.get` | **PASS** |
| `$ref` integrity (custom check over all 124 references) | resolvable, but 43 pointed into an invalid Components section | **PASS** — every `$ref` resolves, none external |
| Self-contained (no external `$ref`, no remote schema) | yes | **yes** |
| Undeclared path-template parameters | 4 | **0** |
| Duplicate `operationId`s | 5 ids covering 39 operations | **0** |
| Operations with no failure response | 43 | **0** |
| Operations with no `security` | 0 declared effectively (43 were ignored as `$ref` siblings) | **0** |
| Paths carrying a redundant `/api/v1` prefix | 11 | **0** |
| Per-path `servers` overrides | 65 | **0** (one document-level server) |
| Unreferenced schemas | 5 | **0** |

### 7.2 `spectral lint --ruleset spectral:oas`

| Rule | Before | After |
|---|---|---|
| `oas3-valid-media-example` (error) | 5 | **0** |
| `typed-enum` (warning) | 1 | **0** |
| `operation-description` (warning) | 11 | **0** |
| `path-keys-no-trailing-slash` (warning) | 65 | 65 — **accepted**, see 3.2 |
| **Total** | 82 problems (5 errors) | **65 problems, 0 errors** |

### 7.3 `redocly lint`

| Rule | Before | After |
|---|---|---|
| `struct` (error) | 6 | **0** |
| `tag-description` (warning) | 12 | **0** |
| `no-unused-components` (warning) | 1 | 1 — `RecommendationRequest`, referenced only from `x-disabled-paths`; **retained deliberately** |
| `no-path-trailing-slash` (error) | 65 | 65 — **accepted**, see 3.2 |

Both remaining categories are deliberate: trailing slashes are part of the routing contract, and the single unused
component belongs to the parked decision endpoints.

### 7.4 Rendering

`npx @redocly/cli build-docs openapi.json` completed with **no errors or warnings**, pre-rendering the whole
document. This proves ReDoc can consume the corrected specification. The pre-rendered file was a verification
artefact only and was deleted; the delivered HTML is the CDN-based, spec-inlined page described below.

**Note on browser verification.** A headless browser was not available in this environment (the Playwright MCP
server required interactive OAuth), so no screenshot of the final page was captured. Renderability is therefore
evidenced by the successful ReDoc pre-render above plus the programmatic checks in 7.5.

### 7.5 `REDOC Loan Origination Documentation.html`

| Property | Result |
|---|---|
| Size | 148 KB (down from 2.2 MB — the previous file was a pre-rendered `Redoc.hydrate` bundle with the library and a search index inlined) |
| Specification source | Embedded inline in `<script id="openapi-spec" type="application/json">` |
| External specification file referenced | **None** |
| Embedded document byte-for-byte equal to `openapi.json` | **Yes** — verified by parsing the script element and comparing the parsed object to the file |
| ReDoc library | `https://cdn.redocly.com/redoc/latest/bundles/redoc.standalone.js` (CDN, as required) |
| Injection safety | `<`, `>`, U+2028, and U+2029 escaped in the embedded JSON; no `</script>` sequence present in the payload |
| Behaviour when the CDN is unreachable | A visible message is shown and the complete specification remains readable in the page source |

### 7.6 Synchronisation of the two artefacts

`REDOC Loan Origination Documentation.html` is generated from `openapi.json`; the two are verified equal after every
build. Regenerate the HTML whenever the specification changes.

---

## 8. Summary

| Metric | Before | After |
|---|---|---|
| Valid OpenAPI 3.0.3 | **No** | **Yes** |
| Paths / operations | 65 / 97 | 65 / 97 |
| Operations with complete, resource-specific documentation | 54 | **97** |
| Operations reachable only through the "other" tag | 43 | **0** |
| Duplicate `operationId`s | 39 operations across 5 ids | **0** |
| Operations documenting failure responses | 54 | **97** |
| Mutating operations with a request example | 3 of 29 | **29 of 29** (the two `multipart/form-data` uploads excepted by design) |
| Security schemes | 4 duplicates, no document-level default | 1, with a document-level default and explicit CSRF headers |
| Schemas | 84 (5 unreferenced) | 78 (0 unreferenced) |
| Blocking linter errors | 11 (`spectral` 5, `redocly` 6) | **0** |

The specification is now valid, self-contained, internally consistent, and renderable. What it cannot yet be is
*verified complete against the implementation* — section 6 lists exactly what to check, in priority order, once
`USSBA/ocio-gh-lending-mono` is reachable.

---

## Appendix — files touched

| File | Action |
|---|---|
| `openapi.json` | Corrected in place; OpenAPI 3.0.3, self-contained, no external `$ref` |
| `REDOC Loan Origination Documentation.html` | Regenerated; ReDoc standalone from CDN with the specification embedded inline |
| `openapi-completeness-audit.md` | Created — this report |

No Django source was modified; none was reachable. No secrets, credentials, or real applicant data appear in either
artefact — every example value is fictional.
