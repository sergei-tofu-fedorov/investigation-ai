# 2026-06-07 — Investigate gcp log trace="projects/inv-project/traces/c23f65a67d88734068d761367a437f0c" (run 3b20f7e113164fcbb973c317c2e87886)

succeeded · by eddy · 3m56s · tags: area:invoices, kind:client-bug, service:invoices-api, source:mixed

## Findings
1. Trace c23f65a67d88734068d761367a437f0c is an expected business 400, not a backend fault: an iOS client (app 3.9.45) PUT /api/invoices with attachments; one attachment contentId ad0aa7ef-... had no object in the temp GCS bucket, so ContentsService.UploadContentFromSignedUrl threw ContentNotFoundException, mapped to HTTP 400 content_not_found. Request took 148ms in-app. Isolated to account sj0fl6vm70 (4 retries/24h), no breadth. Root cause is an incomplete/expired temp upload from the client, surfaced correctly by the BFF. (confidence 95)
   - citations: [{"ref": "trace=\"projects/inv-project/traces/c23f65a67d88734068d761367a437f0c\" (inv-project, invoices-api, 400 content_not_found, 2026-06-07T13:12:20Z)", "kind": "log-query"}, {"ref": "resource.labels.container_name=\"invoices-api\" jsonPayload.properties.ResponseBodyText:\"content_not_found\" freshness=24h -> 4 hits, all account sj0fl6vm70", "kind": "log-query"}, {"ref": "Invoices.Backend/Src/Invoices.Implementation.Services/Contents/ContentsService.cs:104-108", "kind": "code"}, {"ref": "Invoices.Backend/Src/Invoices.Api/Middleware/ApiExceptionHandlingMiddleware.cs:95-97", "kind": "code"}, {"ref": "Invoices.Backend/Src/Invoices.Api/Middleware/ErrorCode.cs:46-48", "kind": "code"}]
   - fingerprint: err:8ff3308d6bde309d05e07b3d16373f81

## Limitations
- RequestBodyText truncated at 10KB; the failing attachment ad0aa7ef-... entry itself is not visible in the log.
- Did not inspect the temp_contents_production GCS bucket to distinguish never-uploaded vs expired temp object; Storage access is outside available read-only sources.
