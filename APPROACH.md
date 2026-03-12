# Approach Document

## 1. Initial Understanding
[cite_start]The core challenge is building a high-throughput PDF service (30k/mo) on a restricted budget ($150/mo). The primary "hard parts" were:
- [cite_start]**Resource Intensity:** Puppeteer's ~400MB RAM usage per render against 1,000 requests/min spikes[cite: 18, 20].
- [cite_start]**Resilience:** Preventing OOM crashes on 500+ line item documents[cite: 31].
- [cite_start]**Data Consistency:** Ensuring "Point-in-Time" accuracy for bulk runs that take several minutes[cite: 35].

## 2. Assumptions & Clarifying Questions
- [cite_start]**Assumption:** The Monolith can be modified to push full JSON snapshots to S3 rather than just document IDs[cite: 28].
- [cite_start]**Assumption:** Customers prefer a resumable ZIP download via a pre-signed link over a direct stream due to connectivity issues[cite: 33].
- **Clarifying Question:** What is the legal retention period for compliant archiving? (Affects long-term S3 storage costs) [cite_start][cite: 21, 37].

## 3. Capacity Planning & Math
- [cite_start]**Peak Load:** ~2,000 PDFs/min (1,000 single + 1,000 from 10 bulk requests)[cite: 18].
- [cite_start]**Compute Needs:** At 2.5s per PDF, we need ~85 concurrent browser instances to handle peak without queuing[cite: 20].
- **Budget Realism:** 34GB+ RAM (required for 85 instances) on a standard instance exceeds the $150 limit.
- [cite_start]**Solution:** Use SQS to buffer spikes and process at a steady state using 2x t3.medium Spot instances to stay within budget.

## 4. Design Decisions

### Decision 1: Claim Check Pattern (S3 Snapshots)
- **Alternatives considered:** Direct DB lookups, RDS Versioning.
- **Chosen approach:** Monolith uploads JSON to S3; SQS message contains the S3 URL.
- [cite_start]**Why:** Solves the "Stale Data" issue (Requirement 4) by freezing data at the moment of request[cite: 35, 36].
- **Tradeoff accepted:** Storage overhead for temporary snapshot files.

### Decision 2: Chunked Rendering for Large Docs
- **Alternatives considered:** Vertically scaling EC2 (Too expensive).
- **Chosen approach:** Split documents with 100+ lines into separate renders and stitch with `pdf-lib`.
- [cite_start]**Why:** Prevents the OOM crashes explicitly mentioned for 500+ line items[cite: 30, 31].
- **Tradeoff accepted:** Increased CPU overhead for PDF merging.

### Decision 3: Worker-Led Streaming ZIP
- **Alternatives considered:** AWS Lambda zipping.
- **Chosen approach:** The worker finishing the final task streams the ZIP from S3 back to S3.
- **Why:** Avoids RDS connection limits and Lambda cold starts. [cite_start]Supports resumable downloads via S3 Range headers for unreliable connections[cite: 33, 34].
- **Tradeoff accepted:** Occupies one worker thread for I/O during the finalization of a bulk run.

## 5. AI Usage Log
- **Tool:** Gemini
- **What I asked:** How to handle 500+ line items causing OOM in Puppeteer while staying under $150.
- **What it suggested:** Chunked rendering and using Spot instances.
- **What I did with it:** Integrated chunking as a core architectural rule for large POs/Invoices.

## 6. Weaknesses & Future Improvements
- **Weakness:** Under 10x load, the single RDS instance will become a bottleneck for status updates.
- **Improvement:** Implement an RDS Proxy and migrate to an auto-scaling ECS Fargate cluster.

## 7. One Thing the Problem Statement Didn't Mention
- **Idempotency Keys:** Without unique request keys, a network retry could trigger duplicate PDF generation and double the S3 storage/bandwidth costs.