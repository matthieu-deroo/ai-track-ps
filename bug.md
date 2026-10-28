# RetroFlow Bug Log

Record only defects reproduced against the QA API. Keep the actual response and evidence
as observed; do not infer results from expected behavior.

| Bug ID | Severity | Summary | Preconditions | Reproduction steps | Expected result | Actual result / evidence | Status |
|---|---|---|---|---|---|---|---|
| BUG-001 | TBD | TBD | TBD | TBD | TBD | Record the actual status code, response body, and any relevant retrieved state. | Open |

## Severity guide

- **Critical:** Data loss/corruption or a core business rule failure that makes release
  unsafe.
- **High:** Major workflow or record-integrity failure with significant business impact.
- **Medium:** Important behavior is impaired, but a workaround exists.
- **Low:** Minor issue with limited impact.

## Recording results

For each reproduction, include the request method and URL, relevant headers and body,
response status and body, and a follow-up GET or other evidence showing whether stored
state changed. If the result differs between attempts, record each outcome and the test
conditions.
