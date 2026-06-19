# SLS Digital Notes & Actions Register

> Single source of truth for notes and actions between Gian & Nic.

---

| # | Date | Title | Owner | Status |
|---|------|-------|-------|--------|
| 1 | 2026-06-19 | L2CoherenceService.php — Review for logic improvement | Gian | Open |
| 2 | 2026-06-19 | Guidance Document Seeding into RAG Pipeline | Gian → Nic | Open |
| 3 | 2026-06-19 | AI Semantic Contradiction Check for Section Header Hierarchy | Gian → Nic | Open |

---

## Detail

### #1 — L2CoherenceService.php Review
- **Date:** 2026-06-19
- **Owner:** Gian
- **Status:** Open
- **Notes:** Review file for potential logic improvement opportunities.

---

### #2 — Guidance Document Seeding into RAG Pipeline
- **Date:** 2026-06-19
- **Raised by:** Gian le Roux
- **Owner:** Nic van der Walt
- **Status:** Open

#### Background

The system classifies BoQ items using AI with CBS code definitions drawn from `industry_standards.prompt_guidance` and `cbs_codes.prompt_guidance`. These rules were written reactively — each one was added after a QS rejection finding. There is no seeded understanding of what each CBS category *fundamentally means* from an authoritative source.

The QS team holds a library of standards and internal guidance documents that directly define the boundaries of each CBS code — what belongs in Substructure vs Superstructure, what constitutes Preliminaries, how ICMS L2 categories are defined, etc. Seeding these into the existing RAG pipeline would give the AI a proper reference base to retrieve from at classification time, rather than relying solely on hand-crafted migration rules.

This item is also a dependency for #3 — the semantic contradiction validator will be materially more accurate when it can retrieve an authoritative CBS definition from a seeded document rather than relying on the AI's general training knowledge.

#### What Already Exists (no new infrastructure needed)

The Phase 10a document pipeline is fully operational:

| Component | File |
|---|---|
| `reference_documents` table | DB — document metadata + processing status |
| `document_chunks` table | DB — parsed text with section headers, clause numbers, page ranges |
| `document_embeddings` table | DB — float32 vectors per chunk (OpenAI `text-embedding-3-large`) |
| `DocumentParserService` | `app/Services/DocumentParserService.php` |
| `DocumentChunkingService` | `app/Services/DocumentChunkingService.php` |
| `DocumentIngestionService` | `app/Services/DocumentIngestionService.php` |
| `DocumentSearchService` | `app/Services/DocumentSearchService.php` |
| Admin upload UI | `public/views/admin/` |

The pipeline is already used by the AI Agent Workspace. The gap is simply that the right documents have not been uploaded yet, and `SectionSignalService` does not yet call `DocumentSearchService` as a fallback.

#### What Needs to Happen

**Action 1 — QS team uploads the following documents via the admin UI:**

| Document | Purpose | Tag on upload |
|---|---|---|
| ICMS 3rd Edition — cost category definitions | Primary CBS L1–L4 boundary definitions | `cbs_classification` |
| NRM1 3rd Edition — elemental categories | NRM1 L1–L4 boundary definitions | `cbs_classification` |
| SLS internal classification guide | Firm-specific edge case rules | `cbs_classification` |
| RICS definitions glossary | Formal construction cost term definitions | `cost_definitions` |
| Any QS review notes on recurring misclassification | Training signal for known error patterns | `qs_review` |

**Action 2 — Nic wires RAG retrieval into `SectionSignalService` as a low-confidence fallback:**

When the service cannot resolve an L2 from sheet code, hierarchy path, section name, or embedding (confidence < 0.60), before accepting the low-confidence result it should query `DocumentSearchService` with the section name, filtered to `primary_use_tag = 'cbs_classification'`. If a high-similarity chunk is retrieved, the matching CBS category name in that chunk can resolve L2 more accurately than the current regex approach alone.

This follows the exact pattern already used in `ClassificationPromptBuilder::buildBatchPrompt()` for item-level RAG.

**No new DB schema required.**

