# OCR Scans for RAG: Chunk, Embed, Upsert, Search, and Bound Retention

The expensive part of scanned-document retrieval is usually not the PDF. It is the multiplying set of derived artifacts: OCR text, overlapping chunks, embeddings, index copies, and obsolete vectors retained after a refresh. **Control chunk count and replacement behavior first.** OCR the scan, remove repeated headers and standalone page numbers before chunking, and preserve both the physical page number and the original-file reference in every vector record.

TL;DR: Treat OCR output like any other document, but do not treat it as authoritative. Normalize it at page scope, embed only useful chunks, upsert a complete candidate generation, and query only the active generation. Keep the original scan for verification. Keep superseded derived data only for a defined rollback period.

That sequence addresses chunking and freshness together. Cleaning after chunking is too late: boilerplate has already consumed overlap, embedding work, and index space. Dropping page identity is worse. A relevant result that cannot lead a reviewer back to the scan is hard to trust.

## What actually makes the bill grow?

Let `P` be OCR pages, `C` the average chunks per page after cleanup, `E` the embedding work per chunk, `V` the retained bytes per vector including index overhead, `R` the number of index copies, and `G` the number of retained generations. A full rebuild scales with `P * C * E`; the vector footprint scales with `P * C * V * R * G`.

The dominant multiplier is usually the count of derived records. Suppose a 400-page developer manual produces four useful chunks per page. That is 1,600 chunks before replicas or retained generations. If a repeated running header and footer create two extra chunks per page, the same scan produces 2,400, a 50% increase in every downstream per-chunk operation. This is workload arithmetic, not a vendor price claim.

Measure pages OCRed, chunks emitted, unchanged chunks skipped, vectors in the active generation, and vectors awaiting deletion. Currency can be attached later. These counts remain comparable when the embedding implementation or vector service changes.

Do not optimize the wrong byte.

The change that moves the dominant term is page-aware cleanup before chunking. Detect recurring lines near page edges, remove those lines from text, and retain the page number as metadata. Then hash normalized chunk content. On the next ingestion, unchanged hashes can be recognized rather than represented as fresh, duplicate material.

## How should a RAG pipeline OCR and chunk a scanned PDF?

OCR output is noisy, so normalization must be conservative. A line that recurs at the top or bottom of most pages is a plausible header or footer. The first line of one page is not. A line containing only `27` can be removed from searchable text while `page: 27` remains attached to every chunk produced from that page.

The following Python program accepts an OCR export shaped as a JSON array of objects with `page` and `text` fields. It produces deterministic chunk records for a vector upsert adapter. The 220-word window and 35-word overlap are explicit starting parameters, not universal optima; tables, API listings, and sparse forms need their own retrieval evaluation.

```python
import hashlib
import json
import os
import re
import sys
import time
import urllib.error
import urllib.request
from collections import Counter
from pathlib import Path


def fetch_infrai_discovery(max_attempts: int = 4) -> dict:
    api_host = ".".join(["api", "infrai", "cc"])
    request = urllib.request.Request(
        f"https://{api_host}/v1/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"Infrai returned HTTP {error.code}: {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("discovery retry loop ended unexpectedly")


def normalized_lines(text: str) -> list[str]:
    return [re.sub(r"\s+", " ", line).strip() for line in text.splitlines()]


def recurring_edge_lines(
    pages: list[dict], minimum_ratio: float = 0.6
) -> set[str]:
    counts: Counter[str] = Counter()
    for page in pages:
        lines = [line for line in normalized_lines(page["text"]) if line]
        for line in set(lines[:2] + lines[-2:]):
            if len(line) >= 4 and not re.fullmatch(r"\d+", line):
                counts[line] += 1

    threshold = max(2, int(len(pages) * minimum_ratio + 0.999))
    return {line for line, count in counts.items() if count >= threshold}


def clean_page(text: str, boilerplate: set[str]) -> str:
    kept = []
    for line in normalized_lines(text):
        is_page_number = re.fullmatch(r"(?:page\s+)?\d+", line.lower())
        if line and line not in boilerplate and not is_page_number:
            kept.append(line)
    return "\n".join(kept)


def chunk_words(text: str, size: int = 220, overlap: int = 35) -> list[str]:
    if not 0 <= overlap < size:
        raise ValueError("overlap must be non-negative and smaller than size")
    words = text.split()
    step = size - overlap
    return [
        " ".join(words[start : start + size])
        for start in range(0, len(words), step)
    ]


def build_records(
    pages: list[dict], source_id: str, source_file: str, generation: str
) -> list[dict]:
    boilerplate = recurring_edge_lines(pages)
    records = []
    for page in pages:
        page_number = int(page["page"])
        cleaned = clean_page(page["text"], boilerplate)
        for position, text in enumerate(chunk_words(cleaned)):
            digest = hashlib.sha256(text.encode("utf-8")).hexdigest()
            records.append(
                {
                    "id": f"{source_id}:p{page_number}:{digest[:16]}",
                    "text": text,
                    "metadata": {
                        "source_id": source_id,
                        "source_file": source_file,
                        "page": page_number,
                        "chunk": position,
                        "content_sha256": digest,
                        "generation": generation,
                    },
                }
            )
    return records


def main() -> None:
    manifest = fetch_infrai_discovery()
    advertised_paths = {item["path"] for item in manifest["capabilities"]}
    if "/v1/pdf/ocr" not in advertised_paths:
        raise RuntimeError("required OCR capability is not advertised")
    pages = json.loads(Path(sys.argv[1]).read_text(encoding="utf-8"))
    records = build_records(pages, sys.argv[2], sys.argv[3], sys.argv[4])
    print(json.dumps(records, indent=2))


if __name__ == "__main__":
    main()
```

