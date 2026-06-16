# Cloud Backend

Google Cloud / Firebase backend. Serverless only — no VMs, no Kubernetes, no
self-hosted inference.

| Service | Purpose |
|---|---|
| Firebase Auth | User authentication |
| Firestore | Scan-session documents enriched with AI labels |
| Google Cloud Storage | Raw RF capture files, firmware images |
| Cloud Functions (Python) | Gemini API proxy, OTA signing, data-enrichment triggers |
| Vertex AI / Gemini API | Heavy inference, report generation |

| Path | Contents |
|---|---|
| `functions/` | Firebase Cloud Functions (Python) |

The Firestore document schema (requirements §7.1) depends on the identifier and
retention decisions in [`../docs/gdpr.md`](../docs/gdpr.md).
