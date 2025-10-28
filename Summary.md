# 🧠 Walr Audience Access — End-to-End Survey Flow

This document describes how the **Audience Access subsystem** coordinates survey distribution, response collection, and quota enforcement.

---

## 🌐 Participants and Components

| Participant | Component | Actions |
|--------------|------------|----------|
| **Walr** | **Orchestrator.Api** | • Creates survey and quotas<br>• Publishes “SurveyOpen” event |
| **Walr** | **ParticipantAdapter** | • Calls each panel provider API synchronously<br><br>**Example payload:**<br>```json<br>{<br>  "surveyId": "SURV-123",<br>  "surveyUrl": "https://survey.walr.io/start?surveyId=SURV-123",<br>  "callbackUrl": "https://edge.walr.io/callbacks/providerX",<br>  "audienceFilters": { "country": "UK", "age": "25-40", "gender": "any" }<br>}``` |
| **Panel Provider** | Email / App / Web | • Shares survey link with respondents |
| **Respondent** | Survey UI | • Opens `https://survey.walr.io/start?surveyId=SURV-123&pid=xyz`<br>• Completes survey<br>• Provider then submits callback to Walr |
| **Walr** | **Edge.CallbackApi** | • `POST /callbacks/{provider}`<br>  ↳ `correlationId = request.Headers["X-Provider-EventId"]`<br>  ↳ `signature = request.Headers["X-Signature"]`<br>• Validate & authenticate request<br>• Publish message → **`callbacks.raw`** topic |
| **Walr** | **Processing.Normalizer.Worker** | • Subscribe → **`callbacks.raw`**<br>• Deserialize provider payload<br>• Transform to normalized format:<br>```csharp<br>public sealed record NormalizedCallbackV1(<br>    string SurveyId,<br>    string Provider,<br>    string ProviderEventId,<br>    OutcomeStatus Status,<br>    string CellKey,<br>    RespondentInfoV1 Respondent,<br>    string Raw<br>);<br>```<br>• Publish → **`callbacks.normalized`** |
| **Walr** | **Processing.Dedup.Worker** | • Subscribe → **`callbacks.normalized`**<br>• Resolve identity:<br>  `externalId = Respondent.ProviderExternalId`<br>  `identity = db.Identities.FirstOrDefaultAsync(...)`<br>• Publish → **`callbacks.deduped`** |
| **Walr** | **Processing.FraudRules.Worker** | • Subscribe → **`callbacks.deduped`**<br>• Apply quality/fraud heuristics<br>• Publish → **`callbacks.qualified`** |
| **Walr** | **Quota.Service.Worker** | • Subscribe → **`callbacks.qualified`**<br>```csharp<br>var idemKey = msg.Provider + \":\" + msg.ProviderEventId;<br>db.Completions.Add(...);<br>var survey = await db.Surveys.Include(s => s.Quotas).FirstOrDefaultAsync(...);<br>var quota = survey.Quotas.First(q => q.CellKey == msg.CellKey);<br>quota.Fulfilled++;<br>await db.SaveChangesAsync();<br>if (survey.Quotas.All(q => q.Fulfilled >= q.Target)) {<br>    survey.State = \"Closed\";<br>    survey.ClosedAtUtc = DateTime.UtcNow;<br>    await db.SaveChangesAsync();<br>}<br>```<br>• Publish → **`progress.updates`** |
| **Walr** | **Progress.Api** | • Subscribe → **`progress.updates`**<br>• Consume `ProgressUpdateV1` and broadcast via **SignalR**<br>• Real-time operator visibility of per-quota completion progress |

---

## Identity deduplication
Handled in Processing.Dedup.Worker:
identity = await db.Identities.FirstOrDefaultAsync(i =>
    i.ProviderExternalId == msg.Respondent.ProviderExternalId
    || i.EmailHash == emailHash
    || i.DeviceHash == deviceHash);


If a match is found → existing IdentityId reused.
Ensures same person = same identity across all callbacks and providers.


## Idempotency (callback replay protection)
Implemented in Quota.Service.Worker:
var idemKey = msg.Provider + ":" + msg.ProviderEventId;
db.Completions.Add(new Completion { IdemKey = idemKey, ... });
await db.SaveChangesAsync();  // UNIQUE constraint on IdemKey

If a provider retries the same callback → SQL UNIQUE constraint blocks duplicate.

Result: callback processed exactly once.


## Quota enforcement (business-level duplicate prevention)

Quota.Service.Worker increments quota only once per unique IdentityId.

If that identity already contributed to that cell, no new count is added.