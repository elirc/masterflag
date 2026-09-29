# Learning Rubrics

Observable behaviours, not vibes. Self-assess monthly; "interview-ready" is the bar for saying it confidently in a loop.

## Codebase navigation

| Level | Observable behaviour | Interview-ready when… |
| --- | --- | --- |
| Junior | Finds the file for a named feature within 10 minutes using rg/IDE search | You can narrate *how* you search (routes → views → models), not just that you found it |
| Mid | Predicts where a change must land across FE service + DRF view + model before opening files | You can sketch the repo map from memory ([01-system-map](../01-codebase-cartography/01-system-map.md)) |
| Senior | Identifies which parts are legacy (Flux stores), generated, or externally owned, and steers work away from them | You can say what you'd delete and in what order |

## API & data modelling

| Level | Observable behaviour | Interview-ready when… |
| --- | --- | --- |
| Junior | Reads a DRF view and names method, auth class, serializer | You can define authn vs authz with a repo example |
| Mid | Explains FeatureState's three-way ownership (environment/segment/identity) and the uniqueness rules it implies | You can whiteboard the flags data model unaided ([08/04](../08-interview-prep/04-system-design-from-this-repo.md)) |
| Senior | Weighs the "resolve priority in Python vs SQL" tradeoff and when it breaks | You can propose and defend an alternative schema |

## Frontend

| Level | Observable behaviour | Interview-ready when… |
| --- | --- | --- |
| Junior | Adds a field to a form using existing RTK Query service + types | You can explain what `invalidatesTags` did for you |
| Mid | Designs a new service with correct tag invalidation; avoids stale-closure bugs in hooks | You can explain the FeaturesPage Flux bridge and why it must die |
| Senior | Plans the Flux→RTK migration incrementally with rollback | You can argue cache-key design tradeoffs abstractly and concretely |

## Testing & debugging

| Level | Observable behaviour | Interview-ready when… |
| --- | --- | --- |
| Junior | Writes a Jest test for a pure util; runs one pytest by name | You follow reproduce→narrow→hypothesize aloud without prompting |
| Mid | Writes a DRF integration test using conftest fixtures incl. a permission-denied case | You can name what each test layer (unit/integration/E2E) is *for* here |
| Senior | Adds regression coverage that pins the invariant, not the implementation | You can critique a flaky test and fix its root cause |

## Async & reliability

| Level | Observable behaviour | Interview-ready when… |
| --- | --- | --- |
| Junior | Locates where a webhook actually fires | You can define idempotency with the webhook-retry example |
| Mid | Explains the audit→task→document-rebuild chain and what happens if the processor is down | You can enumerate the failure modes of fire-and-forget |
| Senior | Designs an outbox or retry-with-dedupe for a new side effect | You can discuss delivery guarantees without hand-waving |

## Security & authorization

| Level | Observable behaviour | Interview-ready when… |
| --- | --- | --- |
| Junior | Names the two auth schemes (user token, environment key) | You can define IDOR and point at [get_feature_state_by_uuid](../../api/features/views.py#L992-L1001) as the defense |
| Mid | Traces a permission check from UI `<Permission>` to DRF permission class | You can write the cross-tenant rejection test from memory |
| Senior | Audits a new endpoint for tenant scoping before reading its body | You can rank this repo's authz risks and defend the ranking |