#### Acceptance Criteria
- [ ] ICMS 3rd Edition and SLS internal classification guide uploaded, status = `ready`
- [ ] `DocumentSearchService` query filtered by `primary_use_tag = 'cbs_classification'` returns relevant chunks for queries like `"Substructure"` or `"Roof structure"`
- [ ] `SectionSignalService` falls back to RAG retrieval when confidence < 0.60
- [ ] Retrieved chunk content is visible in the classification prompt log for an import where the fallback fires

---

### #3 — AI Semantic Contradiction Check for Section Header Hierarchy
- **Date:** 2026-06-19
- **Raised by:** Gian le Roux
- **Owner:** Nic van der Walt
- **Status:** Open

#### Background — The Problem

When a BoQ is imported, `ExcelParserService` builds a section hierarchy using positional and numeric signals — merged cells, numbered codes like `2.02`, ALL CAPS text, word count, etc. This works well for the majority of BoQs.

The failure mode is when a section has no numbering and follows a clearly numbered parent section. The system assigns the unnumbered section to the nearest open parent purely by position. There is no check on whether the child section is semantically valid under that parent.

**Real example that triggered this request:**

A BoQ contained `2.02 Substructure` as a numbered section. Immediately following it, with no independent numbering, was a section called `Roof Gantry Steel`. Because it had no number, the parser treated it as a child of `2.02 Substructure` by position. Every item under `Roof Gantry Steel` received `Substructure` as their L2 hint — a fundamentally incorrect assignment, since a roof structure cannot exist under substructure by definition (substructure covers foundations, below-ground works, basement construction). The AI, receiving this as a strong context signal, classified those items as Substructure throughout.

**Why the existing `L2CoherenceService` does not catch this:**

`L2CoherenceService` works *bottom-up* — it looks at already-classified items and flags individual outliers that disagree with the dominant L2 in their section. In this case all items under `Roof Gantry Steel` agreed with each other (they all got Substructure), so there were no outliers. The error is invisible to the coherence checker because it originates at the section level, not the item level.

**What is needed:**

A *top-down* semantic validation step at the section level, running before any items are classified, asking: *"Is it possible for [child section name] to exist inside [parent CBS L2 category], given what those terms mean?"*

#### Where It Sits in the Pipeline

```
ExcelParserService
  → Builds section hierarchy, stores import_sections
        ↓
SectionSignalService
  → Assigns L2 CBS code to each section
        ↓
◄ NEW ►  SectionCoherenceValidator::validateHierarchy($importId)
  → Checks child section names against parent CBS L2 definitions
  → Flags contradictions, stores corrected L2
        ↓
ClassificationService
  → Items receive corrected section L2 as classification hint
        ↓
L2CoherenceService  (existing — unchanged)
  → Item-level outlier check remains as second safety net
```

#### Database Change

One migration, three columns added to `import_section_l2_assignments`:

```sql
ALTER TABLE import_section_l2_assignments
  ADD COLUMN semantic_flag             TINYINT(1)      NOT NULL DEFAULT 0
      COMMENT '1 = AI detected contradiction with parent section CBS assignment',
  ADD COLUMN semantic_reasoning        VARCHAR(500)    DEFAULT NULL
      COMMENT 'AI one-line explanation of the contradiction',
  ADD COLUMN semantic_suggested_cbs_id BIGINT UNSIGNED DEFAULT NULL
      COMMENT 'AI-suggested corrected L2 cbs_codes.id',
  ADD CONSTRAINT fk_isl2a_suggested_cbs
      FOREIGN KEY (semantic_suggested_cbs_id)
      REFERENCES cbs_codes(id) ON UPDATE CASCADE ON DELETE SET NULL;
```

No new tables required.

#### New Service — `app/Services/SectionCoherenceValidator.php`

Core method: `validateHierarchy(int $importId): void`

**Logic per child section:**

1. Load the child's L2 assignment from `import_section_l2_assignments` and its parent's L2 assignment.
2. **Skip AI call (fast-path)** if:
   - Child assignment source is `sheet_code` (most reliable signal — e.g. `MEC-001` tab names are unambiguous)
   - Child L2 already differs from parent L2 (signal service already detected a difference — no contradiction to check)
