# Design — AI-Powered IPsec VPN Protocol Analyzer

## 1. Scope

This document defines the implementation-level design for the SIH 26160 prototype. The first release focuses on uploaded PCAP/PCAPNG analysis. Live capture and automated testbed generation are later milestones, after offline analysis is reliable.

## 2. User workflow

1. Open the dashboard.
2. Upload an authorized `.pcap` or `.pcapng` capture.
3. Review validation and analysis status.
4. Open the result summary.
5. Inspect protocol observations and evidence-linked findings.
6. Review the security score, evidence coverage, and ML prediction separately.
7. Download the technical report.

## 3. Core modules

### 3.1 Capture intake
Input:
- Capture file.
- Optional analysis label/notes.

Validation:
- Accept only configured file extensions.
- Enforce upload size limits.
- Generate a server-side random ID and safe storage name.
- Reject empty files and malformed captures with a clear error.
- Do not trust the uploaded filename as a filesystem path.

Output:
- `analysis_id`
- status
- upload metadata (file size, hash, received time)

### 3.2 Packet parsing and normalization
The parser should return normalized packet records, not expose parser-specific output directly to the rest of the application.

Initial normalized packet fields:
- `frame_number`
- `timestamp`
- `src_ip`, `dst_ip`
- `ip_version`
- `ip_protocol`
- `src_port`, `dst_port` when present
- `packet_length`
- `is_ike`, `is_esp`, `is_ah`
- `parse_notes`

Not every field exists in every packet. Nullable fields are expected.

### 3.3 Flow and protocol analysis
Group packets into observable flows using available endpoint/protocol metadata. Extract:
- total packets and bytes
- first and last timestamp
- mean/min/max packet size
- inter-arrival-time summaries
- IKE packet/exchange indicators when recognized
- ESP/AH packet counts
- IPv4/IPv6 distribution
- visible endpoint and protocol metadata

Do not assert Tunnel vs Transport mode based only on encrypted ESP payloads unless the parser has sufficient evidence. Record mode as `observed`, `inferred`, or `unknown`, with a reason.

### 3.4 Evidence model
Every meaningful observation should carry provenance. Suggested schema:

```json
{
  "field": "ike.version",
  "value": "IKEv2",
  "status": "observed",
  "source": "pcap",
  "evidence": [
    {
      "frame_number": 12,
      "timestamp": "2026-10-26T10:00:00Z",
      "description": "IKE header parsed"
    }
  ]
}
```

Allowed status values:
- `observed`: directly parsed from the capture.
- `configured`: read from an explicitly supplied lab/config file.
- `inferred`: estimated from indirect features.
- `unknown`: insufficient evidence.
- `not_applicable`: field does not apply.

### 3.5 Security rules
Each rule should be a small, testable function that returns a finding or an insufficient-evidence status.

Suggested finding fields:
- `finding_id`
- `rule_id`
- `title`
- `severity`: `critical`, `high`, `medium`, `low`, `info`
- `status`: `confirmed`, `potential`, `insufficient_evidence`
- `description`
- `evidence_refs`
- `recommendation`
- `rule_version`

Initial rule families:
1. **Algorithm policy** — assess algorithm strength only when the algorithm is known.
2. **Key exchange / DH / PFS** — assess observed or configured negotiation details when available.
3. **SA lifetime** — compare known lifetime settings against documented policy.
4. **Replay protection** — report only when state/configuration can be verified.
5. **Protocol visibility** — identify visible IKE/ESP/AH and non-IPsec traffic without overclaiming.
6. **Metadata exposure** — describe observable endpoints, packet timing, sizes, and volume.
7. **Configuration completeness** — list security properties that could not be verified.

Policy values should live in configuration, not be scattered as hard-coded constants. A rule should link to the policy version used.

### 3.6 Security score
Use a transparent, documented score, initially on a 0–100 scale:
- Start at 100.
- Subtract configured penalties for findings by severity.
- Cap each rule's penalty and the total penalty.
- Never subtract a penalty for an unknown value alone.
- Show evidence coverage beside the score.
- Mark the score `provisional` when key security properties are unknown or coverage is low.

Illustrative starting penalties (tune and document after testing):
- Critical: 30
- High: 20
- Medium: 10
- Low: 4
- Informational: 0

The score is a prototype prioritization aid, not a formal certification or guarantee of security. Display the score methodology in the UI and report.

### 3.7 ML traffic-pattern classifier
First establish a labelled dataset generated from a controlled testbed. Potential classes may include broad traffic-pattern categories such as `interactive`, `bulk_transfer`, `streaming_like`, and `icmp_like`, but only include a class if the dataset can label it reliably.

Candidate features:
- packet size summaries
- inter-arrival-time summaries
- packet/byte rates
- burstiness and direction ratios
- flow duration
- packet count distribution

Avoid using raw identifiers or labels that leak the target class. Split train/test data by capture session or testbed run, not random packets from the same capture, to reduce leakage.

Model output:
- predicted class
- per-class probabilities (where supported)
- confidence
- model version
- supported classes
- warning when confidence is low or the input is out-of-distribution

