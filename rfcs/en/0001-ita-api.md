# ITA API – Initial Proposal

> **Context:** API for centralized access to ITA entrance exam (vestibular) and graduate admission tests.

## Summary

Proposal for a **read-only** API for ITA entrance and graduate admission exams. The expected outcome is a single access point to list tests, fetch test details and questions (with optional answer key), using data extracted from the official website and local files, transformed into standardized JSON and loaded into a structured database.

*What real problem does this solve?* Today there is no centralized, easy-to-consume repository for these exams; access relies on a legacy system and scattered sources.

## Motivation / Problem Statement

There is currently no single place that centralizes ITA exams (entrance and graduate) in a standardized, accessible format. The tests live in a legacy system that is hard to access, which hinders study, research, and tools that depend on this data.

*Why now?* Demand for structured data for study and tools (practice tests, topic-based analysis, etc.) makes it relevant to define the data model and pipeline (ETL + API) now so that future work (e.g., topic tagging, AI-generated practice tests) can build on a stable base.

## Goals / Non-Goals

**Goals:**

- **Phase 1**
  - ETL: extraction (web + local files) → transformation into structured JSON → load into database.
  - Query API: list tests, get test detail and questions with optional answer key.

- **Phase 2**
  - Associate questions with [syllabus topics](https://www.vestibular.ita.br/programa_2026.pdf).
  - Random test generation from question pool.
  - Candidate analysis by region.

- **Phase 3**
  - AI-generated practice tests.

**Non-Goals:**

- Do not replace the official exam website; the API is complementary and for programmatic consumption.
- Do not include, in this proposal, user authentication or access control (can be addressed later).
- Do not cover exams from other institutions; scope is limited to ITA (entrance and graduate).

## Proposed Solution

### Data pipeline overview (ETL)

The first challenge is the ETL: obtaining data (official site and/or local files), normalizing it into a single format, and loading it into a database to serve the API.

Primary reference: exam area at https://www.vestibular.ita.br/.

![ETL flow – Extraction, Transform, and Load](../../.github/imgs/0001-etl.png)  
_Figure 1: ETL of the Tests._

**Figure explanation:**

1. **Extraction**  
   Data is obtained by **web scraping** and/or from **local files** already available. The result is raw files (e.g. PDFs) representing the tests.

2. **Transform**  
   These files are processed and converted into **structured JSON**, with fields such as `title`, `year`, and `questions` (list of questions), enabling a single model regardless of source format.

3. **Load**  
   The generated JSON is **loaded into a structured database**, which the API will query to list tests, details, and questions.

With this pipeline, the API only acts as a read layer on the populated database.

### Test structure (for model and ETL design)

To design the data model and transformers, the actual structure of the exams must be considered:

**ITA entrance exam (vestibular)**

| Period    | Content |
|-----------|---------|
| 2008–2018 | Physics: 30 (20 multiple-choice + 10 open-ended). Portuguese: 20 multiple-choice + essay. English: 20 multiple-choice. Mathematics: 30 (20 multiple-choice + 10 open-ended). Chemistry: 30 (20 multiple-choice + 10 open-ended). |
| 2019–2024 | **Phase 1:** 15 Physics, 15 Portuguese, 10 English, 15 Mathematics, 15 Chemistry (multiple-choice). **Phase 2:** Mathematics, Physics and Chemistry with 10 open-ended each + Essay. |
| 2025–current | **Phase 1:** 12 Mathematics, 12 Physics, 12 Chemistry, 12 English. **Phase 2:** 10 open-ended (Mathematics, Physics, Chemistry), Portuguese 15 multiple-choice + Essay. |

**ITA graduate admission**

- GMAT: 15 or 16 questions (English or Portuguese).
- English: essay or multiple choice.

### API contracts (summary)

Reference enums and schemas (Python example):

```python
class Discipline(Enum):
  FISICA = 'fisica'
  MATEMATICA = 'matematica'
  QUIMICA = 'quimica'
  PORTUGUES = 'portugues'
  INGLES = 'ingles'

class Language(Enum):
  PORTUGUES = 'portugues'
  INGLES = 'ingles'

class TestType(Enum):
  VESTIBULAR = 'vestibular'
  POS = 'pos'

class FormatType(Enum):
  ESSAY = 'essay'
  OBJECTIVE = 'objective'
```

```python
from typing import List, Optional

class TestSummary(Schema):
  id: str
  type: TestType
  year: int
  name: str
  number_of_questions: int
  language: Language
  format: List[FormatType]

ListTestsOut = List[TestSummary]

class TestSummaryParams(Schema):
  year: Optional[int]
  type: Optional[TestType] = TestType.VESTIBULAR.value

class TestDetailSubject(Schema):
  name: str
  number_of_questions: int
  disciplines: List[Discipline]
  with_essay: Optional[bool] = None

class TestDetailPhases(Schema):
  phase: int
  subjects: List[TestDetailSubject]

class TestDetailOut(Schema):
  id: str
  type: TestType
  year: int
  phases: List[TestDetailPhases]

class QuestionParams(Schema):
  with_answers: bool = False

QuestionOut = List[QuestionDetail]

class QuestionOption(Schema):
  id: str
  statement: str
  letter: str
  is_correct: Optional[bool] = None

class QuestionDetail(Schema):
  id: str
  discipline: List[Discipline]
  type: FormatType
  statement: str
  options: List[QuestionOption]
  images: List[str]
```

**Endpoints:**

| Method and route | Query params | Payload | Output |
|------------------|--------------|---------|--------|
| GET /api/v1/tests/ | TestSummaryParams | — | ListTestsOut |
| GET /api/v1/tests/{id} | — | — | TestDetailOut |
| GET /api/v1/tests/{id}/questions | QuestionParams | — | QuestionOut |

## Alternatives Considered

| Alternative | Reason not adopted |
|-------------|--------------------|
| On-demand scraping only (no ETL or database) | Does not scale; latency and load on the site; hard to standardize history and reuse in later phases (topics, practice tests). |
| Keep only local files on disk, no API | Does not provide a stable contract for clients (apps, study tools); hard to version and evolve (filters, pagination, conditional answer key). |
| Include write operations (test upload) in the API in this phase | Increases scope and security surface; current focus is read-only with ETL-validated data. |
| Reuse or extend the [ENEM API](https://docs.enem.dev/) | The ENEM API covers ENEM exams and questions (community open-source project); ITA has a different structure, format, and audience, justifying a dedicated API. It serves as a design reference (public read-only API, documentation, self-hosting). |

## Impact

- **Scope:** New system (ETL pipeline + read API); consumption of the official exam area (read/scraping) and possibly local files.
- **Effort:** Phase 1 depends on implementing the three ETL stages and the three endpoints; varies with source format (PDF/HTML) and data quality.
- **Risks:** Changes to the ITA site may break the scraper; mitigation with integration tests and fallback to local files when available.
- **Rollback:** API and ETL are additive; they do not change the legacy system. If issues arise, the API can be turned off and JSON/database kept for internal use.

## Open Questions

- ETL stack choice (language, scraping tools, database).
- Update strategy: scheduled (cron) vs. on-demand vs. hybrid.
- API versioning policy (e.g. `/api/v1/` vs. header).
- How to handle images in questions (URLs, storage, CDN).
- Quality and validation criteria for data after transformation (e.g. checking question count per discipline).

## References

- [RFC 0001 - ITA API (Portuguese)](../pt/0001-ita-api.md)
- [ITA Vestibular 2026 syllabus](https://www.vestibular.ita.br/programa_2026.pdf)
- Exam area: https://www.vestibular.ita.br/
- [ENEM API (docs.enem.dev)](https://docs.enem.dev/) – Public open-source API for ENEM exams and questions; reference for a similar initiative in the Brazilian exam context (GNU GPL-2.0 license, documentation, rate limits, self-hosting).

---

## Checklist (Good Questions)

- **What real problem does this solve?** → Summary, Motivation
- **Why now?** → Motivation
- **What changes?** → Goals, Proposed Solution, Impact
- **What doesn't change?** → Non-Goals, Impact
- **What are the risks?** → Proposed Solution (ETL), Impact, Alternatives
- **How do we undo this if things go wrong?** → Impact (rollback), Alternatives