3. **Trigger AI call** when:
   - Child has inherited the parent's L2 (same `cbs_code_id`)
   - Assignment source is `hierarchy_path`, `section_name`, or `embedding` (lower reliability)
   - Child section name has low keyword overlap with the parent L2 definition

**AI prompt structure (abbreviated):**

```
You are validating a BoQ section hierarchy against CBS standards.

PARENT: "2.02 Substructure" → CBS L2: Substructure
Definition: [prompt_guidance from cbs_codes for this L2]

CHILD (currently assigned under parent): "Roof Gantry Steel"

Is it semantically valid for "Roof Gantry Steel" to sit under "Substructure"?

Respond as JSON:
{
  "valid": false,
  "reasoning": "Roof structures are above ground and belong to Superstructure, not Substructure which covers foundations and below-ground works.",
  "suggested_l2": "Superstructure"
}
```

Sections are batched up to 10 per AI call (same pattern as `L2CoherenceService::crossBatchL2CoherencePass()`).

**On contradiction confirmed:**
- `semantic_flag = 1`, `semantic_reasoning` stored
- `cbs_code_id` on the row updated to the corrected L2 — downstream classification automatically receives the fix via the existing `loadBatchSections()` query (no changes to `ClassificationPromptBuilder` needed)
- If suggested L2 name cannot be resolved to a `cbs_codes` row, flag and reasoning are stored but `cbs_code_id` is left unchanged (safe fallback)

**When DR-001 is implemented:** the prompt is upgraded to prepend a RAG-retrieved definition chunk from the seeded ICMS/NRM1 document, making the AI's reasoning grounded in the firm's own standards rather than general training knowledge.

#### Integration Point

Single line added in `ClassificationService::classifyImport()` after section pre-processing:

```php
(new SectionCoherenceValidator($this->db, $this->ai, $this->docs))
    ->validateHierarchy($importId);
```

#### UI — Surfacing Flags to QS Users

In the import detail view (section list), flagged sections should show:
- Yellow indicator on the section row
- Expandable note containing `semantic_reasoning`
- Original vs corrected L2 assignment shown side by side
- Option to accept the correction or override it (sets source to `manual`)

#### Files to Create / Modify

| Action | File |
|---|---|
| CREATE | `app/Services/SectionCoherenceValidator.php` |
| CREATE | `db/migrations/XXXX_add_section_coherence_flag.sql` |
| MODIFY | `app/Services/ClassificationService.php` — one line, call validator after section pre-processing |
| MODIFY | `public/views/imports/` — surface flag + reasoning in section detail (exact view file TBD) |

No changes to `ExcelParserService`, `SectionSignalService`, `ClassificationPromptBuilder`, or `L2CoherenceService`.

#### Risk Assessment

| Risk | Level | Mitigation |
|---|---|---|
| Validator overcorrects a valid nesting | LOW | Fast-path skips `sheet_code` assignments; only fires on inherited low-reliability L2 |
| Suggested L2 cannot be resolved to a CBS code | LOW | Unresolvable suggestions store flag + reasoning only, do not overwrite the CBS assignment |
| Added latency per import | LOW | One batched AI call (up to 10 sections), not per item — typical import: < 2 section pairs to check |
| Regression on correct hierarchies | LOW | Only fires where child L2 == parent L2 AND source is not `sheet_code` |
| AI call cost | NEGLIGIBLE | Small model, compact structured prompt — cost per import < $0.01 |

#### Acceptance Criteria
- [ ] `SectionCoherenceValidator::validateHierarchy()` runs for every new import after section pre-processing
- [ ] `Roof Gantry Steel` under `2.02 Substructure` → `semantic_flag = 1`, reasoning references Superstructure, `cbs_code_id` updated to Superstructure L2
- [ ] Items under the corrected section receive the correct L2 as classification hint
- [ ] Fast-path skips sections assigned via `sheet_code` source
- [ ] Fast-path skips sections whose L2 already differs from their parent
- [ ] `semantic_flag` and `semantic_reasoning` visible in import detail view
- [ ] QS user can accept or override the correction (sets source to `manual`)
- [ ] No regressions on imports where section hierarchy is already correct