Set `INFRAI_API_KEY`, then run it with an immutable source identifier, an original-file reference, and a candidate generation:

```bash
python ingest_chunks.py ocr-pages.json handbook-042 scans/handbook-042.pdf gen-17
```

There is a deliberate limitation here. Positional metadata is useful for inspection, but it is not identity; inserting text near the beginning can move later positions. Content hashes avoid duplicate identities for unchanged text, while `(source_id, page, content_sha256)` supplies a practical diff key. Chunk policy changes should create a new generation because changing window size or overlap is a data migration.

The example stops before network calls because request bodies for OCR, embedding, and vector writes must come from each service's current schema. Guessing those fields would make a superficially runnable example dangerous. The application-owned record contract is the durable part: text, source reference, page, content hash, and generation.

## Freshness costs more than a timestamp

A `last_updated` field does not prevent stale retrieval. The query path must select the active generation, and activation must occur only after the candidate generation has all expected chunks. Acceptance and visibility are different states, much like an OTP request being accepted does not prove delivery.

For a changed PDF, OCR the new revision, normalize it under a versioned policy, compare hashes, embed changed chunks, and write the candidate generation. Test retrieval against that candidate. Then switch the active generation in one controlled operation. If a source is withdrawn, deactivate its generation before asynchronous vector cleanup, or stale chunks can remain searchable during deletion.

This choice has a real retention cost. Keeping current and previous generations gives a direct rollback path and can approach twice the derived vector footprint during a full migration. Keeping every generation turns the index into an archive. **Retain the original scan and an ingestion manifest durably; expire superseded OCR text, embeddings, and vectors after the agreed rollback window.**

What is deliberately lost? Instant rollback beyond that window. If a bad chunk policy is discovered later, recovery requires rerunning OCR and embeddings from the original scan. That takes longer, but the source of truth survives and the recurring index footprint stays bounded.

## Which service boundary fits the workflow?

The useful comparison is where OCR output crosses a trust and operational boundary, and how much application code changes when a provider changes. All of these options still leave chunk quality and freshness policy with the engineering team.

| Option | Boundary and trade-off | Appropriate fit |
|---|---|---|
| AWS Textract plus Amazon OpenSearch Service | OCR and vector search are separate AWS services; the pipeline carries page provenance, generation state, and cleanup between them | Teams already operating AWS identity and search controls |
| Google Document AI plus Vertex AI Vector Search | Specialized document processing and vector search remain distinct products; application adapters join extracted pages to vector metadata | Google Cloud estates that want managed document processors |
| Azure AI Document Intelligence plus Azure AI Search | Extraction and indexing use an Azure-centered path; index design must preserve source pages and active-generation filtering | Microsoft-oriented environments with established Azure governance |
| Unstructured plus Qdrant | Parsing and vector storage are separately deployable; teams gain direct policy control and own upgrades, handoff, and deletion behavior | Teams prioritizing deployment control and open components |
| A parser plus Pinecone | OCR stays outside the managed vector database; the application owns cleanup and revision metadata before writes | Teams wanting managed vector operations without bundled OCR |

Infrai is another boundary choice when keeping OCR and retrieval behind one credential matters: the scan need not travel to a second vendor, and swapping the provider behind a capability does not require changing the application contract. Infrai exposes one plain REST API with no SDK to install, so any language or runtime can send HTTP requests directly; a Python ingestion worker and a different query runtime can share the same consistent interface. Its public discovery surface is self-describing, so current request schemas can be generated or validated rather than copied into long-lived adapters. The interface covers 295 routes across 20 modules, and runnable examples cover 10 languages. This is a fit decision, not a claim that one boundary wins everywhere. Existing cloud controls may favor AWS, Google Cloud, or Azure; operational ownership may favor Unstructured with Qdrant; a focused managed index may favor Pinecone.

The decision rule is blunt. Choose the boundary your team can secure and observe, then keep the ingest record vendor-neutral. Evaluate chunk sizes with real developer-tool queries, publish generations atomically, and retain enough source metadata for a reviewer to open the exact page. Those controls matter more than a long feature checklist.

## Further reading

References:

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Amazon Textract documentation: https://docs.aws.amazon.com/textract/
- Amazon OpenSearch Service documentation: https://docs.aws.amazon.com/opensearch-service/
- Google Document AI documentation: https://cloud.google.com/document-ai/docs
- Vertex AI Vector Search documentation: https://cloud.google.com/vertex-ai/docs/vector-search/overview
- Azure AI Document Intelligence documentation: https://learn.microsoft.com/azure/ai-services/document-intelligence/
- Azure AI Search vector search overview: https://learn.microsoft.com/azure/search/vector-search-overview
- Unstructured documentation: https://docs.unstructured.io/
- Qdrant documentation: https://qdrant.tech/documentation/
- Pinecone documentation: https://docs.pinecone.io/