Do not claim that traffic-pattern classification reveals the exact application inside encrypted ESP. Report “traffic pattern estimate,” not “application identified,” unless independently verified.

### 3.8 Assessment aggregation
Combine:
- protocol observations
- security findings
- score and risk band
- evidence coverage
- ML traffic-pattern output (if model available)
- parser/model/rule versions

Keep rule-based security score independent of ML confidence.

### 3.9 Report generation
Report sections:
1. Cover/analysis metadata.
2. Executive summary.
3. Capture and protocol overview.
4. Security score, risk band, and methodology.
5. Findings table (severity, status, evidence, recommendation).
6. Traffic-pattern estimates and confidence, if available.
7. Evidence coverage and unknown fields.
8. Limitations and disclaimer.

HTML is the first output format. PDF can be added after HTML output is stable.

## 4. API design

Base path: `/api/v1`

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | API health check |
| `POST` | `/analyses` | Upload capture and create analysis |
| `GET` | `/analyses/{analysis_id}` | Read status and metadata |
| `GET` | `/analyses/{analysis_id}/summary` | Get protocol summary and score |
| `GET` | `/analyses/{analysis_id}/findings` | List security findings |
| `GET` | `/analyses/{analysis_id}/traffic` | Get traffic/flow statistics |
| `GET` | `/analyses/{analysis_id}/report` | Download generated report when ready |
| `DELETE` | `/analyses/{analysis_id}` | Delete analysis and associated artifacts |

For the first local prototype, analysis may run synchronously for small captures. Keep the response shape compatible with job-based processing so background execution can be added without redesigning the API.

### Example analysis summary response

```json
{
  "analysis_id": "server-generated-id",
  "status": "completed",
  "capture": {
    "packet_count": 1250,
    "ip_versions": ["IPv4", "IPv6"],
    "ike_packets": 14,
    "esp_packets": 1100,
    "ah_packets": 0
  },
  "security": {
    "score": 78,
    "risk_level": "medium",
    "score_status": "provisional",
    "evidence_coverage": 0.62
  },
  "ml": {
    "available": false,
    "reason": "Model not configured"
  }
}
```

Example numbers are illustrative only; never return these as computed results unless supported by the uploaded capture.

## 5. Data model

### Analysis
- `id`
- `original_filename` (display-only; sanitized)
- `stored_path`
- `sha256`
- `status`
- `created_at`, `completed_at`
- `parser_version`
- `error_code` (safe, non-sensitive)

### ProtocolObservation
- `id`
- `analysis_id`
- `field`
- `value_json`
- `status`
- `source`
- `evidence_json`

### Finding
- `id`
- `analysis_id`
- `rule_id`
- `severity`
- `status`
- `title`
- `description`
- `recommendation`
- `evidence_json`
- `rule_version`

### TrafficSummary
- `id`
- `analysis_id`
- `summary_json`

### ModelRun
- `id`
- `analysis_id`
- `model_name`
- `model_version`
- `prediction_json`
- `metrics_reference`

Use database migrations once the schema begins changing; do not rely on manually editing an existing database.

## 6. Frontend design

Pages:
- **Overview:** total analyses, latest analyses, API/system status.
- **New Analysis:** capture upload, validation messages, progress/status.
- **Analysis Results:** score, risk band, evidence coverage, protocol summary.
- **Findings:** filter by severity/status and expand evidence/recommendation.
- **Traffic Analysis:** packet/byte trends and flow summaries.
- **Report:** preview and download.

UI principles:
- Clearly label “Observed,” “Configured,” “Inferred,” and “Unknown.”
- Do not use colour alone to communicate severity; include text labels.
- Always show why a finding exists and what evidence supports it.
- Show ML confidence separately from security score.
- Make empty states and parser errors understandable.

## 7. Error handling

Use stable error codes such as:
- `INVALID_FILE_TYPE`
- `FILE_TOO_LARGE`
- `EMPTY_CAPTURE`
- `UNSUPPORTED_CAPTURE`
- `PARSER_UNAVAILABLE`
- `ANALYSIS_FAILED`
- `REPORT_NOT_READY`
- `ANALYSIS_NOT_FOUND`

Return a short message to the UI and log technical details on the server without exposing stack traces to the user.

## 8. Testing strategy

- Unit tests for parsing normalization, evidence status, each security rule, scoring, and schema validation.
- API tests for upload validation, status/result retrieval, deletion, and error responses.
- Fixture captures from a controlled lab with known expected characteristics.
- ML tests for schema consistency, low-confidence behaviour, and train/test separation.
- Integration tests from upload to report generation.
- Security tests for oversized uploads, path traversal filenames, malformed captures, and command injection resistance.

## 9. Definition of done for the first MVP

- User can upload a valid PCAP/PCAPNG.
- API returns a normalized protocol/traffic summary.
- Findings are evidence-linked and distinguish unknown from confirmed.
- Score methodology and evidence coverage are visible.
- ML is optional and does not fabricate a result when unavailable.
- Dashboard can display results and download an HTML report.
- Tests cover the main success and failure paths.
- Documentation states limitations and how to run the project locally.
