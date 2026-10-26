# RepoScout - Complete Source Bundle

Generated: 2026-10-26 06:35 UTC

This single file contains all 29 hand-written project files (9,291 source lines).

Generated artifacts under `.reposcout/` (SQLite index, HTML/Markdown reports), `__pycache__/`, and compiled `.pyc` files are intentionally excluded.

## Contents

1. [`pyproject.toml`](#pyprojecttoml) - 35 lines
2. [`.gitignore`](#gitignore) - 22 lines
3. [`.github/workflows/tests.yml`](#githubworkflowstestsyml) - 20 lines
4. [`reposcout/__init__.py`](#reposcout__init__py) - 9 lines
5. [`reposcout/__main__.py`](#reposcout__main__py) - 6 lines
6. [`reposcout/models.py`](#reposcoutmodelspy) - 502 lines
7. [`reposcout/util.py`](#reposcoututilpy) - 337 lines
8. [`reposcout/tokens.py`](#reposcouttokenspy) - 366 lines
9. [`reposcout/scanning.py`](#reposcoutscanningpy) - 683 lines
10. [`reposcout/parsing.py`](#reposcoutparsingpy) - 1,004 lines
11. [`reposcout/indexing.py`](#reposcoutindexingpy) - 779 lines
12. [`reposcout/retrieval.py`](#reposcoutretrievalpy) - 756 lines
13. [`reposcout/checks.py`](#reposcoutcheckspy) - 873 lines
14. [`reposcout/planning.py`](#reposcoutplanningpy) - 794 lines
15. [`reposcout/agent.py`](#reposcoutagentpy) - 366 lines
16. [`reposcout/rendering.py`](#reposcoutrenderingpy) - 875 lines
17. [`reposcout/cli.py`](#reposcoutclipy) - 573 lines
18. [`tests/support.py`](#testssupportpy) - 196 lines
19. [`tests/test_models.py`](#teststest_modelspy) - 70 lines
20. [`tests/test_tokens.py`](#teststest_tokenspy) - 66 lines
21. [`tests/test_scanning.py`](#teststest_scanningpy) - 68 lines
22. [`tests/test_indexing.py`](#teststest_indexingpy) - 89 lines
23. [`tests/test_checks.py`](#teststest_checkspy) - 182 lines
24. [`tests/test_agent.py`](#teststest_agentpy) - 94 lines
25. [`tests/test_rendering.py`](#teststest_renderingpy) - 96 lines
26. [`tests/test_cli.py`](#teststest_clipy) - 174 lines
27. [`CONTRIBUTING.md`](#contributingmd) - 49 lines
28. [`README.md`](#readmemd) - 186 lines
29. [`LICENSE`](#license) - 21 lines

---

## pyproject.toml

`pyproject.toml` - 35 lines

```toml
[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"

[project]
name = "reposcout"
version = "0.1.0"
description = "Evidence-first repository mapping and beginner-friendly contribution planning"
readme = "README.md"
requires-python = ">=3.11"
license = { text = "MIT" }
authors = [{ name = "RepoScout contributors" }]
keywords = ["repository", "code-search", "bm25", "contributing", "agent", "static-analysis"]
classifiers = [
  "Development Status :: 4 - Beta",
  "Environment :: Console",
  "Intended Audience :: Developers",
  "License :: OSI Approved :: MIT License",
  "Programming Language :: Python :: 3",
  "Programming Language :: Python :: 3.11",
  "Programming Language :: Python :: 3.12",
  "Programming Language :: Python :: 3.13",
  "Topic :: Software Development :: Quality Assurance",
  "Topic :: Software Development :: Documentation",
]
dependencies = []

[project.scripts]
reposcout = "reposcout.cli:main"

[project.urls]
Homepage = "https://github.com/example/reposcout"

[tool.setuptools.packages.find]
include = ["reposcout*"]
```

## .gitignore

`.gitignore` - 22 lines

```text
.reposcout/
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
env/
.pytest_cache/
.mypy_cache/
.ruff_cache/
.tox/
.nox/
build/
dist/
*.egg-info/
.coverage
htmlcov/
.DS_Store
Thumbs.db
.env
.env.*
!.env.example
```

## .github/workflows/tests.yml

`.github/workflows/tests.yml` - 20 lines

```yml
name: tests

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: Run unit tests
        run: python -m unittest discover -s tests -v
```

## reposcout/__init__.py

`reposcout/__init__.py` - 9 lines

```py
"""RepoScout: evidence-first repository mapping and contribution planning.

The public entry points live in the submodules; importing the package itself is
intentionally cheap so ``reposcout --version`` does not pay for the indexer.
"""

__version__ = "0.1.0"

__all__ = ["__version__"]
```

## reposcout/__main__.py

`reposcout/__main__.py` - 6 lines

```py
"""Allow ``python -m reposcout ...``."""

from .cli import main

if __name__ == "__main__":
    raise SystemExit(main())
```

## reposcout/models.py

`reposcout/models.py` - 502 lines

```py
"""Typed data model shared by every stage of RepoScout.

The central design rule of this project is that facts and guesses never share
a shelf.  A :class:`Claim` is either ``verified`` (backed by at least one
:class:`Source` pointing at a file/line) or ``suggested`` (backed by an
explicit rationale in ``basis``).  Constructing an unsupported claim raises,
so the distinction cannot silently rot.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any, Literal

Status = Literal["verified", "suggested"]
Role = Literal["primary", "supporting", "test", "doc", "config"]


@dataclass(frozen=True)
class Source:
    """A concrete pointer into the repository."""

    path: str
    line: int | None = None
    snippet: str | None = None

    def label(self) -> str:
        if self.line is None:
            return self.path
        return f"{self.path}:{self.line}"


@dataclass
class Claim:
    """A statement that is explicitly either grounded or speculative."""

    text: str
    status: Status
    sources: list[Source] = field(default_factory=list)
    basis: str | None = None

    def __post_init__(self) -> None:
        if self.status not in ("verified", "suggested"):
            raise ValueError(f"unknown claim status: {self.status!r}")
        if self.status == "verified" and not self.sources:
            raise ValueError(f"verified claim requires a source: {self.text!r}")
        if self.status == "suggested" and not self.basis:
            raise ValueError(f"suggested claim requires a basis: {self.text!r}")


@dataclass
class Check:
    """A command a contributor could run, with provenance."""

    id: str
    command: str
    kind: str  # test | lint | typecheck | format | build | docs | security
    status: Status
    sources: list[Source] = field(default_factory=list)
    basis: str = ""
    scope: list[str] = field(default_factory=list)
    declared_in: str | None = None
    runnable: bool = False
    tool_notes: list[str] = field(default_factory=list)

    def __post_init__(self) -> None:
        if self.status == "verified" and not self.sources:
            raise ValueError(f"verified check requires a source: {self.command!r}")
        if self.status == "suggested" and not self.basis:
            raise ValueError(f"suggested check requires a basis: {self.command!r}")


@dataclass
class Symbol:
    name: str
    kind: str
    line: int
    end_line: int | None = None
    signature: str = ""
    doc: str = ""
    exported: bool = True
    container: str = ""

    @property
    def qualified(self) -> str:
        return f"{self.container}.{self.name}" if self.container else self.name

    def to_dict(self) -> dict[str, Any]:
        return {
            "name": self.name,
            "kind": self.kind,
            "line": self.line,
            "end_line": self.end_line,
            "signature": self.signature,
            "doc": self.doc,
            "exported": self.exported,
            "container": self.container,
            "qualified": self.qualified,
        }


@dataclass
class ImportRef:
    module: str
    line: int
    raw: str
    resolved: str | None = None

    def to_dict(self) -> dict[str, Any]:
        return {
            "module": self.module,
            "line": self.line,
            "raw": self.raw,
            "resolved": self.resolved,
        }


@dataclass
class FileInfo:
    """One indexed file and everything we observed about it."""

    path: str
    lang: str
    lines: int
    bytes: int
    is_test: bool = False
    is_doc: bool = False
    is_config: bool = False
    sha: str = ""
    symbols: list[Symbol] = field(default_factory=list)
    imports: list[ImportRef] = field(default_factory=list)
    summary: str = ""

    def to_dict(self) -> dict[str, Any]:
        return {
            "path": self.path,
            "lang": self.lang,
            "lines": self.lines,
            "bytes": self.bytes,
            "is_test": self.is_test,
            "is_doc": self.is_doc,
            "is_config": self.is_config,
            "sha": self.sha,
            "summary": self.summary,
            "symbols": [s.to_dict() for s in self.symbols],
            "imports": [i.to_dict() for i in self.imports],
        }


@dataclass
class Manifest:
    path: str
    kind: str
    name: str = ""
    description: str = ""

    def to_dict(self) -> dict[str, Any]:
        return {
            "path": self.path,
            "kind": self.kind,
            "name": self.name,
            "description": self.description,
        }


@dataclass
class RepoProfile:
    root: str
    name: str
    files_indexed: int = 0
    total_lines: int = 0
    languages: dict[str, int] = field(default_factory=dict)
    manifests: list[Manifest] = field(default_factory=list)
    docs: list[str] = field(default_factory=list)
    guidelines: list[str] = field(default_factory=list)
    issue_templates: list[str] = field(default_factory=list)
    ci_workflows: list[str] = field(default_factory=list)
    test_files: int = 0
    entry_points: list[str] = field(default_factory=list)
    package_name: str = ""
    description: str = ""
    license: str = ""
    is_git: bool = False
    branch: str = ""
    commit: str = ""
    remote: str = ""
    top_level: list[str] = field(default_factory=list)

    def to_dict(self) -> dict[str, Any]:
        data = {k: v for k, v in self.__dict__.items()}
        data["manifests"] = [m.to_dict() for m in self.manifests]
        return data


@dataclass
class QueryTerm:
    term: str
    weight: float = 1.0
    origin: str = "task"  # task | expansion | identifier | llm
    note: str = ""

    def to_dict(self) -> dict[str, Any]:
        return {
            "term": self.term,
            "weight": self.weight,
            "origin": self.origin,
            "note": self.note,
        }


@dataclass
class Snippet:
    line: int
    text: str
    terms: list[str] = field(default_factory=list)
    symbol: str | None = None

    def to_dict(self) -> dict[str, Any]:
        return {
            "line": self.line,
            "text": self.text,
            "terms": self.terms,
            "symbol": self.symbol,
        }


@dataclass
class Reason:
    text: str
    kind: str  # match | symbol | graph | role | test-link | path
    strength: float = 0.0
    sources: list[Source] = field(default_factory=list)
    inferred: bool = False

    def to_dict(self) -> dict[str, Any]:
        return {
            "text": self.text,
            "kind": self.kind,
            "strength": round(self.strength, 4),
            "sources": [s.__dict__ for s in self.sources],
            "inferred": self.inferred,
        }


@dataclass
class FileHit:
    path: str
    role: Role
    score: float
    lang: str = ""
    lines: int = 0
    reasons: list[Reason] = field(default_factory=list)
    snippets: list[Snippet] = field(default_factory=list)
    symbols: list[Symbol] = field(default_factory=list)
    imports: list[str] = field(default_factory=list)
    imported_by: list[str] = field(default_factory=list)
    module: str = ""

    def to_dict(self) -> dict[str, Any]:
        return {
            "path": self.path,
            "role": self.role,
            "score": round(self.score, 3),
            "lang": self.lang,
            "lines": self.lines,
            "reasons": [r.to_dict() for r in self.reasons],
            "snippets": [s.to_dict() for s in self.snippets],
            "symbols": [s.to_dict() for s in self.symbols],
            "imports": self.imports,
            "imported_by": self.imported_by,
            "module": self.module,
        }


@dataclass
class Mapping:
    task: str
    primaries: list[FileHit] = field(default_factory=list)
    supporting: list[FileHit] = field(default_factory=list)
    tests: list[FileHit] = field(default_factory=list)
    docs: list[FileHit] = field(default_factory=list)
    query_terms: list[QueryTerm] = field(default_factory=list)
    identifiers: list[str] = field(default_factory=list)
    phrases: list[str] = field(default_factory=list)
    confidence: float = 0.0
    confidence_label: str = "low"
    confidence_rationale: list[str] = field(default_factory=list)
    notes: list[str] = field(default_factory=list)
    graph_expansions: int = 0

    def all_hits(self) -> list[FileHit]:
        seen: set[str] = set()
        ordered: list[FileHit] = []
        for hit in [*self.primaries, *self.supporting, *self.tests, *self.docs]:
            if hit.path not in seen:
                seen.add(hit.path)
                ordered.append(hit)
        return ordered

    def to_dict(self) -> dict[str, Any]:
        return {
            "task": self.task,
            "primaries": [h.to_dict() for h in self.primaries],
            "supporting": [h.to_dict() for h in self.supporting],
            "tests": [h.to_dict() for h in self.tests],
            "docs": [h.to_dict() for h in self.docs],
            "query_terms": [t.to_dict() for t in self.query_terms],
            "identifiers": self.identifiers,
            "phrases": self.phrases,
            "confidence": round(self.confidence, 3),
            "confidence_label": self.confidence_label,
            "confidence_rationale": self.confidence_rationale,
            "notes": self.notes,
            "graph_expansions": self.graph_expansions,
        }


@dataclass
class Action:
    kind: str  # read | run | edit | inspect | ask
    detail: str
    status: Status = "suggested"
    sources: list[Source] = field(default_factory=list)
    basis: str | None = None
    command: str | None = None
    check_id: str | None = None

    def __post_init__(self) -> None:
        if self.status == "verified" and not self.sources:
            raise ValueError(f"verified action requires a source: {self.detail!r}")
        if self.status == "suggested" and not self.basis:
            raise ValueError(f"suggested action requires a basis: {self.detail!r}")

    def to_dict(self) -> dict[str, Any]:
        return {
            "kind": self.kind,
            "detail": self.detail,
            "status": self.status,
            "sources": [s.__dict__ for s in self.sources],
            "basis": self.basis,
            "command": self.command,
            "check_id": self.check_id,
        }


@dataclass
class PlanStep:
    number: int
    title: str
    goal: str
    duration_minutes: int = 15
    actions: list[Action] = field(default_factory=list)
    rationale: list[Claim] = field(default_factory=list)
    risks: list[str] = field(default_factory=list)

    def to_dict(self) -> dict[str, Any]:
        return {
            "number": self.number,
            "title": self.title,
            "goal": self.goal,
            "duration_minutes": self.duration_minutes,
            "actions": [a.to_dict() for a in self.actions],
            "rationale": [
                {
                    "text": c.text,
                    "status": c.status,
                    "sources": [s.__dict__ for s in c.sources],
                    "basis": c.basis,
                }
                for c in self.rationale
            ],
            "risks": self.risks,
        }


@dataclass
class ContributionPlan:
    task: str
    difficulty_score: int = 0
    difficulty_label: str = "moderate"
    estimated_minutes: tuple[int, int] = (30, 120)
    summary: str = ""
    steps: list[PlanStep] = field(default_factory=list)
    checklist: list[Claim] = field(default_factory=list)
    unknowns: list[str] = field(default_factory=list)
    files_to_touch: list[str] = field(default_factory=list)
    test_files: list[str] = field(default_factory=list)

    def to_dict(self) -> dict[str, Any]:
        return {
            "task": self.task,
            "difficulty_score": self.difficulty_score,
            "difficulty_label": self.difficulty_label,
            "estimated_minutes": list(self.estimated_minutes),
            "summary": self.summary,
            "steps": [s.to_dict() for s in self.steps],
            "checklist": [
                {
                    "text": c.text,
                    "status": c.status,
                    "sources": [s.__dict__ for s in c.sources],
                    "basis": c.basis,
                }
                for c in self.checklist
            ],
            "unknowns": self.unknowns,
            "files_to_touch": self.files_to_touch,
            "test_files": self.test_files,
        }


@dataclass
class TraceStep:
    index: int
    tool: str
    input: str
    output: str
    duration_ms: int = 0

    def to_dict(self) -> dict[str, Any]:
        return {
            "index": self.index,
            "tool": self.tool,
            "input": self.input,
            "output": self.output,
            "duration_ms": self.duration_ms,
        }


@dataclass
class CheckResult:
    """A command that was actually executed, as opposed to just declared."""

    check_id: str
    command: str
    returncode: int
    duration_ms: int = 0
    timed_out: bool = False
    output: str = ""

    @property
    def ok(self) -> bool:
        return self.returncode == 0 and not self.timed_out

    def to_dict(self) -> dict[str, Any]:
        return {
            "check_id": self.check_id,
            "command": self.command,
            "returncode": self.returncode,
            "ok": self.ok,
            "timed_out": self.timed_out,
            "duration_ms": self.duration_ms,
            "output": self.output,
        }


@dataclass
class Report:
    task: str
    profile: RepoProfile
    mapping: Mapping
    checks: list[Check]
    plan: ContributionPlan
    trace: list[TraceStep] = field(default_factory=list)
    check_results: list[CheckResult] = field(default_factory=list)
    limitations: list[str] = field(default_factory=list)
    llm_used: bool = False
    llm_notes: list[str] = field(default_factory=list)
    generated_at: str = ""
    version: str = ""
    repo_root: str = ""

    def to_dict(self) -> dict[str, Any]:
        return {
            "task": self.task,
            "generated_at": self.generated_at,
            "version": self.version,
            "repo_root": self.repo_root,
            "profile": self.profile.to_dict(),
            "mapping": self.mapping.to_dict(),
            "checks": [
                {
                    "id": c.id,
                    "command": c.command,
                    "kind": c.kind,
                    "status": c.status,
                    "sources": [s.__dict__ for s in c.sources],
                    "basis": c.basis,
                    "scope": c.scope,
                    "declared_in": c.declared_in,
                    "runnable": c.runnable,
                    "tool_notes": c.tool_notes,
                }
                for c in self.checks
            ],
            "plan": self.plan.to_dict(),
            "trace": [t.to_dict() for t in self.trace],
            "check_results": [r.to_dict() for r in self.check_results],
            "limitations": self.limitations,
            "llm_used": self.llm_used,
            "llm_notes": self.llm_notes,
        }
```

## reposcout/util.py

`reposcout/util.py` - 337 lines

````py
"""Small, dependency-free helpers used across RepoScout."""

from __future__ import annotations

import datetime as _dt
import hashlib
import os
import re
import subprocess
from pathlib import Path

TEXT_ENCODINGS = ("utf-8", "utf-8-sig", "latin-1")
MAX_FILE_BYTES = 2_000_000


def posix(path: str | Path) -> str:
    return str(path).replace("\\", "/")


def relative(root: Path, path: Path) -> str:
    try:
        return posix(path.relative_to(root))
    except ValueError:
        return posix(path)


def read_bytes(path: Path, limit: int = MAX_FILE_BYTES) -> bytes | None:
    try:
        with path.open("rb") as handle:
            return handle.read(limit)
    except (OSError, PermissionError):
        return None


def is_binary(data: bytes) -> bool:
    if not data:
        return False
    if b"\x00" in data[:8192]:
        return True
    # Heuristic: large share of non-text control bytes.
    sample = data[:8192]
    control = sum(1 for byte in sample if byte < 9 or 13 < byte < 32)
    return control / max(1, len(sample)) > 0.30


def read_text(path: Path, limit: int = MAX_FILE_BYTES) -> str | None:
    data = read_bytes(path, limit)
    if data is None or is_binary(data):
        return None
    for encoding in TEXT_ENCODINGS:
        try:
            return data.decode(encoding)
        except UnicodeDecodeError:
            continue
    return data.decode("utf-8", errors="replace")


def split_lines(text: str) -> list[str]:
    return text.replace("\r\n", "\n").replace("\r", "\n").split("\n")


def sha(text: str) -> str:
    return hashlib.sha1(text.encode("utf-8", errors="replace")).hexdigest()[:16]


def truncate(text: str, width: int, suffix: str = "...") -> str:
    text = " ".join(text.split())
    if len(text) <= width:
        return text
    return text[: max(0, width - len(suffix))].rstrip() + suffix


def first_sentence(text: str, width: int = 240) -> str:
    """Return the first meaningful sentence of prose, truncated."""
    cleaned_lines: list[str] = []
    for line in split_lines(text):
        stripped = line.strip()
        if not stripped:
            if cleaned_lines:
                break
            continue
        if stripped.startswith(("#", "```", "---", "===", "<!--", "|")):
            if cleaned_lines:
                break
            continue
        cleaned_lines.append(stripped)
        if len(" ".join(cleaned_lines)) > width:
            break
    sentence = " ".join(cleaned_lines)
    match = re.search(r"(?<=[.!?])\s+", sentence)
    if match:
        sentence = sentence[: match.start()]
    return truncate(sentence, width)


def find_line(lines: list[str], needle: str, start: int = 0) -> int | None:
    """1-based line number of the first line containing ``needle``."""
    lowered = needle.lower()
    for index in range(start, len(lines)):
        if lowered in lines[index].lower():
            return index + 1
    return None


def line_text(lines: list[str], line: int | None) -> str:
    if line is None or line < 1 or line > len(lines):
        return ""
    return lines[line - 1].strip()


def statement_at(text: str, line: int, width: int = 200) -> str:
    lines = split_lines(text)
    return truncate(line_text(lines, line), width)


def detect_language(path: str, filename: str) -> str:
    ext = Path(filename).suffix.lower()
    name = filename.lower()
    special = {
        "dockerfile": "docker",
        "makefile": "make",
        "cmakelists.txt": "cmake",
        "justfile": "just",
        "procfile": "procfile",
        "gemfile": "ruby",
        "rakefile": "ruby",
        "requirements.txt": "text",
        "license": "text",
        "notice": "text",
        "codeowners": "text",
    }
    if name in special:
        return special[name]
    return {
        ".py": "python",
        ".pyi": "python",
        ".js": "javascript",
        ".mjs": "javascript",
        ".cjs": "javascript",
        ".jsx": "javascript",
        ".ts": "typescript",
        ".tsx": "typescript",
        ".mts": "typescript",
        ".cts": "typescript",
        ".go": "go",
        ".rs": "rust",
        ".java": "java",
        ".kt": "kotlin",
        ".kts": "kotlin",
        ".scala": "scala",
        ".rb": "ruby",
        ".php": "php",
        ".cs": "csharp",
        ".c": "c",
        ".h": "c",
        ".cc": "cpp",
        ".cpp": "cpp",
        ".cxx": "cpp",
        ".hpp": "cpp",
        ".hh": "cpp",
        ".m": "objective-c",
        ".mm": "objective-c",
        ".swift": "swift",
        ".lua": "lua",
        ".pl": "perl",
        ".pm": "perl",
        ".r": "r",
        ".jl": "julia",
        ".sh": "shell",
        ".bash": "shell",
        ".zsh": "shell",
        ".fish": "shell",
        ".ps1": "powershell",
        ".bat": "batch",
        ".cmd": "batch",
        ".html": "html",
        ".htm": "html",
        ".css": "css",
        ".scss": "css",
        ".sass": "css",
        ".less": "css",
        ".vue": "vue",
        ".svelte": "svelte",
        ".sql": "sql",
        ".graphql": "graphql",
        ".gql": "graphql",
        ".proto": "protobuf",
        ".toml": "toml",
        ".yaml": "yaml",
        ".yml": "yaml",
        ".json": "json",
        ".jsonc": "json",
        ".xml": "xml",
        ".md": "markdown",
        ".mdx": "markdown",
        ".rst": "rst",
        ".txt": "text",
        ".tex": "latex",
        ".ipynb": "notebook",
        ".env": "dotenv",
        ".ini": "ini",
        ".cfg": "ini",
        ".conf": "ini",
        ".gradle": "gradle",
        ".tf": "terraform",
        ".hcl": "hcl",
        ".nix": "nix",
        ".zig": "zig",
        ".ex": "elixir",
        ".exs": "elixir",
        ".erl": "erlang",
        ".hs": "haskell",
        ".ml": "ocaml",
        ".clj": "clojure",
        ".groovy": "groovy",
        ".dart": "dart",
        ".sol": "solidity",
        ".asm": "assembly",
        ".s": "assembly",
    }.get(ext, "text" if ext in {".log", ".csv"} else "other")


PROGRAMMING_LANGS = {
    "python",
    "javascript",
    "typescript",
    "go",
    "rust",
    "java",
    "kotlin",
    "scala",
    "ruby",
    "php",
    "csharp",
    "c",
    "cpp",
    "objective-c",
    "swift",
    "lua",
    "perl",
    "r",
    "julia",
    "shell",
    "powershell",
    "vue",
    "svelte",
    "dart",
    "elixir",
    "haskell",
    "clojure",
    "zig",
}


def module_label(path: str) -> str:
    """Human-friendly module name for a file path."""
    cleaned = posix(path)
    for prefix in ("src/", "lib/", "source/"):
        if cleaned.startswith(prefix):
            cleaned = cleaned[len(prefix) :]
    stem = cleaned.rsplit(".", 1)[0]
    stem = stem.replace("/", ".")
    if stem.endswith(".__init__"):
        stem = stem[: -len(".__init__")]
    return stem


def run_git(root: Path, *args: str, timeout: int = 8) -> str:
    try:
        completed = subprocess.run(
            ["git", *args],
            cwd=str(root),
            capture_output=True,
            text=True,
            timeout=timeout,
            check=False,
        )
    except (OSError, subprocess.SubprocessError):
        return ""
    if completed.returncode != 0:
        return ""
    return completed.stdout.strip()


def now_iso() -> str:
    return _dt.datetime.now(_dt.timezone.utc).replace(microsecond=0).isoformat()


def human_bytes(size: int) -> str:
    value = float(size)
    for unit in ("B", "KB", "MB", "GB"):
        if value < 1024 or unit == "GB":
            return f"{value:.0f} {unit}" if unit == "B" else f"{value:.1f} {unit}"
        value /= 1024
    return f"{size} B"


def clamp(value: float, low: float = 0.0, high: float = 1.0) -> float:
    return max(low, min(high, value))


def slug(text: str, limit: int = 60) -> str:
    cleaned = re.sub(r"[^a-zA-Z0-9]+", "-", text.lower()).strip("-")
    return cleaned[:limit] or "report"


def which(program: str) -> str | None:
    from shutil import which as _which

    return _which(program)


def is_within(path: Path, parent: Path) -> bool:
    try:
        path.resolve().relative_to(parent.resolve())
        return True
    except (ValueError, OSError):
        return False


def common_ancestor(paths: list[str]) -> str:
    if not paths:
        return ""
    parts = [p.split("/") for p in paths]
    prefix = parts[0]
    for other in parts[1:]:
        new: list[str] = []
        for left, right in zip(prefix, other):
            if left != right:
                break
            new.append(left)
        prefix = new
    return "/".join(prefix)


def env_flag(name: str) -> bool:
    return os.environ.get(name, "").strip().lower() in {"1", "true", "yes", "on"}
````

## reposcout/tokens.py

`reposcout/tokens.py` - 366 lines

```py
"""Tokenization, light stemming and query expansion.

RepoScout deliberately avoids embeddings and external models for its core
retrieval.  A transparent lexical index means every ranking decision can be
traced back to a term that literally appears in the repository.
"""

from __future__ import annotations

import re
from dataclasses import dataclass, field

# Task-tracker boilerplate and English function words.  These are removed from
# queries, not from the index: code may legitimately contain the word "issue".
STOPWORDS: frozenset[str] = frozenset(
    """
    a an the and or but if then else when while of to in on at by for with from as is are was were be been being
    this that these those it its it's their there here you your yours we our ours they them he she his her
    i me my mine do does did done doing have has had having can could should would will shall may might must
    not no yes so such than too very just also only about into over under again further once all any both each
    few more most other some own same up down out off above below between through during before after
    what which who whom whose why how where
    please thanks thank hi hello regards
    issue bug feature request problem bugfix fixed fixes fixing
    add adds added adding support supports supported allow allows make makes making use uses used using
    implement implements implemented implementation provide provides provided
    expected actual behavior behaviour steps reproduce reproduction description context
    possible easy good first new need needs needed want wants like update updates updated change changes changed
    currently existing instead able unable unable work works working
    user users case cases example examples way ways thing things
    """.split()
)

# Words that are too common in code to carry retrieval signal on their own.
CODE_STOPWORDS: frozenset[str] = frozenset(
    """
    var let const function return import export from default class def self this super true false none null
    nil new public private protected static final void int float double bool string str list dict object
    print console log logging getter setter module package src lib test tests spec
    """.split()
)

_TOKEN_RE = re.compile(r"[A-Za-z_][A-Za-z0-9_]*|\d+")
_ACRONYM_RE = re.compile(r"[A-Z]+(?=[A-Z][a-z])|[A-Z]?[a-z]+|[A-Z]+|\d+")
_BACKTICK_RE = re.compile(r"`([^`\n]{2,120})`")
_QUOTE_RE = re.compile(r"[\"“]([^\"”\n]{4,160})[\"”]")
_PATH_RE = re.compile(
    r"(?<![\w/])((?:[\w.-]+/)*[\w.-]+\.(?:py|pyi|js|mjs|cjs|jsx|ts|tsx|go|rs|java|kt|rb|php|cs|c|h|cc|cpp|hpp|"
    r"md|rst|toml|ya?ml|json|cfg|ini|txt|sh|ps1|html|css|scss|sql|proto))\b"
)
_CALL_RE = re.compile(r"\b([A-Za-z_][A-Za-z0-9_]{2,})\s*\(\s*\)")
_ERROR_RE = re.compile(r"\b([A-Z][A-Za-z0-9]{2,}(?:Error|Exception|Warning))\b")
_IDENTIFIER_RE = re.compile(r"^[A-Za-z_][A-Za-z0-9_.]*$")

SYNONYM_GROUPS: tuple[tuple[str, ...], ...] = (
    ("crash", "exception", "traceback", "panic", "segfault", "failure", "fail"),
    ("error", "err", "exception", "traceback", "failure", "raise"),
    ("config", "configuration", "configure", "settings", "option", "options", "env", "environment"),
    ("login", "auth", "authentication", "signin", "token", "session"),
    ("test", "tests", "pytest", "unittest", "spec", "specs"),
    ("doc", "docs", "documentation", "readme", "guide", "tutorial"),
    ("slow", "performance", "perf", "latency", "timeout", "timing", "optimization", "optimize"),
    ("memory", "leak", "cache", "gc"),
    ("ui", "frontend", "view", "render", "template", "interface"),
    ("api", "endpoint", "route", "handler", "server", "http"),
    ("database", "db", "sql", "query", "sqlite", "postgres", "mysql"),
    ("file", "path", "filename", "filepath", "io", "filesystem"),
    ("install", "installation", "dependency", "dependencies", "requirements", "package", "pip"),
    ("windows", "win32", "nt", "powershell"),
    ("unicode", "utf", "utf8", "encoding", "decode", "encode"),
    ("print", "output", "stdout", "log", "logging", "console"),
    ("parse", "parser", "parsing", "tokenize", "tokenizer"),
    ("rank", "ranking", "ranked", "score", "scoring", "relevance", "search"),
    ("plan", "planning", "steps", "roadmap", "checklist"),
    ("document", "documentation", "docs", "readme", "typo", "wording"),
    ("build", "compile", "bundle", "package", "dist"),
    ("dependency", "dependencies", "import", "imports", "require"),
)


def split_identifier(word: str) -> list[str]:
    """Split ``loadConfigFile`` / ``load_config_file`` into lowercase words."""
    word = word.strip("`'\"()[]{}.,:;!?<>")
    pieces = re.split(r"[\s_\-./\\]+", word)
    parts: list[str] = []
    for piece in pieces:
        for match in _ACRONYM_RE.finditer(piece):
            parts.append(match.group(0).lower())
    return [part for part in parts if part]


def stem(word: str) -> str:
    """A conservative suffix stripper (not a full Porter stemmer)."""
    if len(word) <= 3 or word.isdigit():
        return word

    def undouble(base: str) -> str:
        if len(base) > 3 and base[-1] == base[-2] and base[-1] not in "aeiou":
            return base[:-1]
        return base

    if word.endswith("ies") and len(word) > 4:
        return word[:-3] + "y"
    if word.endswith(("sses", "shes", "ches", "xes")):
        return word[:-2]
    if word.endswith("s") and not word.endswith(("ss", "us", "is")) and len(word) > 4:
        return word[:-1]
    if word.endswith("ing") and len(word) > 5:
        base = word[:-3]
        if len(base) >= 4:
            return undouble(base)
    if word.endswith("ed") and len(word) > 4:
        base = word[:-2]
        if len(base) >= 4:
            return undouble(base)
    return word


def tokens(text: str, *, include_stopwords: bool = False) -> list[str]:
    """Tokenize prose or code into normalized, lightly stemmed terms."""
    output: list[str] = []
    for match in _TOKEN_RE.finditer(text):
        raw = match.group(0)
        for piece in split_identifier(raw):
            if len(piece) < 2 and not piece.isdigit():
                continue
            if not include_stopwords and piece in STOPWORDS:
                continue
            output.append(stem(piece))
    return output


def unique_tokens(text: str, *, include_stopwords: bool = False) -> list[str]:
    seen: set[str] = set()
    ordered: list[str] = []
    for token in tokens(text, include_stopwords=include_stopwords):
        if token not in seen:
            seen.add(token)
            ordered.append(token)
    return ordered


def raw_tokens(text: str, *, include_stopwords: bool = False) -> list[str]:
    """Lowercase surface words (no stemming), useful for human-facing output."""
    output: list[str] = []
    for match in _TOKEN_RE.finditer(text):
        for piece in split_identifier(match.group(0)):
            if len(piece) < 2 and not piece.isdigit():
                continue
            if not include_stopwords and piece in STOPWORDS:
                continue
            output.append(piece.lower())
    return output


def unique_raw_tokens(text: str, *, include_stopwords: bool = False) -> list[str]:
    seen: set[str] = set()
    ordered: list[str] = []
    for token in raw_tokens(text, include_stopwords=include_stopwords):
        if token not in seen:
            seen.add(token)
            ordered.append(token)
    return ordered


def path_tokens(path: str) -> list[str]:
    """Tokens from a repository path, with the filename stem repeated once."""
    cleaned = path.replace("\\", "/")
    parts = [part for part in re.split(r"[/._\-]+", cleaned) if part]
    output: list[str] = []
    for part in parts:
        output.extend(split_identifier(part))
    if parts:
        output.extend(split_identifier(parts[-1]))
    seen: set[str] = set()
    result: list[str] = []
    for token in output:
        if len(token) >= 2 and token not in STOPWORDS and token not in seen:
            seen.add(token)
            result.append(stem(token))
    return result


def _group_index() -> dict[str, set[str]]:
    index: dict[str, set[str]] = {}
    for group in SYNONYM_GROUPS:
        normalized = {stem(item) for item in group}
        for item in normalized:
            index.setdefault(item, set()).update(normalized - {item})
    return index


_SYNONYMS = _group_index()
_SYNONYM_SURFACES: dict[str, list[str]] = {}
for _group in SYNONYM_GROUPS:
    for _item in _group:
        _SYNONYM_SURFACES.setdefault(stem(_item), [])
        for _other in _group:
            if _other != _item and _other not in _SYNONYM_SURFACES[stem(_item)]:
                _SYNONYM_SURFACES[stem(_item)].append(_other)


def expand_term(term: str) -> list[str]:
    """Return synonyms for a term (already stemmed, deduplicated)."""
    return sorted(_SYNONYMS.get(stem(term), set()))


def synonyms_for(term: str) -> list[str]:
    """Return human-readable synonyms for a surface term."""
    return sorted(_SYNONYM_SURFACES.get(stem(term), []))


@dataclass
class TaskSignals:
    identifiers: list[str] = field(default_factory=list)
    phrases: list[str] = field(default_factory=list)
    paths: list[str] = field(default_factory=list)
    terms: list[str] = field(default_factory=list)

    def to_dict(self) -> dict[str, object]:
        return {
            "identifiers": self.identifiers,
            "phrases": self.phrases,
            "paths": self.paths,
            "terms": self.terms,
        }


def extract_task_signals(task: str) -> TaskSignals:
    """Pull the parts of a task description that deserve special weight."""
    identifiers: list[str] = []
    phrases: list[str] = []
    paths: list[str] = []

    for match in _BACKTICK_RE.finditer(task):
        span = match.group(1).strip()
        if " " in span or "/" in span:
            if span not in phrases:
                phrases.append(span)
            inner_paths = _PATH_RE.findall(span)
            for inner in inner_paths:
                if inner not in paths:
                    paths.append(inner)
        elif _IDENTIFIER_RE.match(span) and span not in identifiers:
            identifiers.append(span)

    for match in _QUOTE_RE.finditer(task):
        phrase = match.group(1).strip()
        if phrase and phrase not in phrases:
            phrases.append(phrase)

    for match in _PATH_RE.finditer(task):
        path = match.group(1)
        if path not in paths:
            paths.append(path)
        stem_name = path.rsplit("/", 1)[-1].rsplit(".", 1)[0]
        if stem_name and stem_name not in identifiers:
            identifiers.append(stem_name)

    for match in _CALL_RE.finditer(task):
        name = match.group(1)
        if name not in identifiers and name.lower() not in STOPWORDS:
            identifiers.append(name)

    for match in _ERROR_RE.finditer(task):
        name = match.group(1)
        if name not in identifiers:
            identifiers.append(name)

    for match in _TOKEN_RE.finditer(task):
        raw = match.group(0)
        if "_" in raw or re.search(r"[a-z][A-Z]", raw):
            if raw not in identifiers and raw.lower() not in STOPWORDS and len(raw) >= 4:
                identifiers.append(raw)

    terms = unique_tokens(task)
    return TaskSignals(identifiers=identifiers, phrases=phrases, paths=paths, terms=terms)


def looks_like_test_path(path: str) -> bool:
    lowered = "/" + path.lower()
    name = path.rsplit("/", 1)[-1].lower()
    if any(
        segment in lowered
        for segment in ("/tests/", "/test/", "/spec/", "/__tests__/", "/testdata/", "/fixtures/", "/e2e/")
    ):
        return True
    if name.startswith(("test_", "tests_")) or name.endswith(("_test.py", "_test.go")):
        return True
    if name in {"conftest.py", "pytest.ini", "tox.ini"}:
        return True
    if re.search(r"\.(test|spec)\.[a-z]+$", name):
        return True
    if re.search(r"_spec\.rb$", name):
        return True
    return False


def looks_like_test_case_path(path: str) -> bool:
    """True only for files a test runner would collect as tests.

    ``tests/support.py`` and ``conftest.py`` are test *support*: useful evidence,
    but "the test to extend" should usually be a file that actually contains
    test cases.
    """
    name = path.rsplit("/", 1)[-1].lower()
    if name.startswith(("test_", "tests_", "spec_")):
        return True
    if name.endswith(("_test.py", "_test.go", "_spec.rb")):
        return True
    return bool(re.search(r"\.(test|spec)\.[a-z]+$", name))


def looks_like_doc_path(path: str) -> bool:
    lowered = path.lower()
    name = lowered.rsplit("/", 1)[-1]
    if name.startswith(("readme", "changelog", "changes", "history", "contributing", "code_of_conduct",
                        "security", "support", "authors", "maintainers")):
        return name.endswith((".md", ".rst", ".txt", ".adoc")) or "." not in name
    if lowered.startswith(("docs/", "doc/", "documentation/")):
        return name.endswith((".md", ".rst", ".txt", ".adoc"))
    return name.endswith((".md", ".rst", ".adoc")) and "/" not in path


def looks_like_config_path(path: str) -> bool:
    name = path.rsplit("/", 1)[-1].lower()
    if name in {
        "pyproject.toml",
        "setup.py",
        "setup.cfg",
        "tox.ini",
        "noxfile.py",
        "package.json",
        "package-lock.json",
        "yarn.lock",
        "pnpm-lock.yaml",
        "cargo.toml",
        "cargo.lock",
        "go.mod",
        "go.sum",
        "makefile",
        "justfile",
        "dockerfile",
        "docker-compose.yml",
        "docker-compose.yaml",
        "requirements.txt",
        "requirements-dev.txt",
        "gemfile",
        "composer.json",
        "pom.xml",
        "build.gradle",
        "build.gradle.kts",
        "tsconfig.json",
        ".pre-commit-config.yaml",
        ".editorconfig",
        "ruff.toml",
        ".ruff.toml",
        "mypy.ini",
        ".flake8",
    }:
        return True
    if name.endswith((".yml", ".yaml")) and ".github/workflows/" in path.replace("\\", "/").lower():
        return True
    if "/.github/" in "/" + path.lower():
        return True
    return False
```

## reposcout/scanning.py

`reposcout/scanning.py` - 683 lines

```py
"""Repository discovery: walk the tree, honour ignore rules, build a profile."""

from __future__ import annotations

import configparser
import json
import os
import re
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Iterable

from . import tokens as tok
from .models import Manifest, RepoProfile
from .util import (
    PROGRAMMING_LANGS,
    detect_language,
    find_line,
    first_sentence,
    now_iso,
    read_text,
    relative,
    run_git,
    sha,
    split_lines,
    truncate,
)

try:  # Python 3.11+
    import tomllib
except ImportError:  # pragma: no cover - handled in the CLI
    tomllib = None  # type: ignore[assignment]

MAX_INDEX_BYTES = 1_500_000
MAX_FILES = 30_000

SKIP_DIRS = {
    ".git",
    ".hg",
    ".svn",
    "node_modules",
    "bower_components",
    "vendor",
    "third_party",
    "site-packages",
    "__pycache__",
    ".venv",
    "venv",
    "env",
    ".env",
    ".tox",
    ".nox",
    ".pytest_cache",
    ".mypy_cache",
    ".ruff_cache",
    ".gradle",
    ".idea",
    ".vscode",
    ".next",
    ".nuxt",
    ".cache",
    ".parcel-cache",
    "dist",
    "build",
    "target",
    "coverage",
    "htmlcov",
    ".coverage",
    "eggs",
    ".eggs",
    "bin",
    "obj",
    "out",
    ".terraform",
    ".serverless",
    ".yarn",
    ".pnpm-store",
    "Pods",
    "DerivedData",
    "tmp",
    "temp",
}

SKIP_FILES = {
    "package-lock.json",
    "yarn.lock",
    "pnpm-lock.yaml",
    "poetry.lock",
    "Pipfile.lock",
    "Cargo.lock",
    "go.sum",
    "composer.lock",
    "Gemfile.lock",
}

SKIP_SUFFIXES = (".min.js", ".min.css", ".bundle.js", ".map", ".chunk.js")

# Never index credential material.  Documentation variants (``.env.example``,
# ``.env.sample``) are safe and stay indexable.
SECRET_FILENAMES = {
    ".env",
    ".env.local",
    ".env.development",
    ".env.production",
    ".env.test",
    ".npmrc",
    ".pypirc",
    ".netrc",
    "id_rsa",
    "id_dsa",
    "id_ecdsa",
    "id_ed25519",
    "credentials",
    "credentials.json",
    "secrets.yml",
    "secrets.yaml",
}
SECRET_SUFFIXES = (".pem", ".key", ".p12", ".pfx", ".keystore")


def is_secret_filename(name: str) -> bool:
    lower = name.lower()
    if lower.endswith((".example", ".sample", ".template", ".dist", ".md")):
        return False
    if lower in SECRET_FILENAMES:
        return True
    if lower.startswith(".env.") and lower.count(".") >= 2:
        return True
    return lower.endswith(SECRET_SUFFIXES)

CONFIG_FILENAMES = {
    "pyproject.toml",
    "setup.py",
    "setup.cfg",
    "tox.ini",
    "noxfile.py",
    "package.json",
    "tsconfig.json",
    "cargo.toml",
    "go.mod",
    "makefile",
    "justfile",
    "dockerfile",
    "docker-compose.yml",
    "docker-compose.yaml",
    "requirements.txt",
    "requirements-dev.txt",
    "gemfile",
    "composer.json",
    "pom.xml",
    "build.gradle",
    "build.gradle.kts",
    ".pre-commit-config.yaml",
    ".editorconfig",
    "mkdocs.yml",
    "docusaurus.config.js",
    "readthedocs.yml",
    ".readthedocs.yaml",
}


def _translate_glob(pattern: str) -> str:
    out: list[str] = []
    index = 0
    while index < len(pattern):
        char = pattern[index]
        if char == "*":
            if pattern[index : index + 2] == "**":
                index += 2
                if pattern[index : index + 1] == "/":
                    out.append("(?:.*/)?")
                    index += 1
                else:
                    out.append(".*")
                continue
            out.append("[^/]*")
        elif char == "?":
            out.append("[^/]")
        elif char == "[":
            end = pattern.find("]", index + 1)
            if end == -1:
                out.append(r"\[")
            else:
                body = pattern[index + 1 : end]
                if body.startswith("!"):
                    body = "^" + body[1:]
                out.append("[" + body + "]")
                index = end
        else:
            out.append(re.escape(char))
        index += 1
    return "".join(out)


class GitIgnore:
    """A pragmatic subset of gitignore semantics (last matching rule wins)."""

    def __init__(self, root: Path) -> None:
        self.rules: list[tuple[re.Pattern[str], bool]] = []
        for candidate in (root / ".gitignore", root / ".git" / "info" / "exclude"):
            text = read_text(candidate, limit=200_000) if candidate.exists() else None
            if text:
                self._load(text)

    def _load(self, text: str) -> None:
        for raw in split_lines(text):
            line = raw.strip()
            if not line or line.startswith("#"):
                continue
            negated = line.startswith("!")
            if negated:
                line = line[1:].strip()
            if not line:
                continue
            directory_only = line.endswith("/")
            line = line.rstrip("/")
            anchored = line.startswith("/")
            line = line.lstrip("/")
            if not line:
                continue
            prefix = "" if (anchored or "/" in line) else r"(?:.*/)?"
            regex = "^" + prefix + _translate_glob(line)
            if directory_only:
                regex += r"(?:/.*)?"
            regex += "$"
            try:
                self.rules.append((re.compile(regex), negated))
            except re.error:
                continue

    def ignored(self, rel_path: str, is_dir: bool = False) -> bool:
        target = rel_path.replace("\\", "/")
        if is_dir:
            target = target.rstrip("/") + "/"
        state = False
        for regex, negated in self.rules:
            if regex.match(target.rstrip("/")) or (is_dir and regex.match(target)):
                state = not negated
        return state


@dataclass
class ScannedFile:
    path: str
    abs_path: Path
    lang: str
    lines: int
    bytes: int
    is_test: bool = False
    is_doc: bool = False
    is_config: bool = False
    digest: str = ""

    @property
    def stem(self) -> str:
        return Path(self.path).stem


@dataclass
class ScanResult:
    root: Path
    files: list[ScannedFile]
    profile: RepoProfile
    skipped: dict[str, int] = field(default_factory=dict)

    def programming_files(self) -> list[ScannedFile]:
        return [f for f in self.files if f.lang in PROGRAMMING_LANGS]


def _should_skip_dir(name: str) -> bool:
    return name in SKIP_DIRS or name == ".git"


def discover(root: Path, *, max_files: int = MAX_FILES) -> ScanResult:
    root = root.resolve()
    ignore = GitIgnore(root)
    scanned: list[ScannedFile] = []
    skipped: dict[str, int] = {"ignored": 0, "binary_or_too_large": 0, "generated": 0, "limit": 0}

    for current, dirs, names in os.walk(root):
        current_path = Path(current)
        kept_dirs: list[str] = []
        for name in sorted(dirs):
            child = current_path / name
            rel = relative(root, child)
            if _should_skip_dir(name):
                skipped["ignored"] += 1
                continue
            if ignore.ignored(rel, is_dir=True):
                skipped["ignored"] += 1
                continue
            if child.is_symlink():
                continue
            kept_dirs.append(name)
        dirs[:] = kept_dirs

        for name in sorted(names):
            if len(scanned) >= max_files:
                skipped["limit"] += 1
                continue
            if name in SKIP_FILES or name.endswith(SKIP_SUFFIXES):
                skipped["generated"] += 1
                continue
            if is_secret_filename(name):
                skipped["ignored"] += 1
                continue
            abs_path = current_path / name
            if abs_path.is_symlink() or not abs_path.is_file():
                continue
            rel = relative(root, abs_path)
            if ignore.ignored(rel, is_dir=False):
                skipped["ignored"] += 1
                continue
            try:
                size = abs_path.stat().st_size
            except OSError:
                continue
            if size > MAX_INDEX_BYTES:
                skipped["binary_or_too_large"] += 1
                continue
            text = read_text(abs_path)
            if text is None:
                skipped["binary_or_too_large"] += 1
                continue
            lang = detect_language(rel, name)
            lower_name = name.lower()
            is_test = tok.looks_like_test_path(rel)
            is_doc = tok.looks_like_doc_path(rel)
            is_config = lower_name in CONFIG_FILENAMES or tok.looks_like_config_path(rel)
            line_count = text.count("\n") + (0 if text.endswith("\n") else 1)
            scanned.append(
                ScannedFile(
                    path=rel,
                    abs_path=abs_path,
                    lang=lang,
                    lines=line_count,
                    bytes=size,
                    is_test=is_test,
                    is_doc=is_doc,
                    is_config=is_config,
                    digest=sha(text),
                )
            )

    profile = build_profile(root, scanned)
    return ScanResult(root=root, files=scanned, profile=profile, skipped=skipped)


def detect_repo_root(start: Path) -> Path:
    """Prefer the nearest enclosing git work tree, else the start directory."""
    start = start.resolve()
    if start.is_file():
        start = start.parent
    current = start
    for _ in range(40):
        if (current / ".git").exists():
            return current
        if current.parent == current:
            break
        current = current.parent
    return start


def build_profile(root: Path, files: list[ScannedFile]) -> RepoProfile:
    language_counts: dict[str, int] = {}
    total_lines = 0
    docs: list[str] = []
    guidelines: list[str] = []
    issue_templates: list[str] = []
    workflows: list[str] = []
    manifests: list[Manifest] = []
    test_files = 0

    for item in files:
        total_lines += item.lines
        if item.lang in PROGRAMMING_LANGS:
            language_counts[item.lang] = language_counts.get(item.lang, 0) + item.lines
        if item.is_test:
            test_files += 1
        lower = item.path.lower()
        name = lower.rsplit("/", 1)[-1]
        if item.is_doc and not item.is_test:
            docs.append(item.path)
        if (
            (
                name.startswith(("contributing", "code_of_conduct", "code-of-conduct", "security", "support",
                                 "pull_request_template", "pull-request-template", "governance"))
                and (item.is_doc or "." not in name)
            )
            or lower.startswith((".github/pull_request_template", ".github/pull-request-template",
                                 ".github/issue_template/", ".github/issue-templates/"))
            or "contributing" in lower
        ):
            guidelines.append(item.path)
            if "issue_template" in lower:
                issue_templates.append(item.path)
        if (
            lower.startswith(".github/workflows/")
            or lower in {".gitlab-ci.yml", ".travis.yml", "azure-pipelines.yml", ".circleci/config.yml"}
            or lower.startswith(".buildkite/")
        ):
            workflows.append(item.path)
        manifest = _manifest_for(item.abs_path, item.path)
        if manifest:
            manifests.append(manifest)

    docs.sort(key=lambda path: (path.count("/"), path.lower()))
    guidelines.sort(key=lambda path: (path.count("/"), path.lower()))
    workflows.sort()
    manifests.sort(key=lambda m: (m.path.count("/"), m.path))
    languages = dict(sorted(language_counts.items(), key=lambda kv: (-kv[1], kv[0]))[:10])

    name, description, package_name, entry_points, license_name = _project_identity(
        root, files, manifests
    )

    if not description:
        description = _readme_summary(root, docs)

    top_level = sorted(
        {
            path.split("/")[0] + ("/" if "/" in path else "")
            for path in (item.path for item in files)
        }
    )[:40]

    profile = RepoProfile(
        root=str(root),
        name=name or root.name,
        files_indexed=len(files),
        total_lines=total_lines,
        languages=languages,
        manifests=manifests,
        docs=docs[:30],
        guidelines=guidelines[:20],
        issue_templates=issue_templates[:10],
        ci_workflows=workflows[:10],
        test_files=test_files,
        entry_points=entry_points,
        package_name=package_name,
        description=description,
        license=license_name,
        top_level=top_level,
    )
    _attach_git(root, profile)
    return profile


def _attach_git(root: Path, profile: RepoProfile) -> None:
    if not (root / ".git").exists():
        return
    profile.is_git = True
    profile.branch = run_git(root, "rev-parse", "--abbrev-ref", "HEAD")
    profile.commit = run_git(root, "rev-parse", "--short", "HEAD")
    profile.remote = run_git(root, "config", "--get", "remote.origin.url")


def _manifest_for(abs_path: Path, rel_path: str) -> Manifest | None:
    name = abs_path.name.lower()
    kind = {
        "pyproject.toml": "pyproject",
        "setup.py": "setup.py",
        "setup.cfg": "setup.cfg",
        "package.json": "package.json",
        "cargo.toml": "cargo.toml",
        "go.mod": "go.mod",
        "composer.json": "composer.json",
        "pom.xml": "pom.xml",
        "gemfile": "gemfile",
        "build.gradle": "gradle",
        "build.gradle.kts": "gradle",
        "makefile": "makefile",
        "justfile": "justfile",
        "tox.ini": "tox.ini",
        "noxfile.py": "noxfile.py",
        "requirements.txt": "requirements.txt",
        "environment.yml": "environment.yml",
        "dockerfile": "dockerfile",
        "pubspec.yaml": "pubspec.yaml",
        "mix.exs": "mix.exs",
    }.get(name)
    if not kind and rel_path.count("/") <= 1 and (name.endswith(".csproj") or name.endswith(".sln")):
        kind = "dotnet"
    if not kind:
        return None
    text = read_text(abs_path, limit=500_000) or ""
    parsed = _parse_manifest(abs_path, rel_path, kind, text)
    return Manifest(path=rel_path, kind=kind, name=parsed.get("name", ""), description=parsed.get("description", ""))


def _parse_toml(text: str) -> dict[str, Any]:
    if tomllib is None:
        return {}
    try:
        return tomllib.loads(text)
    except (tomllib.TOMLDecodeError, ValueError):
        return {}


def _parse_manifest(abs_path: Path, rel_path: str, kind: str, text: str) -> dict[str, str]:
    info: dict[str, str] = {}
    try:
        if kind == "pyproject":
            data = _parse_toml(text)
            project = data.get("project", {}) if isinstance(data, dict) else {}
            poetry = data.get("tool", {}).get("poetry", {}) if isinstance(data, dict) else {}
            source = project or poetry
            info["name"] = str(source.get("name", "") or "")
            info["description"] = str(source.get("description", "") or "")
            info["license"] = _license_value(source.get("license"))
        elif kind == "package.json":
            data = json.loads(text)
            info["name"] = str(data.get("name", "") or "")
            info["description"] = str(data.get("description", "") or "")
            info["license"] = str(data.get("license", "") or "")
        elif kind == "cargo.toml":
            data = _parse_toml(text)
            package = data.get("package", {}) if isinstance(data, dict) else {}
            info["name"] = str(package.get("name", "") or "")
            info["description"] = str(package.get("description", "") or "")
            info["license"] = str(package.get("license", "") or "")
        elif kind == "go.mod":
            match = re.search(r"^module\s+(\S+)", text, re.MULTILINE)
            info["name"] = match.group(1) if match else ""
        elif kind == "setup.cfg":
            parser = configparser.ConfigParser()
            parser.read_string(text)
            if parser.has_section("metadata"):
                info["name"] = parser.get("metadata", "name", fallback="")
                info["description"] = parser.get("metadata", "description", fallback="")
                info["license"] = parser.get("metadata", "license", fallback="")
        elif kind == "setup.py":
            info["name"] = _regex_value(text, r"name\s*=\s*[\"']([^\"']+)")
            info["description"] = _regex_value(text, r"description\s*=\s*[\"']([^\"']+)")
            info["license"] = _regex_value(text, r"license\s*=\s*[\"']([^\"']+)")
        elif kind == "composer.json":
            data = json.loads(text)
            info["name"] = str(data.get("name", "") or "")
            info["description"] = str(data.get("description", "") or "")
            info["license"] = str(data.get("license", "") or "")
        elif kind == "pom.xml":
            match = re.search(r"<artifactId>([^<]+)</artifactId>", text)
            info["name"] = match.group(1) if match else ""
            info["description"] = _regex_value(text, r"<description>([^<]+)</description>")
        elif kind == "gemfile":
            match = re.search(r"gemspec", text)
            info["name"] = "" if match else ""
        else:
            info["description"] = ""
    except (ValueError, KeyError, configparser.Error, json.JSONDecodeError):
        pass
    if not info.get("name"):
        info["name"] = Path(rel_path).stem
    info["description"] = truncate(info.get("description", ""), 300)
    return info


def _license_value(value: Any) -> str:
    if isinstance(value, str):
        return value
    if isinstance(value, dict):
        return str(value.get("text", "") or value.get("file", "") or "")
    return ""


def _regex_value(text: str, pattern: str) -> str:
    match = re.search(pattern, text)
    return match.group(1).strip() if match else ""


def _project_identity(
    root: Path, files: list[ScannedFile], manifests: list[Manifest]
) -> tuple[str, str, str, list[str], str]:
    name = ""
    description = ""
    package_name = ""
    entry_points: list[str] = []
    license_name = ""

    for manifest in manifests:
        if manifest.kind in {"pyproject", "package.json", "cargo.toml", "go.mod", "setup.cfg", "setup.py", "composer.json"}:
            if not name and manifest.name:
                name = manifest.name
            if not description and manifest.description:
                description = manifest.description
            if not package_name and manifest.name:
                package_name = manifest.name
        text = read_text(root / manifest.path, limit=500_000) or ""
        license_name = license_name or _manifest_license(text, manifest.kind)
        entry_points.extend(_manifest_entry_points(text, manifest.kind))

    if not name:
        for item in files:
            if item.path.lower().startswith("readme"):
                text = read_text(item.abs_path) or ""
                match = re.search(r"^#\s+(.+)$", text, re.MULTILINE)
                if match:
                    name = truncate(match.group(1), 80)
                break

    if not license_name:
        for item in files:
            lower = item.path.lower().rsplit("/", 1)[-1]
            if lower.startswith(("license", "licence", "copying")):
                text = read_text(item.abs_path, limit=20_000) or ""
                first = next((line.strip() for line in split_lines(text) if line.strip()), "")
                license_name = truncate(first, 60)
                break

    return name, description, package_name, sorted(set(entry_points))[:15], license_name


def _manifest_entry_points(text: str, kind: str) -> list[str]:
    found: list[str] = []
    if kind == "pyproject":
        data = _parse_toml(text)
        scripts = {}
        if isinstance(data, dict):
            project = data.get("project", {})
            scripts = project.get("scripts", {}) if isinstance(project, dict) else {}
            if not scripts:
                poetry = data.get("tool", {}).get("poetry", {})
                scripts = poetry.get("scripts", {}) if isinstance(poetry, dict) else {}
        if isinstance(scripts, dict):
            for key, value in list(scripts.items())[:20]:
                found.append(f"python -m {key}" if isinstance(value, str) else str(key))
    elif kind == "package.json":
        try:
            data = json.loads(text)
        except json.JSONDecodeError:
            return []
        bins = data.get("bin", {})
        if isinstance(bins, str):
            found.append(bins)
        elif isinstance(bins, dict):
            found.extend(str(value) for value in list(bins.values())[:10])
    elif kind == "setup.cfg":
        section = re.search(r"\[options\.entry_points\](.*?)(?:\n\[|\Z)", text, re.S)
        if section:
            for match in re.finditer(r"^\s*([\w.-]+)\s*=", section.group(1), re.MULTILINE):
                found.append(match.group(1))
    return found


def _manifest_license(text: str, kind: str) -> str:
    if kind == "pyproject":
        data = _parse_toml(text)
        project = data.get("project", {}) if isinstance(data, dict) else {}
        value = project.get("license") if isinstance(project, dict) else None
        if isinstance(value, dict):
            return str(value.get("text", "") or value.get("file", ""))
        return str(value or "")
    patterns = (
        r"license\s*=\s*[\"']([^\"']+)",
        r"^license\s*[:=]\s*(.+)$",
        r"\"license\"\s*:\s*\"([^\"]+)\"",
        r"<license[^>]*>([^<]+)</license>",
    )
    for pattern in patterns:
        match = re.search(pattern, text, re.IGNORECASE | re.MULTILINE)
        if match:
            return truncate(match.group(1), 60)
    return ""


def _readme_summary(root: Path, docs: Iterable[str]) -> str:
    for path in docs:
        name = path.lower().rsplit("/", 1)[-1]
        if name.startswith("readme"):
            text = read_text(root / path, limit=200_000) or ""
            lines = split_lines(text)
            start = find_line(lines, "## ") or len(lines)
            body = "\n".join(lines[: max(1, start - 1)])
            return first_sentence(body)
    return ""


def collect_profile(root: Path) -> RepoProfile:
    """Convenience wrapper used by ``overview`` when an index already exists."""
    return build_profile(root, list(discover(root).files))


STAMP = now_iso
```

## reposcout/parsing.py

`reposcout/parsing.py` - 1,004 lines

````py
"""Best-effort, dependency-free symbol and import extraction.

This is intentionally *not* a compiler front end.  The goal is a searchable,
explainable map of a repository: names, locations, imports and short doc
snippets.  Python uses :mod:`ast`; other languages use conservative line
patterns, and every extraction is marked with the line it came from so a
human can check it.
"""

from __future__ import annotations

import ast
import re
from dataclasses import dataclass, field

from .models import ImportRef, Symbol
from .util import first_sentence, split_lines, truncate

MAX_SYMBOLS = 400
MAX_IMPORTS = 150

JS_KEYWORDS = {
    "if",
    "for",
    "while",
    "switch",
    "catch",
    "return",
    "function",
    "constructor",
    "super",
    "await",
    "else",
    "do",
    "try",
    "typeof",
    "new",
}


@dataclass
class ParseResult:
    symbols: list[Symbol] = field(default_factory=list)
    imports: list[ImportRef] = field(default_factory=list)
    summary: str = ""


def parse_file(lang: str, text: str, path: str = "") -> ParseResult:
    parser = {
        "python": _parse_python,
        "javascript": _parse_js,
        "typescript": _parse_js,
        "vue": _parse_js,
        "svelte": _parse_js,
        "go": _parse_go,
        "rust": _parse_rust,
        "java": _parse_c_family,
        "kotlin": _parse_c_family,
        "scala": _parse_c_family,
        "csharp": _parse_c_family,
        "c": _parse_c_family,
        "cpp": _parse_c_family,
        "objective-c": _parse_c_family,
        "swift": _parse_swift,
        "ruby": _parse_ruby,
        "php": _parse_php,
        "shell": _parse_shell,
        "powershell": _parse_shell,
        "elixir": _parse_elixir,
        "haskell": _parse_haskell,
        "markdown": _parse_markdown,
        "rst": _parse_rst,
        "html": _parse_html,
        "lua": _parse_lua,
        "perl": _parse_perl,
        "r": _parse_generic_functions,
        "julia": _parse_generic_functions,
        "dart": _parse_c_family,
    }.get(lang)
    if parser is None:
        return ParseResult()
    try:
        result = parser(text)
    except (RecursionError, ValueError, re.error):
        return ParseResult()
    result.symbols = _dedupe(result.symbols)[:MAX_SYMBOLS]
    result.imports = result.imports[:MAX_IMPORTS]
    result.summary = truncate(result.summary, 240)
    return result


def _dedupe(symbols: list[Symbol]) -> list[Symbol]:
    seen: set[tuple[str, int, str]] = set()
    output: list[Symbol] = []
    for symbol in symbols:
        key = (symbol.name, symbol.line, symbol.kind)
        if key in seen:
            continue
        seen.add(key)
        output.append(symbol)
    return output


def _leading_doc(lines: list[str], line: int) -> str:
    """Collect a JS/Ruby-style comment block immediately above ``line``."""
    index = line - 2
    collected: list[str] = []
    while index >= 0 and len(collected) < 6:
        stripped = lines[index].strip()
        if stripped.startswith(("//", "#", "*", "/*", "///", "//!")):
            cleaned = stripped.lstrip("/*#! ").strip()
            if cleaned and not cleaned.startswith("@"):
                collected.append(cleaned)
            index -= 1
            continue
        break
    if not collected:
        return ""
    return first_sentence(" ".join(reversed(collected)), 160)


# ---------------------------------------------------------------------------
# Python
# ---------------------------------------------------------------------------


def _parse_python(text: str) -> ParseResult:
    try:
        tree = ast.parse(text)
    except (SyntaxError, ValueError):
        return _parse_python_regex(text)

    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    stack: list[tuple[str, str]] = []  # (name, kind)

    class Visitor(ast.NodeVisitor):
        def visit_ClassDef(self, node: ast.ClassDef) -> None:
            container = ".".join(name for name, _ in stack)
            bases = []
            for base in node.bases:
                if isinstance(base, ast.Name):
                    bases.append(base.id)
                elif isinstance(base, ast.Attribute):
                    bases.append(base.attr)
            signature = "class " + node.name + (f"({', '.join(bases)})" if bases else "")
            symbols.append(
                Symbol(
                    name=node.name,
                    kind="class",
                    line=node.lineno,
                    end_line=getattr(node, "end_lineno", None),
                    signature=truncate(signature, 120),
                    doc=_doc(node),
                    exported=not node.name.startswith("_"),
                    container=container,
                )
            )
            stack.append((node.name, "class"))
            self.generic_visit(node)
            stack.pop()

        def _function(self, node: ast.AST, kind: str) -> None:
            if not isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
                return
            container = ".".join(name for name, _ in stack)
            immediate = stack[-1][1] if stack else ""
            actual_kind = "method" if immediate == "class" else "function"
            symbols.append(
                Symbol(
                    name=node.name,
                    kind=actual_kind,
                    line=node.lineno,
                    end_line=getattr(node, "end_lineno", None),
                    signature=truncate(_py_signature(node), 160),
                    doc=_doc(node),
                    exported=not node.name.startswith("_"),
                    container=container,
                )
            )
            stack.append((node.name, "function"))
            self.generic_visit(node)
            stack.pop()

        def visit_FunctionDef(self, node: ast.FunctionDef) -> None:
            self._function(node, "function")

        def visit_AsyncFunctionDef(self, node: ast.AsyncFunctionDef) -> None:
            self._function(node, "function")

        def visit_Assign(self, node: ast.Assign) -> None:
            if not stack:
                for target in node.targets:
                    if isinstance(target, ast.Name) and target.id.isupper():
                        symbols.append(
                            Symbol(
                                name=target.id,
                                kind="constant",
                                line=node.lineno,
                                signature=truncate(ast.get_source_segment(text, node) or target.id, 120),
                                exported=not target.id.startswith("_"),
                            )
                        )
            self.generic_visit(node)

        def visit_Import(self, node: ast.Import) -> None:
            for alias in node.names:
                imports.append(ImportRef(module=alias.name, line=node.lineno, raw=f"import {alias.name}"))

        def visit_ImportFrom(self, node: ast.ImportFrom) -> None:
            module = "." * (node.level or 0) + (node.module or "")
            imports.append(
                ImportRef(
                    module=module,
                    line=node.lineno,
                    raw=f"from {module} import {', '.join(alias.name for alias in node.names)}",
                )
            )

    Visitor().visit(tree)
    summary = _doc(tree)
    return ParseResult(symbols=symbols, imports=imports, summary=summary)


def _doc(node: ast.AST) -> str:
    try:
        value = ast.get_docstring(node, clean=True)
    except TypeError:
        return ""
    if not value:
        return ""
    return first_sentence(value, 180)


def _py_signature(node: ast.FunctionDef | ast.AsyncFunctionDef) -> str:
    args: list[str] = []
    for arg in [*node.args.posonlyargs, *node.args.args]:
        args.append(arg.arg)
    if node.args.vararg:
        args.append("*" + node.args.vararg.arg)
    for arg in node.args.kwonlyargs:
        args.append(arg.arg)
    if node.args.kwarg:
        args.append("**" + node.args.kwarg.arg)
    prefix = "async def" if isinstance(node, ast.AsyncFunctionDef) else "def"
    return f"{prefix} {node.name}({', '.join(args)})"


_PY_DEF_RE = re.compile(r"^\s*(?:async\s+)?def\s+(\w+)\s*\(([^)]*)\)")
_PY_CLASS_RE = re.compile(r"^\s*class\s+(\w+)(?:\(([^)]*)\))?")
_PY_IMPORT_RE = re.compile(r"^\s*from\s+([\w.]+)\s+import\s+(.+)$|^\s*import\s+([\w., ]+)$")


def _parse_python_regex(text: str) -> ParseResult:
    """Fallback for files with syntax errors (e.g. Python 2 sources)."""
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    container = ""
    for number, line in enumerate(lines, start=1):
        class_match = _PY_CLASS_RE.match(line)
        def_match = _PY_DEF_RE.match(line)
        if class_match:
            container = class_match.group(1)
            symbols.append(
                Symbol(
                    name=container,
                    kind="class",
                    line=number,
                    signature=truncate(line.strip(), 120),
                    doc=_leading_doc(lines, number),
                    exported=not container.startswith("_"),
                )
            )
        elif def_match:
            name = def_match.group(1)
            symbols.append(
                Symbol(
                    name=name,
                    kind="method" if container else "function",
                    line=number,
                    signature=truncate(line.strip(), 160),
                    doc=_leading_doc(lines, number),
                    exported=not name.startswith("_"),
                    container=container,
                )
            )
        elif line and not line[0].isspace():
            container = ""
        import_match = _PY_IMPORT_RE.match(line)
        if import_match:
            module = import_match.group(1) or (import_match.group(3) or "").strip()
            imports.append(ImportRef(module=module, line=number, raw=line.strip()))
    summary = first_sentence("\n".join(line for line in lines[:20] if line.strip()), 200)
    return ParseResult(symbols=symbols, imports=imports, summary=summary)


# ---------------------------------------------------------------------------
# JavaScript / TypeScript
# ---------------------------------------------------------------------------

_JS_CLASS_RE = re.compile(
    r"^\s*(?:export\s+)?(?:default\s+)?(?:abstract\s+)?(class)\s+([A-Za-z_$][\w$]*)"
)
_JS_FUNCTION_RE = re.compile(
    r"^\s*(?:export\s+)?(?:default\s+)?(?:async\s+)?function\s*\*?\s*([A-Za-z_$][\w$]*)\s*\("
)
_JS_VAR_FN_RE = re.compile(
    r"^\s*(?:export\s+)?(?:const|let|var)\s+([A-Za-z_$][\w$]*)\s*(?::\s*[^=]+)?=\s*"
    r"(?:async\s+)?(?:function\b|(?:\([^)]*\)|[A-Za-z_$][\w$]*)\s*(?::[^=]+)?=>)"
)
_JS_TYPE_RE = re.compile(
    r"^\s*(?:export\s+)?(?:default\s+)?(interface|type|enum|namespace)\s+([A-Za-z_$][\w$]*)"
)
_JS_METHOD_RE = re.compile(
    r"^\s{2,}(?:(?:public|private|protected|static|async|get|set|readonly|override|abstract)\s+)*"
    r"([A-Za-z_$][\w$]*)\s*\(([^)]*)\)\s*(?::\s*[^{;]+)?\{\s*$"
)
_JS_IMPORT_RE = re.compile(
    r"^\s*import\s+(?:type\s+)?(?:[\w$*{}\s,]+?\s+from\s+)?[\"']([^\"']+)[\"']"
)
_JS_EXPORT_FROM_RE = re.compile(r"^\s*export\s+(?:type\s+)?(?:\*|\{[^}]*\})\s+from\s+[\"']([^\"']+)[\"']")
_JS_REQUIRE_RE = re.compile(r"(?:require|import)\(\s*[\"']([^\"']+)[\"']\s*\)")


def _brace_delta(line: str) -> int:
    cleaned = re.sub(r"\"(?:\\.|[^\"\\])*\"|'(?:\\.|[^'\\])*'|`(?:\\.|[^`\\])*`", "", line)
    cleaned = cleaned.split("//", 1)[0]
    return cleaned.count("{") - cleaned.count("}")


def _js_class_ranges(lines: list[str]) -> list[tuple[int, int, str]]:
    ranges: list[tuple[int, int, str]] = []
    for index, line in enumerate(lines):
        match = _JS_CLASS_RE.match(line)
        if not match:
            continue
        name = match.group(2)
        depth = 0
        started = False
        end = len(lines) - 1
        for cursor in range(index, min(len(lines), index + 4000)):
            delta = _brace_delta(lines[cursor])
            if delta:
                started = True
            depth += delta
            if started and depth <= 0:
                end = cursor
                break
        ranges.append((index, end, name))
    return ranges


def _parse_js(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    ranges = _js_class_ranges(lines)

    for index, line in enumerate(lines):
        stripped = line.strip()
        if not stripped or stripped.startswith(("//", "/*", "*")):
            continue

        class_match = _JS_CLASS_RE.match(line)
        if class_match:
            name = class_match.group(2)
            symbols.append(
                Symbol(
                    name=name,
                    kind="class",
                    line=index + 1,
                    signature=truncate(stripped, 140),
                    doc=_leading_doc(lines, index + 1),
                    exported="export" in line,
                )
            )
            continue

        function_match = _JS_FUNCTION_RE.match(line)
        if function_match:
            name = function_match.group(1)
            symbols.append(
                Symbol(
                    name=name,
                    kind="function",
                    line=index + 1,
                    signature=truncate(stripped, 160),
                    doc=_leading_doc(lines, index + 1),
                    exported="export" in line,
                )
            )

        var_match = _JS_VAR_FN_RE.match(line)
        if var_match:
            name = var_match.group(1)
            symbols.append(
                Symbol(
                    name=name,
                    kind="function",
                    line=index + 1,
                    signature=truncate(stripped, 160),
                    doc=_leading_doc(lines, index + 1),
                    exported="export" in line,
                )
            )

        type_match = _JS_TYPE_RE.match(line)
        if type_match:
            symbols.append(
                Symbol(
                    name=type_match.group(2),
                    kind=type_match.group(1),
                    line=index + 1,
                    signature=truncate(stripped, 140),
                    doc=_leading_doc(lines, index + 1),
                    exported="export" in line,
                )
            )

        for match in _JS_IMPORT_RE.finditer(line):
            imports.append(ImportRef(module=match.group(1), line=index + 1, raw=stripped))
        for match in _JS_EXPORT_FROM_RE.finditer(line):
            imports.append(ImportRef(module=match.group(1), line=index + 1, raw=stripped))
        for match in _JS_REQUIRE_RE.finditer(line):
            imports.append(ImportRef(module=match.group(1), line=index + 1, raw=stripped))

    for start, end, class_name in ranges:
        for index in range(start + 1, min(end, start + 500) + 1):
            if index >= len(lines):
                break
            method_match = _JS_METHOD_RE.match(lines[index])
            if not method_match:
                continue
            name = method_match.group(1)
            if name in JS_KEYWORDS:
                continue
            symbols.append(
                Symbol(
                    name=name,
                    kind="method",
                    line=index + 1,
                    signature=truncate(lines[index].strip(), 160),
                    doc=_leading_doc(lines, index + 1),
                    exported=True,
                    container=class_name,
                )
            )

    summary = _leading_doc(lines, 1)
    if not summary:
        for line in lines[:30]:
            stripped = line.strip()
            if stripped.startswith(("//", "/*", "*", "#!")):
                summary = stripped.lstrip("/*#! ").strip()
                if summary:
                    break
    return ParseResult(symbols=symbols, imports=imports, summary=summary)


# ---------------------------------------------------------------------------
# Go
# ---------------------------------------------------------------------------

_GO_FUNC_RE = re.compile(r"^func\s+(?:\(([^)]*)\)\s+)?(\w+)\s*\(")
_GO_TYPE_RE = re.compile(r"^type\s+(\w+)\s+(struct|interface|func|map|\[\]|chan|\w+)")
_GO_IMPORT_RE = re.compile(r"^\s*(?:import\s+)?([\w.-]*)\s*[\"`]([^\"`]+)[\"`]")


def _parse_go(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    in_import_block = False
    for index, line in enumerate(lines):
        stripped = line.strip()
        if stripped.startswith("import ("):
            in_import_block = True
            continue
        if in_import_block:
            if stripped.startswith(")"):
                in_import_block = False
                continue
            match = _GO_IMPORT_RE.match(line)
            if match:
                imports.append(ImportRef(module=match.group(2), line=index + 1, raw=stripped))
        elif stripped.startswith("import "):
            match = _GO_IMPORT_RE.match(line)
            if match:
                imports.append(ImportRef(module=match.group(2), line=index + 1, raw=stripped))

        func_match = _GO_FUNC_RE.match(line)
        if func_match:
            receiver = func_match.group(1) or ""
            name = func_match.group(2)
            container = ""
            if receiver:
                parts = receiver.replace("*", " ").split()
                container = parts[-1] if parts else ""
            symbols.append(
                Symbol(
                    name=name,
                    kind="method" if receiver else "function",
                    line=index + 1,
                    signature=truncate(stripped, 160),
                    doc=_leading_doc(lines, index + 1),
                    exported=name[:1].isupper(),
                    container=container,
                )
            )
            continue

        type_match = _GO_TYPE_RE.match(line)
        if type_match:
            name = type_match.group(1)
            kind = type_match.group(2)
            symbols.append(
                Symbol(
                    name=name,
                    kind="type" if kind in {"struct", "interface"} else "alias",
                    line=index + 1,
                    signature=truncate(stripped, 140),
                    doc=_leading_doc(lines, index + 1),
                    exported=name[:1].isupper(),
                )
            )

    summary = ""
    for line in lines[:30]:
        if line.startswith("// "):
            summary = line[3:].strip()
            break
    return ParseResult(symbols=symbols, imports=imports, summary=summary)


# ---------------------------------------------------------------------------
# Rust
# ---------------------------------------------------------------------------

_RS_FN_RE = re.compile(r"^\s*(?:pub(?:\([^)]*\))?\s+)?(?:async\s+)?(?:unsafe\s+)?fn\s+(\w+)")
_RS_TYPE_RE = re.compile(
    r"^\s*(?:pub(?:\([^)]*\))?\s+)?(struct|enum|trait|type|union)\s+(\w+)"
)
_RS_IMPL_RE = re.compile(r"^\s*impl(?:<[^>]*>)?\s+(?:[\w:<>]+\s+for\s+)?([\w:<>]+)")
_RS_MOD_RE = re.compile(r"^\s*(?:pub(?:\([^)]*\))?\s+)?mod\s+(\w+)")
_RS_USE_RE = re.compile(r"^\s*(?:pub\s+)?use\s+([^;]+);")


def _parse_rust(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    for index, line in enumerate(lines):
        stripped = line.strip()
        fn_match = _RS_FN_RE.match(line)
        if fn_match:
            name = fn_match.group(1)
            symbols.append(
                Symbol(
                    name=name,
                    kind="function",
                    line=index + 1,
                    signature=truncate(stripped, 160),
                    doc=_leading_doc(lines, index + 1),
                    exported="pub" in stripped,
                )
            )
        type_match = _RS_TYPE_RE.match(line)
        if type_match:
            symbols.append(
                Symbol(
                    name=type_match.group(2),
                    kind=type_match.group(1),
                    line=index + 1,
                    signature=truncate(stripped, 140),
                    doc=_leading_doc(lines, index + 1),
                    exported="pub" in stripped,
                )
            )
        impl_match = _RS_IMPL_RE.match(line)
        if impl_match:
            target = impl_match.group(1).split("<", 1)[0].split("::")[-1]
            symbols.append(
                Symbol(
                    name=target,
                    kind="impl",
                    line=index + 1,
                    signature=truncate(stripped, 140),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        mod_match = _RS_MOD_RE.match(line)
        if mod_match and "fn " not in line:
            symbols.append(
                Symbol(
                    name=mod_match.group(1),
                    kind="module",
                    line=index + 1,
                    signature=truncate(stripped, 120),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        use_match = _RS_USE_RE.match(line)
        if use_match:
            module = use_match.group(1).split("{", 1)[0].strip().replace("::", ".")
            imports.append(ImportRef(module=module, line=index + 1, raw=stripped))
    summary = _leading_doc(lines, 1)
    return ParseResult(symbols=symbols, imports=imports, summary=summary)


# ---------------------------------------------------------------------------
# C-family (Java, Kotlin, C#, C, C++, Scala, Dart)
# ---------------------------------------------------------------------------

_CF_TYPE_RE = re.compile(
    r"^\s*(?:(?:public|private|protected|internal|abstract|final|static|sealed|data|open|partial|"
    r"export|declare|unsafe|readonly)\s+)*(class|interface|enum|record|struct|object|protocol|extension|trait)"
    r"\s+([A-Za-z_]\w*)"
)
_CF_METHOD_RE = re.compile(
    r"^\s*(?:(?:public|private|protected|internal|static|final|abstract|virtual|override|synchronized|"
    r"native|async|inline|operator|extern|constexpr|friend|virtual)\s+)*"
    r"(?:[A-Za-z_][\w<>\[\],.?*&:]*\s+)+"
    r"([A-Za-z_]\w*)\s*\([^;]*\)\s*(?:const\s*)?(?:\{|;|=>)"
)
_CF_IMPORT_RE = re.compile(r"^\s*(?:import|using|use)\s+(?:static\s+)?([\w.:*]+)\s*;?")


def _parse_c_family(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    for index, line in enumerate(lines):
        stripped = line.strip()
        if stripped.startswith(("//", "/*", "*")):
            continue
        type_match = _CF_TYPE_RE.match(line)
        if type_match:
            name = type_match.group(2)
            symbols.append(
                Symbol(
                    name=name,
                    kind=type_match.group(1),
                    line=index + 1,
                    signature=truncate(stripped, 140),
                    doc=_leading_doc(lines, index + 1),
                    exported=not stripped.startswith("private"),
                )
            )
            continue
        method_match = _CF_METHOD_RE.match(line)
        if method_match:
            name = method_match.group(1)
            if name not in JS_KEYWORDS and name not in {"if", "for", "while", "switch", "catch", "sizeof"}:
                symbols.append(
                    Symbol(
                        name=name,
                        kind="function",
                        line=index + 1,
                        signature=truncate(stripped, 160),
                        doc=_leading_doc(lines, index + 1),
                    )
                )
        import_match = _CF_IMPORT_RE.match(line)
        if import_match:
            imports.append(ImportRef(module=import_match.group(1), line=index + 1, raw=stripped))
    summary = _leading_doc(lines, 1)
    return ParseResult(symbols=symbols, imports=imports, summary=summary)


# ---------------------------------------------------------------------------
# Swift
# ---------------------------------------------------------------------------

_SWIFT_TYPE_RE = re.compile(r"^\s*(?:public\s+|private\s+|internal\s+|open\s+|final\s+)*(class|struct|enum|protocol|extension)\s+(\w+)")
_SWIFT_FUNC_RE = re.compile(r"^\s*(?:public\s+|private\s+|internal\s+|open\s+|static\s+|class\s+)?func\s+(\w+)")
_SWIFT_IMPORT_RE = re.compile(r"^\s*import\s+(\w+)")


def _parse_swift(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    for index, line in enumerate(lines):
        type_match = _SWIFT_TYPE_RE.match(line)
        if type_match:
            symbols.append(
                Symbol(
                    name=type_match.group(2),
                    kind=type_match.group(1),
                    line=index + 1,
                    signature=truncate(line.strip(), 140),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        func_match = _SWIFT_FUNC_RE.match(line)
        if func_match:
            symbols.append(
                Symbol(
                    name=func_match.group(1),
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        import_match = _SWIFT_IMPORT_RE.match(line)
        if import_match:
            imports.append(ImportRef(module=import_match.group(1), line=index + 1, raw=line.strip()))
    return ParseResult(symbols=symbols, imports=imports, summary=_leading_doc(lines, 1))


# ---------------------------------------------------------------------------
# Ruby / PHP
# ---------------------------------------------------------------------------

_RB_DEF_RE = re.compile(r"^\s*def\s+(?:self\.)?([\w?!]+)")
_RB_TYPE_RE = re.compile(r"^\s*(class|module)\s+([\w:]+)")
_RB_REQUIRE_RE = re.compile(r"^\s*require(?:_relative)?\s+[\"']([^\"']+)[\"']")


def _parse_ruby(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    container = ""
    for index, line in enumerate(lines):
        type_match = _RB_TYPE_RE.match(line)
        if type_match:
            name = type_match.group(2)
            container = name
            symbols.append(
                Symbol(
                    name=name,
                    kind=type_match.group(1),
                    line=index + 1,
                    signature=truncate(line.strip(), 140),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        def_match = _RB_DEF_RE.match(line)
        if def_match:
            name = def_match.group(1)
            symbols.append(
                Symbol(
                    name=name,
                    kind="method" if container else "function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                    doc=_leading_doc(lines, index + 1),
                    container=container,
                )
            )
        require_match = _RB_REQUIRE_RE.match(line)
        if require_match:
            imports.append(ImportRef(module=require_match.group(1), line=index + 1, raw=line.strip()))
    return ParseResult(symbols=symbols, imports=imports, summary=_leading_doc(lines, 1))


_PHP_FUNC_RE = re.compile(
    r"^\s*(?:(?:public|private|protected|static|final|abstract)\s+)*function\s+(\w+)\s*\("
)
_PHP_TYPE_RE = re.compile(r"^\s*(?:final\s+|abstract\s+)?(class|interface|trait|enum)\s+(\w+)")
_PHP_USE_RE = re.compile(r"^\s*use\s+([\w\\]+)")


def _parse_php(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    for index, line in enumerate(lines):
        type_match = _PHP_TYPE_RE.match(line)
        if type_match:
            symbols.append(
                Symbol(
                    name=type_match.group(2),
                    kind=type_match.group(1),
                    line=index + 1,
                    signature=truncate(line.strip(), 140),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        func_match = _PHP_FUNC_RE.match(line)
        if func_match:
            symbols.append(
                Symbol(
                    name=func_match.group(1),
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        use_match = _PHP_USE_RE.match(line)
        if use_match:
            imports.append(ImportRef(module=use_match.group(1), line=index + 1, raw=line.strip()))
    return ParseResult(symbols=symbols, imports=imports, summary=_leading_doc(lines, 1))


# ---------------------------------------------------------------------------
# Shell / Elixir / Haskell / Lua / Perl / generic
# ---------------------------------------------------------------------------

_SH_FUNC_RE = re.compile(r"^\s*(?:function\s+)?([A-Za-z_][\w-]*)\s*\(\)\s*\{?")
_EX_DEF_RE = re.compile(r"^\s*def(?:p|module|macro)?\s+([\w!?]+)")
_EX_MODULE_RE = re.compile(r"^\s*defmodule\s+([\w.]+)")
_HS_SIG_RE = re.compile(r"^([a-z_][\w']*)\s*::")
_LUA_FUNC_RE = re.compile(r"^\s*(?:local\s+)?function\s+([\w.:]+)")
_PL_SUB_RE = re.compile(r"^\s*sub\s+(\w+)")


def _parse_shell(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    for index, line in enumerate(lines):
        match = _SH_FUNC_RE.match(line)
        if match and match.group(1) not in {"if", "for", "while", "case", "do"}:
            symbols.append(
                Symbol(
                    name=match.group(1),
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 120),
                    doc=_leading_doc(lines, index + 1),
                )
            )
    return ParseResult(symbols=symbols, summary=_leading_doc(lines, 1))


def _parse_elixir(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    for index, line in enumerate(lines):
        module_match = _EX_MODULE_RE.match(line)
        if module_match:
            symbols.append(
                Symbol(
                    name=module_match.group(1).split(".")[-1],
                    kind="module",
                    line=index + 1,
                    signature=truncate(line.strip(), 140),
                    container=module_match.group(1),
                )
            )
            continue
        def_match = _EX_DEF_RE.match(line)
        if def_match:
            symbols.append(
                Symbol(
                    name=def_match.group(1),
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                    doc=_leading_doc(lines, index + 1),
                )
            )
        for keyword in ("alias", "import", "require", "use"):
            if line.strip().startswith(keyword + " "):
                module = line.strip().split(None, 1)[1].split(",", 1)[0]
                imports.append(ImportRef(module=module, line=index + 1, raw=line.strip()))
    return ParseResult(symbols=symbols, imports=imports, summary=_leading_doc(lines, 1))


def _parse_haskell(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    for index, line in enumerate(lines):
        if line.startswith("import "):
            module = line.split()[1] if len(line.split()) > 1 else ""
            imports.append(ImportRef(module=module, line=index + 1, raw=line.strip()))
            continue
        match = _HS_SIG_RE.match(line)
        if match:
            symbols.append(
                Symbol(
                    name=match.group(1),
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                    doc=_leading_doc(lines, index + 1),
                )
            )
    return ParseResult(symbols=symbols, imports=imports, summary=_leading_doc(lines, 1))


def _parse_lua(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    for index, line in enumerate(lines):
        match = _LUA_FUNC_RE.match(line)
        if match:
            symbols.append(
                Symbol(
                    name=match.group(1).split(".")[-1],
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                    container=match.group(1).rsplit(".", 1)[0] if "." in match.group(1) else "",
                )
            )
    return ParseResult(symbols=symbols, summary=_leading_doc(lines, 1))


def _parse_perl(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    imports: list[ImportRef] = []
    for index, line in enumerate(lines):
        if line.startswith("use ") or line.startswith("require "):
            module = line.split()[1].rstrip(";") if len(line.split()) > 1 else ""
            imports.append(ImportRef(module=module, line=index + 1, raw=line.strip()))
        match = _PL_SUB_RE.match(line)
        if match:
            symbols.append(
                Symbol(
                    name=match.group(1),
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                )
            )
    return ParseResult(symbols=symbols, imports=imports, summary=_leading_doc(lines, 1))


def _parse_generic_functions(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    for index, line in enumerate(lines):
        match = re.match(r"^\s*(?:function\s+)?([A-Za-z_]\w*)\s*(?:<-|=)\s*function", line)
        if match:
            symbols.append(
                Symbol(
                    name=match.group(1),
                    kind="function",
                    line=index + 1,
                    signature=truncate(line.strip(), 160),
                )
            )
    return ParseResult(symbols=symbols, summary=_leading_doc(lines, 1))


# ---------------------------------------------------------------------------
# Docs
# ---------------------------------------------------------------------------


def _parse_markdown(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    in_fence = False
    for index, line in enumerate(lines):
        if line.strip().startswith("```"):
            in_fence = not in_fence
            continue
        if in_fence:
            continue
        match = re.match(r"^(#{1,6})\s+(.+?)\s*#*\s*$", line)
        if match:
            level = len(match.group(1))
            title = match.group(2).strip()
            symbols.append(
                Symbol(
                    name=truncate(title, 100),
                    kind=f"h{level}",
                    line=index + 1,
                    signature="#" * level + " " + truncate(title, 100),
                )
            )
    summary = ""
    for symbol in symbols:
        if symbol.kind == "h1":
            summary = symbol.name
            break
    return ParseResult(symbols=symbols, summary=summary)


def _parse_rst(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    for index, line in enumerate(lines[:-1]):
        stripped = line.strip()
        underline = lines[index + 1].strip()
        if stripped and re.fullmatch(r"[=\-~^\"'`#*+]{3,}", underline) and len(underline) >= len(stripped) - 2:
            symbols.append(
                Symbol(name=truncate(stripped, 100), kind="section", line=index + 1, signature=stripped)
            )
    summary = symbols[0].name if symbols else ""
    return ParseResult(symbols=symbols, summary=summary)


def _parse_html(text: str) -> ParseResult:
    lines = split_lines(text)
    symbols: list[Symbol] = []
    for index, line in enumerate(lines):
        for match in re.finditer(r"<h([1-6])[^>]*>(.*?)</h\1>", line, re.IGNORECASE):
            title = re.sub(r"<[^>]+>", "", match.group(2)).strip()
            if title:
                symbols.append(
                    Symbol(name=truncate(title, 100), kind=f"h{match.group(1)}", line=index + 1, signature=title)
                )
    summary = symbols[0].name if symbols else ""
    return ParseResult(symbols=symbols, summary=summary)
````

## reposcout/indexing.py

`reposcout/indexing.py` - 779 lines

```py
"""SQLite-backed repository index and BM25 retrieval primitives."""

from __future__ import annotations

import math
import os
import posixpath
import sqlite3
from collections import Counter, defaultdict
from dataclasses import dataclass, field
from pathlib import Path
from typing import Iterable, Sequence

from . import parsing
from . import tokens as tok
from .models import FileInfo, Symbol
from .scanning import ScanResult, ScannedFile, discover
from .util import detect_language, now_iso, posix, read_text

SCHEMA_VERSION = "1"
INDEX_DIR = ".reposcout"
INDEX_FILE = "index.sqlite3"
FIELD_WEIGHTS = {"body": 1.0, "symbol": 2.2, "path": 1.8}
K1 = 1.2
B = 0.75

SCHEMA = """
CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);
CREATE TABLE files (
    id INTEGER PRIMARY KEY,
    path TEXT NOT NULL UNIQUE,
    lang TEXT NOT NULL,
    lines INTEGER NOT NULL,
    bytes INTEGER NOT NULL,
    is_test INTEGER NOT NULL DEFAULT 0,
    is_doc INTEGER NOT NULL DEFAULT 0,
    is_config INTEGER NOT NULL DEFAULT 0,
    sha TEXT NOT NULL DEFAULT '',
    summary TEXT NOT NULL DEFAULT ''
);
CREATE TABLE symbols (
    id INTEGER PRIMARY KEY,
    file_id INTEGER NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    kind TEXT NOT NULL,
    line INTEGER NOT NULL,
    end_line INTEGER,
    signature TEXT NOT NULL DEFAULT '',
    doc TEXT NOT NULL DEFAULT '',
    exported INTEGER NOT NULL DEFAULT 1,
    container TEXT NOT NULL DEFAULT ''
);
CREATE TABLE imports (
    id INTEGER PRIMARY KEY,
    file_id INTEGER NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    module TEXT NOT NULL,
    line INTEGER NOT NULL,
    raw TEXT NOT NULL DEFAULT '',
    resolved_id INTEGER
);
CREATE TABLE edges (
    src INTEGER NOT NULL,
    dst INTEGER NOT NULL,
    kind TEXT NOT NULL DEFAULT 'import',
    PRIMARY KEY (src, dst, kind)
);
CREATE TABLE terms (
    file_id INTEGER NOT NULL,
    field TEXT NOT NULL,
    term TEXT NOT NULL,
    tf INTEGER NOT NULL,
    PRIMARY KEY (file_id, field, term)
) WITHOUT ROWID;
CREATE TABLE term_df (
    field TEXT NOT NULL,
    term TEXT NOT NULL,
    df INTEGER NOT NULL,
    PRIMARY KEY (field, term)
);
CREATE TABLE file_lens (
    file_id INTEGER NOT NULL,
    field TEXT NOT NULL,
    length INTEGER NOT NULL,
    PRIMARY KEY (file_id, field)
);
CREATE TABLE avgdl (field TEXT PRIMARY KEY, avg REAL NOT NULL);
CREATE INDEX idx_symbols_name ON symbols(name);
CREATE INDEX idx_symbols_file ON symbols(file_id);
CREATE INDEX idx_imports_file ON imports(file_id);
CREATE INDEX idx_imports_resolved ON imports(resolved_id);
CREATE INDEX idx_edges_dst ON edges(dst);
CREATE INDEX idx_edges_src ON edges(src);
CREATE INDEX idx_terms_term ON terms(term);
"""


class IndexUnavailable(RuntimeError):
    pass


@dataclass
class TermHit:
    term: str
    field: str
    tf: int
    contribution: float


@dataclass
class SearchHit:
    file_id: int
    path: str
    score: float
    term_hits: list[TermHit] = field(default_factory=list)

    def matched_terms(self) -> set[str]:
        return {hit.term for hit in self.term_hits}

    def fields_for(self, term: str) -> list[str]:
        return [hit.field for hit in self.term_hits if hit.term == term]


def default_db_path(root: Path) -> Path:
    return root / INDEX_DIR / INDEX_FILE


class Index:
    """Read-only view over a built index."""

    def __init__(self, connection: sqlite3.Connection, root: Path, db_path: Path) -> None:
        self.conn = connection
        self.root = root
        self.db_path = db_path
        self._meta: dict[str, str] | None = None
        self._symbol_by_name: dict[str, list[tuple[int, str, Symbol]]] | None = None

    # -- lifecycle ---------------------------------------------------------

    @classmethod
    def open(cls, db_path: Path, root: Path | None = None) -> "Index":
        if not db_path.exists():
            raise IndexUnavailable(f"No index found at {db_path}")
        connection = sqlite3.connect(str(db_path))
        connection.row_factory = sqlite3.Row
        connection.execute("PRAGMA query_only = ON")
        index = cls(connection, root or db_path.parent.parent, db_path)
        version = index.meta("schema_version", "")
        if version != SCHEMA_VERSION:
            index.close()
            raise IndexUnavailable(
                f"Index schema {version or 'unknown'} does not match {SCHEMA_VERSION}; rebuild it."
            )
        return index

    def close(self) -> None:
        self.conn.close()

    def __enter__(self) -> "Index":
        return self

    def __exit__(self, *exc: object) -> None:
        self.close()

    # -- metadata ----------------------------------------------------------

    def meta(self, key: str, default: str = "") -> str:
        if self._meta is None:
            self._meta = {row["key"]: row["value"] for row in self.conn.execute("SELECT key, value FROM meta")}
        return self._meta.get(key, default)

    def meta_int(self, key: str, default: int = 0) -> int:
        try:
            return int(self.meta(key, str(default)))
        except ValueError:
            return default

    @property
    def file_count(self) -> int:
        return self.meta_int("file_count")

    @property
    def indexed_at(self) -> str:
        return self.meta("created_at")

    @property
    def commit(self) -> str:
        return self.meta("git_commit")

    def stats(self) -> dict[str, object]:
        return {
            "root": self.meta("root"),
            "files": self.file_count,
            "symbols": self.conn.execute("SELECT COUNT(*) FROM symbols").fetchone()[0],
            "imports": self.conn.execute("SELECT COUNT(*) FROM imports").fetchone()[0],
            "terms": self.conn.execute("SELECT COUNT(*) FROM terms").fetchone()[0],
            "created_at": self.indexed_at,
            "commit": self.commit,
            "schema_version": self.meta("schema_version"),
            "db_path": str(self.db_path),
        }

    # -- file access -------------------------------------------------------

    def file_row(self, file_id: int) -> sqlite3.Row:
        row = self.conn.execute("SELECT * FROM files WHERE id = ?", (file_id,)).fetchone()
        if row is None:
            raise KeyError(file_id)
        return row

    def file_id(self, path: str) -> int | None:
        row = self.conn.execute("SELECT id FROM files WHERE path = ?", (path,)).fetchone()
        return int(row["id"]) if row else None

    def file_paths(self) -> list[str]:
        return [row["path"] for row in self.conn.execute("SELECT path FROM files ORDER BY path")]

    def files_info(self) -> list[FileInfo]:
        infos: list[FileInfo] = []
        for row in self.conn.execute("SELECT * FROM files ORDER BY path"):
            infos.append(
                FileInfo(
                    path=row["path"],
                    lang=row["lang"],
                    lines=row["lines"],
                    bytes=row["bytes"],
                    is_test=bool(row["is_test"]),
                    is_doc=bool(row["is_doc"]),
                    is_config=bool(row["is_config"]),
                    sha=row["sha"],
                    summary=row["summary"],
                )
            )
        return infos

    def read(self, path: str, limit: int = 500_000) -> str:
        return read_text(self.root / path, limit=limit) or ""

    def symbols_for(self, file_id: int) -> list[Symbol]:
        rows = self.conn.execute(
            "SELECT name, kind, line, end_line, signature, doc, exported, container "
            "FROM symbols WHERE file_id = ? ORDER BY line",
            (file_id,),
        ).fetchall()
        return [
            Symbol(
                name=row["name"],
                kind=row["kind"],
                line=row["line"],
                end_line=row["end_line"],
                signature=row["signature"],
                doc=row["doc"],
                exported=bool(row["exported"]),
                container=row["container"],
            )
            for row in rows
        ]

    def symbols_for_path(self, path: str) -> list[Symbol]:
        file_id = self.file_id(path)
        return self.symbols_for(file_id) if file_id is not None else []

    def imports_for(self, file_id: int) -> list[sqlite3.Row]:
        return list(
            self.conn.execute(
                "SELECT module, line, raw, resolved_id FROM imports WHERE file_id = ? ORDER BY line",
                (file_id,),
            )
        )

    def imported_paths(self, file_id: int, limit: int = 30) -> list[str]:
        rows = self.conn.execute(
            "SELECT f.path FROM imports i JOIN files f ON f.id = i.resolved_id "
            "WHERE i.file_id = ? AND i.resolved_id IS NOT NULL GROUP BY f.path LIMIT ?",
            (file_id, limit),
        )
        return [row["path"] for row in rows]

    def importers(self, file_id: int, limit: int = 30) -> list[str]:
        rows = self.conn.execute(
            "SELECT f.path FROM edges e JOIN files f ON f.id = e.src "
            "WHERE e.dst = ? GROUP BY f.path LIMIT ?",
            (file_id, limit),
        )
        return [row["path"] for row in rows]

    def neighbor_ids(self, file_id: int) -> list[int]:
        rows = self.conn.execute(
            "SELECT dst AS id FROM edges WHERE src = ? UNION SELECT src AS id FROM edges WHERE dst = ?",
            (file_id, file_id),
        )
        return [int(row["id"]) for row in rows]

    # -- symbols -----------------------------------------------------------

    def find_symbols(self, names: Iterable[str], limit: int = 30) -> list[tuple[str, Symbol]]:
        wanted = [name.lower() for name in names if name]
        if not wanted:
            return []
        if self._symbol_by_name is None:
            cache: dict[str, list[tuple[int, str, Symbol]]] = defaultdict(list)
            for row in self.conn.execute(
                "SELECT s.*, f.path FROM symbols s JOIN files f ON f.id = s.file_id"
            ):
                symbol = Symbol(
                    name=row["name"],
                    kind=row["kind"],
                    line=row["line"],
                    end_line=row["end_line"],
                    signature=row["signature"],
                    doc=row["doc"],
                    exported=bool(row["exported"]),
                    container=row["container"],
                )
                cache[row["name"].lower()].append((row["file_id"], row["path"], symbol))
            self._symbol_by_name = dict(cache)
        found: list[tuple[str, Symbol]] = []
        seen: set[tuple[str, int, str]] = set()
        for name in wanted:
            for file_id, path, symbol in self._symbol_by_name.get(name, []):
                key = (path, symbol.line, symbol.name)
                if key in seen:
                    continue
                seen.add(key)
                found.append((path, symbol))
                if len(found) >= limit:
                    return found
        return found

    def symbol_context(self, file_id: int, line: int) -> Symbol | None:
        """Innermost symbol whose body contains ``line``."""
        symbols = self.symbols_for(file_id)
        candidates = [
            symbol
            for symbol in symbols
            if symbol.line <= line and (symbol.end_line is None or line <= symbol.end_line)
        ]
        if not candidates:
            return None
        return max(candidates, key=lambda symbol: (symbol.line, symbol.end_line or symbol.line))

    # -- retrieval ---------------------------------------------------------

    def search(self, weights: dict[str, float], limit: int = 60) -> list[SearchHit]:
        """Weighted BM25 across the body, symbol and path fields."""
        terms = [term for term, weight in weights.items() if weight > 0]
        if not terms:
            return []
        placeholders = ",".join("?" for _ in terms)
        total_docs = max(1, self.file_count)
        averages = {
            row["field"]: float(row["avg"])
            for row in self.conn.execute("SELECT field, avg FROM avgdl")
        }
        dfs = {
            (row["field"], row["term"]): int(row["df"])
            for row in self.conn.execute(
                f"SELECT field, term, df FROM term_df WHERE term IN ({placeholders})", terms
            )
        }
        rows = list(
            self.conn.execute(
                f"SELECT file_id, field, term, tf FROM terms WHERE term IN ({placeholders})", terms
            )
        )
        lengths = {
            (row["file_id"], row["field"]): int(row["length"])
            for row in self.conn.execute(f"SELECT file_id, field, length FROM file_lens")
        }
        scores: dict[int, float] = defaultdict(float)
        hits: dict[int, list[TermHit]] = defaultdict(list)
        for row in rows:
            term = row["term"]
            field = row["field"]
            query_weight = weights.get(term, 0.0)
            if query_weight <= 0:
                continue
            df = dfs.get((field, term), 1)
            idf = math.log(1 + (total_docs - df + 0.5) / (df + 0.5))
            average = averages.get(field, 1.0) or 1.0
            doc_length = lengths.get((row["file_id"], field), 0)
            denominator = row["tf"] + K1 * (1 - B + B * doc_length / average)
            if denominator <= 0:
                continue
            contribution = (
                query_weight
                * FIELD_WEIGHTS.get(field, 1.0)
                * idf
                * (row["tf"] * (K1 + 1))
                / denominator
            )
            scores[row["file_id"]] += contribution
            hits[row["file_id"]].append(
                TermHit(term=term, field=field, tf=row["tf"], contribution=contribution)
            )

        ordered = sorted(scores.items(), key=lambda item: (-item[1], item[0]))[:limit]
        output: list[SearchHit] = []
        for file_id, score in ordered:
            row = self.file_row(file_id)
            output.append(
                SearchHit(
                    file_id=file_id,
                    path=row["path"],
                    score=score,
                    term_hits=sorted(hits[file_id], key=lambda hit: -hit.contribution)[:24],
                )
            )
        return output

    def idf(self, field: str, term: str) -> float:
        row = self.conn.execute(
            "SELECT df FROM term_df WHERE field = ? AND term = ?", (field, term)
        ).fetchone()
        df = int(row["df"]) if row else 0
        total = max(1, self.file_count)
        return math.log(1 + (total - df + 0.5) / (df + 0.5))

    def has_term(self, term: str) -> bool:
        row = self.conn.execute("SELECT 1 FROM term_df WHERE term = ? LIMIT 1", (term,)).fetchone()
        return row is not None

    def role_counts(self) -> dict[str, int]:
        row = self.conn.execute(
            "SELECT SUM(is_test) AS tests, SUM(is_doc) AS docs, SUM(is_config) AS configs FROM files"
        ).fetchone()
        return {
            "tests": int(row["tests"] or 0),
            "docs": int(row["docs"] or 0),
            "configs": int(row["configs"] or 0),
        }


# ---------------------------------------------------------------------------
# Building
# ---------------------------------------------------------------------------


def build_index(root: Path, db_path: Path | None = None, *, scanned: ScanResult | None = None) -> Index:
    root = root.resolve()
    db_path = db_path or default_db_path(root)
    db_path.parent.mkdir(parents=True, exist_ok=True)
    scan = scanned or discover(root)
    tmp_path = db_path.with_suffix(".building")
    for candidate in (tmp_path, Path(str(tmp_path) + "-wal"), Path(str(tmp_path) + "-shm")):
        try:
            candidate.unlink()
        except FileNotFoundError:
            pass

    started = now_iso()
    connection = sqlite3.connect(str(tmp_path))
    connection.row_factory = sqlite3.Row
    connection.execute("PRAGMA journal_mode = OFF")
    connection.execute("PRAGMA synchronous = OFF")
    connection.executescript(SCHEMA)

    path_to_id: dict[str, int] = {}
    file_rows: list[tuple[object, ...]] = []
    symbol_rows: list[tuple[object, ...]] = []
    import_rows: list[tuple[object, ...]] = []
    term_rows: list[tuple[int, str, str, int]] = []
    pending_imports: list[tuple[int, int, str, int, str]] = []

    file_id = 0
    for item in scan.files:
        text = read_text(item.abs_path) or ""
        if not text:
            continue
        file_id += 1
        path_to_id[item.path] = file_id
        parsed = parsing.parse_file(item.lang, text, item.path)
        summary = parsed.summary or _fallback_summary(text, item.lang)
        file_rows.append(
            (
                file_id,
                item.path,
                item.lang,
                item.lines,
                item.bytes,
                int(item.is_test),
                int(item.is_doc),
                int(item.is_config),
                item.digest,
                summary,
            )
        )
        for symbol in parsed.symbols:
            symbol_rows.append(
                (
                    file_id,
                    symbol.name,
                    symbol.kind,
                    symbol.line,
                    symbol.end_line,
                    symbol.signature,
                    symbol.doc,
                    int(symbol.exported),
                    symbol.container,
                )
            )
        for ref in parsed.imports:
            pending_imports.append((file_id, item.path, ref.module, ref.line, ref.raw))

        body_counts = Counter(tok.tokens(text))
        symbol_text = " ".join(
            f"{symbol.name} {symbol.qualified} {symbol.doc} {symbol.signature}" for symbol in parsed.symbols
        )
        symbol_counts = Counter(tok.tokens(symbol_text))
        path_counts = Counter(tok.path_tokens(item.path))
        for field, counts in (("body", body_counts), ("symbol", symbol_counts), ("path", path_counts)):
            for term, tf in counts.items():
                if len(term) < 2 and not term.isdigit():
                    continue
                term_rows.append((file_id, field, term, tf))

    connection.executemany(
        "INSERT INTO files (id, path, lang, lines, bytes, is_test, is_doc, is_config, sha, summary) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)",
        file_rows,
    )
    connection.executemany(
        "INSERT INTO symbols (file_id, name, kind, line, end_line, signature, doc, exported, container) "
        "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
        symbol_rows,
    )
    connection.executemany(
        "INSERT INTO terms (file_id, field, term, tf) VALUES (?, ?, ?, ?)", term_rows
    )

    resolver = _ImportResolver(scan.files, path_to_id)
    edges: set[tuple[int, int, str]] = set()
    for src, src_path, module, line, raw in pending_imports:
        resolved_path = resolver.resolve(src_path, module)
        resolved_id = path_to_id.get(resolved_path) if resolved_path else None
        import_rows.append((src, module, line, raw, resolved_id))
        if resolved_id and resolved_id != src:
            edges.add((src, resolved_id, "import"))
    connection.executemany(
        "INSERT INTO imports (file_id, module, line, raw, resolved_id) VALUES (?, ?, ?, ?, ?)",
        import_rows,
    )
    connection.executemany("INSERT INTO edges (src, dst, kind) VALUES (?, ?, ?)", list(edges))

    connection.execute(
        "INSERT INTO term_df (field, term, df) "
        "SELECT field, term, COUNT(DISTINCT file_id) FROM terms GROUP BY field, term"
    )
    connection.execute(
        "INSERT INTO file_lens (file_id, field, length) "
        "SELECT file_id, field, SUM(tf) FROM terms GROUP BY file_id, field "
        "ON CONFLICT(file_id, field) DO UPDATE SET length = excluded.length"
    )
    connection.execute(
        "INSERT INTO avgdl (field, avg) "
        "SELECT field, AVG(length) FROM file_lens GROUP BY field"
    )
    meta = {
        "schema_version": SCHEMA_VERSION,
        "root": str(root),
        "created_at": started,
        "file_count": str(len(file_rows)),
        "git_commit": scan.profile.commit,
        "git_branch": scan.profile.branch,
        "builder": "reposcout",
    }
    connection.executemany("INSERT INTO meta (key, value) VALUES (?, ?)", list(meta.items()))
    connection.commit()
    connection.execute("PRAGMA optimize")
    connection.close()

    os.replace(tmp_path, db_path)
    index = Index.open(db_path, root)
    return index


def _fallback_summary(text: str, lang: str) -> str:
    for line in text.split("\n")[:15]:
        stripped = line.strip()
        if not stripped or stripped.startswith(("#", "//", "/*", "*", "import ", "from ", "use ", "package ")):
            continue
        return stripped[:200]
    return ""


def ensure_index(
    root: Path,
    *,
    refresh: bool = False,
    db_path: Path | None = None,
    scanned: ScanResult | None = None,
) -> tuple[Index, bool]:
    """Open an existing index or build one. Returns ``(index, built)``."""
    target = db_path or default_db_path(root)
    if refresh or not target.exists():
        index = build_index(root, target, scanned=scanned)
        return index, True
    try:
        return Index.open(target, root), False
    except IndexUnavailable:
        index = build_index(root, target, scanned=scanned)
        return index, True


def _module_key(path: str) -> str:
    cleaned = path
    for suffix in (".py", ".pyi"):
        if cleaned.endswith(suffix):
            cleaned = cleaned[: -len(suffix)]
            break
    if cleaned.endswith("/__init__"):
        cleaned = cleaned[: -len("/__init__")]
    return cleaned.replace("/", ".")


class _ImportResolver:
    """Resolve imports to indexed files using language-specific conventions."""

    JS_EXTENSIONS = (".ts", ".tsx", ".js", ".jsx", ".mjs", ".cjs", ".vue", ".svelte", ".json")
    PY_EXTENSIONS = (".py", ".pyi")

    def __init__(self, files: Sequence[ScannedFile], path_to_id: dict[str, int]) -> None:
        self.files = {item.path: item for item in files}
        self.path_to_id = path_to_id
        self.by_path = set(path_to_id)
        self.py_modules: dict[str, str] = {}
        self.by_basename: dict[str, list[str]] = defaultdict(list)
        self.by_dir: dict[str, list[str]] = defaultdict(list)
        for item in files:
            if item.lang in {"python"}:
                self.py_modules.setdefault(_module_key(item.path), item.path)
            self.by_basename[Path(item.path).stem.lower()].append(item.path)
            self.by_dir[posix(Path(item.path).parent)].append(item.path)

    def resolve(self, src_path: str, module: str) -> str | None:
        src = self.files.get(src_path)
        lang = src.lang if src else detect_language(src_path, Path(src_path).name)
        if not module:
            return None
        if lang == "python":
            return self._resolve_python(src_path, module)
        if lang in {"javascript", "typescript", "vue", "svelte"}:
            return self._resolve_js(src_path, module)
        if lang == "go":
            return self._resolve_by_suffix(module, {".go"})
        if lang == "rust":
            return self._resolve_rust(src_path, module)
        if lang in {"java", "kotlin", "scala"}:
            return self._resolve_by_class(module, {".java", ".kt", ".kts", ".scala"})
        if lang == "csharp":
            return self._resolve_by_class(module, {".cs"})
        if lang == "ruby":
            return self._resolve_relative_or_suffix(src_path, module, {".rb"})
        if lang == "php":
            return self._resolve_by_class(module.replace("\\", "/"), {".php"})
        if lang in {"c", "cpp", "objective-c"}:
            return self._resolve_relative_or_suffix(src_path, module, {".h", ".hpp"})
        if lang == "swift":
            return self._resolve_by_class(module, {".swift"})
        return self._resolve_by_class(module, {".py", ".js", ".ts", ".go", ".rs", ".rb"})

    def _resolve_python(self, src_path: str, module: str) -> str | None:
        if module.startswith("."):
            level = len(module) - len(module.lstrip("."))
            remainder = module.lstrip(".")
            base = posix(Path(src_path).parent)
            for _ in range(max(0, level - 1)):
                base = posix(Path(base).parent) if base not in ("", ".") else ""
            prefix = (base + "." if base else "") + remainder
            key = prefix if remainder else base
        else:
            key = module
        key = key.strip(".")
        if key in self.py_modules:
            return self.py_modules[key]
        # ``from package import name`` may target a submodule.
        for candidate in (f"{key}.__init__", f"{key}"):
            if candidate in self.py_modules:
                return self.py_modules[candidate]
        return self._resolve_by_suffix(key.replace(".", "/"), set(self.PY_EXTENSIONS))

    def _resolve_js(self, src_path: str, module: str) -> str | None:
        if not module.startswith((".", "/")):
            if module.startswith("@/"):
                base = "src/" + module[2:]
            else:
                return None
        else:
            base = posix(posixpath.normpath(posixpath.join(posix(Path(src_path).parent), module)))
        if base.startswith("/"):
            base = base.lstrip("/")
        candidates = [base]
        stem, ext = posixpath.splitext(base)
        if ext:
            candidates.extend([stem, stem + "/index"])
        for extension in self.JS_EXTENSIONS:
            candidates.append(base + extension)
            candidates.append(base + "/index" + extension)
        for candidate in candidates:
            if candidate in self.by_path:
                return candidate
        return None

    def _resolve_rust(self, src_path: str, module: str) -> str | None:
        cleaned = module.replace("::", "/")
        base = posix(Path(src_path).parent)
        if cleaned.startswith("crate/"):
            cleaned = cleaned[len("crate/") :]
            base = ""
        elif cleaned.startswith("self/"):
            cleaned = cleaned[len("self/") :]
        elif cleaned.startswith("super/"):
            while cleaned.startswith("super/"):
                cleaned = cleaned[len("super/") :]
                base = posix(Path(base).parent) if base not in ("", ".") else ""
        else:
            return None
        for candidate in (
            posixpath.join(base, cleaned + ".rs"),
            posixpath.join(base, cleaned, "mod.rs"),
            posixpath.join("src", cleaned + ".rs"),
            posixpath.join("src", cleaned, "mod.rs"),
        ):
            if candidate in self.by_path:
                return candidate
        return None

    def _resolve_by_class(self, module: str, extensions: set[str]) -> str | None:
        last = module.replace("\\", "/").split("/")[-1].split(".")[-1]
        for candidate in self.by_basename.get(last.lower(), []):
            if Path(candidate).suffix.lower() in extensions:
                return candidate
        return None

    def _resolve_relative_or_suffix(self, src_path: str, module: str, extensions: set[str]) -> str | None:
        base = posix(Path(src_path).parent)
        candidates = [posixpath.normpath(posixpath.join(base, module))]
        candidates.append(posixpath.normpath(module))
        for candidate in candidates:
            for extension in extensions:
                for option in (candidate + extension, candidate):
                    if option in self.by_path:
                        return option
        suffix = module.replace("\\", "/").lstrip("./")
        for path in self.by_path:
            if Path(path).suffix.lower() not in extensions:
                continue
            if path.endswith(suffix) or path.endswith("/" + suffix):
                return path
        return None

    def _resolve_by_suffix(self, module: str, extensions: set[str]) -> str | None:
        cleaned = module.strip("/").replace(".", "/") if extensions == self.PY_EXTENSIONS else module.strip("/")
        parts = [part for part in cleaned.split("/") if part]
        if not parts:
            return None
        for length in range(min(6, len(parts)), 0, -1):
            suffix = "/".join(parts[-length:])
            for item in self.files.values():
                if Path(item.path).suffix.lower() not in extensions:
                    continue
                directory = posix(Path(item.path).parent)
                if directory.endswith(suffix):
                    return item.path
        return None


def index_overview(index: Index) -> dict[str, object]:
    stats = index.stats()
    rows = list(
        index.conn.execute(
            "SELECT lang, COUNT(*) AS files, SUM(lines) AS lines FROM files "
            "GROUP BY lang ORDER BY lines DESC LIMIT 12"
        )
    )
    stats["languages"] = [
        {"lang": row["lang"], "files": row["files"], "lines": row["lines"]} for row in rows
    ]
    stats["roles"] = index.role_counts()
    return stats
```

## reposcout/retrieval.py

`reposcout/retrieval.py` - 756 lines

```py
"""Task-to-code mapping with line-level evidence.

Given a prose task, RepoScout combines:

* weighted BM25 over three fields (file text, symbol names/docs, path tokens);
* hard boosts for identifiers and paths that appear verbatim in the task;
* one hop of the resolved import graph;
* test-file association by module name and import edges.

Every ranked file carries the reasons it surfaced and, where possible, the
exact lines that support those reasons.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from pathlib import Path
from typing import Iterable, Sequence

from . import tokens as tok
from .indexing import Index, TermHit
from .models import (
    FileHit,
    Mapping,
    QueryTerm,
    Reason,
    Snippet,
    Source,
    Symbol,
)
from .util import clamp, module_label, split_lines, truncate

GENERIC_MODULE_TOKENS = {"test", "tests", "spec", "specs", "src", "lib", "main", "init", "index", "pkg"}
MAX_READ_BYTES = 400_000


@dataclass
class RetrievalOptions:
    max_primary: int = 5
    max_supporting: int = 6
    max_tests: int = 4
    max_docs: int = 4
    expand_graph: bool = True
    read_limit: int = 48


@dataclass
class _Candidate:
    file_id: int
    path: str
    row: object
    score: float = 0.0
    reasons: list[Reason] = field(default_factory=list)
    text: str = ""
    snippets: list[Snippet] = field(default_factory=list)
    term_hits: list[TermHit] = field(default_factory=list)
    graph_from: str | None = None
    relevant: list[Symbol] = field(default_factory=list)

    @property
    def is_test(self) -> bool:
        return bool(self.row["is_test"])

    @property
    def is_doc(self) -> bool:
        return bool(self.row["is_doc"])

    @property
    def is_config(self) -> bool:
        return bool(self.row["is_config"])


def build_query(
    index: Index, task: str, extra_terms: Iterable[str] | None = None
) -> tuple[object, list[QueryTerm], dict[str, float]]:
    """Turn a prose task into weighted, deduplicated index terms."""
    signals = tok.extract_task_signals(task)
    query_terms: list[QueryTerm] = []
    weights: dict[str, float] = {}
    seen_stems: dict[str, str] = {}

    def add(surface: str, weight: float, origin: str, note: str = "") -> None:
        surface = surface.strip()
        if not surface:
            return
        stem = tok.stem(surface.lower())
        if not stem or (len(stem) < 2 and not stem.isdigit()):
            return
        if stem in tok.STOPWORDS:
            return
        if stem in seen_stems:
            return
        seen_stems[stem] = surface
        query_terms.append(QueryTerm(surface, weight, origin, note))
        weights[stem] = weight

    for raw in tok.unique_raw_tokens(task):
        add(raw, 1.0, "task")

    for identifier in signals.identifiers:
        for piece in tok.split_identifier(identifier):
            add(piece, 1.25, "identifier", f"from `{identifier}`")

    expansion_pool: list[tuple[float, str, str]] = []
    for item in list(query_terms):
        for synonym in tok.synonyms_for(item.term):
            stem = tok.stem(synonym)
            if stem in seen_stems or not index.has_term(stem):
                continue
            expansion_pool.append((index.idf("body", stem), synonym, item.term))
    expansion_pool.sort(key=lambda entry: (-entry[0], entry[1]))
    for _, synonym, parent in expansion_pool[:16]:
        add(synonym, 0.45, "expansion", f"synonym of '{parent}' present in this repo")

    for raw in extra_terms or []:
        stem = tok.stem(raw)
        if stem in seen_stems or not index.has_term(stem):
            continue
        add(raw, 0.8, "llm", "model-suggested term; verified to exist in the index")

    return signals, query_terms, weights


def map_task(
    index: Index,
    task: str,
    *,
    options: RetrievalOptions | None = None,
    extra_terms: Iterable[str] | None = None,
) -> Mapping:
    options = options or RetrievalOptions()
    signals, query_terms, weights = build_query(index, task, extra_terms)
    mapping = Mapping(task=task, query_terms=query_terms, identifiers=signals.identifiers, phrases=signals.phrases)
    if not weights:
        mapping.notes.append(
            "The task contains no searchable terms after removing filler words. "
            "Try including a symbol name, filename or error message."
        )
        mapping.confidence_label = "low"
        mapping.confidence_rationale.append("No query terms were available.")
        return mapping

    hits = index.search(weights, limit=120)
    candidates: dict[str, _Candidate] = {}
    for hit in hits:
        row = index.file_row(hit.file_id)
        candidates[hit.path] = _Candidate(
            file_id=hit.file_id, path=hit.path, row=row, score=hit.score, term_hits=hit.term_hits
        )

    _apply_path_boosts(index, signals, candidates, mapping)
    _apply_symbol_boosts(index, signals, candidates)

    for candidate in candidates.values():
        _add_match_reason(candidate, query_terms, weights)

    if options.expand_graph:
        mapping.graph_expansions = _expand_graph(index, candidates, options)

    # Read the most promising files once; snippets and phrase bonuses need text.
    ordered = sorted(candidates.values(), key=lambda c: (-c.score, c.path))
    for candidate in ordered[: options.read_limit]:
        candidate.text = index.read(candidate.path, MAX_READ_BYTES)
        _apply_phrase_boosts(candidate, mapping)

    # Build evidence snippets and relevant symbols for the strongest files.
    for candidate in sorted(candidates.values(), key=lambda c: (-c.score, c.path))[: options.read_limit]:
        if not candidate.text:
            candidate.text = index.read(candidate.path, MAX_READ_BYTES)
        symbols = index.symbols_for(candidate.file_id)
        candidate.snippets = _extract_snippets(candidate, weights, signals.identifiers, signals.phrases, symbols)
        candidate.relevant = _relevant_symbols(
            symbols, weights, signals.identifiers, {snippet.line for snippet in candidate.snippets}
        )
        candidate.reasons.extend(_symbol_reasons(candidate, symbols, weights, signals.identifiers))

    _classify(index, candidates, mapping, options)

    _attach_fan_in(index, mapping)
    _link_tests(index, mapping, weights, signals.identifiers, signals.phrases, options)

    _score_confidence(mapping, weights, query_terms)
    _add_mapping_notes(mapping)
    return mapping


# ---------------------------------------------------------------------------
# Boosts
# ---------------------------------------------------------------------------


def _add_candidate(index: Index, candidates: dict[str, _Candidate], path: str) -> _Candidate | None:
    if path in candidates:
        return candidates[path]
    file_id = index.file_id(path)
    if file_id is None:
        return None
    row = index.file_row(file_id)
    candidate = _Candidate(file_id=file_id, path=path, row=row, score=0.0)
    candidates[path] = candidate
    return candidate


def _apply_path_boosts(
    index: Index, signals: object, candidates: dict[str, _Candidate], mapping: Mapping
) -> None:
    all_paths = index.file_paths()
    for referenced in getattr(signals, "paths", []):
        matches = [path for path in all_paths if path == referenced or path.endswith("/" + referenced)]
        if not matches:
            mapping.notes.append(f"The referenced path `{referenced}` is not in the index.")
            continue
        for path in matches[:3]:
            candidate = _add_candidate(index, candidates, path)
            if candidate is None:
                continue
            candidate.score += 40.0
            candidate.reasons.append(
                Reason(
                    text=f"Task references this path directly (`{referenced}`)",
                    kind="path",
                    strength=40.0,
                    sources=[Source(path=path)],
                )
            )

    identifiers = [name.lower() for name in getattr(signals, "identifiers", [])]
    if not identifiers:
        return
    for candidate in list(candidates.values()):
        filename = Path(candidate.path).name.lower()
        stem = Path(candidate.path).stem.lower()
        if filename in identifiers or stem in identifiers:
            candidate.score += 9.0
            candidate.reasons.append(
                Reason(
                    text=f"Filename matches task identifier `{stem}`",
                    kind="path",
                    strength=9.0,
                    sources=[Source(candidate.path)],
                )
            )


def _apply_symbol_boosts(index: Index, signals: object, candidates: dict[str, _Candidate]) -> None:
    identifiers = {name.lower() for name in getattr(signals, "identifiers", [])}
    if not identifiers:
        return
    found = index.find_symbols([name for name in identifiers], limit=80)
    per_file: dict[str, float] = {}
    for path, symbol in found:
        exact = symbol.name.lower() in identifiers or symbol.qualified.lower() in identifiers
        if not exact:
            continue
        candidate = _add_candidate(index, candidates, path)
        if candidate is None:
            continue
        boost = min(24.0, 13.0 + (symbol.end_line or symbol.line) / 400.0)
        added = per_file.get(path, 0.0)
        if added >= 30:
            continue
        boost = min(boost, 30.0 - added)
        per_file[path] = added + boost
        candidate.score += boost
        candidate.reasons.append(
            Reason(
                text=f"Defines task identifier `{symbol.qualified}` ({symbol.kind})",
                kind="symbol",
                strength=boost,
                sources=[Source(path=path, line=symbol.line, snippet=symbol.signature or symbol.name)],
            )
        )


def _expand_graph(index: Index, candidates: dict[str, _Candidate], options: RetrievalOptions) -> int:
    added = 0
    seeds = [
        candidate
        for candidate in sorted(candidates.values(), key=lambda c: -c.score)
        if candidate.score > 0 and not candidate.is_test and not candidate.is_doc
    ][:6]
    for seed in seeds:
        for neighbor_id in index.neighbor_ids(seed.file_id)[:6]:
            row = index.file_row(neighbor_id)
            if row["is_doc"] or row["is_config"]:
                continue
            path = row["path"]
            bonus = 0.8 + 0.22 * seed.score
            existing = candidates.get(path)
            if existing is None:
                if added >= 10:
                    continue
                existing = _Candidate(file_id=neighbor_id, path=path, row=row, score=bonus, graph_from=seed.path)
                candidates[path] = existing
                added += 1
            elif existing.score < bonus * 0.5:
                existing.score += bonus * 0.35
                existing.graph_from = existing.graph_from or seed.path
            else:
                continue
            existing.reasons.append(
                Reason(
                    text=f"Connected to `{seed.path}` through a resolved import",
                    kind="graph",
                    strength=round(bonus, 2),
                    sources=[Source(path=seed.path)],
                    inferred=True,
                )
            )
    return added


def _apply_phrase_boosts(candidate: _Candidate, mapping: Mapping) -> None:
    if not candidate.text or not mapping.phrases:
        return
    lowered = candidate.text.lower()
    for phrase in mapping.phrases:
        if phrase.lower() not in lowered:
            continue
        candidate.score += 3.0
        line = _line_containing(candidate.text, phrase)
        candidate.reasons.append(
            Reason(
                text=f"Contains quoted phrase \"{truncate(phrase, 80)}\"",
                kind="match",
                strength=3.0,
                sources=[Source(path=candidate.path, line=line, snippet=truncate(phrase, 120))],
            )
        )


def _line_containing(text: str, phrase: str) -> int | None:
    lowered = phrase.lower()
    for index, line in enumerate(split_lines(text), start=1):
        if lowered in line.lower():
            return index
    return None


# ---------------------------------------------------------------------------
# Evidence
# ---------------------------------------------------------------------------


def _add_match_reason(candidate: _Candidate, query_terms: Sequence[QueryTerm], weights: dict[str, float]) -> None:
    if not candidate.term_hits:
        return
    fields: dict[str, list[str]] = {}
    for hit in candidate.term_hits:
        surface = _surface_for(hit.term, query_terms)
        fields.setdefault(hit.field, []).append(surface)
    labels = {"body": "file text", "symbol": "symbols", "path": "path"}

    def unique(names: list[str]) -> list[str]:
        seen: set[str] = set()
        ordered: list[str] = []
        for name in names:
            if name not in seen:
                seen.add(name)
                ordered.append(name)
        return ordered[:5]

    pieces = [f"{labels.get(field, field)}: {', '.join(unique(names))}" for field, names in fields.items()]
    candidate.reasons.append(
        Reason(
            text="Matched " + "; ".join(pieces),
            kind="match",
            strength=min(6.0, candidate.score / 6),
        )
    )


def _surface_for(term: str, query_terms: Sequence[QueryTerm]) -> str:
    for query_term in query_terms:
        if tok.stem(query_term.term) == term:
            return query_term.term
    return term


def _relevant_symbols(
    symbols: Sequence[Symbol], weights: dict[str, float], identifiers: Sequence[str], snippet_lines: set[int]
) -> list[Symbol]:
    identifier_set = {name.lower() for name in identifiers}
    scored: list[tuple[float, Symbol]] = []
    for symbol in symbols:
        name_tokens = set(tok.tokens(symbol.name + " " + symbol.qualified))
        overlap = len(name_tokens & set(weights))
        exact = symbol.name.lower() in identifier_set or symbol.qualified.lower() in identifier_set
        near = any(abs(symbol.line - line) <= 2 for line in snippet_lines)
        doc_overlap = len(set(tok.tokens(symbol.doc)) & set(weights))
        score = overlap * 2.5 + doc_overlap * 0.8 + (6.0 if exact else 0.0) + (1.0 if near else 0.0)
        if symbol.kind in {"h1", "h2", "h3", "section"}:
            score += 0.5
        scored.append((score, symbol))
    scored.sort(key=lambda pair: (-pair[0], pair[1].line))
    selected = [symbol for score, symbol in scored if score > 0][:6]
    if not selected:
        selected = sorted(symbols, key=lambda symbol: symbol.line)[:4]
    return sorted(selected, key=lambda symbol: symbol.line)


def _extract_snippets(
    candidate: _Candidate,
    weights: dict[str, float],
    identifiers: Sequence[str],
    phrases: Sequence[str],
    symbols: Sequence[Symbol],
) -> list[Snippet]:
    if not candidate.text:
        return []
    terms = set(weights)
    identifier_set = [name.lower() for name in identifiers]
    symbol_at = {symbol.line: symbol for symbol in symbols}
    lines = split_lines(candidate.text)
    scored: list[tuple[float, int, set[str], str]] = []
    for index, line in enumerate(lines):
        stripped = line.strip()
        if not stripped or len(stripped) > 1200:
            continue
        line_tokens = set(tok.tokens(line))
        matched = line_tokens & terms
        lowers = stripped.lower()
        phrase_hit = any(phrase.lower() in lowers for phrase in phrases)
        ident_hit = any(name in lowers for name in identifier_set)
        if not matched and not phrase_hit and not ident_hit:
            continue
        score = 2.0 * len(matched)
        if index + 1 in symbol_at:
            score += 2.5
        if phrase_hit:
            score += 3.0
        if ident_hit:
            score += 2.0
        if stripped.startswith(("#", "//", "/*", "*")):
            score -= 0.4
        scored.append((score, index, matched, stripped))

    scored.sort(key=lambda item: (-item[0], item[1]))
    snippets: list[Snippet] = []
    used_text: set[str] = set()
    covered: set[str] = set()
    for _, index, matched, stripped in scored:
        if len(snippets) >= 4:
            break
        if stripped in used_text:
            continue
        used_text.add(stripped)
        covered |= matched
        symbol = _symbol_containing(symbols, index + 1)
        snippets.append(
            Snippet(
                line=index + 1,
                text=truncate(stripped, 220),
                terms=sorted(matched)[:8],
                symbol=symbol.qualified if symbol else None,
            )
        )

    # Guarantee at least a little coverage of every meaningful query term.
    for term in sorted(terms):
        if term in covered or len(snippets) >= 4:
            continue
        for index, line in enumerate(lines):
            if term in set(tok.tokens(line)):
                snippets.append(
                    Snippet(
                        line=index + 1,
                        text=truncate(line.strip(), 220),
                        terms=[term],
                        symbol=(_symbol_containing(symbols, index + 1).qualified if _symbol_containing(symbols, index + 1) else None),
                    )
                )
                covered.add(term)
                break
    return sorted(snippets, key=lambda snippet: snippet.line)


def _symbol_containing(symbols: Sequence[Symbol], line: int) -> Symbol | None:
    matches = [
        symbol
        for symbol in symbols
        if symbol.line <= line and (symbol.end_line is None or line <= symbol.end_line)
    ]
    if not matches:
        return None
    return max(matches, key=lambda symbol: (symbol.line, symbol.end_line or symbol.line))


def _symbol_reasons(
    candidate: _Candidate, symbols: Sequence[Symbol], weights: dict[str, float], identifiers: Sequence[str]
) -> list[Reason]:
    reasons: list[Reason] = []
    identifier_set = {name.lower() for name in identifiers}
    for symbol in symbols:
        exact = symbol.name.lower() in identifier_set or symbol.qualified.lower() in identifier_set
        tokens = set(tok.tokens(symbol.name + " " + symbol.qualified)) & set(weights)
        if not exact and not tokens:
            continue
        if candidate.is_doc and symbol.kind not in {"h1", "h2", "h3", "section"}:
            continue
        reasons.append(
            Reason(
                text=f"Defines `{symbol.qualified}` ({symbol.kind}) at line {symbol.line}",
                kind="symbol",
                strength=3.0 if exact else 1.0,
                sources=[Source(candidate.path, symbol.line, symbol.signature or symbol.name)],
            )
        )
        if len(reasons) >= 4:
            break
    return reasons


# ---------------------------------------------------------------------------
# Classification and confidence
# ---------------------------------------------------------------------------


def _classify(
    index: Index,
    candidates: dict[str, _Candidate],
    mapping: Mapping,
    options: RetrievalOptions,
) -> None:
    ordered = sorted(candidates.values(), key=lambda c: (-c.score, c.path))
    threshold = max(0.9, (ordered[0].score * 0.06) if ordered else 0)

    code = [c for c in ordered if not c.is_test and not c.is_doc and not c.is_config and c.score >= threshold]
    docs = [c for c in ordered if c.is_doc and c.score >= threshold]
    configs = [c for c in ordered if c.is_config and c.score >= threshold]
    tests = [c for c in ordered if c.is_test and c.score >= threshold]

    # Documentation- or configuration-only tasks still deserve a target.
    if not code and docs:
        code = docs[:1]
    if not code and configs:
        code = configs[:1]

    selected = code[: options.max_primary]
    mapping.primaries = [
        _to_hit(hit, "doc" if hit.is_doc else "config" if hit.is_config else "primary", index)
        for hit in selected
    ]
    primary_paths = {hit.path for hit in mapping.primaries}

    remaining_code = [c for c in code[options.max_primary:] if c.score > threshold]
    supporting = list(remaining_code)
    for candidate in [*configs, *ordered]:
        if len(supporting) >= options.max_supporting:
            break
        if candidate.path in primary_paths or any(c.path == candidate.path for c in supporting):
            continue
        if candidate.is_test or candidate.is_doc:
            continue
        if candidate.score <= threshold:
            continue
        supporting.append(candidate)
    mapping.supporting = [_to_hit(c, "supporting", index) for c in supporting]

    mapping.tests = [_to_hit(c, "test", index) for c in tests[: options.max_tests]]
    seen = primary_paths | {hit.path for hit in mapping.tests} | {hit.path for hit in mapping.supporting}
    mapping.docs = [
        _to_hit(c, "doc", index) for c in docs if c.path not in seen
    ][: options.max_docs]


def _to_hit(candidate: _Candidate, role: str, index: Index) -> FileHit:
    relevant = candidate.relevant or index.symbols_for(candidate.file_id)[:4]
    return FileHit(
        path=candidate.path,
        role=role,  # type: ignore[arg-type]
        score=round(candidate.score, 3),
        lang=candidate.row["lang"],
        lines=candidate.row["lines"],
        reasons=candidate.reasons,
        snippets=candidate.snippets,
        symbols=relevant,
        imports=index.imported_paths(candidate.file_id, limit=8),
        imported_by=[],
        module=module_label(candidate.path),
    )


def _attach_fan_in(index: Index, mapping: Mapping) -> None:
    for hit in mapping.primaries + mapping.supporting:
        importers = index.importers(index.file_id(hit.path) or -1, limit=20)
        hit.imported_by = importers[:8]
        if len(importers) >= 2:
            hit.reasons.append(
                Reason(
                    text=f"Imported by {len(importers)} indexed file(s), including {', '.join(importers[:3])}",
                    kind="graph",
                    strength=min(4.0, len(importers) * 0.5),
                    sources=[Source(hit.path)],
                )
            )


def _link_tests(
    index: Index,
    mapping: Mapping,
    weights: dict[str, float],
    identifiers: Sequence[str],
    phrases: Sequence[str],
    options: RetrievalOptions,
) -> None:
    test_rows = list(index.conn.execute("SELECT id, path FROM files WHERE is_test = 1 ORDER BY path"))
    if not test_rows:
        return
    existing = {hit.path for hit in mapping.tests}
    for primary in list(mapping.primaries):
        if primary.role not in {"primary"}:
            continue
        primary_tokens = set(tok.path_tokens(primary.path)) - GENERIC_MODULE_TOKENS
        if not primary_tokens:
            continue
        importers = index.importers(index.file_id(primary.path) or -1, limit=30)
        importer_set = set(importers)
        for row in test_rows:
            path = row["path"]
            test_tokens = set(tok.path_tokens(path)) - {"tests", "test", "spec", "specs"}
            shared = primary_tokens & test_tokens
            imports_primary = path in importer_set
            if not shared and not imports_primary:
                continue
            if len(mapping.tests) >= options.max_tests and path not in existing:
                continue
            hit = next((item for item in mapping.tests if item.path == path), None)
            if hit is None:
                text = index.read(path, MAX_READ_BYTES)
                line = _first_import_line(index, path, primary.path)
                lines = split_lines(text)
                line_snippet = (
                    truncate(lines[line - 1].strip(), 160) if line and 1 <= line <= len(lines) else None
                )
                reason = Reason(
                    text=(
                        f"Imports the target module `{primary.module}`"
                        if imports_primary
                        else f"Test filename mirrors the target module ({', '.join(sorted(shared))})"
                    ),
                    kind="test-link",
                    strength=5.0 if imports_primary else 2.5,
                    sources=[Source(path, line, line_snippet)],
                    inferred=not imports_primary,
                )
                artifact = _Candidate(
                    file_id=row["id"], path=path, row=index.file_row(row["id"]), score=primary.score * 0.25 + 2
                )
                artifact.text = text
                artifact.snippets = _extract_snippets(
                    artifact, weights, [*identifiers, Path(primary.path).stem], phrases, index.symbols_for(row["id"])
                )[:3]
                artifact.reasons = [reason]
                hit = _to_hit(artifact, "test", index)
                mapping.tests.append(hit)
                existing.add(path)
            else:
                hit.reasons.append(
                    Reason(
                        text=f"Covers target module `{primary.module}`",
                        kind="test-link",
                        strength=2.0,
                        sources=[Source(path)],
                        inferred=True,
                    )
                )
            primary.reasons.append(
                Reason(
                    text=f"Existing test file: `{path}`",
                    kind="test-link",
                    strength=2.0,
                    sources=[Source(path)],
                )
            )
        if len(mapping.tests) >= options.max_tests:
            break


def _first_import_line(index: Index, test_path: str, target_path: str) -> int | None:
    test_id = index.file_id(test_path)
    target_id = index.file_id(target_path)
    if test_id is None or target_id is None:
        return None
    for row in index.imports_for(test_id):
        if row["resolved_id"] == target_id:
            return int(row["line"])
    return None


def _score_confidence(mapping: Mapping, weights: dict[str, float], query_terms: Sequence[QueryTerm]) -> None:
    if not mapping.primaries:
        mapping.confidence = 0.0
        mapping.confidence_label = "low"
        mapping.confidence_rationale.append("No source file accumulated enough evidence to be a target.")
        return
    top = mapping.primaries[0]
    important = {tok.stem(term.term) for term in query_terms if term.weight >= 0.9}
    covered: set[str] = set()
    for snippet in top.snippets:
        covered |= {tok.stem(term) for term in snippet.terms}
    covered |= {tok.stem(symbol.name) for symbol in top.symbols}
    coverage = len(important & covered) / max(1, len(important))

    second = mapping.primaries[1].score if len(mapping.primaries) > 1 else 0.0
    separation = clamp((top.score / max(1.0, second) - 1) / 2) if second else 1.0
    normalized = clamp(top.score / 30)
    confidence = 0.45 * normalized + 0.40 * coverage + 0.15 * separation

    explicit = any(
        reason.kind in {"path", "symbol"} and reason.strength >= 9.0
        for reason in top.reasons
    )
    if explicit:
        confidence = max(confidence, 0.68)
    if mapping.graph_expansions and not top.snippets and not explicit:
        confidence *= 0.7
    mapping.confidence = round(clamp(confidence), 3)
    mapping.confidence_label = "high" if mapping.confidence >= 0.7 else "medium" if mapping.confidence >= 0.45 else "low"
    mapping.confidence_rationale.extend(
        [
            f"Top candidate scored {top.score:.1f} (weighted BM25).",
            f"{len(important & covered)}/{len(important)} high-weight task terms appear in the top candidate's evidence.",
            "Clear separation from the runner-up."
            if separation >= 0.6
            else "Runner-up is close; compare candidates before editing.",
        ]
    )
    if explicit:
        mapping.confidence_rationale.append("The task names a path or symbol that exists in this repository.")


def _add_mapping_notes(mapping: Mapping) -> None:
    mapping.notes.append(
        "Ranking is lexical (BM25 over text, symbols and paths), not semantic. "
        "Tasks that paraphrase code vocabulary may rank poorly; adding an error message, symbol or filename helps."
    )
    if not mapping.identifiers:
        mapping.notes.append(
            "No explicit symbol or filename was found in the task text, so no identifier-level boost was applied."
        )
    if mapping.graph_expansions:
        mapping.notes.append(
            f"Added {mapping.graph_expansions} file(s) through one hop of the resolved import graph; those links are inferred, not declared by the task."
        )
    if mapping.confidence_label == "low":
        mapping.notes.append(
            "Low confidence: treat the ordering below as a shortlist to investigate, not as an answer."
        )


__all__ = [
    "RetrievalOptions",
    "build_query",
    "map_task",
]
```

## reposcout/checks.py

`reposcout/checks.py` - 873 lines

````py
"""Discovery and auditing of verification commands.

The contract:

``verified``
    The command is written down in this repository (config, CI workflow,
    Makefile, package script, or contributor docs).  ``sources`` points at the
    file and line.  Verified does **not** mean "ran successfully"; runnability
    is tracked separately.

``suggested``
    The command is inferred from ecosystem conventions or file names.  It has
    an explicit ``basis`` and is never presented as a repository fact.
"""

from __future__ import annotations

import configparser
import importlib.util
import json
import os
import re
import shlex
import subprocess
import sys
import time
from dataclasses import dataclass, field
from pathlib import Path
from typing import Iterable

from . import tokens as tok
from .models import Check, RepoProfile, Source
from .util import find_line, read_text, split_lines, truncate, which

KIND_ORDER = ["test", "lint", "typecheck", "format", "build", "docs", "security"]
COMMAND_PREFIXES = (
    "pytest",
    "py.test",
    "tox",
    "nox",
    "python -m pytest",
    "python -m unittest",
    "python3 -m pytest",
    "python3 -m unittest",
    "uv run pytest",
    "poetry run pytest",
    "hatch run test",
    "make ",
    "npm ",
    "npm run ",
    "yarn ",
    "pnpm ",
    "bun ",
    "cargo ",
    "go test",
    "go vet",
    "mvn ",
    "./mvnw ",
    "gradle ",
    "./gradlew ",
    "ruff ",
    "black ",
    "mypy ",
    "pyright",
    "flake8",
    "isort ",
    "pre-commit ",
    "eslint ",
    "prettier ",
    "tsc ",
    "npx ",
    "bundle exec ",
    "rake ",
    "composer ",
    "php ",
    "dotnet ",
    "swift test",
    "mix test",
    "stack test",
    "cabal test",
    "zig build",
    "mkdocs ",
    "sphinx-build ",
    "doc8 ",
    "vale ",
    "semgrep ",
    "bandit ",
    "gitleaks ",
)


@dataclass
class CheckSet:
    checks: list[Check] = field(default_factory=list)
    notes: list[str] = field(default_factory=list)

    @property
    def verified(self) -> list[Check]:
        return [check for check in self.checks if check.status == "verified"]

    @property
    def suggested(self) -> list[Check]:
        return [check for check in self.checks if check.status == "suggested"]

    def by_kind(self) -> dict[str, list[Check]]:
        grouped: dict[str, list[Check]] = {}
        for check in self.checks:
            grouped.setdefault(check.kind, []).append(check)
        return grouped

    def get(self, check_id: str) -> Check | None:
        return next((check for check in self.checks if check.id == check_id), None)


@dataclass
class CheckRun:
    check_id: str
    command: str
    returncode: int
    output: str
    duration_ms: int
    timed_out: bool = False

    @property
    def ok(self) -> bool:
        return self.returncode == 0 and not self.timed_out


def discover_checks(root: Path, profile: RepoProfile, paths: Iterable[str]) -> CheckSet:
    root = root.resolve()
    path_list = [p.replace("\\", "/") for p in paths]
    result = CheckSet()
    lookup = {path.lower(): path for path in path_list}
    lines_cache: dict[str, list[str]] = {}

    def lines_of(rel_path: str) -> list[str]:
        if rel_path not in lines_cache:
            text = read_text(root / rel_path, limit=800_000) or ""
            lines_cache[rel_path] = split_lines(text)
        return lines_cache[rel_path]

    def add(
        command: str,
        kind: str,
        status: str,
        *,
        basis: str = "",
        sources: list[Source] | None = None,
        scope: list[str] | None = None,
        declared_in: str | None = None,
    ) -> None:
        command = _clean_command(command)
        if not command:
            return
        actual_kind = kind if kind in KIND_ORDER else classify_command(command) or kind
        if status == "verified" and not sources:
            status = "suggested"
            basis = basis or "command was inferred without a repository source"
        if status == "suggested" and not basis:
            basis = "inferred from repository contents; not declared as a command"
        result.checks.append(
            Check(
                id="pending",
                command=command,
                kind=actual_kind,
                status=status,  # type: ignore[arg-type]
                sources=sources or [],
                basis=basis,
                scope=scope or [],
                declared_in=declared_in,
            )
        )

    # --- Python / polyglot manifests -------------------------------------
    for manifest in profile.manifests:
        text = read_text(root / manifest.path, limit=500_000) or ""
        if not text:
            continue
        lines = lines_of(manifest.path)
        if manifest.kind == "pyproject":
            _pyproject_checks(add, lines, manifest.path)
        elif manifest.kind == "setup.cfg":
            _setup_cfg_checks(add, lines, manifest.path)
        elif manifest.kind == "tox.ini":
            _tox_checks(add, lines, manifest.path)
        elif manifest.kind == "noxfile.py":
            _nox_checks(add, lines, manifest.path)
        elif manifest.kind == "makefile":
            _make_checks(add, lines, manifest.path)
        elif manifest.kind == "package.json":
            _package_json_checks(add, root, lines, manifest.path, lookup)
        elif manifest.kind == "cargo.toml":
            for command, kind in (
                ("cargo test", "test"),
                ("cargo clippy --all-targets --all-features", "lint"),
                ("cargo fmt --check", "format"),
                ("cargo build", "build"),
            ):
                add(
                    command,
                    kind,
                    "suggested",
                    basis="Cargo.toml declares a Rust package; this is the cargo default, not a declared task.",
                )
        elif manifest.kind == "go.mod":
            add(
                "go test ./...",
                "test",
                "suggested",
                basis="go.mod declares a Go module; `go test ./...` is the ecosystem default, not declared.",
            )
            add(
                "go vet ./...",
                "lint",
                "suggested",
                basis="go.mod declares a Go module; `go vet` is a standard check, not declared.",
            )
        elif manifest.kind == "pom.xml":
            add("mvn test", "test", "suggested", basis="pom.xml declares a Maven build; no test command is configured here.")
        elif manifest.kind == "gradle":
            add("./gradlew test", "test", "suggested", basis="A Gradle build file is present; no test task is declared.")
        elif manifest.kind == "composer.json":
            _composer_checks(add, text, lines, manifest.path)
        elif manifest.kind == "gemfile":
            add(
                "bundle exec rake test",
                "test",
                "suggested",
                basis="A Gemfile is present; no Rake test task was found to declare this command.",
            )

    # --- CI workflows and contributor docs --------------------------------
    for workflow in profile.ci_workflows:
        _ci_checks(add, lines_of(workflow), workflow)
    for doc in profile.docs + profile.guidelines:
        _documented_checks(add, lines_of(doc), doc)

    # --- Test-file hints --------------------------------------------------
    test_paths = [path for path in path_list if tok.looks_like_test_path(path)]
    hints = _test_hints(root, test_paths, profile)
    for check in hints:
        add(
            check["command"],
            check["kind"],
            check["status"],
            basis=check["basis"],
            sources=check.get("sources", []),
            scope=check.get("scope", []),
            declared_in=check.get("declared_in"),
        )

    # --- Merge duplicates, audit, assign stable identifiers --------------
    result.checks = _merge(result.checks)
    for check in result.checks:
        audit_check(check, root)
    result.checks.sort(key=lambda check: (check.status != "verified", KIND_ORDER.index(check.kind) if check.kind in KIND_ORDER else 99, check.command))
    counts: dict[str, int] = {}
    for check in result.checks:
        counts[check.kind] = counts.get(check.kind, 0) + 1
        check.id = f"{check.kind}-{counts[check.kind]}"

    if not result.checks:
        result.notes.append(
            "No test, lint or build command is declared in this repository, and no test files were found. "
            "Ask the maintainers how they validate changes before proposing a patch."
        )
    elif not any(check.status == "verified" for check in result.checks):
        result.notes.append(
            "Every command below is a suggestion. Nothing in the repository declares how this project is checked."
        )
    if not test_paths:
        result.notes.append("No test files were detected, so the plan cannot point at an existing test to extend.")
    return result


# ---------------------------------------------------------------------------
# Config parsers
# ---------------------------------------------------------------------------


def _pyproject_checks(add, lines: list[str], path: str) -> None:
    def section(name: str) -> int | None:
        pattern = re.compile(r"^\s*\[" + re.escape(name) + r"\]\s*$")
        for index, line in enumerate(lines):
            if pattern.match(line):
                return index + 1
        return None

    specs = [
        ("tool.pytest.ini_options", "python -m pytest", "test", "pytest is configured in pyproject.toml"),
        ("tool.ruff", "ruff check .", "lint", "ruff is configured in pyproject.toml"),
        ("tool.black", "black --check .", "format", "black is configured in pyproject.toml"),
        ("tool.mypy", "mypy .", "typecheck", "mypy is configured in pyproject.toml"),
        ("tool.isort", "isort --check-only .", "format", "isort is configured in pyproject.toml"),
        ("tool.pyright", "pyright", "typecheck", "pyright is configured in pyproject.toml"),
        ("tool.tox", "tox", "test", "tox is configured in pyproject.toml"),
    ]
    for section_name, command, kind, label in specs:
        line = section(section_name)
        if line:
            add(
                command,
                kind,
                "verified",
                sources=[Source(path, line, f"[{section_name}]")],
                declared_in=f"{path}:{line}",
                basis=label,
            )


def _setup_cfg_checks(add, lines: list[str], path: str) -> None:
    parser = configparser.ConfigParser()
    try:
        parser.read_string("\n".join(lines))
    except configparser.Error:
        return
    specs = [
        ("tool:pytest", "python -m pytest", "test"),
        ("flake8", "flake8", "lint"),
        ("mypy", "mypy .", "typecheck"),
        ("isort", "isort --check-only .", "format"),
    ]
    for section, command, kind in specs:
        if parser.has_section(section):
            line = find_line(lines, f"[{section}]")
            add(
                command,
                kind,
                "verified",
                sources=[Source(path, line or 1, f"[{section}]")],
                declared_in=f"{path}:{line or 1}",
            )


def _tox_checks(add, lines: list[str], path: str) -> None:
    env = ""
    for index, raw in enumerate(lines):
        line = raw.strip()
        if line.startswith("[") and line.endswith("]"):
            env = line[1:-1]
            continue
        if not env.startswith("testenv"):
            continue
        if line.startswith("commands"):
            remainder = line.split("=", 1)[1].strip() if "=" in line else ""
            if remainder:
                _add_inline_commands(add, remainder, path, index + 1, "tox.ini commands")
                continue
            cursor = index + 1
            while cursor < len(lines):
                command_line = lines[cursor]
                if command_line.strip().startswith("["):
                    break
                if command_line.startswith((" ", "\t")) and command_line.strip():
                    command = command_line.strip()
                    if not command.startswith(("#", "-")):
                        kind = classify_command(command)
                        if kind:
                            add(
                                command,
                                kind,
                                "verified",
                                sources=[Source(path, cursor + 1, command)],
                                declared_in=f"{path}:{cursor + 1}",
                            )
                cursor += 1


def _nox_checks(add, lines: list[str], path: str) -> None:
    session_re = re.compile(r"^\s*def\s+([A-Za-z_]\w*)\s*\(\s*session")
    for index, line in enumerate(lines):
        match = session_re.match(line)
        if not match:
            continue
        name = match.group(1)
        add(
            f"nox -s {name}",
            "test" if "test" in name else "lint" if "lint" in name else "build",
            "verified",
            sources=[Source(path, index + 1, line.strip())],
            declared_in=f"{path}:{index + 1}",
            basis=f"nox session `{name}` is declared",
        )


def _make_checks(add, lines: list[str], path: str) -> None:
    target_re = re.compile(r"^([A-Za-z0-9][\w.-]*)\s*:(?!=)")
    interesting = re.compile(r"(test|check|lint|format|fmt|type|build|doc|style|cover|audit)")
    for index, line in enumerate(lines):
        match = target_re.match(line)
        if not match:
            continue
        target = match.group(1)
        if target.startswith(".") or not interesting.search(target.lower()):
            continue
        add(
            f"make {target}",
            classify_command(target) or "test",
            "verified",
            sources=[Source(path, index + 1, line.strip())],
            declared_in=f"{path}:{index + 1}",
            basis=f"Makefile target `{target}`",
        )


def _package_json_checks(add, root: Path, lines: list[str], path: str, lookup: dict[str, str]) -> None:
    text = "\n".join(lines)
    try:
        data = json.loads(text)
    except json.JSONDecodeError:
        return
    scripts = data.get("scripts", {})
    if not isinstance(scripts, dict):
        return
    # Lockfiles are scanned but deliberately not indexed, so they may be absent
    # from ``lookup``.  Check the working tree as well.
    runner = "npm"
    for candidate, name in (
        ("pnpm-lock.yaml", "pnpm"),
        ("yarn.lock", "yarn"),
        ("bun.lockb", "bun"),
        ("bun.lock", "bun"),
    ):
        if candidate in lookup or (root / candidate).is_file():
            runner = name
            break
    for name, value in scripts.items():
        if not isinstance(value, str):
            continue
        kind = classify_command(f"{name} {value}", declared_name=name)
        if kind is None:
            continue
        line = find_line(lines, f'"{name}"')
        add(
            f"{runner} run {name}",
            kind,
            "verified",
            sources=[Source(path, line or 1, truncate(f'"{name}": "{value}"', 160))],
            declared_in=f"{path}:{line or 1}",
            basis=f"package.json script `{name}` (runner inferred from lockfile)",
        )


def _composer_checks(add, text: str, lines: list[str], path: str) -> None:
    try:
        data = json.loads(text)
    except json.JSONDecodeError:
        return
    scripts = data.get("scripts", {})
    if not isinstance(scripts, dict):
        return
    for name in scripts:
        kind = classify_command(str(name))
        if not kind:
            continue
        line = find_line(lines, f'"{name}"')
        add(
            f"composer run {name}",
            kind,
            "verified",
            sources=[Source(path, line or 1, f'"{name}"')],
            declared_in=f"{path}:{line or 1}",
            basis=f"Composer script `{name}`",
        )


_CI_RUN_RE = re.compile(r"^(\s*)(?:-\s+)?run:\s*(.+?)\s*$")
_CI_SCRIPT_ITEM_RE = re.compile(r"^\s*-\s+(.+?)\s*$")


def _ci_checks(add, lines: list[str], path: str) -> None:
    is_gitlab = path.lower().endswith((".gitlab-ci.yml", ".travis.yml", "azure-pipelines.yml"))
    for index, raw in enumerate(lines):
        match = _CI_RUN_RE.match(raw)
        if match:
            value = match.group(2).strip().strip("'\"")
            if value in {"|", ">", "|-", ">-", "|+", ">+"}:
                parent_indent = len(match.group(1))
                cursor = index + 1
                while cursor < len(lines):
                    candidate = lines[cursor]
                    if not candidate.strip():
                        cursor += 1
                        continue
                    indent = len(candidate) - len(candidate.lstrip())
                    if indent <= parent_indent:
                        break
                    command = candidate.strip()
                    if not command.startswith(("#", "echo ", "export ", "cd ", "set ", "if ", "fi", "then", "do", "done", "}")):
                        kind = classify_command(command)
                        if kind:
                            add(
                                command,
                                kind,
                                "verified",
                                sources=[Source(path, cursor + 1, truncate(command, 200))],
                                declared_in=f"{path}:{cursor + 1}",
                                basis="GitHub Actions step",
                            )
                    cursor += 1
            else:
                kind = classify_command(value)
                if kind:
                    add(
                        value,
                        kind,
                        "verified",
                        sources=[Source(path, index + 1, truncate(value, 200))],
                        declared_in=f"{path}:{index + 1}",
                        basis="CI workflow step",
                    )
        elif is_gitlab:
            match = _CI_SCRIPT_ITEM_RE.match(raw)
            if match:
                command = match.group(1).strip().strip("'\"")
                kind = classify_command(command)
                if kind:
                    add(
                        command,
                        kind,
                        "verified",
                        sources=[Source(path, index + 1, truncate(command, 200))],
                        declared_in=f"{path}:{index + 1}",
                        basis="CI script step",
                    )


# A fenced block in a document is usually a directory tree, an example, or
# program output.  Only lines that begin with a real command runner are treated
# as documented checks.
_COMMAND_PREFIXES = frozenset(
    {
        "python", "python3", "py", "pip", "pip3", "pipx", "poetry", "uv", "pdm",
        "hatch", "pipenv", "conda", "pytest", "tox", "nox", "coverage", "coverage3",
        "ruff", "mypy", "pyright", "black", "isort", "flake8", "pylint", "bandit",
        "semgrep", "pip-audit", "safety", "npm", "pnpm", "yarn", "bun", "npx",
        "node", "deno", "eslint", "prettier", "tsc", "jest", "vitest", "mocha",
        "cargo", "rustc", "rustup", "go", "gofmt", "golangci-lint", "staticcheck",
        "mvn", "mvnw", "gradle", "gradlew", "dotnet", "msbuild", "ant", "sbt",
        "composer", "phpunit", "bundle", "rake", "rspec", "ruby", "make", "cmake",
        "ctest", "bazel", "bazelisk", "docker", "docker-compose",
        "gitleaks", "trivy", "swift", "mix", "rebar3", "stack", "cabal", "ghc",
        "perl", "prove", "Rscript", "julia", "zig", "nimble", "dune", "opam",
        "lein", "clojure", "elm", "task", "just", "mage",
    }
)

_VERIFICATION_HEADING_RE = re.compile(
    r"\b(test\w*|lint\w*|check\w*|verif\w*|validat\w*|build\w*|develop\w*|"
    r"contribut\w*|quality|format\w*)\b",
    re.IGNORECASE,
)

_CONTRIBUTION_DOC_RE = re.compile(
    r"(contribut|hacking|develop|testing|test-?guide|maintainer)", re.IGNORECASE
)


def _looks_like_command(line: str) -> bool:
    stripped = line.lstrip("$ ").strip()
    if not stripped or stripped.startswith(("#", "//")):
        return False
    if any(character in stripped for character in "├└│┌┐┘┬┴┼"):
        return False
    first = stripped.split()[0]
    if first.endswith(("/", "\\", ":", "=")) or first.startswith(("-", "=")):
        return False
    return first in _COMMAND_PREFIXES


def _documented_checks(add, lines: list[str], path: str) -> None:
    in_fence = False
    heading = ""
    allow_any_fence = bool(_CONTRIBUTION_DOC_RE.search(path))
    for index, raw in enumerate(lines):
        stripped = raw.strip()
        if stripped.startswith("#"):
            if not in_fence:
                heading = stripped.lstrip("#").strip()
            continue
        if stripped.startswith("```"):
            in_fence = not in_fence
            continue
        if not in_fence:
            continue
        if not stripped or stripped.startswith(("#", "//")):
            continue
        if not allow_any_fence and not _VERIFICATION_HEADING_RE.search(heading):
            continue
        if not _looks_like_command(stripped):
            continue
        command = stripped.lstrip("$ ").strip()
        if any(placeholder in command for placeholder in ("<", ">", "...", "your_", "YOUR_")):
            continue
        kind = classify_command(command)
        if not kind:
            continue
        add(
            command,
            kind,
            "verified",
            sources=[Source(path, index + 1, truncate(command, 200))],
            declared_in=f"{path}:{index + 1}",
            basis=f"documented command in {path} (may be stale)",
        )


def _add_inline_commands(add, text: str, path: str, line: int, basis: str) -> None:
    for piece in text.split("&&"):
        command = piece.strip()
        kind = classify_command(command)
        if kind:
            add(
                command,
                kind,
                "verified",
                sources=[Source(path, line, command)],
                declared_in=f"{path}:{line}",
                basis=basis,
            )


# ---------------------------------------------------------------------------
# Suggestions based on files
# ---------------------------------------------------------------------------


def _test_hints(root: Path, test_paths: list[str], profile: RepoProfile) -> list[dict]:
    hints: list[dict] = []
    if not test_paths:
        return hints

    sample_text = ""
    for path in test_paths[:15]:
        sample_text += read_text(root / path, limit=80_000) or ""
    python_tests = [path for path in test_paths if path.endswith((".py", ".pyi"))]
    js_tests = [path for path in test_paths if path.endswith((".js", ".jsx", ".ts", ".tsx", ".mjs", ".cjs"))]

    declares_pytest = any(
        "tool.pytest" in (read_text(root / manifest.path, limit=100_000) or "")
        for manifest in profile.manifests
    )
    if python_tests:
        pytest_basis = "pytest imports appear in test files; no pytest configuration declares the command"
        if "import pytest" in sample_text or "from pytest" in sample_text or "pytest.mark" in sample_text:
            hints.append(
                {
                    "command": "python -m pytest",
                    "kind": "test",
                    "status": "suggested",
                    "basis": pytest_basis,
                    "scope": python_tests[:8],
                }
            )
        elif "unittest" in sample_text:
            test_dir = _common_test_dir(python_tests)
            hints.append(
                {
                    "command": f"python -m unittest discover -s {test_dir} -v",
                    "kind": "test",
                    "status": "suggested",
                    "basis": "test files import unittest; this is the stdlib default, not a declared command",
                    "scope": python_tests[:8],
                }
            )
        elif not declares_pytest:
            hints.append(
                {
                    "command": "python -m pytest",
                    "kind": "test",
                    "status": "suggested",
                    "basis": "Python test files exist; pytest is the common runner but is not configured here",
                    "scope": python_tests[:8],
                }
            )

    if js_tests:
        package_text = ""
        for manifest in profile.manifests:
            if manifest.kind == "package.json":
                package_text = read_text(root / manifest.path, limit=200_000) or ""
                break
        runner = None
        if '"vitest"' in package_text:
            runner = "npx vitest run"
        elif '"jest"' in package_text:
            runner = "npx jest"
        elif '"mocha"' in package_text:
            runner = "npx mocha"
        if runner:
            hints.append(
                {
                    "command": runner,
                    "kind": "test",
                    "status": "suggested",
                    "basis": "the runner is listed as a dependency but no test script declares it",
                    "scope": js_tests[:8],
                }
            )

    if any(path.endswith(".go") for path in test_paths):
        pass  # go test is already suggested from go.mod
    return hints


def _common_test_dir(paths: list[str]) -> str:
    directories = {str(Path(path).parent).replace("\\", "/") for path in paths}
    if len(directories) == 1:
        directory = directories.pop()
        return directory if directory != "." else "."
    return "tests" if any(path.startswith("tests/") for path in paths) else "."


# ---------------------------------------------------------------------------
# Auditing and running
# ---------------------------------------------------------------------------

_COMPOUND_RE = re.compile(r"[|;&><`]|\$\(")


def audit_check(check: Check, root: Path) -> None:
    check.tool_notes = []
    check.runnable = False
    command = check.command.strip()
    if not command:
        check.tool_notes.append("empty command")
        return
    if "${{" in command or _COMPOUND_RE.search(command):
        check.tool_notes.append("compound shell command; run it manually")
        return
    try:
        argv = shlex.split(command)
    except ValueError:
        check.tool_notes.append("command could not be parsed; run it manually")
        return
    if not argv:
        return
    executable = argv[0]
    if executable in {"python", "python3", "py"}:
        module = None
        if "-m" in argv:
            position = argv.index("-m")
            if position + 1 < len(argv):
                module = argv[position + 1]
        if module:
            try:
                available = importlib.util.find_spec(module) is not None
            except (ImportError, ValueError):
                available = False
            if not available:
                check.tool_notes.append(
                    f"Python module `{module}` is not installed in the current interpreter; the command is still declared."
                )
                return
            check.tool_notes.append("will run with this Python interpreter")
            check.runnable = True
            return
        check.tool_notes.append("will run with this Python interpreter")
        check.runnable = True
        return
    if executable.startswith("./") or executable.startswith(".\\"):
        target = root / executable[2:]
        if target.exists():
            check.runnable = True
        else:
            check.tool_notes.append(f"`{executable}` does not exist")
        return
    located = which(executable)
    if located:
        check.runnable = True
    else:
        check.tool_notes.append(f"`{executable}` is not installed on this machine; the command is still declared.")


def run_check(check: Check, root: Path, timeout: int = 180) -> CheckRun:
    if check.status != "verified":
        raise ValueError("Refusing to run an unverified suggestion. Confirm it manually first.")
    if not check.runnable:
        raise ValueError("This command is declared but is not runnable in the current environment.")
    argv = shlex.split(check.command)
    if argv and argv[0] in {"python", "python3", "py"}:
        argv[0] = sys.executable
    started = time.perf_counter()
    try:
        completed = subprocess.run(
            argv,
            cwd=str(root),
            capture_output=True,
            text=True,
            encoding="utf-8",
            errors="replace",
            timeout=timeout,
            check=False,
        )
        duration = int((time.perf_counter() - started) * 1000)
        output = (completed.stdout or "") + (completed.stderr or "")
        return CheckRun(
            check_id=check.id,
            command=check.command,
            returncode=completed.returncode,
            output=truncate(output.strip(), 8000),
            duration_ms=duration,
        )
    except subprocess.TimeoutExpired as exc:
        duration = int((time.perf_counter() - started) * 1000)
        output = (exc.stdout or "") if isinstance(exc.stdout, str) else ""
        return CheckRun(
            check_id=check.id,
            command=check.command,
            returncode=124,
            output=truncate(output + "\n[timed out]", 8000),
            duration_ms=duration,
            timed_out=True,
        )


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------


def classify_command(command: str, declared_name: str = "") -> str | None:
    text = f"{declared_name} {command}".lower()
    if not text.strip():
        return None
    if re.search(r"\b(gitleaks|trivy|bandit|semgrep|npm audit|yarn audit|pnpm audit|safety check|pip-audit)\b", text):
        return "security"
    if re.search(r"\b(test|tests|pytest|unittest|jest|vitest|mocha|tox|nox|spec|rspec|phpunit|ctest)\b", text):
        if re.search(r"\b(install|setup|download)\b", text):
            return None
        return "test"
    if re.search(r"\b(lint|eslint|ruff|flake8|pylint|rubocop|stylelint|clippy|golangci|staticcheck|checkstyle)\b", text):
        return "lint"
    if re.search(r"\b(typecheck|type-check|mypy|pyright|tsc|typescript --noemit)\b", text):
        return "typecheck"
    if re.search(r"\b(format|fmt|prettier|black|isort|rustfmt|gofmt|dprint)\b", text):
        return "format"
    if re.search(r"\b(build|compile|bundle|package|dist|wheel|sdist)\b", text):
        return "build"
    if re.search(r"\b(docs|documentation|mkdocs|sphinx|docusaurus|doc8|vale)\b", text):
        return "docs"
    return None


def _clean_command(command: str) -> str:
    command = command.strip().strip("`")
    command = re.sub(r"^\$\s+", "", command)
    return truncate(command, 240)


def _merge(checks: list[Check]) -> list[Check]:
    merged: dict[str, Check] = {}
    order: list[str] = []
    for check in checks:
        key = re.sub(r"\s+", " ", check.command.strip()).lower()
        if key not in merged:
            merged[key] = check
            order.append(key)
            continue
        existing = merged[key]
        if check.status == "verified" and existing.status == "suggested":
            check.id = existing.id
            merged[key] = check
            existing = check
        see = {(source.path, source.line) for source in existing.sources}
        for source in check.sources:
            if (source.path, source.line) not in see:
                existing.sources.append(source)
                see.add((source.path, source.line))
        existing.scope = sorted(set(existing.scope) | set(check.scope))
        if not existing.basis and check.basis:
            existing.basis = check.basis
    return [merged[key] for key in order]
````

## reposcout/planning.py

`reposcout/planning.py` - 794 lines

```py
"""Turn a mapping plus declared checks into a beginner-sized contribution plan.

Every step contains actions whose status is explicit.  Reads and citations are
usually ``verified`` because the file/line exists; proposed edits and strategy
advice are ``suggested`` because no static tool can prove the right fix.
"""

from __future__ import annotations

import re
from dataclasses import dataclass
from pathlib import Path
from typing import Iterable

from .checks import CheckSet
from .models import (
    Action,
    Check,
    Claim,
    ContributionPlan,
    FileHit,
    Mapping,
    PlanStep,
    RepoProfile,
    Source,
)
from .util import clamp, common_ancestor, read_text, split_lines, truncate
from .tokens import looks_like_test_case_path

DEFAULT_TOTAL_MINUTES = (45, 120)


def build_plan(
    task: str,
    profile: RepoProfile,
    mapping: Mapping,
    checks: CheckSet,
    *,
    test_paths: Iterable[str] = (),
    root: Path | None = None,
) -> ContributionPlan:
    root = root or Path(profile.root)
    plan = ContributionPlan(task=task)
    plan.difficulty_score = _difficulty(mapping, checks, profile)
    plan.difficulty_label = (
        "starter" if plan.difficulty_score <= 30 else "moderate" if plan.difficulty_score <= 60 else "advanced"
    )
    low, high = _estimate_minutes(plan.difficulty_score)
    plan.estimated_minutes = (low, high)
    plan.files_to_touch = [hit.path for hit in mapping.primaries]
    plan.test_files = sorted(
        (hit.path for hit in mapping.tests),
        key=lambda path: (not looks_like_test_case_path(path), path),
    )
    plan.summary = _summary(mapping, checks, plan)

    steps = [
        _step_orient(profile, mapping, root),
        _step_confirm(profile, mapping, checks, test_paths, root),
        _step_read(mapping),
        _step_change(mapping),
        _step_tests(mapping, checks, profile, test_paths, root),
        _step_verify(mapping, checks, profile),
        _step_submit(mapping, profile, root),
    ]
    for index, step in enumerate(steps, start=1):
        step.number = index
    plan.steps = steps
    plan.checklist = _checklist(profile, root)
    plan.unknowns = _unknowns(mapping, checks, profile)
    return plan


# ---------------------------------------------------------------------------
# Steps
# ---------------------------------------------------------------------------


def _step_orient(profile: RepoProfile, mapping: Mapping, root: Path) -> PlanStep:
    actions: list[Action] = []
    rationale: list[Claim] = []

    readme = next((path for path in profile.docs if Path(path).name.lower().startswith("readme")), None)
    if readme:
        actions.append(
            Action(
                kind="read",
                detail=f"Skim `{readme}` for setup, scope and support channels.",
                status="verified",
                sources=[Source(readme)],
            )
        )
    else:
        actions.append(
            Action(
                kind="ask",
                detail="No README is indexed. Ask the maintainers where setup instructions live.",
                status="suggested",
                basis="no README file was found in the indexed tree",
            )
        )

    guide = next((path for path in profile.guidelines if "contribut" in path.lower()), None)
    if guide:
        actions.append(
            Action(
                kind="read",
                detail=f"Read `{guide}` before writing code; it is the project's own contribution contract.",
                status="verified",
                sources=[Source(guide)],
            )
        )
        rationale.append(
            Claim(
                text=f"The project publishes contribution guidance at `{guide}`.",
                status="verified",
                sources=[Source(guide)],
            )
        )
    else:
        rationale.append(
            Claim(
                text="No CONTRIBUTING guide was found, so the review expectations below are generic.",
                status="suggested",
                basis="no contributing guide exists in the indexed tree",
            )
        )

    if mapping.primaries:
        top = mapping.primaries[0]
        anchor = _anchor(top)
        actions.append(
            Action(
                kind="read",
                detail=(
                    f"Open `{top.path}` at {anchor.label()} first"
                    + (f" (symbol `{top.symbols[0].qualified}`)" if top.symbols else "")
                    + "."
                ),
                status="verified",
                sources=[anchor],
            )
        )

    rationale.append(
        Claim(
            text=f"The repository has {profile.files_indexed} indexed files across {len(profile.languages)} language(s), {profile.test_files} of them test files.",
            status="verified",
            sources=[Source(_profile_source(profile))],
        )
    )
    return PlanStep(
        number=0,
        title="Orient yourself",
        goal="Understand the project's rules and where the task lives before touching code.",
        duration_minutes=10,
        actions=actions,
        rationale=rationale,
        risks=["Guides and READMEs can lag behind the code; trust the source you cite in the PR."],
    )


def _step_confirm(
    profile: RepoProfile,
    mapping: Mapping,
    checks: CheckSet,
    test_paths: Iterable[str],
    root: Path,
) -> PlanStep:
    actions: list[Action] = []
    rationale: list[Claim] = []
    test_checks = [check for check in checks.checks if check.kind == "test"]
    verified_tests = [check for check in test_checks if check.status == "verified"]
    suggested_tests = [check for check in test_checks if check.status == "suggested"]

    if verified_tests:
        check = verified_tests[0]
        actions.append(_run_action(check))
        rationale.append(
            Claim(
                text=f"The project declares a test command: `{check.command}` ({check.declared_in}).",
                status="verified",
                sources=check.sources[:1],
            )
        )
    elif suggested_tests:
        check = suggested_tests[0]
        actions.append(
            Action(
                kind="run",
                detail=f"Try `{check.command}` once to see the current baseline (unverified suggestion).",
                status="suggested",
                basis=check.basis,
                command=check.command,
                check_id=check.id,
            )
        )
    else:
        rationale.append(
            Claim(
                text="No test command could be found or inferred, so reproducing the issue is an open question.",
                status="suggested",
                basis="the repository declares no test command and no test files were detected",
            )
        )

    if mapping.tests:
        test = mapping.tests[0]
        anchor = _anchor(test)
        actions.append(
            Action(
                kind="read",
                detail=f"Read `{test.path}` near {anchor.label()} to see the existing expectations and fixtures.",
                status="verified",
                sources=[anchor],
            )
        )
        rationale.append(
            Claim(
                text=f"An existing test file is associated with the target: `{test.path}`.",
                status="verified",
                sources=[Source(test.path)],
            )
        )
    else:
        actions.append(
            Action(
                kind="ask",
                detail="Ask in the issue thread how maintainers would like a regression test; none was detected.",
                status="suggested",
                basis="no test file matched the target module and no test directory was detected",
            )
        )

    return PlanStep(
        number=0,
        title="Reproduce or pin down the behavior",
        goal="Get a concrete before/after signal that the change can be judged against.",
        duration_minutes=15,
        actions=actions,
        rationale=rationale,
        risks=[
            "Running tests may need dependencies that are not installed; use the project's documented setup.",
        ],
    )


def _step_read(mapping: Mapping) -> PlanStep:
    actions: list[Action] = []
    rationale: list[Claim] = []
    for hit in mapping.primaries[:4]:
        anchor = _anchor(hit)
        symbol = hit.symbols[0].qualified if hit.symbols else None
        detail = f"Read `{hit.path}` starting at {anchor.label()}"
        if symbol:
            detail += f"; follow `{symbol}` and its callers."
        else:
            detail += "; follow the matching lines into their enclosing functions."
        actions.append(Action(kind="read", detail=detail, status="verified", sources=[anchor]))
        if hit.role == "doc":
            rationale.append(
                Claim(
                    text=f"The task vocabulary matches the documentation file `{hit.path}` (score {hit.score:.1f}).",
                    status="verified",
                    sources=[anchor],
                )
            )
        elif hit.snippets:
            snippet = hit.snippets[0]
            rationale.append(
                Claim(
                    text=f"`{hit.path}:{snippet.line}` is direct evidence for the task: \"{truncate(snippet.text, 70)}\".",
                    status="verified",
                    sources=[Source(hit.path, snippet.line, snippet.text)],
                )
            )
        else:
            graph_reason = next((reason for reason in hit.reasons if reason.kind == "graph"), None)
            rationale.append(
                Claim(
                    text=(
                        f"`{hit.path}` is relevant through the import graph"
                        + (f" ({graph_reason.text})" if graph_reason else "")
                        + ", not through a direct line match."
                    ),
                    status="suggested",
                    basis="the file was reached by graph expansion, so relevance is inferred rather than observed",
                )
            )

    for hit in mapping.supporting[:2]:
        actions.append(
            Action(
                kind="read",
                detail=f"Read `{hit.path}` for surrounding context before editing; it is connected to the targets.",
                status="verified",
                sources=[Source(hit.path)],
            )
        )

    risks = []
    if any(hit.imported_by for hit in mapping.primaries):
        risks.append("Callers exist in the index; changing a signature can break files outside this list.")
    if any(reason.inferred for hit in mapping.primaries for reason in hit.reasons):
        risks.append("Some links are inferred from imports or file names; confirm them with search before relying on them.")

    return PlanStep(
        number=0,
        title="Read the code in evidence order",
        goal="Build a mental model from the files that actually matched the task.",
        duration_minutes=20,
        actions=actions,
        rationale=rationale,
        risks=risks,
    )


def _step_change(mapping: Mapping) -> PlanStep:
    actions: list[Action] = []
    rationale: list[Claim] = []
    for hit in mapping.primaries[:4]:
        anchor = _anchor(hit)
        symbol = next(
            (item for item in hit.symbols if any(snippet.line >= item.line and (item.end_line is None or snippet.line <= item.end_line) for snippet in hit.snippets)),
            hit.symbols[0] if hit.symbols else None,
        )
        location = f"`{symbol.qualified}`" if symbol else "the matched block"
        actions.append(
            Action(
                kind="edit",
                detail=f"Change {location} in `{hit.path}` at {anchor.label()}, keeping the diff limited to this behavior.",
                status="suggested",
                sources=[anchor],
                basis="proposed edit site derived from ranked evidence; the correct fix still needs human judgement",
            )
        )
        if hit.role == "doc":
            rationale.append(
                Claim(
                    text=f"Documentation target: `{hit.path}`.",
                    status="verified",
                    sources=[anchor],
                )
            )
        else:
            rationale.append(
                Claim(
                    text=f"Primary implementation target: `{hit.path}` ({hit.lines} lines, {hit.lang}).",
                    status="verified",
                    sources=[anchor],
                )
            )

    actions.append(
        Action(
            kind="inspect",
            detail="Search the repository for other call sites before renaming or changing a public signature.",
            status="suggested",
            basis="static evidence is limited to indexed files and one import hop",
        )
    )
    rationale.append(
        Claim(
            text="Keep the patch minimal and behavior-preserving; a focused diff is easier for maintainers to review.",
            status="suggested",
            basis="review heuristic, not a repository rule",
        )
    )

    risks = [
        "Avoid drive-by refactors; they raise review cost and hide the behavioral change.",
    ]
    if any("__init__" in hit.path for hit in mapping.primaries):
        risks.append("This is a package entry point; changing exports can break downstream imports.")
    return PlanStep(
        number=0,
        title="Make the smallest change that satisfies the task",
        goal="Move from reading to one coherent, reviewable edit.",
        duration_minutes=25,
        actions=actions,
        rationale=rationale,
        risks=risks,
    )


def _step_tests(
    mapping: Mapping,
    checks: CheckSet,
    profile: RepoProfile,
    test_paths: Iterable[str],
    root: Path,
) -> PlanStep:
    actions: list[Action] = []
    rationale: list[Claim] = []
    target = mapping.primaries[0] if mapping.primaries else None

    if mapping.tests:
        for test in mapping.tests[:2]:
            actions.append(
                Action(
                    kind="edit",
                    detail=f"Extend `{test.path}` with a focused regression test for the task behavior.",
                    status="suggested",
                    sources=[Source(test.path)],
                    basis="existing test file mapped to the target module; the exact assertion depends on the intended behavior",
                )
            )
        test = mapping.tests[0]
        command = _targeted_test_command(test, checks)
        actions.append(
            Action(
                kind="run",
                detail=f"Run the focused test first: `{command}`.",
                status="suggested",
                basis="targeted invocation derived from the mapped test file; not declared as a repository command",
                command=command,
            )
        )
        rationale.append(
            Claim(
                text=f"A matching test exists at `{test.path}`; extending it is cheaper than creating a new suite.",
                status="verified",
                sources=[Source(test.path)],
            )
        )
    elif target:
        suggestion = _suggest_test_path(target.path, target.lang, profile, list(test_paths))
        actions.append(
            Action(
                kind="edit",
                detail=f"Add a focused regression test at the conventional path `{suggestion}`.",
                status="suggested",
                basis="derived from the target filename and language convention; not an existing file",
            )
        )
        rationale.append(
            Claim(
                text="No existing test maps to this target, so a new regression test is likely required.",
                status="suggested",
                basis="no test file shared a module name or imported the target",
            )
        )

    rationale.append(
        Claim(
            text="A regression test should fail before the fix and pass after it.",
            status="suggested",
            basis="testing heuristic, independent of this repository",
        )
    )
    return PlanStep(
        number=0,
        title="Add or extend a regression test",
        goal="Prove the behavior change and protect it from regressions.",
        duration_minutes=20,
        actions=actions,
        rationale=rationale,
        risks=[
            "If the project has no tests, maintainers may prefer a manual reproduction; confirm before adding a framework.",
        ],
    )


def _step_verify(mapping: Mapping, checks: CheckSet, profile: RepoProfile) -> PlanStep:
    actions: list[Action] = []
    rationale: list[Claim] = []
    verified = [check for check in checks.checks if check.status == "verified"][:8]
    suggested = [check for check in checks.checks if check.status == "suggested"][:4]

    for check in verified:
        actions.append(_run_action(check))
        note = f" ({'; '.join(check.tool_notes)})" if check.tool_notes else ""
        rationale.append(
            Claim(
                text=f"`{check.command}` is declared at {check.declared_in or check.sources[0].label()}.{note}",
                status="verified",
                sources=check.sources[:1],
            )
        )
    for check in suggested:
        actions.append(
            Action(
                kind="run",
                detail=f"Optional, unverified: `{check.command}`. {check.basis}.",
                status="suggested",
                basis=check.basis,
                command=check.command,
                check_id=check.id,
            )
        )
    if not verified:
        rationale.append(
            Claim(
                text="Nothing in the repository declares how to validate the change. Treat every command here as a suggestion.",
                status="suggested",
                basis="no verification command was found in config, CI or contributor docs",
            )
        )

    return PlanStep(
        number=0,
        title="Run the declared checks",
        goal="Separate what the repository requires from what we merely think you should run.",
        duration_minutes=15,
        actions=actions,
        rationale=rationale,
        risks=[
            "A command can be declared but fail for environmental reasons (missing tools, dependencies or services).",
        ],
    )


def _step_submit(mapping: Mapping, profile: RepoProfile, root: Path) -> PlanStep:
    actions: list[Action] = []
    rationale: list[Claim] = []

    template = next(
        (path for path in profile.guidelines if "pull_request" in path.lower() or "contributing" in path.lower()),
        None,
    )
    if template:
        actions.append(
            Action(
                kind="read",
                detail=f"Copy the pull-request expectations from `{template}` into your own checklist.",
                status="verified",
                sources=[Source(template)],
            )
        )
        rationale.append(
            Claim(
                text=f"The repository publishes a pull-request/contribution process at `{template}`.",
                status="verified",
                sources=[Source(template)],
            )
        )
    else:
        rationale.append(
            Claim(
                text="No pull-request template or contributing guide was found; use the generic checklist below.",
                status="suggested",
                basis="no contribution documents were detected in the repository",
            )
        )

    if any(hit.role == "doc" for hit in mapping.primaries):
        rationale.append(
            Claim(
                text="Because the target is documentation, check spelling and links as well as the rendered formatting.",
                status="suggested",
                basis="documentation review heuristic",
            )
        )

    actions.append(
        Action(
            kind="ask",
            detail="Open the pull request against the default branch, reference the issue, and describe the verification you ran.",
            status="suggested",
            basis="standard open-source workflow; the exact branch and template come from the repository guidelines",
        )
    )
    return PlanStep(
        number=0,
        title="Prepare the pull request",
        goal="Make the contribution easy to review and easy to trust.",
        duration_minutes=10,
        actions=actions,
        rationale=rationale,
        risks=[],
    )


# ---------------------------------------------------------------------------
# Difficulty, estimates and checklist
# ---------------------------------------------------------------------------


def _difficulty(mapping: Mapping, checks: CheckSet, profile: RepoProfile) -> int:
    score = 12
    primaries = mapping.primaries
    score += 6 * max(0, len(primaries) - 1)
    line_total = sum(hit.lines for hit in primaries)
    score += min(18, line_total / 120)
    fan_in = sum(len(hit.imported_by) for hit in primaries)
    score += min(15, fan_in * 2)
    if not mapping.tests:
        score += 9
    if not checks.verified:
        score += 10
    if not profile.guidelines:
        score += 4
    if mapping.confidence_label == "low":
        score += 8
    if len({hit.lang for hit in primaries}) > 1:
        score += 5
    if mapping.primaries and all(hit.role == "doc" for hit in mapping.primaries):
        score -= 8
    if checks.verified:
        score -= 5
    return int(clamp(score, 5, 100))


def _estimate_minutes(difficulty: int) -> tuple[int, int]:
    base = 20 + difficulty * 1.6
    low = max(15, int(round(base * 0.6 / 15) * 15))
    high = max(low + 15, int(round(base * 1.4 / 15) * 15))
    return low, high


def _summary(mapping: Mapping, checks: CheckSet, plan: ContributionPlan) -> str:
    if not mapping.primaries:
        return (
            "No confident code target was found. Start by narrowing the task with a filename, symbol or error message; "
            "the evidence below is a shortlist, not an answer."
        )
    top = mapping.primaries[0]
    anchor = _anchor(top)
    parts = [
        f"Start with `{top.path}` ({anchor.label()})",
        f"{len(mapping.primaries)} primary target(s)",
    ]
    if mapping.tests:
        parts.append(f"{len(mapping.tests)} mapped test file(s)")
    verified_checks = len(checks.verified)
    parts.append(f"{verified_checks} declared check(s)" if verified_checks else "no declared checks")
    parts.append(
        f"estimated {plan.estimated_minutes[0]}-{plan.estimated_minutes[1]} min"
    )
    parts.append(f"confidence: {mapping.confidence_label}")
    return "; ".join(parts) + "."


def _checklist(profile: RepoProfile, root: Path) -> list[Claim]:
    claims: list[Claim] = []
    # Only read checklists from contribution material.  README task lists and
    # issue templates are not PR checklists, and treating them as such would
    # turn unrelated prose into "verified" project requirements.
    candidates = [
        path
        for path in profile.guidelines
        if path.lower().endswith((".md", ".rst", ".txt", ".adoc", ""))
        and any(
            marker in path.lower()
            for marker in ("contribut", "pull_request", "pull-request", "pr_template", "review")
        )
    ]
    seen: set[str] = set()
    for path in candidates[:6]:
        text = read_text(root / path, limit=200_000) or ""
        lines = split_lines(text)
        for index, line in enumerate(lines):
            match = re.match(r"^\s*[-*]\s*\[[ xX]\]\s+(.+?)\s*$", line)
            if not match:
                continue
            item = truncate(match.group(1), 160)
            key = item.lower()
            if key in seen:
                continue
            seen.add(key)
            claims.append(
                Claim(
                    text=f"Project checklist: {item}",
                    status="verified",
                    sources=[Source(path, index + 1, item)],
                )
            )
            if len(claims) >= 10:
                break
        if len(claims) >= 10:
            break

    generic = [
        Claim(
            text="The change includes or references a regression test (or explains why one is not practical).",
            status="suggested",
            basis="review heuristic; no checklist item for this was found in the repository",
        ),
        Claim(
            text="The pull request is scoped to one behavior and contains no unrelated refactors.",
            status="suggested",
            basis="review heuristic",
        ),
        Claim(
            text="Commands you ran are listed, including which ones were declared by the project and which were your own.",
            status="suggested",
            basis="transparency heuristic; RepoScout itself separates verified from suggested checks",
        ),
    ]
    for claim in generic:
        if not any(claim.text.lower() == existing.text.lower() for existing in claims):
            claims.append(claim)
    return claims


def _unknowns(mapping: Mapping, checks: CheckSet, profile: RepoProfile) -> list[str]:
    unknowns: list[str] = []
    if mapping.confidence_label == "low":
        unknowns.append("The target files are a low-confidence shortlist; confirm with a maintainer before coding.")
    if not mapping.tests:
        unknowns.append("No test file maps to the target, so the project's expected test style is unknown.")
    if not checks.verified:
        unknowns.append("No verification command is declared, so acceptance criteria are unknown.")
    if not profile.guidelines:
        unknowns.append("No contribution guide or PR template was found.")
    if not profile.is_git:
        unknowns.append("The directory is not a git repository, so branch/PR conventions cannot be inferred.")
    if profile.test_files == 0:
        unknowns.append("No test files were detected at all; validation may be manual.")
    if any(check.status == "verified" and not check.runnable for check in checks.checks):
        unknowns.append("Some declared checks cannot run here because their tools are not installed.")
    return unknowns


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------


def _anchor(hit: FileHit) -> Source:
    if hit.snippets:
        snippet = hit.snippets[0]
        return Source(hit.path, snippet.line, snippet.text)
    if hit.symbols:
        symbol = hit.symbols[0]
        return Source(hit.path, symbol.line, symbol.signature)
    return Source(hit.path, 1)


def _run_action(check: Check) -> Action:
    detail = f"Run `{check.command}`"
    if check.tool_notes:
        detail += f" ({'; '.join(check.tool_notes)})"
    return Action(
        kind="run",
        detail=detail,
        status=check.status,
        sources=check.sources[:1],
        basis=check.basis if check.status == "suggested" else None,
        command=check.command,
        check_id=check.id,
    )


def _targeted_test_command(test: FileHit, checks: CheckSet) -> str:
    commands = " ".join(check.command for check in checks.checks if check.status == "verified").lower()
    if test.lang == "python":
        if "unittest" in commands:
            module = test.path[:-3].replace("/", ".") if test.path.endswith(".py") else test.path
            return f"python -m unittest {module} -v"
        return f"python -m pytest {test.path} -q"
    if test.lang in {"javascript", "typescript", "vue", "svelte"}:
        if "vitest" in commands:
            return f"npx vitest run {test.path}"
        return f"npx jest {test.path}"
    if test.lang == "go":
        return f"go test ./{test.path.rsplit('/', 1)[0] if '/' in test.path else '.'}"
    if test.lang == "rust":
        return f"cargo test --test {Path(test.path).stem}"
    if test.lang == "ruby":
        return f"bundle exec rspec {test.path}"
    if test.lang == "php":
        return f"php vendor/bin/phpunit {test.path}"
    return f"run the focused test in {test.path}"


def _suggest_test_path(primary: str, lang: str, profile: RepoProfile, test_paths: list[str]) -> str:
    stem = Path(primary).stem
    existing_dirs = sorted({str(Path(path).parent).replace("\\", "/") for path in test_paths})
    base = "tests" if not existing_dirs else (common_ancestor(existing_dirs) or existing_dirs[0])
    if base == ".":
        base = ""
    if lang == "python":
        candidate = f"{base + '/' if base else ''}test_{stem}.py"
    elif lang in {"javascript", "typescript", "vue", "svelte"}:
        extension = ".ts" if lang == "typescript" else ".js"
        candidate = f"{base + '/' if base else ''}{stem}.test{extension}"
    elif lang == "go":
        candidate = str(Path(primary).with_name(f"{stem}_test.go")).replace("\\", "/")
    elif lang == "rust":
        candidate = f"tests/{stem}.rs"
    elif lang in {"java", "kotlin", "csharp"}:
        candidate = f"{base + '/' if base else ''}{stem}Test{Path(primary).suffix}"
    elif lang == "ruby":
        candidate = f"spec/{stem}_spec.rb"
    else:
        candidate = f"{base + '/' if base else ''}{stem}_test{Path(primary).suffix or '.txt'}"
    return candidate


def _profile_source(profile: RepoProfile) -> str:
    for manifest in profile.manifests:
        return manifest.path
    return profile.docs[0] if profile.docs else "."
```

## reposcout/agent.py

`reposcout/agent.py` - 366 lines

````py
"""The agent loop: scan, retrieve, inspect, verify, plan.

The loop is deterministic by default; an optional LLM (any OpenAI-compatible
chat endpoint) can propose extra search terms and a short grounded narrative.
Model output is accepted only when it points at retrieved evidence.
"""

from __future__ import annotations

import json
import os
import re
import time
import urllib.error
import urllib.request
from dataclasses import dataclass, field
from pathlib import Path

from . import __version__
from .checks import CheckRun, CheckSet, discover_checks, run_check
from .indexing import Index, ensure_index
from .models import CheckResult, Report, TraceStep
from .planning import build_plan
from .retrieval import RetrievalOptions, map_task
from .scanning import ScanResult, detect_repo_root, discover
from .tokens import looks_like_test_path
from .util import now_iso

MAX_NARRATIVE = 1500


@dataclass
class AgentOptions:
    refresh: bool = False
    use_llm: bool = False
    expand_graph: bool = True
    db_path: Path | None = None
    max_primary: int = 5
    run_checks: bool = False
    timeout: int = 180
    llm_timeout: int = 45


@dataclass
class _Timer:
    started: float = field(default_factory=time.perf_counter)

    def ms(self) -> int:
        return int((time.perf_counter() - self.started) * 1000)


class LLMClient:
    """Minimal OpenAI-compatible client; no third-party dependencies."""

    def __init__(self, timeout: int = 45) -> None:
        self.base_url = (os.environ.get("REPOSCOUT_LLM_BASE_URL") or "https://api.openai.com/v1").rstrip("/")
        self.api_key = os.environ.get("REPOSCOUT_LLM_API_KEY") or os.environ.get("OPENAI_API_KEY") or ""
        self.model = os.environ.get("REPOSCOUT_LLM_MODEL") or "gpt-4o-mini"
        self.timeout = timeout

    @property
    def available(self) -> bool:
        return bool(self.api_key)

    def complete(self, system: str, user: str) -> str:
        payload = {
            "model": self.model,
            "temperature": 0.1,
            "messages": [
                {"role": "system", "content": system},
                {"role": "user", "content": user},
            ],
        }
        request = urllib.request.Request(
            f"{self.base_url}/chat/completions",
            data=json.dumps(payload).encode("utf-8"),
            headers={
                "Content-Type": "application/json",
                "Authorization": f"Bearer {self.api_key}",
            },
            method="POST",
        )
        with urllib.request.urlopen(request, timeout=self.timeout) as response:
            data = json.loads(response.read().decode("utf-8", errors="replace"))
        choices = data.get("choices") or []
        if not choices:
            raise ValueError("no choices in LLM response")
        return str(choices[0].get("message", {}).get("content", ""))


@dataclass
class AgentRun:
    report: Report
    index: Index
    checks: CheckSet
    scan: ScanResult
    check_runs: list[CheckRun] = field(default_factory=list)

    def close(self) -> None:
        self.index.close()


class RepoAgent:
    def __init__(self, root: Path, options: AgentOptions | None = None) -> None:
        self.root = detect_repo_root(root)
        self.options = options or AgentOptions()
        self.trace: list[TraceStep] = []
        self.llm = LLMClient(timeout=self.options.llm_timeout) if self.options.use_llm else None
        self.llm_notes: list[str] = []

    def _log(self, tool: str, tool_input: str, output: str, timer: _Timer) -> None:
        self.trace.append(
            TraceStep(
                index=len(self.trace) + 1,
                tool=tool,
                input=tool_input,
                output=output,
                duration_ms=timer.ms(),
            )
        )

    def run(self, task: str, *, run_checks: bool = False) -> AgentRun:
        task = " ".join(task.split())
        if not task:
            raise ValueError("task description is empty")

        timer = _Timer()
        scan = discover(self.root)
        index, built = ensure_index(
            self.root, refresh=self.options.refresh, db_path=self.options.db_path, scanned=scan
        )
        stats = index.stats()
        self._log(
            "index",
            f"root={self.root}",
            f"{'built' if built else 'loaded'} {stats['files']} files, {stats['symbols']} symbols, "
            f"{stats['imports']} imports ({stats['created_at']})",
            timer,
        )

        timer = _Timer()
        mapping = map_task(
            index,
            task,
            options=RetrievalOptions(
                max_primary=self.options.max_primary, expand_graph=self.options.expand_graph
            ),
        )
        top = ", ".join(f"{hit.path} ({hit.score:.1f})" for hit in mapping.primaries[:3]) or "none"
        self._log(
            "map_task",
            task,
            f"confidence={mapping.confidence_label} ({mapping.confidence:.2f}); top: {top}",
            timer,
        )

        if self.llm and self.llm.available:
            timer = _Timer()
            try:
                terms, narrative = self._llm_pass(task, index, mapping)
                if terms:
                    mapping = map_task(
                        index,
                        task,
                        options=RetrievalOptions(
                            max_primary=self.options.max_primary, expand_graph=self.options.expand_graph
                        ),
                        extra_terms=terms,
                    )
                    mapping.notes.append(
                        "An optional LLM proposed extra search terms; only terms already present in the index were used."
                    )
                if narrative:
                    self.llm_notes.append(narrative)
                self._log(
                    "llm",
                    f"model={self.llm.model}",
                    f"{len(terms)} grounded term(s), narrative={'yes' if narrative else 'no'}",
                    timer,
                )
            except (urllib.error.URLError, TimeoutError, ValueError, KeyError, json.JSONDecodeError, OSError) as exc:
                self.llm_notes.append(f"LLM pass skipped: {type(exc).__name__}: {exc}")
                self._log("llm", f"model={self.llm.model}", f"skipped ({type(exc).__name__})", timer)
        elif self.options.use_llm:
            self.llm_notes.append(
                "LLM requested but no API key was found (set REPOSCOUT_LLM_API_KEY or OPENAI_API_KEY). "
                "The deterministic pipeline produced this report."
            )
            self._log("llm", "api key check", "skipped: no credentials", _Timer())

        timer = _Timer()
        paths = [item.path for item in scan.files]
        check_set = discover_checks(self.root, scan.profile, paths)
        self._log(
            "discover_checks",
            f"{len(paths)} files",
            f"{len(check_set.verified)} verified, {len(check_set.suggested)} suggested, "
            f"{sum(1 for check in check_set.checks if check.runnable)} runnable here",
            timer,
        )

        check_runs: list[CheckRun] = []
        if run_checks:
            timer = _Timer()
            for check in check_set.verified:
                if not check.runnable:
                    continue
                try:
                    check_runs.append(run_check(check, self.root, timeout=self.options.timeout))
                except (ValueError, OSError) as exc:
                    check.tool_notes.append(f"execution failed: {exc}")
            passed = sum(1 for run in check_runs if run.ok)
            self._log("run_checks", "all verified + runnable", f"{passed}/{len(check_runs)} passed", timer)

        timer = _Timer()
        plan = build_plan(
            task,
            scan.profile,
            mapping,
            check_set,
            test_paths=[path for path in paths if looks_like_test_path(path)],
            root=self.root,
        )
        self._log(
            "build_plan",
            f"{len(mapping.primaries)} primaries",
            f"{len(plan.steps)} steps, difficulty={plan.difficulty_label} ({plan.difficulty_score}), "
            f"{plan.estimated_minutes[0]}-{plan.estimated_minutes[1]} min",
            timer,
        )

        report = Report(
            task=task,
            profile=scan.profile,
            mapping=mapping,
            checks=check_set.checks,
            plan=plan,
            trace=self.trace,
            check_results=[
                CheckResult(
                    check_id=run.check_id,
                    command=run.command,
                    returncode=run.returncode,
                    duration_ms=run.duration_ms,
                    timed_out=run.timed_out,
                    output=run.output,
                )
                for run in check_runs
            ],
            llm_used=bool(self.llm and self.llm.available),
            llm_notes=self.llm_notes,
            generated_at=now_iso(),
            version=__version__,
            repo_root=str(self.root),
        )
        self._self_check(report, index.indexed_at)
        return AgentRun(report=report, index=index, checks=check_set, scan=scan, check_runs=check_runs)

    # -- LLM grounding -----------------------------------------------------

    def _llm_pass(self, task: str, index: Index, mapping: object):
        assert self.llm is not None
        evidence = _evidence_pack(mapping)
        system = (
            "You are RepoScout, a repository analysis assistant. You receive retrieved evidence only. "
            "Never invent file paths, symbols or commands. Return strict JSON with keys "
            "'terms' (array of at most 6 additional search words that are likely to appear verbatim in this repository) "
            "and 'narrative' (at most 4 sentences; every claim must cite [path:line] taken from the evidence). "
            "If the evidence is insufficient, say so in the narrative."
        )
        user = f"TASK:\n{task}\n\nEVIDENCE:\n{evidence}"
        raw = self.llm.complete(system, user)
        data = _parse_json_object(raw)
        raw_terms = data.get("terms", [])
        terms = [str(term) for term in raw_terms if isinstance(term, str)][:6] if isinstance(raw_terms, list) else []
        # Grounding check: a term is usable only if it occurs in the index.
        usable: list[str] = []
        from .tokens import stem as _stem

        for term in terms:
            cleaned = term.strip()
            if not cleaned:
                continue
            if index.has_term(_stem(cleaned.lower())) or index.has_term(cleaned.lower()):
                usable.append(cleaned)
        narrative = data.get("narrative")
        narrative_text = str(narrative).strip() if isinstance(narrative, str) else ""
        if narrative_text:
            narrative_text = _validate_narrative(narrative_text, mapping)
        return usable, narrative_text

    def _self_check(self, report: Report, indexed_at: str) -> None:
        limitations = [
            f"The index was built at {indexed_at}; changes made after indexing are invisible. "
            "Re-run `reposcout index --refresh` after editing.",
            "Python symbols come from the AST; other languages use line-based heuristics, so a missed symbol is possible.",
            "Only static evidence is used: dynamic imports, generated code and files outside the index are not analysed.",
            "Ranking is lexical (BM25 and import proximity), not an embedding model; paraphrased tasks can rank poorly.",
        ]
        if not report.mapping.primaries:
            limitations.append("The mapping produced no confident code target.")
        for hit in report.mapping.primaries:
            if not hit.snippets and not any(reason.kind in {"path", "symbol"} for reason in hit.reasons):
                limitations.append(
                    f"`{hit.path}` is ranked, but no line-level evidence was captured for it; treat the suggestion as unverified."
                )
        if not report.checks:
            limitations.append("No verification command exists in the repository, so acceptance criteria are unverified.")
        report.limitations = limitations
        for item in limitations:
            if "index" not in item.lower() and item not in report.plan.unknowns:
                report.plan.unknowns.append(item)


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------


def _evidence_pack(mapping: object) -> str:
    primaries = getattr(mapping, "primaries", [])
    supporting = getattr(mapping, "supporting", [])
    lines: list[str] = []
    for hit in [*primaries, *supporting][:6]:
        lines.append(f"FILE {hit.path} (role={hit.role}, score={hit.score})")
        for reason in hit.reasons[:3]:
            lines.append(f"  why: {reason.text}")
        for snippet in hit.snippets[:3]:
            lines.append(f"  {snippet.line}: {snippet.text}")
        lines.append("")
    pack = "\n".join(lines)
    return pack[:6000]


_CITATION_RE = re.compile(r"\[([\w./\\-]+?):(\d+)\]")


def _validate_narrative(text: str, mapping: object) -> str:
    text = text.replace("```", "").strip()
    if len(text) > MAX_NARRATIVE:
        text = text[:MAX_NARRATIVE].rsplit(" ", 1)[0] + "..."
    allowed = {hit.path for hit in getattr(mapping, "all_hits", lambda: [])()}
    citations = _CITATION_RE.findall(text)
    if not citations:
        return ""
    if any(path.replace("\\", "/") not in allowed for path, _ in citations):
        return ""
    return text


def _parse_json_object(raw: str) -> dict:
    cleaned = raw.strip()
    if cleaned.startswith("```"):
        cleaned = re.sub(r"^```[a-zA-Z]*\s*", "", cleaned)
        cleaned = re.sub(r"```\s*$", "", cleaned)
    start = cleaned.find("{")
    end = cleaned.rfind("}")
    if start == -1 or end == -1:
        raise ValueError("no JSON object in LLM response")
    data = json.loads(cleaned[start : end + 1])
    if not isinstance(data, dict):
        raise ValueError("LLM response is not a JSON object")
    return data


__all__ = ["AgentOptions", "AgentRun", "RepoAgent", "LLMClient"]
````

## reposcout/rendering.py

`reposcout/rendering.py` - 875 lines

````py
"""Human-readable and machine-readable report rendering."""

from __future__ import annotations

import html
import json
import re
from pathlib import Path

from .models import Claim, FileHit, Report
from .tokens import looks_like_test_case_path

BADGE = {"verified": "verified", "suggested": "suggested"}
STATUS_LABEL = {"verified": "VERIFIED", "suggested": "SUGGESTED"}


# ---------------------------------------------------------------------------
# Markdown
# ---------------------------------------------------------------------------


def render_markdown(report: Report) -> str:
    out: list[str] = []
    profile = report.profile
    mapping = report.mapping

    out.append("# RepoScout contribution brief")
    out.append("")
    out.append(f"**Task:** {_md(report.task)}  ")
    repo_line = f"**Repository:** `{profile.name}`"
    if profile.is_git and profile.branch:
        repo_line += f" on `{profile.branch}`"
        if profile.commit:
            repo_line += f" @ `{profile.commit}`"
    repo_line += f" · {profile.files_indexed} files · {profile.total_lines:,} lines"
    out.append(repo_line + "  ")
    out.append(
        f"**Generated:** {report.generated_at} · RepoScout v{report.version} · "
        f"**Confidence:** `{mapping.confidence_label}` ({mapping.confidence:.2f})"
    )
    out.append("")
    out.append(f"> {_md(report.plan.summary)}")
    out.append("")
    out.append(
        "**How to read this report:** `VERIFIED` means a file/line in this repository states it. "
        "`SUGGESTED` means RepoScout inferred it. Declared checks have not been executed unless you asked for that."
    )
    out.append("")
    out.append("## At a glance")
    out.append("")
    if mapping.primaries:
        top = mapping.primaries[0]
        out.append(f"- **Start here:** `{top.path}` (line {_anchor_line(top)}), score {top.score:.1f}")
    else:
        out.append("- **Start here:** no confident target was found; see the shortlist below.")
    if mapping.tests:
        test_hits = [hit for hit in mapping.tests if looks_like_test_case_path(hit.path)] or mapping.tests
        out.append("- **Test to extend:** " + ", ".join(f"`{hit.path}`" for hit in test_hits[:3]))
    else:
        out.append("- **Test to extend:** none mapped")
    verified_checks = [check for check in report.checks if check.status == "verified"]
    suggested_checks = [check for check in report.checks if check.status == "suggested"]
    out.append(
        f"- **Checks:** {len(verified_checks)} declared, {len(suggested_checks)} suggested, "
        f"{sum(1 for check in report.checks if check.runnable)} runnable on this machine"
    )
    out.append(
        f"- **Effort:** {report.plan.estimated_minutes[0]}-{report.plan.estimated_minutes[1]} minutes "
        f"(`{report.plan.difficulty_label}`)"
    )
    out.append(f"- **Files likely to change:** {len(report.plan.files_to_touch)}")
    out.append("")

    out.extend(_md_confidence(mapping))
    out.extend(_md_query(mapping))
    out.extend(_md_code_map(mapping))
    out.extend(_md_checks(report))
    out.extend(_md_plan(report))
    out.extend(_md_checklist(report))
    out.extend(_md_unknowns(report))
    out.extend(_md_trace(report))
    out.extend(_md_method(report))
    return "\n".join(out).rstrip() + "\n"


def _md_confidence(mapping) -> list[str]:
    if not mapping.confidence_rationale:
        return []
    lines = ["## Why this confidence", ""]
    for item in mapping.confidence_rationale:
        lines.append(f"- {_md(item)}")
    for note in mapping.notes:
        lines.append(f"- {_md(note)}")
    lines.append("")
    return lines


def _md_query(mapping) -> list[str]:
    if not mapping.query_terms and not mapping.identifiers:
        return []
    lines = ["## How the task was interpreted", ""]
    if mapping.identifiers:
        lines.append("**Identifiers given extra weight:** " + ", ".join(f"`{name}`" for name in mapping.identifiers))
        lines.append("")
    if mapping.phrases:
        lines.append("**Quoted phrases:** " + ", ".join(f"\"{_md(phrase)}\"" for phrase in mapping.phrases))
        lines.append("")
    if mapping.query_terms:
        terms = mapping.query_terms[:18]
        rendered = ", ".join(f"`{_md(term.term)}`" for term in terms)
        lines.append(f"**Search terms:** {rendered}")
        expansions = [term for term in mapping.query_terms if term.origin == "expansion"][:8]
        if expansions:
            lines.append(
                "**Expanded via repository vocabulary:** "
                + ", ".join(f"`{_md(term.term)}`" for term in expansions)
            )
        lines.append("")
    return lines


def _md_code_map(mapping) -> list[str]:
    lines = ["## Relevant code (with evidence)", ""]
    if not mapping.primaries and not mapping.supporting:
        lines.append("No file matched the task strongly enough to be recommended.")
        lines.append("")
        return lines

    sections = [
        ("Primary targets", mapping.primaries),
        ("Supporting files", mapping.supporting),
        ("Existing tests", mapping.tests),
        ("Documentation and configuration", mapping.docs),
    ]
    for title, hits in sections:
        if not hits:
            continue
        lines.append(f"### {title}")
        lines.append("")
        for index, hit in enumerate(hits, start=1):
            lines.append(f"#### {index}. `{hit.path}` · score {hit.score:.1f} · {hit.role}")
            lines.append("")
            if hit.reasons:
                for reason in hit.reasons[:6]:
                    suffix = " _(inferred)_" if reason.inferred else ""
                    lines.append(f"- {_md(reason.text)}{suffix}")
                lines.append("")
            if hit.snippets:
                lines.append("Evidence:")
                lines.append("")
                lines.append("```text")
                for snippet in hit.snippets[:4]:
                    prefix = f"{hit.path}:{snippet.line}"
                    lines.append(prefix)
                    lines.append(f"  {snippet.text}")
                lines.append("```")
                lines.append("")
            if hit.symbols:
                rendered = ", ".join(
                    f"`{_md(symbol.qualified)}` ({symbol.kind}, L{symbol.line})" for symbol in hit.symbols[:5]
                )
                lines.append(f"Symbols: {rendered}")
                lines.append("")
            if hit.imports or hit.imported_by:
                relation: list[str] = []
                if hit.imports:
                    relation.append("imports " + ", ".join(f"`{path}`" for path in hit.imports[:4]))
                if hit.imported_by:
                    relation.append("imported by " + ", ".join(f"`{path}`" for path in hit.imported_by[:4]))
                lines.append("Graph: " + "; ".join(relation))
                lines.append("")
    return lines


def _md_checks(report: Report) -> list[str]:
    lines = ["## Checks", ""]
    verified = [check for check in report.checks if check.status == "verified"]
    suggested = [check for check in report.checks if check.status == "suggested"]

    lines.append("### Verified: declared inside this repository")
    lines.append("")
    if verified:
        lines.append("| Command | Kind | Declared at | Runnable here | Notes |")
        lines.append("| --- | --- | --- | --- | --- |")
        for check in verified:
            sources = ", ".join(source.label() for source in check.sources[:2])
            notes = "; ".join(check.tool_notes) or ""
            lines.append(
                f"| `{_md(check.command)}` | {check.kind} | {sources} | "
                f"{'yes' if check.runnable else 'no'} | {_md(notes)} |"
            )
    else:
        lines.append("None. The repository does not declare how to test or lint it.")
    lines.append("")
    if report.check_results:
        lines.append("### Execution results (explicitly requested)")
        lines.append("")
        lines.append("| Command | Result | Exit | Time | Output (truncated) |")
        lines.append("| --- | --- | --- | --- | --- |")
        for result in report.check_results:
            status = "PASS" if result.ok else ("TIMEOUT" if result.timed_out else "FAIL")
            command = _md(result.command).replace("|", "\\|")
            output = _md(_short(result.output, 300)).replace("|", "\\|")
            lines.append(
                f"| `{command}` | {status} | "
                f"{result.returncode} | {result.duration_ms} ms | {output} |"
            )
        lines.append("")
    lines.append("### Suggested: inferred, not declared")
    lines.append("")
    if suggested:
        lines.append("| Command | Kind | Why it is only a suggestion |")
        lines.append("| --- | --- | --- |")
        for check in suggested:
            basis = check.basis or "inferred from repository contents"
            lines.append(f"| `{_md(check.command)}` | {check.kind} | {_md(basis)} |")
    else:
        lines.append("None.")
    lines.append("")
    for note in _check_notes(report):
        lines.append(f"- {_md(note)}")
    if _check_notes(report):
        lines.append("")
    if report.check_results:
        lines.append(
            "Execution results above come from an explicit run. Commands outside that table are still "
            "declared only, not verified by execution."
        )
    else:
        lines.append(
            "Verified commands are executed only when you explicitly run `reposcout checks --run --yes`; "
            "this report does not claim they passed."
        )
    lines.append("")
    if report.llm_notes:
        lines.append("### AI narrative")
        lines.append("")
        for note in report.llm_notes:
            lines.append(f"- {_md(note)}")
        lines.append("")
        lines.append("_Citations were validated against the retrieved evidence; the narrative can still be wrong._")
        lines.append("")
    return lines


def _md_plan(report: Report) -> list[str]:
    plan = report.plan
    lines = [
        "## Contribution plan",
        "",
        f"Difficulty: **{plan.difficulty_label}** ({plan.difficulty_score}/100) · "
        f"estimate {plan.estimated_minutes[0]}-{plan.estimated_minutes[1]} minutes · "
        f"{sum(step.duration_minutes for step in plan.steps)} minutes of guided steps",
        "",
    ]
    for step in plan.steps:
        lines.append(f"### Step {step.number}: {_md(step.title)}")
        lines.append("")
        lines.append(f"_{_md(step.goal)}_ · ~{step.duration_minutes} min")
        lines.append("")
        for action in step.actions:
            marker = "VERIFIED" if action.status == "verified" else "SUGGESTED"
            detail = _md(action.detail)
            if action.sources and action.status == "verified":
                detail += " — " + ", ".join(f"`{source.label()}`" for source in action.sources[:2])
            lines.append(f"- `{marker}` **{action.kind}**: {detail}")
            if action.basis and action.status == "suggested":
                lines.append(f"  - _basis: {_md(action.basis)}_")
        if step.rationale:
            lines.append("")
            lines.append("Why:")
            for claim in step.rationale:
                lines.append(f"- {_md_claim(claim)}")
        if step.risks:
            lines.append("")
            lines.append("Risks:")
            for risk in step.risks:
                lines.append(f"- {_md(risk)}")
        lines.append("")
    return lines


def _md_checklist(report: Report) -> list[str]:
    lines = ["## Pull request checklist", ""]
    for claim in report.plan.checklist:
        marker = "VERIFIED" if claim.status == "verified" else "SUGGESTED"
        suffix = ""
        if claim.sources:
            suffix = " — " + ", ".join(f"`{source.label()}`" for source in claim.sources[:2])
        elif claim.basis:
            suffix = f" — _{_md(claim.basis)}_"
        lines.append(f"- [ ] `{marker}` {_md(claim.text)}{suffix}")
    lines.append("")
    return lines


def _md_unknowns(report: Report) -> list[str]:
    items: list[str] = []
    for item in [*report.plan.unknowns, *report.limitations]:
        if item not in items:
            items.append(item)
    if not items:
        return []
    lines = ["## What this report does not verify", ""]
    for item in items:
        lines.append(f"- {_md(item)}")
    lines.append("")
    return lines


def _md_trace(report: Report) -> list[str]:
    if not report.trace:
        return []
    lines = ["<details>", "<summary>Agent trace</summary>", ""]
    lines.append("| # | Tool | Input | Output | ms |")
    lines.append("| --- | --- | --- | --- | --- |")
    for step in report.trace:
        lines.append(
            f"| {step.index} | `{step.tool}` | {_md(_short(step.input, 80))} | "
            f"{_md(_short(step.output, 120))} | {step.duration_ms} |"
        )
    lines.append("")
    lines.append("</details>")
    lines.append("")
    return lines


def _md_method(report: Report) -> list[str]:
    languages = ", ".join(f"{lang} ({lines:,})" for lang, lines in list(report.profile.languages.items())[:5])
    lines = [
        "## Method",
        "",
        "- Walked the working tree, honoured `.gitignore`, skipped vendored and generated directories.",
        f"- Indexed {report.profile.files_indexed} files ({languages or 'languages unknown'}) into SQLite.",
        "- Scored files with weighted BM25 over body text, symbols and paths; expanded one hop through resolved imports.",
        "- Extracted every verification command with its declaration site; unmarked ecosystem defaults stay suggestions.",
        "- Generated the plan from observed symbols, snippets, tests and check declarations.",
        "",
        f"RepoScout v{report.version} · index root `{report.repo_root}` · source available under the MIT licence.",
    ]
    return lines


def _check_notes(report: Report) -> list[str]:
    notes = []
    if not any(check.status == "verified" for check in report.checks):
        notes.append("Nothing in the repository declares a verification command; every check is a suggestion.")
    if not any(check.runnable for check in report.checks):
        notes.append("No check is runnable on this machine (tools or dependencies are missing).")
    return notes


def _md_claim(claim: Claim) -> str:
    marker = "VERIFIED" if claim.status == "verified" else "SUGGESTED"
    text = f"`{marker}` {_md(claim.text)}"
    if claim.sources:
        text += " — " + ", ".join(f"`{source.label()}`" for source in claim.sources[:2])
    elif claim.basis:
        text += f" — _{_md(claim.basis)}_"
    return text


def _md(text: str) -> str:
    return text.replace("\n", " ").strip()


def _short(text: str, width: int) -> str:
    return text if len(text) <= width else text[: width - 1] + "…"


def _anchor_line(hit: FileHit) -> int:
    if hit.snippets:
        return hit.snippets[0].line
    if hit.symbols:
        return hit.symbols[0].line
    return 1


# ---------------------------------------------------------------------------
# JSON
# ---------------------------------------------------------------------------


def render_json(report: Report) -> str:
    return json.dumps(report.to_dict(), indent=2, ensure_ascii=False)


# ---------------------------------------------------------------------------
# HTML
# ---------------------------------------------------------------------------

CSS = """
:root {
  --bg: #f7f7f4; --panel: #ffffff; --ink: #1b1d1c; --muted: #5c6360; --line: #e2e3dd;
  --accent: #145c46; --accent-soft: #e6f2ec; --warn: #8a5200; --warn-soft: #fdf1dc;
  --code: #f0f1ea; --shadow: 0 1px 2px rgba(20,30,25,.06), 0 8px 24px rgba(20,30,25,.05);
}
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #141715; --panel: #1b1f1c; --ink: #e8ece7; --muted: #a4ada7; --line: #2c332e;
    --accent: #7fd0ad; --accent-soft: #1d332a; --warn: #f0b45c; --warn-soft: #332816;
    --code: #222823; --shadow: 0 1px 2px rgba(0,0,0,.3), 0 8px 24px rgba(0,0,0,.25);
  }
}
* { box-sizing: border-box; }
html { scroll-behavior: smooth; }
body {
  margin: 0; background: var(--bg); color: var(--ink);
  font: 16px/1.6 ui-sans-serif, -apple-system, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}
.wrap { max-width: 1000px; margin: 0 auto; padding: 32px 20px 80px; }
header.hero {
  background: linear-gradient(140deg, var(--accent-soft), transparent 70%);
  border: 1px solid var(--line); border-radius: 18px; padding: 30px 30px 24px; margin-bottom: 26px;
  box-shadow: var(--shadow);
}
.eyebrow { text-transform: uppercase; letter-spacing: .14em; font-size: 11px; font-weight: 700; color: var(--accent); }
h1 { font-size: clamp(26px, 4vw, 38px); line-height: 1.15; margin: 8px 0 12px; letter-spacing: -.02em; }
h2 { font-size: 22px; margin: 42px 0 14px; letter-spacing: -.01em; }
h3 { font-size: 17px; margin: 26px 0 10px; }
h4 { font-size: 15px; margin: 18px 0 8px; }
p { margin: 10px 0; }
a { color: var(--accent); text-decoration: none; }
a:hover { text-decoration: underline; }
code, pre { font-family: ui-monospace, SFMono-Regular, Consolas, "Liberation Mono", monospace; }
code { background: var(--code); padding: 1px 5px; border-radius: 5px; font-size: .88em; overflow-wrap: anywhere; }
pre { background: var(--code); padding: 14px 16px; border-radius: 10px; overflow-x: auto; font-size: 13px; line-height: 1.5; }
pre code { background: none; padding: 0; }
.task { font-size: 18px; margin: 0 0 14px; max-width: 70ch; }
.meta { color: var(--muted); font-size: 13.5px; display: flex; flex-wrap: wrap; gap: 6px 16px; }
.meta strong { color: var(--ink); font-weight: 600; }
.summary { margin-top: 16px; padding-top: 16px; border-top: 1px solid var(--line); font-size: 15.5px; }
.legend { display: flex; flex-wrap: wrap; gap: 10px 22px; margin: 16px 0 0; font-size: 13px; color: var(--muted); }
.legend span { display: inline-flex; align-items: center; gap: 7px; }
nav.toc { display: flex; flex-wrap: wrap; gap: 8px; margin: 0 0 8px; }
nav.toc a {
  border: 1px solid var(--line); background: var(--panel); border-radius: 999px;
  padding: 6px 12px; font-size: 13px; color: var(--muted);
}
nav.toc a:hover { color: var(--accent); border-color: var(--accent); text-decoration: none; }
.badge {
  display: inline-block; border-radius: 999px; padding: 2px 9px; font-size: 10.5px;
  font-weight: 700; letter-spacing: .06em; text-transform: uppercase; vertical-align: 1px;
}
.badge.verified { background: var(--accent-soft); color: var(--accent); border: 1px solid color-mix(in srgb, var(--accent) 30%, transparent); }
.badge.suggested { background: var(--warn-soft); color: var(--warn); border: 1px solid color-mix(in srgb, var(--warn) 30%, transparent); }
.badge.role { background: var(--code); color: var(--muted); border: 1px solid var(--line); }
.panel { background: var(--panel); border: 1px solid var(--line); border-radius: 14px; padding: 20px 22px; box-shadow: var(--shadow); }
.grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 12px; margin: 18px 0; }
.stat { background: var(--panel); border: 1px solid var(--line); border-radius: 12px; padding: 13px 15px; }
.stat .k { font-size: 11px; text-transform: uppercase; letter-spacing: .1em; color: var(--muted); font-weight: 700; }
.stat .v { font-size: 17px; font-weight: 650; margin-top: 4px; }
.meter { height: 7px; background: var(--code); border-radius: 99px; overflow: hidden; margin-top: 8px; }
.meter > i { display: block; height: 100%; background: var(--accent); border-radius: 99px; }
.file { background: var(--panel); border: 1px solid var(--line); border-radius: 14px; margin: 14px 0; overflow: hidden; }
.file > summary { cursor: pointer; padding: 14px 18px; display: flex; flex-wrap: wrap; gap: 8px 12px; align-items: center; list-style: none; }
.file > summary::-webkit-details-marker { display: none; }
.file > summary .path { font-weight: 650; font-family: ui-monospace, Consolas, monospace; font-size: 14px; }
.file > summary .score { color: var(--muted); font-size: 12.5px; margin-left: auto; }
.file .body { padding: 0 18px 18px; border-top: 1px solid var(--line); }
ul.why { margin: 12px 0; padding-left: 20px; }
ul.why li { margin: 5px 0; }
ul.why .inferred { color: var(--warn); font-size: 11.5px; font-weight: 700; letter-spacing: .04em; text-transform: uppercase; }
.evidence { margin: 12px 0; border-left: 3px solid var(--accent); background: var(--code); padding: 10px 14px; border-radius: 0 8px 8px 0; }
.evidence .loc { color: var(--muted); font-size: 11.5px; font-family: ui-monospace, Consolas, monospace; }
.evidence .line { font-family: ui-monospace, Consolas, monospace; font-size: 12.5px; margin-top: 3px; word-break: break-word; }
.symbols { color: var(--muted); font-size: 13.5px; margin: 10px 0; }
table { width: 100%; border-collapse: collapse; margin: 12px 0; font-size: 13.5px; }
th, td { text-align: left; padding: 9px 12px; border-bottom: 1px solid var(--line); vertical-align: top; }
th { color: var(--muted); font-size: 11px; letter-spacing: .06em; text-transform: uppercase; }
tr:last-child td { border-bottom: none; }
.tablewrap { overflow-x: auto; border: 1px solid var(--line); border-radius: 12px; background: var(--panel); }
.tablewrap table { margin: 0; }
.step { background: var(--panel); border: 1px solid var(--line); border-radius: 14px; padding: 18px 20px; margin: 12px 0; }
.step h3 { margin-top: 0; display: flex; flex-wrap: wrap; gap: 10px; align-items: baseline; }
.step h3 .num {
  display: inline-flex; align-items: center; justify-content: center; width: 26px; height: 26px;
  border-radius: 50%; background: var(--accent-soft); color: var(--accent); font-size: 13px; font-weight: 800;
}
.step h3 .mins { margin-left: auto; color: var(--muted); font-size: 12px; font-weight: 500; }
.goal { color: var(--muted); margin: -4px 0 12px; }
.action { padding: 7px 0; border-bottom: 1px dashed var(--line); }
.action:last-child { border-bottom: 0; }
.action .kind { font-weight: 700; font-size: 12px; text-transform: uppercase; letter-spacing: .05em; margin-right: 5px; }
.basis { color: var(--muted); font-size: 12.5px; margin: 3px 0 0 0; }
.risk { color: var(--warn); background: var(--warn-soft); border-radius: 9px; padding: 9px 12px; font-size: 13px; margin-top: 8px; }
.checklist { list-style: none; padding: 0; }
.checklist li { padding: 8px 0 8px 28px; position: relative; border-bottom: 1px solid var(--line); }
.checklist li::before { content: ""; position: absolute; left: 0; top: 12px; width: 15px; height: 15px; border: 1.5px solid var(--muted); border-radius: 4px; }
.note { color: var(--muted); font-size: 13px; }
details.trace summary, details.more summary { cursor: pointer; color: var(--accent); font-weight: 600; font-size: 14px; padding: 6px 0; }
.ok { color: var(--accent); font-weight: 700; }
.no { color: var(--warn); font-weight: 700; }
footer { margin-top: 50px; padding-top: 18px; border-top: 1px solid var(--line); color: var(--muted); font-size: 12.5px; }
@media print {
  body { background: #fff; } .wrap { padding: 0; max-width: none; }
  .file, .panel, .step { box-shadow: none; break-inside: avoid; }
  nav.toc, .legend { display: none; }
}
"""


def render_html(report: Report) -> str:
    profile = report.profile
    mapping = report.mapping
    plan = report.plan
    e = html.escape

    def badge(status: str) -> str:
        return f'<span class="badge {status}">{STATUS_LABEL[status]}</span>'

    def inline(text: str) -> str:
        escaped = e(text)
        escaped = re.sub(r"`([^`]+)`", lambda match: f"<code>{match.group(1)}</code>", escaped)
        return escaped

    verified = [check for check in report.checks if check.status == "verified"]
    suggested = [check for check in report.checks if check.status == "suggested"]

    parts: list[str] = []
    parts.append("<!doctype html>")
    parts.append('<html lang="en"><head><meta charset="utf-8">')
    parts.append('<meta name="viewport" content="width=device-width, initial-scale=1">')
    parts.append(f"<title>RepoScout - {e(profile.name)}</title>")
    parts.append(f"<style>{CSS}</style></head><body>")
    parts.append('<div class="wrap">')

    # Hero
    parts.append('<header class="hero">')
    parts.append('<div class="eyebrow">RepoScout contribution brief</div>')
    parts.append(f"<h1>{inline(report.task)}</h1>")
    parts.append('<div class="meta">')
    repo_meta = f"<strong>{e(profile.name)}</strong>"
    if profile.is_git and profile.branch:
        repo_meta += f" on <code>{e(profile.branch)}</code>"
        if profile.commit:
            repo_meta += f" @ <code>{e(profile.commit)}</code>"
    parts.append(f"<span>{repo_meta}</span>")
    parts.append(f"<span>{profile.files_indexed:,} files</span>")
    parts.append(f"<span>{profile.total_lines:,} lines</span>")
    parts.append(f"<span>{len(profile.languages)} languages</span>")
    parts.append(f"<span>Generated {e(report.generated_at)}</span>")
    parts.append(f"<span>v{e(report.version)}</span>")
    parts.append("</div>")
    parts.append(f'<div class="summary">{inline(plan.summary)}</div>')
    parts.append(
        '<div class="legend">'
        f'<span>{badge("verified")} backed by a file and line in this repository</span>'
        f'<span>{badge("suggested")} inferred by RepoScout; confirm before acting</span>'
        "</div>"
    )
    parts.append("</header>")

    parts.append(
        '<nav class="toc">'
        '<a href="#overview">Overview</a><a href="#code">Code map</a><a href="#checks">Checks</a>'
        '<a href="#plan">Plan</a><a href="#checklist">PR checklist</a><a href="#limits">Limits</a>'
        "</nav>"
    )

    # Overview stats
    parts.append('<section id="overview"><h2>Overview</h2>')
    parts.append('<div class="grid">')
    confidence_pct = int(round(mapping.confidence * 100))
    parts.append(
        f'<div class="stat"><div class="k">Mapping confidence</div>'
        f'<div class="v">{e(mapping.confidence_label)} · {confidence_pct}%</div>'
        f'<div class="meter"><i style="width:{confidence_pct}%"></i></div></div>'
    )
    parts.append(
        f'<div class="stat"><div class="k">Primary targets</div><div class="v">{len(mapping.primaries)} files</div>'
        f'<div class="note">{len(plan.files_to_touch)} likely to change</div></div>'
    )
    parts.append(
        f'<div class="stat"><div class="k">Mapped tests</div><div class="v">{len(mapping.tests)}</div>'
        f'<div class="note">{"ready to extend" if mapping.tests else "no test found"}</div></div>'
    )
    parts.append(
        f'<div class="stat"><div class="k">Declared checks</div><div class="v">{len(verified)}</div>'
        f'<div class="note">{sum(1 for check in verified if check.runnable)} runnable here</div></div>'
    )
    parts.append(
        f'<div class="stat"><div class="k">Estimated effort</div>'
        f'<div class="v">{plan.estimated_minutes[0]}-{plan.estimated_minutes[1]} min</div>'
        f'<div class="note">{e(plan.difficulty_label)} · score {plan.difficulty_score}/100</div></div>'
    )
    parts.append("</div>")

    if mapping.confidence_rationale or mapping.notes:
        parts.append('<div class="panel">')
        parts.append("<h3>Why this confidence</h3><ul class=\"why\">")
        for item in mapping.confidence_rationale:
            parts.append(f"<li>{inline(item)}</li>")
        for note in mapping.notes:
            parts.append(f'<li class="note">{inline(note)}</li>')
        parts.append("</ul></div>")

    if mapping.identifiers or mapping.query_terms:
        parts.append("<details class=\"more\"><summary>How the task was interpreted</summary><div class=\"panel\">")
        if mapping.identifiers:
            parts.append(
                "<p><strong>Identifiers:</strong> "
                + " ".join(f"<code>{e(name)}</code>" for name in mapping.identifiers)
                + "</p>"
            )
        if mapping.phrases:
            parts.append(
                "<p><strong>Quoted phrases:</strong> "
                + " ".join(f"<code>{e(phrase)}</code>" for phrase in mapping.phrases)
                + "</p>"
            )
        parts.append(
            "<p><strong>Index terms:</strong> "
            + " ".join(
                f'<code title="{e(term.origin)} · weight {term.weight}">{e(term.term)}</code>'
                for term in mapping.query_terms[:24]
            )
            + "</p>"
        )
        parts.append("</div></details>")
    parts.append("</section>")

    # Code map
    parts.append('<section id="code"><h2>Relevant code</h2>')
    if not mapping.all_hits():
        parts.append('<div class="panel">No file matched strongly enough. Narrow the task with a filename, symbol or error text.</div>')
    for role_title, hits in (
        ("Primary targets", mapping.primaries),
        ("Supporting files", mapping.supporting),
        ("Existing tests", mapping.tests),
        ("Docs and configuration", mapping.docs),
    ):
        if not hits:
            continue
        parts.append(f"<h3>{role_title}</h3>")
        for number, hit in enumerate(hits, start=1):
            open_attr = " open" if number == 1 and role_title == "Primary targets" else ""
            parts.append(f'<details class="file"{open_attr}>')
            parts.append("<summary>")
            parts.append(f'<span class="path">{e(hit.path)}</span>')
            parts.append(f'<span class="badge role">{e(hit.role)}</span>')
            parts.append(f'<span class="badge role">{e(hit.lang or "text")}</span>')
            if hit.module and hit.module != Path(hit.path).stem:
                parts.append(f'<span class="badge role">{e(hit.module)}</span>')
            parts.append(f'<span class="score">score {hit.score:.1f} · {hit.lines:,} lines</span>')
            parts.append("</summary>")
            parts.append('<div class="body">')
            if hit.reasons:
                parts.append('<ul class="why">')
                for reason in hit.reasons[:7]:
                    inferred = ' <span class="inferred">inferred</span>' if reason.inferred else ""
                    parts.append(f"<li>{inline(reason.text)}{inferred}</li>")
                parts.append("</ul>")
            for snippet in hit.snippets[:4]:
                parts.append('<div class="evidence">')
                parts.append(f'<div class="loc">{e(hit.path)}:{snippet.line}'
                             + (f" · {e(snippet.symbol)}" if snippet.symbol else "")
                             + "</div>")
                parts.append(f'<div class="line">{e(snippet.text)}</div>')
                parts.append("</div>")
            if hit.symbols:
                parts.append('<div class="symbols"><strong>Symbols:</strong> ')
                parts.append(
                    ", ".join(
                        f"<code>{e(symbol.qualified)}</code> <span class=\"note\">{e(symbol.kind)} · L{symbol.line}</span>"
                        for symbol in hit.symbols[:6]
                    )
                )
                parts.append("</div>")
            relations: list[str] = []
            if hit.imports:
                relations.append("imports " + ", ".join(f"<code>{e(path)}</code>" for path in hit.imports[:4]))
            if hit.imported_by:
                relations.append("imported by " + ", ".join(f"<code>{e(path)}</code>" for path in hit.imported_by[:4]))
            if relations:
                parts.append('<div class="symbols">' + "; ".join(relations) + "</div>")
            parts.append("</div></details>")
    parts.append("</section>")

    # Checks
    parts.append('<section id="checks"><h2>Checks</h2>')
    if report.check_results:
        parts.append(
            '<p class="note">Verified means the command is written down in this repository. '
            "The execution table below lists commands you explicitly asked RepoScout to run; "
            "everything else is declared but unexecuted.</p>"
        )
    else:
        parts.append(
            '<p class="note">Verified means the command is written down in this repository, not that it passed. '
            "RepoScout executes nothing unless you run <code>reposcout checks --run --yes</code>.</p>"
        )
    parts.append(f"<h3>{badge('verified')} Declared in this repository</h3>")
    if verified:
        parts.append('<div class="tablewrap"><table><thead><tr>'
                     "<th>Command</th><th>Kind</th><th>Declared at</th><th>Runnable here</th><th>Notes</th>"
                     "</tr></thead><tbody>")
        for check in verified:
            sources = "<br>".join(
                f'<code>{e(source.path)}:{source.line}</code>' for source in check.sources[:3]
            )
            runnable = f'<span class="ok">yes</span>' if check.runnable else '<span class="no">no</span>'
            notes = "; ".join(check.tool_notes)
            parts.append(
                f"<tr><td><code>{e(check.command)}</code></td><td>{e(check.kind)}</td><td>{sources}</td>"
                f"<td>{runnable}</td><td class=\"note\">{e(notes)}</td></tr>"
            )
        parts.append("</tbody></table></div>")
    else:
        parts.append('<div class="panel">None. This repository declares no verification command.</div>')
    if report.check_results:
        parts.append("<h3>Execution results (explicitly requested)</h3>")
        parts.append(
            '<div class="tablewrap"><table><thead><tr>'
            "<th>Result</th><th>Command</th><th>Exit</th><th>Time</th><th>Output (truncated)</th>"
            "</tr></thead><tbody>"
        )
        for result in report.check_results:
            if result.ok:
                status = '<span class="ok">PASS</span>'
            elif result.timed_out:
                status = '<span class="no">TIMEOUT</span>'
            else:
                status = '<span class="no">FAIL</span>'
            parts.append(
                f"<tr><td>{status}</td><td><code>{e(result.command)}</code></td>"
                f"<td>{result.returncode}</td><td>{result.duration_ms} ms</td>"
                f"<td class=\"note\">{e(_short(result.output, 240))}</td></tr>"
            )
        parts.append("</tbody></table></div>")
    parts.append(f"<h3>{badge('suggested')} Inferred, not declared</h3>")
    if suggested:
        parts.append('<div class="tablewrap"><table><thead><tr>'
                     "<th>Command</th><th>Kind</th><th>Why it stays a suggestion</th>"
                     "</tr></thead><tbody>")
        for check in suggested:
            parts.append(
                f"<tr><td><code>{e(check.command)}</code></td><td>{e(check.kind)}</td>"
                f"<td>{inline(check.basis)}</td></tr>"
            )
        parts.append("</tbody></table></div>")
    else:
        parts.append('<div class="panel">No inferred commands.</div>')
    for note in _check_notes(report):
        parts.append(f'<p class="note">{inline(note)}</p>')
    if report.llm_notes:
        parts.append('<div class="panel"><h3>AI narrative</h3>')
        for note in report.llm_notes:
            parts.append(f"<p>{inline(note)}</p>")
        parts.append(
            '<p class="note">Citations were validated against the retrieved evidence. '
            "The narrative is optional and can still be wrong.</p></div>"
        )
    parts.append("</section>")

    # Plan
    parts.append('<section id="plan"><h2>Contribution plan</h2>')
    parts.append(
        f'<p class="note">Difficulty <strong>{e(plan.difficulty_label)}</strong> '
        f"({plan.difficulty_score}/100) · estimate {plan.estimated_minutes[0]}-{plan.estimated_minutes[1]} minutes.</p>"
    )
    for step in plan.steps:
        parts.append('<div class="step">')
        parts.append(
            f'<h3><span class="num">{step.number}</span> {inline(step.title)}'
            f'<span class="mins">~{step.duration_minutes} min</span></h3>'
        )
        parts.append(f'<p class="goal">{inline(step.goal)}</p>')
        for action in step.actions:
            parts.append('<div class="action">')
            parts.append(f"{badge(action.status)} <span class=\"kind\">{e(action.kind)}</span>")
            parts.append(inline(action.detail))
            if action.sources and action.status == "verified":
                parts.append(
                    ' <span class="note">→ '
                    + ", ".join(f'<code>{e(source.label())}</code>' for source in action.sources[:2])
                    + "</span>"
                )
            if action.basis and action.status == "suggested":
                parts.append(f'<div class="basis">basis: {inline(action.basis)}</div>')
            parts.append("</div>")
        if step.rationale:
            parts.append('<div class="basis"><strong>Why:</strong><ul class="why">')
            for claim in step.rationale:
                parts.append(f"<li>{_html_claim(claim, badge)}</li>")
            parts.append("</ul></div>")
        for risk in step.risks:
            parts.append(f'<div class="risk">{inline(risk)}</div>')
        parts.append("</div>")
    parts.append("</section>")

    # Checklist
    parts.append('<section id="checklist"><h2>Pull request checklist</h2><ul class="checklist">')
    for claim in plan.checklist:
        source_note = ""
        if claim.sources:
            source_note = " <span class=\"note\">— " + ", ".join(
                f"<code>{e(source.label())}</code>" for source in claim.sources[:2]
            ) + "</span>"
        elif claim.basis:
            source_note = f' <span class="note">— {inline(claim.basis)}</span>'
        parts.append(f"<li>{badge(claim.status)} {inline(claim.text)}{source_note}</li>")
    parts.append("</ul></section>")

    # Limits
    parts.append('<section id="limits"><h2>What this report does not verify</h2><div class="panel">')
    seen: set[str] = set()
    for item in [*plan.unknowns, *report.limitations]:
        if item in seen:
            continue
        seen.add(item)
        parts.append(f"<p>• {inline(item)}</p>")
    if not seen:
        parts.append("<p>No additional caveats were recorded.</p>")
    parts.append("</div></section>")

    # Trace
    if report.trace:
        parts.append('<details class="trace"><summary>Agent trace</summary><div class="tablewrap"><table>')
        parts.append("<thead><tr><th>#</th><th>Tool</th><th>Input</th><th>Output</th><th>ms</th></tr></thead><tbody>")
        for step in report.trace:
            parts.append(
                f"<tr><td>{step.index}</td><td><code>{e(step.tool)}</code></td>"
                f"<td>{inline(_short(step.input, 90))}</td>"
                f"<td>{inline(_short(step.output, 160))}</td><td>{step.duration_ms}</td></tr>"
            )
        parts.append("</tbody></table></div></details>")

    parts.append("<footer>")
    parts.append(
        f"RepoScout v{e(report.version)} · generated {e(report.generated_at)} · "
        f"index root <code>{e(report.repo_root)}</code><br>"
        "Lexical retrieval + static analysis. Verified means declared in the repository; "
        "suggested means inferred. Nothing was executed without an explicit request."
    )
    parts.append("</footer>")
    parts.append("</div></body></html>")
    return "\n".join(parts)


def _html_claim(claim: Claim, badge) -> str:
    escaped = html.escape(claim.text)
    escaped = re.sub(r"`([^`]+)`", lambda match: f"<code>{match.group(1)}</code>", escaped)
    suffix = ""
    if claim.sources:
        suffix = " <span class=\"note\">→ " + ", ".join(
            f"<code>{html.escape(source.label())}</code>" for source in claim.sources[:2]
        ) + "</span>"
    elif claim.basis:
        suffix = f' <span class="note">— {html.escape(claim.basis)}</span>'
    return f'{badge(claim.status)} {escaped}{suffix}'


# ---------------------------------------------------------------------------
# Files
# ---------------------------------------------------------------------------


def write_report(report: Report, path: Path, fmt: str) -> Path:
    fmt = fmt.lower().lstrip(".")
    renderers = {"md": render_markdown, "markdown": render_markdown, "html": render_html, "json": render_json}
    if fmt not in renderers:
        raise ValueError(f"unsupported format: {fmt}")
    path = path.resolve()
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(renderers[fmt](report), encoding="utf-8")
    return path


def suggest_output_path(root: Path, task: str, default: Path | None = None) -> Path:
    from .util import slug

    if default is not None:
        return default
    return root / ".reposcout" / "reports" / f"{slug(task)}.html"
````

## reposcout/cli.py

`reposcout/cli.py` - 573 lines

```py
"""Command-line interface for RepoScout.

Every command is read-only with respect to the target repository except for
``index`` (which writes ``.reposcout/index.sqlite3``) and ``plan`` (which writes
a report when an output path is requested).  Declared checks run only with
``checks --run --yes`` or ``plan --run-checks --yes``.
"""

from __future__ import annotations

import argparse
import json
import sys
from pathlib import Path

from . import __version__
from .agent import AgentOptions, RepoAgent
from .checks import CheckRun, CheckSet, discover_checks, run_check
from .indexing import Index, ensure_index, index_overview
from .models import Mapping, RepoProfile
from .rendering import (
    render_json,
    render_markdown,
    suggest_output_path,
    write_report,
)
from .retrieval import RetrievalOptions, map_task
from .scanning import ScanResult, detect_repo_root, discover
from .util import human_bytes, now_iso, truncate

FORMATS = ("md", "html", "json")


# ---------------------------------------------------------------------------
# Output helpers
# ---------------------------------------------------------------------------


def _configure_stdio() -> None:
    for stream in (sys.stdout, sys.stderr):
        try:
            stream.reconfigure(encoding="utf-8", errors="replace")  # type: ignore[union-attr]
        except (AttributeError, ValueError):
            pass


def _out(text: str = "") -> None:
    print(text)


def _err(text: str) -> None:
    print(text, file=sys.stderr)


def _rule(title: str = "") -> None:
    if title:
        _out(f"== {title} " + "=" * max(0, 72 - len(title)))
    else:
        _out("=" * 76)


def _read_path(raw: str | None) -> Path:
    if raw:
        return Path(raw).expanduser().resolve()
    return Path.cwd().resolve()


def _resolve_root(raw: str | None, *, nearest_git: bool = False) -> Path:
    root = _read_path(raw)
    if not root.is_dir():
        raise SystemExit(f"error: not a directory: {root}")
    return detect_repo_root(root) if nearest_git else root


def _require_task(task: list[str]) -> str:
    text = " ".join(task).strip()
    if not text:
        raise SystemExit("error: a task description is required")
    return text


# ---------------------------------------------------------------------------
# Commands
# ---------------------------------------------------------------------------


def cmd_index(args: argparse.Namespace) -> int:
    root = _resolve_root(getattr(args, "path", None) or args.root)
    scan = discover(root)
    index, built = ensure_index(
        root, refresh=args.refresh, db_path=_db_path(args.db), scanned=scan
    )
    try:
        stats = index_overview(index)
        if args.json:
            _out(json.dumps(stats, indent=2))
            return 0
        _out(f"Repository: {scan.profile.name}")
        _out(f"Root:       {root}")
        _out(f"Index:      {'built' if built else 'reused'} at {index.db_path}")
        _out(f"Files:      {stats['files']:,} ({human_bytes(sum(f.bytes for f in scan.files))})")
        _out(f"Lines:      {scan.profile.total_lines:,}")
        _out(f"Symbols:    {stats['symbols']:,}")
        _out(f"Imports:    {stats['imports']:,} ({index.conn.execute('SELECT COUNT(*) FROM edges').fetchone()[0]:,} resolved edges)")
        _out(f"Tests:      {scan.profile.test_files:,} flagged test files")
        if stats.get("languages"):
            rendered = ", ".join(
                f"{row['lang']} ({row['lines']:,})" for row in stats["languages"][:6]  # type: ignore[index]
            )
            _out(f"Languages:  {rendered}")
        if scan.skipped:
            _out(
                "Skipped:    "
                + ", ".join(f"{key}={value}" for key, value in scan.skipped.items() if value)
            )
        _out("")
        _out("Next: reposcout map \"<task>\" or reposcout plan \"<task>\" -o report.html")
        return 0
    finally:
        index.close()


def cmd_overview(args: argparse.Namespace) -> int:
    root = _resolve_root(getattr(args, "path", None) or args.root)
    scan = discover(root)
    profile = scan.profile
    index, built = ensure_index(
        root, refresh=args.refresh, db_path=_db_path(args.db), scanned=scan
    )
    try:
        underlying = Index.stats(index)
        data = {
            "profile": profile.to_dict(),
            "index": underlying,
            "checks": _checks_summary(root, profile, scan),
            "skipped": scan.skipped,
        }
        if args.json:
            _out(json.dumps(data, indent=2))
            return 0

        _rule(profile.name or "repository")
        if profile.description:
            _out(profile.description)
            _out("")
        _out(f"Root:       {root}")
        _out(f"Index:      {'built' if built else 'reused'} at {index.db_path}")
        _out(f"Files:      {profile.files_indexed:,} indexed, {profile.test_files:,} tests")
        _out(f"Lines:      {profile.total_lines:,}")
        _out(f"Git:        {'yes' if profile.is_git else 'no'}"
             + (f" ({profile.branch} @ {profile.commit})" if profile.commit else ""))
        if profile.languages:
            _out("Languages:  " + ", ".join(f"{k} ({v:,})" for k, v in profile.languages.items()))
        if profile.manifests:
            _out("Manifests:  " + ", ".join(
                f"{m.path}" + (f" [{m.name}]" if m.name else "") for m in profile.manifests[:8]
            ))
        if profile.entry_points:
            _out("Entry:      " + ", ".join(profile.entry_points[:6]))
        if profile.docs:
            _out("Docs:       " + ", ".join(profile.docs[:6]))
        if profile.guidelines:
            _out("Guides:     " + ", ".join(profile.guidelines[:6]))
        if profile.ci_workflows:
            _out("CI:         " + ", ".join(profile.ci_workflows[:6]))
        checks = data["checks"]
        _out(
            f"Checks:     {checks['verified']} declared, {checks['suggested']} suggested, "
            f"{checks['runnable']} runnable here"
        )
        if profile.top_level:
            _out("Top level:  " + ", ".join(profile.top_level[:12]))
        _out("")
        _out(f"Indexed at {underlying['created_at']} ({underlying['files']:,} files, "
             f"{underlying['symbols']:,} symbols).")
        return 0
    finally:
        index.close()


def cmd_map(args: argparse.Namespace) -> int:
    task = _require_task(args.task)
    root = _resolve_root(args.root, nearest_git=True)
    scan = discover(root)
    index, _built = ensure_index(
        root, refresh=args.refresh, db_path=_db_path(args.db), scanned=scan
    )
    try:
        mapping = map_task(
            index,
            task,
            options=RetrievalOptions(
                max_primary=args.limit,
                expand_graph=not args.no_graph,
            ),
        )
        if args.json:
            payload = {
                "task": task,
                "repo_root": str(root),
                "generated_at": now_iso(),
                "mapping": mapping.to_dict(),
            }
            _out(json.dumps(payload, indent=2))
            return 0
        _print_mapping(mapping)
        return 0
    finally:
        index.close()


def cmd_plan(args: argparse.Namespace) -> int:
    task = _require_task(args.task)
    root = _resolve_root(args.root, nearest_git=True)
    if args.run_checks and not args.yes:
        raise SystemExit("error: --run-checks executes declared commands; pass --yes to confirm")
    options = AgentOptions(
        refresh=args.refresh,
        use_llm=args.llm,
        expand_graph=not args.no_graph,
        db_path=_db_path(args.db),
        max_primary=args.limit,
        run_checks=args.run_checks,
        timeout=args.timeout,
    )
    agent = RepoAgent(root, options)
    run = agent.run(task, run_checks=args.run_checks)
    try:
        fmt = args.format
        if args.output:
            target = Path(args.output).expanduser().resolve()
            if fmt == "md" and target.suffix.lower() in {".html", ".json"}:
                fmt = "html" if target.suffix.lower() == ".html" else "json"
            path = write_report(run.report, target, fmt)
            _err(f"Wrote {fmt} report: {path}")
            if run.check_runs:
                _print_check_runs(run.check_runs)
            return 0

        if fmt == "html":
            target = suggest_output_path(root, task)
            path = write_report(run.report, target, "html")
            _err(f"Wrote HTML report: {path}")
            _out(str(path))
            if run.check_runs:
                _print_check_runs(run.check_runs)
            return 0

        if fmt == "json":
            _out(render_json(run.report))
            return 0

        _out(render_markdown(run.report))
        if run.check_runs:
            _print_check_runs(run.check_runs)
        return 0
    finally:
        run.close()


def cmd_checks(args: argparse.Namespace) -> int:
    root = _resolve_root(args.root, nearest_git=True)
    scan = discover(root)
    check_set = discover_checks(root, scan.profile, [item.path for item in scan.files])
    runs: list[CheckRun] = []
    if args.run:
        if not args.yes:
            raise SystemExit(
                "error: --run executes declared commands in this repository; pass --yes to confirm"
            )
        for check in check_set.verified:
            if not check.runnable:
                continue
            try:
                runs.append(run_check(check, root, timeout=args.timeout))
            except (ValueError, OSError) as exc:
                check.tool_notes.append(f"execution failed: {exc}")

    if args.json:
        payload = {
            "repo_root": str(root),
            "verified": [
                {
                    "id": check.id,
                    "command": check.command,
                    "kind": check.kind,
                    "declared_in": check.declared_in,
                    "sources": [s.__dict__ for s in check.sources],
                    "runnable": check.runnable,
                    "notes": check.tool_notes,
                }
                for check in check_set.verified
            ],
            "suggested": [
                {
                    "id": check.id,
                    "command": check.command,
                    "kind": check.kind,
                    "basis": check.basis,
                    "scope": check.scope,
                }
                for check in check_set.suggested
            ],
            "runs": [
                {
                    "id": run.check_id,
                    "command": run.command,
                    "returncode": run.returncode,
                    "ok": run.ok,
                    "timed_out": run.timed_out,
                    "duration_ms": run.duration_ms,
                    "output": run.output,
                }
                for run in runs
            ],
            "notes": check_set.notes,
        }
        _out(json.dumps(payload, indent=2))
        return 0 if all(run.ok for run in runs) else 1

    _print_checks(check_set, show_run_hint=not args.run)
    if runs:
        _print_check_runs(runs)
    return 0 if all(run.ok for run in runs) else 1


def cmd_version(_args: argparse.Namespace) -> int:
    _out(f"RepoScout {__version__}")
    return 0


# ---------------------------------------------------------------------------
# Printing
# ---------------------------------------------------------------------------


def _print_mapping(mapping: Mapping) -> None:
    _rule("Task")
    _out(mapping.task)
    _out("")
    _out(
        f"Confidence: {mapping.confidence_label} ({mapping.confidence:.2f})"
        + (f"  identifiers: {', '.join(mapping.identifiers[:6])}" if mapping.identifiers else "")
    )
    for item in mapping.confidence_rationale:
        _out(f"  - {item}")
    if mapping.query_terms:
        terms = ", ".join(term.term for term in mapping.query_terms[:16])
        _out(f"Terms: {terms}")
    _out("")

    for title, hits in (
        ("PRIMARY TARGETS", mapping.primaries),
        ("SUPPORTING FILES", mapping.supporting),
        ("EXISTING TESTS", mapping.tests),
        ("DOCS / CONFIG", mapping.docs),
    ):
        if not hits:
            continue
        _rule(title)
        for position, hit in enumerate(hits, start=1):
            _out(f"{position}. {hit.path}  [score {hit.score:.1f}, {hit.role}, {hit.lines} lines]")
            for reason in hit.reasons[:4]:
                suffix = " (inferred)" if reason.inferred else ""
                _out(f"   - {reason.text}{suffix}")
            for snippet in hit.snippets[:3]:
                _out(f"   {hit.path}:{snippet.line}: {truncate(snippet.text, 160)}")
            if hit.symbols:
                rendered = ", ".join(f"{symbol.qualified} (L{symbol.line})" for symbol in hit.symbols[:4])
                _out(f"   symbols: {rendered}")
            if hit.imported_by:
                _out(f"   imported by: {', '.join(hit.imported_by[:3])}")
            _out("")
    if mapping.notes:
        _rule("NOTES")
        for note in mapping.notes:
            _out(f"- {note}")
        _out("")


def _print_checks(check_set: CheckSet, *, show_run_hint: bool = True) -> None:
    _rule("VERIFIED (declared in this repository)")
    if check_set.verified:
        for check in check_set.verified:
            where = check.declared_in or (check.sources[0].label() if check.sources else "?")
            flags = []
            flags.append("runnable here" if check.runnable else "not runnable here")
            if check.tool_notes:
                flags.append("; ".join(check.tool_notes))
            _out(f"[{check.kind}] {check.command}")
            _out(f"    declared at {where} ({', '.join(flags)})")
    else:
        _out("None. This repository does not declare how to verify changes.")
    _out("")

    _rule("SUGGESTED (inferred, not declared)")
    if check_set.suggested:
        for check in check_set.suggested:
            _out(f"[{check.kind}] {check.command}")
            _out(f"    basis: {check.basis}")
            if check.scope:
                _out(f"    scope: {', '.join(check.scope[:4])}")
    else:
        _out("None.")
    _out("")
    for note in check_set.notes:
        _out(f"- {note}")
    if show_run_hint and any(check.runnable for check in check_set.verified):
        _out("")
        _out("Run declared+runnable checks with: reposcout checks --run --yes")


def _print_check_runs(runs: list[CheckRun]) -> None:
    _rule("CHECK RESULTS")
    for run in runs:
        status = "PASS" if run.ok else "FAIL"
        _out(f"[{status}] {run.command}  ({run.duration_ms} ms, exit {run.returncode})")
        if run.output:
            output_lines = run.output.splitlines()
            for line in output_lines[:40]:
                _out(f"    {truncate(line, 300)}")
            if len(output_lines) > 40:
                _out("    ... output truncated")
        _out("")
    passed = sum(1 for run in runs if run.ok)
    _out(f"{passed}/{len(runs)} executed checks passed. These are real results, not declarations.")


def _checks_summary(root: Path, profile: RepoProfile, scan: ScanResult) -> dict[str, int]:
    try:
        check_set = discover_checks(root, profile, [item.path for item in scan.files])
    except (OSError, ValueError):
        return {"verified": 0, "suggested": 0, "runnable": 0}
    return {
        "verified": len(check_set.verified),
        "suggested": len(check_set.suggested),
        "runnable": sum(1 for check in check_set.checks if check.runnable),
    }


def _db_path(raw: str | None) -> Path | None:
    return Path(raw).expanduser().resolve() if raw else None


# ---------------------------------------------------------------------------
# Parser
# ---------------------------------------------------------------------------


def build_parser() -> argparse.ArgumentParser:
    common = argparse.ArgumentParser(add_help=False)
    common.add_argument(
        "-C",
        "--root",
        metavar="PATH",
        help="repository directory (default: current directory)",
    )
    common.add_argument("--db", metavar="PATH", help="alternate SQLite index path")
    common.add_argument(
        "--refresh",
        action="store_true",
        help="rebuild the index before running",
    )

    parser = argparse.ArgumentParser(
        prog="reposcout",
        description=(
            "Map a task to repository code with evidence, and separate declared checks "
            "from unverified suggestions."
        ),
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog=(
            "examples:\n"
            "  reposcout index .\n"
            "  reposcout map \"fix crash in config loader\"\n"
            "  reposcout plan \"add timeout option\" -o brief.html\n"
            "  reposcout checks --run --yes\n"
        ),
    )
    parser.add_argument("--version", action="version", version=f"RepoScout {__version__}")
    subparsers = parser.add_subparsers(dest="command", metavar="COMMAND")

    index_parser = subparsers.add_parser(
        "index", parents=[common], help="build or refresh the SQLite index"
    )
    index_parser.add_argument(
        "path", nargs="?", help="repository directory (default: current directory)"
    )
    index_parser.add_argument("--json", action="store_true", help="emit JSON statistics")
    index_parser.set_defaults(func=cmd_index)

    overview_parser = subparsers.add_parser(
        "overview", parents=[common], help="describe the repository and its verification setup"
    )
    overview_parser.add_argument(
        "path", nargs="?", help="repository directory (default: current directory)"
    )
    overview_parser.add_argument("--json", action="store_true", help="emit JSON")
    overview_parser.set_defaults(func=cmd_overview)

    map_parser = subparsers.add_parser(
        "map", parents=[common], help="rank files for a task and show the evidence"
    )
    map_parser.add_argument("task", nargs="+", help="task or issue description")
    map_parser.add_argument("--limit", type=int, default=5, help="maximum primary targets (default: 5)")
    map_parser.add_argument("--json", action="store_true", help="emit JSON")
    map_parser.add_argument("--no-graph", action="store_true", help="disable import-graph expansion")
    map_parser.set_defaults(func=cmd_map)

    plan_parser = subparsers.add_parser(
        "plan", parents=[common], help="produce a contribution plan and report"
    )
    plan_parser.add_argument("task", nargs="+", help="task or issue description")
    plan_parser.add_argument(
        "--format", choices=FORMATS, default="md", help="report format (default: md)"
    )
    plan_parser.add_argument("-o", "--output", help="write the report to this path")
    plan_parser.add_argument("--llm", action="store_true", help="opt in to an OpenAI-compatible model")
    plan_parser.add_argument("--limit", type=int, default=5, help="maximum primary targets (default: 5)")
    plan_parser.add_argument("--no-graph", action="store_true", help="disable import-graph expansion")
    plan_parser.add_argument(
        "--run-checks",
        action="store_true",
        help="execute declared+runnable checks (requires --yes)",
    )
    plan_parser.add_argument("--yes", action="store_true", help="confirm executing checks")
    plan_parser.add_argument(
        "--timeout", type=int, default=180, help="per-check timeout in seconds (default: 180)"
    )
    plan_parser.set_defaults(func=cmd_plan)

    checks_parser = subparsers.add_parser(
        "checks", parents=[common], help="list declared and suggested verification commands"
    )
    checks_parser.add_argument("--json", action="store_true", help="emit JSON")
    checks_parser.add_argument(
        "--run", action="store_true", help="execute declared+runnable checks (requires --yes)"
    )
    checks_parser.add_argument("--yes", action="store_true", help="confirm executing checks")
    checks_parser.add_argument(
        "--timeout", type=int, default=180, help="per-check timeout in seconds (default: 180)"
    )
    checks_parser.set_defaults(func=cmd_checks)

    version_parser = subparsers.add_parser("version", help="print the version")
    version_parser.set_defaults(func=cmd_version)

    return parser


def main(argv: list[str] | None = None) -> int:
    _configure_stdio()
    parser = build_parser()
    args = parser.parse_args(argv)
    if not getattr(args, "command", None):
        parser.print_help()
        return 0
    try:
        return int(args.func(args))
    except KeyboardInterrupt:
        _err("interrupted")
        return 130
    except BrokenPipeError:
        return 0
    except SystemExit:
        raise
    except (OSError, ValueError, RuntimeError) as exc:
        _err(f"error: {exc}")
        return 1


if __name__ == "__main__":  # pragma: no cover
    raise SystemExit(main())
```

## tests/support.py

`tests/support.py` - 196 lines

````py
"""Shared fixtures for the RepoScout test suite.

Fixtures are created in the system temporary directory and removed by the test
that created them.  Nothing here touches the RepoScout checkout itself.
"""

from __future__ import annotations

import json
import shutil
import tempfile
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parents[1]


class RepoFixture:
    def __init__(self, prefix: str = "reposcout-test-") -> None:
        self.root = Path(tempfile.mkdtemp(prefix=prefix))

    def write(self, relative: str, text: str) -> Path:
        path = self.root / relative
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(text, encoding="utf-8", newline="\n")
        return path

    def cleanup(self) -> None:
        shutil.rmtree(self.root, ignore_errors=True)


def python_repo() -> RepoFixture:
    """A tiny Python project with a known ranking target and declared checks."""
    repo = RepoFixture("reposcout-py-")
    repo.write(
        "README.md",
        "# SampleLib\n\n"
        "SampleLib loads JSON configuration files and makes small HTTP requests.\n\n"
        "## Usage\n\n"
        "Run the command line interface to load a configuration.\n",
    )
    repo.write(
        "pyproject.toml",
        '[project]\n'
        'name = "samplelib"\n'
        'version = "0.3.0"\n'
        'description = "Load JSON configuration files with clear errors."\n'
        'license = { text = "MIT" }\n'
        "\n"
        "[tool.pytest.ini_options]\n"
        'testpaths = ["tests"]\n'
        "\n"
        "[tool.ruff]\n"
        "line-length = 100\n",
    )
    repo.write(
        "CONTRIBUTING.md",
        "# Contributing\n\n"
        "Use a topic branch and keep patches small.\n\n"
        "## Pull request checklist\n\n"
        "- [ ] A regression test is included.\n"
        "- [ ] The change is limited to one behavior.\n"
        "- [ ] The PR description lists the commands that were run.\n\n"
        "Run the suite with:\n\n"
        "```bash\n"
        "python -m unittest discover -s tests -v\n"
        "```\n",
    )
    repo.write(
        ".github/workflows/tests.yml",
        "name: tests\n"
        "on: [push, pull_request]\n"
        "jobs:\n"
        "  test:\n"
        "    runs-on: ubuntu-latest\n"
        "    steps:\n"
        "      - uses: actions/checkout@v4\n"
        "      - uses: actions/setup-python@v5\n"
        "        with:\n"
        '          python-version: "3.12"\n'
        "      - name: Unit tests\n"
        "        run: python -m unittest discover -s tests -v\n",
    )
    repo.write(
        "samplelib/__init__.py",
        '"""SampleLib package."""\n\n'
        "from .config import ConfigError, load_config\n\n"
        '__all__ = ["ConfigError", "load_config"]\n',
    )
    repo.write(
        "samplelib/config.py",
        '"""Configuration loading helpers."""\n\n'
        "import json\n"
        "from pathlib import Path\n\n"
        "DEFAULT_TIMEOUT = 30\n\n\n"
        "class ConfigError(Exception):\n"
        '    """Raised when a configuration file cannot be read."""\n\n\n'
        "def load_config(path):\n"
        '    """Load a JSON configuration file from disk."""\n'
        "    file_path = Path(path)\n"
        "    if not file_path.exists():\n"
        '        raise ConfigError(f"configuration file not found: {file_path}")\n'
        '    with file_path.open("r", encoding="utf-8") as handle:\n'
        "        return json.load(handle)\n",
    )
    repo.write(
        "samplelib/client.py",
        '"""HTTP client with a configurable timeout."""\n\n'
        "from .config import DEFAULT_TIMEOUT\n\n\n"
        "class HttpClient:\n"
        '    """Minimal client wrapper used by the tests."""\n\n'
        "    def __init__(self, timeout=DEFAULT_TIMEOUT):\n"
        "        self.timeout = timeout\n\n"
        "    def get(self, url):\n"
        '        """Return the request metadata for a URL."""\n'
        '        return {"url": url, "timeout": self.timeout}\n',
    )
    repo.write(
        "samplelib/cli.py",
        '"""Command line entry point."""\n\n'
        "from .client import HttpClient\n"
        "from .config import load_config\n\n\n"
        "def main(argv):\n"
        '    """Load a configuration file and make one request."""\n'
        "    config = load_config(argv[1])\n"
        '    client = HttpClient(timeout=config.get("timeout", 30))\n'
        '    return client.get(config["url"])\n',
    )
    repo.write(
        "tests/test_config.py",
        "import json\n"
        "import tempfile\n"
        "import unittest\n"
        "from pathlib import Path\n\n"
        "from samplelib.config import ConfigError, load_config\n\n\n"
        "class LoadConfigTests(unittest.TestCase):\n"
        "    def test_missing_file_raises_config_error(self):\n"
        '        with self.assertRaises(ConfigError):\n'
        '            load_config("/definitely/not/here.json")\n\n'
        "    def test_reads_json_object(self):\n"
        "        with tempfile.TemporaryDirectory() as directory:\n"
        '            path = Path(directory) / "config.json"\n'
        '            path.write_text(json.dumps({"timeout": 5}), encoding="utf-8")\n'
        '            self.assertEqual(load_config(path)["timeout"], 5)\n\n\n'
        'if __name__ == "__main__":\n'
        "    unittest.main()\n",
    )
    repo.write(
        "tests/test_client.py",
        "import unittest\n\n"
        "from samplelib.client import HttpClient\n\n\n"
        "class HttpClientTests(unittest.TestCase):\n"
        "    def test_timeout_is_kept(self):\n"
        "        client = HttpClient(timeout=12)\n"
        '        self.assertEqual(client.get("https://example.test")["timeout"], 12)\n\n\n'
        'if __name__ == "__main__":\n'
        "    unittest.main()\n",
    )
    repo.write("docs/usage.md", "# Usage\n\nCall `load_config` before constructing the client.\n")
    repo.write(".gitignore", ".venv/\n*.log\n.reposcout/\n")
    repo.write(".env", "SAMPLELIB_API_KEY=super-secret-value\n")
    repo.write(".env.example", "SAMPLELIB_API_KEY=replace-me\n")
    repo.write("logs/run.log", "this file is ignored\n")
    return repo


def js_repo() -> RepoFixture:
    repo = RepoFixture("reposcout-js-")
    repo.write("README.md", "# Tiny JS project\n\nA package used to test script discovery.\n")
    repo.write(
        "package.json",
        json.dumps(
            {
                "name": "tiny-js",
                "version": "1.0.0",
                "description": "Tiny JavaScript fixture",
                "license": "MIT",
                "scripts": {
                    "test": "jest",
                    "lint": "eslint .",
                    "build": "tsc",
                },
                "devDependencies": {"jest": "^29.0.0", "eslint": "^8.0.0"},
            },
            indent=2,
        ),
    )
    repo.write("src/math.js", "export function add(a, b) {\n  return a + b;\n}\n")
    repo.write(
        "src/math.test.js",
        'import { add } from "./math.js";\n\n'
        'test("adds numbers", () => {\n'
        "  expect(add(1, 2)).toBe(3);\n"
        "});\n",
    )
    repo.write("yarn.lock", "# lockfile placeholder\n")
    return repo
````

## tests/test_models.py

`tests/test_models.py` - 70 lines

```py
import unittest

from reposcout.models import Action, Check, Claim, Source


class SourceTests(unittest.TestCase):
    def test_label_uses_line_when_present(self):
        self.assertEqual(Source("a/b.py").label(), "a/b.py")
        self.assertEqual(Source("a/b.py", 12).label(), "a/b.py:12")


class ClaimTests(unittest.TestCase):
    def test_verified_requires_source(self):
        with self.assertRaises(ValueError):
            Claim("the parser exists", "verified")

    def test_suggested_requires_basis(self):
        with self.assertRaises(ValueError):
            Claim("the parser should move", "suggested")

    def test_unknown_status_rejected(self):
        with self.assertRaises(ValueError):
            Claim("x", "probably", sources=[Source("x")])

    def test_valid_claims(self):
        verified = Claim("exists", "verified", sources=[Source("a.py", 1)])
        suggested = Claim("could move", "suggested", basis="review heuristic")
        self.assertEqual(verified.sources[0].line, 1)
        self.assertEqual(suggested.status, "suggested")


class CheckTests(unittest.TestCase):
    def test_verified_check_requires_source(self):
        with self.assertRaises(ValueError):
            Check("test-1", "python -m unittest", "test", "verified")

    def test_suggested_check_requires_basis(self):
        with self.assertRaises(ValueError):
            Check("test-1", "pytest", "test", "suggested")

    def test_valid_check(self):
        check = Check(
            "test-1",
            "python -m unittest",
            "test",
            "verified",
            sources=[Source("pyproject.toml", 4)],
            declared_in="pyproject.toml:4",
        )
        self.assertEqual(check.declared_in, "pyproject.toml:4")


class ActionTests(unittest.TestCase):
    def test_verified_action_requires_source(self):
        with self.assertRaises(ValueError):
            Action("edit", "change it", status="verified")

    def test_suggested_action_requires_basis(self):
        with self.assertRaises(ValueError):
            Action("edit", "change it")

    def test_valid_action(self):
        action = Action(
            "read", "read it", status="verified", sources=[Source("a.py", 3)]
        )
        self.assertEqual(action.sources[0].line, 3)


if __name__ == "__main__":
    unittest.main()
```

## tests/test_tokens.py

`tests/test_tokens.py` - 66 lines

```py
import unittest

from reposcout import tokens as tok


class TokenTests(unittest.TestCase):
    def test_split_identifier(self):
        self.assertEqual(tok.split_identifier("loadConfigFile"), ["load", "config", "file"])
        self.assertEqual(tok.split_identifier("HTTPClient"), ["http", "client"])
        self.assertEqual(tok.split_identifier("snake_case_name"), ["snake", "case", "name"])

    def test_stem_is_conservative(self):
        self.assertEqual(tok.stem("configurations"), "configuration")
        self.assertEqual(tok.stem("loading"), "load")
        self.assertEqual(tok.stem("files"), "file")
        self.assertEqual(tok.stem("class"), "class")

    def test_tokens_drop_filler(self):
        self.assertNotIn("the", tok.tokens("the config file"))
        self.assertIn("config", tok.tokens("the config file"))
        self.assertIn("the", tok.tokens("the config file", include_stopwords=True))

    def test_raw_tokens_keep_surface_forms(self):
        self.assertEqual(tok.raw_tokens("LoadConfig configs"), ["load", "config", "configs"])

    def test_synonyms_are_human_readable(self):
        synonyms = tok.synonyms_for("config")
        self.assertIn("settings", synonyms)
        self.assertIn("configuration", synonyms)

    def test_task_signals(self):
        task = (
            "`load_config` crashes in `samplelib/config.py` with ConfigError"
        )
        signals = tok.extract_task_signals(task)
        self.assertIn("load_config", signals.identifiers)
        self.assertIn("ConfigError", signals.identifiers)
        self.assertIn("samplelib/config.py", signals.paths)
        self.assertIn("config", signals.identifiers)

    def test_quoted_phrase(self):
        signals = tok.extract_task_signals('show "configuration file not found" instead')
        self.assertIn("configuration file not found", signals.phrases)

    def test_test_case_classifier_excludes_helpers(self):
        self.assertTrue(tok.looks_like_test_case_path("tests/test_config.py"))
        self.assertTrue(tok.looks_like_test_case_path("src/math.test.js"))
        self.assertTrue(tok.looks_like_test_case_path("internal/store/store_test.go"))
        self.assertFalse(tok.looks_like_test_case_path("tests/support.py"))
        self.assertFalse(tok.looks_like_test_case_path("tests/conftest.py"))

    def test_path_classifiers(self):
        self.assertTrue(tok.looks_like_test_path("tests/test_config.py"))
        self.assertTrue(tok.looks_like_test_path("src/math.test.js"))
        self.assertTrue(tok.looks_like_test_path("internal/store/store_test.go"))
        self.assertFalse(tok.looks_like_test_path("src/contest.py"))
        self.assertTrue(tok.looks_like_doc_path("README.md"))
        self.assertTrue(tok.looks_like_doc_path("docs/usage.md"))
        self.assertTrue(tok.looks_like_doc_path("SUPPORT.md"))
        self.assertFalse(tok.looks_like_doc_path("tests/support.py"))
        self.assertTrue(tok.looks_like_config_path("pyproject.toml"))
        self.assertTrue(tok.looks_like_config_path(".github/workflows/tests.yml"))


if __name__ == "__main__":
    unittest.main()
```

## tests/test_scanning.py

`tests/test_scanning.py` - 68 lines

```py
import unittest

from reposcout.scanning import discover, is_secret_filename
from support import RepoFixture, python_repo


class SecretTests(unittest.TestCase):
    def test_secret_files_are_rejected(self):
        self.assertTrue(is_secret_filename(".env"))
        self.assertTrue(is_secret_filename(".env.production"))
        self.assertTrue(is_secret_filename("id_rsa"))
        self.assertTrue(is_secret_filename("server.pem"))

    def test_documentation_variants_are_allowed(self):
        self.assertFalse(is_secret_filename(".env.example"))
        self.assertFalse(is_secret_filename(".env.sample"))
        self.assertFalse(is_secret_filename("secrets.md"))


class DiscoveryTests(unittest.TestCase):
    def setUp(self):
        self.repo = python_repo()
        self.addCleanup(self.repo.cleanup)
        self.scan = discover(self.repo.root)

    def test_skips_secrets_and_gitignored_files(self):
        paths = {item.path for item in self.scan.files}
        self.assertNotIn(".env", paths)
        self.assertIn(".env.example", paths)
        self.assertNotIn("logs/run.log", paths)
        self.assertGreater(self.scan.skipped["ignored"], 0)

    def test_profile_identity_and_structure(self):
        profile = self.scan.profile
        self.assertEqual(profile.name, "samplelib")
        self.assertIn("Load JSON configuration", profile.description)
        self.assertEqual(profile.test_files, 2)
        self.assertTrue(profile.docs)
        self.assertIn("README.md", profile.docs)
        self.assertIn("CONTRIBUTING.md", profile.guidelines)
        self.assertIn(".github/workflows/tests.yml", profile.ci_workflows)
        self.assertEqual(list(profile.languages)[0], "python")
        self.assertFalse(profile.is_git)

    def test_manifest_detection(self):
        kinds = {manifest.kind for manifest in self.scan.profile.manifests}
        self.assertIn("pyproject", kinds)

    def test_test_classification(self):
        by_path = {item.path: item for item in self.scan.files}
        self.assertTrue(by_path["tests/test_config.py"].is_test)
        self.assertFalse(by_path["samplelib/config.py"].is_test)
        self.assertTrue(by_path["README.md"].is_doc)


class GuidelineTests(unittest.TestCase):
    def test_test_support_module_is_not_a_guide(self):
        repo = RepoFixture("reposcout-guides-")
        self.addCleanup(repo.cleanup)
        repo.write("SUPPORT.md", "# Support\n")
        repo.write("tests/support.py", "VALUE = 1\n")
        scan = discover(repo.root)
        self.assertIn("SUPPORT.md", scan.profile.guidelines)
        self.assertNotIn("tests/support.py", scan.profile.guidelines)


if __name__ == "__main__":
    unittest.main()
```

## tests/test_indexing.py

`tests/test_indexing.py` - 89 lines

```py
import unittest

from reposcout.indexing import ensure_index, index_overview
from reposcout.retrieval import RetrievalOptions, map_task
from support import python_repo


class IndexTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.repo = python_repo()
        cls.index, cls.built = ensure_index(cls.repo.root)
        cls.indexed_at = cls.index.indexed_at

    @classmethod
    def tearDownClass(cls):
        cls.index.close()
        cls.repo.cleanup()

    def test_index_contains_files_and_symbols(self):
        paths = self.index.file_paths()
        self.assertIn("samplelib/config.py", paths)
        self.assertIn("tests/test_config.py", paths)
        self.assertNotIn(".env", paths)
        self.assertGreater(self.index.file_count, 5)
        found = self.index.find_symbols(["load_config", "ConfigError"])
        names = {symbol.name for _path, symbol in found}
        self.assertIn("load_config", names)
        self.assertIn("ConfigError", names)

    def test_relative_imports_resolve(self):
        cli_id = self.index.file_id("samplelib/cli.py")
        self.assertIsNotNone(cli_id)
        imported = self.index.imported_paths(cli_id or -1)
        self.assertIn("samplelib/config.py", imported)
        self.assertIn("samplelib/client.py", imported)
        importers = self.index.importers(self.index.file_id("samplelib/config.py") or -1)
        self.assertIn("samplelib/__init__.py", importers)

    def test_search_and_stats(self):
        hits = self.index.search({"configuration": 1.0, "missing": 0.8})
        self.assertTrue(hits)
        self.assertIn("samplelib/config.py", [hit.path for hit in hits[:5]])
        stats = index_overview(self.index)
        self.assertGreater(stats["symbols"], 0)
        self.assertEqual(stats["languages"][0]["lang"], "python")

    def test_ensure_index_reuses_existing_database(self):
        reopened, built = ensure_index(self.repo.root)
        try:
            self.assertFalse(built)
            self.assertEqual(reopened.indexed_at, self.indexed_at)
        finally:
            reopened.close()

    def test_map_task_returns_evidence_and_tests(self):
        mapping = map_task(
            self.index,
            "missing configuration file raises ConfigError when loading JSON",
            options=RetrievalOptions(max_primary=4),
        )
        self.assertTrue(mapping.primaries)
        self.assertEqual(mapping.primaries[0].path, "samplelib/config.py")
        self.assertIn("samplelib/config.py", [hit.path for hit in mapping.primaries])
        self.assertTrue(mapping.tests)
        self.assertIn("tests/test_config.py", [hit.path for hit in mapping.tests])
        top = mapping.primaries[0]
        self.assertTrue(top.snippets, "the top target must carry line-level evidence")
        self.assertTrue(any(reason.kind in {"match", "symbol", "path"} for reason in top.reasons))
        self.assertGreaterEqual(mapping.confidence, 0.45)
        self.assertIn(mapping.confidence_label, {"medium", "high"})

    def test_map_task_explains_missing_terms(self):
        mapping = map_task(self.index, "the and of to a")
        self.assertFalse(mapping.primaries)
        self.assertEqual(mapping.confidence_label, "low")
        self.assertTrue(mapping.notes)

    def test_confidence_does_not_claim_a_missing_identifier(self):
        mapping = map_task(self.index, "make loading configuration friendlier for newcomers")
        if mapping.identifiers:
            self.skipTest("task unexpectedly produced explicit identifiers")
        rationale = " ".join(mapping.confidence_rationale)
        self.assertNotIn("names a path or symbol", rationale)
        self.assertIn("No explicit symbol or filename", " ".join(mapping.notes))


if __name__ == "__main__":
    unittest.main()
```

## tests/test_checks.py

`tests/test_checks.py` - 182 lines

````py
import unittest

from reposcout.checks import CheckSet, discover_checks, run_check
from reposcout.models import Check, Source
from reposcout.scanning import discover
from support import RepoFixture, js_repo, python_repo


class PythonCheckTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.repo = python_repo()
        cls.scan = discover(cls.repo.root)
        cls.checks = discover_checks(
            cls.repo.root, cls.scan.profile, [item.path for item in cls.scan.files]
        )

    @classmethod
    def tearDownClass(cls):
        cls.repo.cleanup()

    def test_commands_are_found_and_provenanced(self):
        commands = {check.command for check in self.checks.checks}
        self.assertIn("python -m unittest discover -s tests -v", commands)
        self.assertIn("python -m pytest", commands)
        self.assertIn("ruff check .", commands)
        for check in self.checks.verified:
            self.assertTrue(check.sources, check.command)
            self.assertEqual(check.status, "verified")
        for check in self.checks.suggested:
            self.assertTrue(check.basis, check.command)

    def test_ci_and_docs_merge_into_one_declared_test_command(self):
        check = next(
            check
            for check in self.checks.verified
            if check.command == "python -m unittest discover -s tests -v"
        )
        paths = {source.path for source in check.sources}
        self.assertIn(".github/workflows/tests.yml", paths)
        self.assertIn("CONTRIBUTING.md", paths)
        self.assertEqual(check.kind, "test")

    def test_pytest_config_is_verified_even_without_pytest_installed(self):
        check = next(check for check in self.checks.verified if check.command == "python -m pytest")
        self.assertEqual(check.declared_in.split(":")[0], "pyproject.toml")
        self.assertIn("pyproject.toml", {source.path for source in check.sources})

    def test_suggestion_cannot_be_executed(self):
        suggestion = Check(
            "test-1",
            "python -m pytest",
            "test",
            "suggested",
            basis="inferred from file names",
        )
        with self.assertRaises(ValueError):
            run_check(suggestion, self.repo.root)

    def test_declared_unittest_command_runs(self):
        check = next(
            check
            for check in self.checks.verified
            if check.command == "python -m unittest discover -s tests -v"
        )
        self.assertTrue(check.runnable, check.tool_notes)
        result = run_check(check, self.repo.root, timeout=120)
        self.assertTrue(result.ok, result.output)
        self.assertEqual(result.returncode, 0)
        self.assertIn("OK", result.output)


class JavaScriptCheckTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.repo = js_repo()
        cls.scan = discover(cls.repo.root)
        cls.checks = discover_checks(
            cls.repo.root, cls.scan.profile, [item.path for item in cls.scan.files]
        )

    @classmethod
    def tearDownClass(cls):
        cls.repo.cleanup()

    def test_package_scripts_use_lockfile_runner(self):
        commands = {check.command for check in self.checks.verified}
        self.assertIn("yarn run test", commands)
        self.assertIn("yarn run lint", commands)
        self.assertIn("yarn run build", commands)
        check = next(check for check in self.checks.verified if check.command == "yarn run test")
        self.assertEqual(check.kind, "test")
        self.assertIn("package.json", {source.path for source in check.sources})

    def test_js_runner_is_only_suggested(self):
        suggested = {check.command for check in self.checks.suggested}
        self.assertIn("npx jest", suggested)
        check = next(check for check in self.checks.suggested if check.command == "npx jest")
        self.assertTrue(check.basis)


class DocumentationCheckTests(unittest.TestCase):
    """Fenced blocks are full of examples and directory trees, not commands."""

    def setUp(self):
        self.repo = RepoFixture("reposcout-docs-")
        self.addCleanup(self.repo.cleanup)

    def discover(self):
        scan = discover(self.repo.root)
        return discover_checks(
            self.repo.root, scan.profile, [item.path for item in scan.files]
        )

    def test_usage_examples_are_not_declared_checks(self):
        self.repo.write(
            "README.md",
            "# Tool\n\n"
            "## Quick start\n\n"
            "```bash\n"
            "python -m tool plan --format html\n"
            "```\n",
        )
        self.repo.write("tool/__init__.py", "")
        commands = {check.command for check in self.discover().verified}
        self.assertNotIn("python -m tool plan --format html", commands)

    def test_directory_trees_are_not_commands(self):
        self.repo.write(
            "README.md",
            "# Tool\n\n"
            "## Development\n\n"
            "```bash\n"
            "python -m unittest discover -s tests -v\n"
            "```\n\n"
            "Layout:\n\n"
            "```\n"
            "tool/\n"
            "  core.py\n"
            "```\n",
        )
        self.repo.write("tool/__init__.py", "")
        self.repo.write("tests/test_core.py", "import unittest\n")
        commands = {check.command for check in self.discover().verified}
        self.assertIn("python -m unittest discover -s tests -v", commands)
        self.assertNotIn("tool/", commands)
        self.assertNotIn("core.py", commands)

    def test_contribution_docs_do_not_need_a_heading(self):
        self.repo.write(
            "CONTRIBUTING.md",
            "# Contributing\n\n"
            "```bash\n"
            "python -m unittest discover -s tests\n"
            "```\n",
        )
        self.repo.write("tests/test_core.py", "import unittest\n")
        commands = {check.command for check in self.discover().verified}
        self.assertIn("python -m unittest discover -s tests", commands)


class CheckSetTests(unittest.TestCase):
    def test_partition_helpers(self):
        verified = Check(
            "test-1",
            "python -m unittest",
            "test",
            "verified",
            sources=[Source("README.md", 3)],
        )
        suggested = Check(
            "lint-1", "ruff check .", "lint", "suggested", basis="ruff.toml exists"
        )
        check_set = CheckSet(checks=[verified, suggested])
        self.assertEqual([check.id for check in check_set.verified], ["test-1"])
        self.assertEqual([check.id for check in check_set.suggested], ["lint-1"])
        self.assertEqual(check_set.get("lint-1"), suggested)
        self.assertEqual(set(check_set.by_kind()), {"test", "lint"})


if __name__ == "__main__":
    unittest.main()
````

## tests/test_agent.py

`tests/test_agent.py` - 94 lines

```py
import unittest

from reposcout.agent import AgentOptions, RepoAgent
from support import python_repo


class AgentTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.repo = python_repo()
        cls.agent = RepoAgent(cls.repo.root, AgentOptions(max_primary=4))
        cls.agent_run = cls.agent.run(
            "missing configuration file should raise a clear ConfigError with the path"
        )

    @classmethod
    def tearDownClass(cls):
        cls.agent_run.close()
        cls.repo.cleanup()

    def test_report_has_all_sections(self):
        report = self.agent_run.report
        self.assertEqual(report.task, "missing configuration file should raise a clear ConfigError with the path")
        self.assertTrue(report.mapping.primaries)
        self.assertEqual(report.mapping.primaries[0].path, "samplelib/config.py")
        self.assertEqual(len(report.plan.steps), 7)
        self.assertTrue(report.plan.checklist)
        self.assertTrue(report.limitations)
        self.assertTrue(report.trace)
        self.assertEqual(report.version, "0.1.0")

    def test_every_action_and_claim_has_provenance(self):
        for step in self.agent_run.report.plan.steps:
            self.assertTrue(step.actions, f"step {step.number} has no actions")
            for action in step.actions:
                if action.status == "verified":
                    self.assertTrue(action.sources, action.detail)
                else:
                    self.assertTrue(action.basis, action.detail)
            for claim in step.rationale:
                if claim.status == "verified":
                    self.assertTrue(claim.sources, claim.text)
                else:
                    self.assertTrue(claim.basis, claim.text)
        for claim in self.agent_run.report.plan.checklist:
            if claim.status == "verified":
                self.assertTrue(claim.sources, claim.text)
            else:
                self.assertTrue(claim.basis, claim.text)

    def test_plan_uses_real_files_and_checks(self):
        plan = self.agent_run.report.plan
        self.assertIn("samplelib/config.py", plan.files_to_touch)
        self.assertIn("tests/test_config.py", plan.test_files)
        verified_commands = {
            check.command for check in self.agent_run.report.checks if check.status == "verified"
        }
        self.assertIn("python -m unittest discover -s tests -v", verified_commands)
        self.assertGreaterEqual(plan.difficulty_score, 5)
        self.assertLessEqual(plan.difficulty_score, 100)
        self.assertGreaterEqual(plan.estimated_minutes[0], 15)

    def test_checklist_includes_project_items(self):
        texts = [claim.text for claim in self.agent_run.report.plan.checklist]
        self.assertTrue(any("regression test is included" in text for text in texts))
        self.assertTrue(any(claim.status == "suggested" for claim in self.agent_run.report.plan.checklist))

    def test_trace_records_the_loop(self):
        tools = [step.tool for step in self.agent_run.report.trace]
        self.assertIn("index", tools)
        self.assertIn("map_task", tools)
        self.assertIn("discover_checks", tools)
        self.assertIn("build_plan", tools)
        self.assertTrue(all(step.index == position for position, step in enumerate(self.agent_run.report.trace, start=1)))

    def test_no_llm_without_opt_in(self):
        self.assertFalse(self.agent_run.report.llm_used)
        self.assertFalse(any("LLM" in note for note in self.agent_run.report.llm_notes))

    def test_requested_checks_are_executed_and_recorded(self):
        run = RepoAgent(self.repo.root, AgentOptions(max_primary=2)).run(
            "missing configuration file raises ConfigError", run_checks=True
        )
        try:
            results = run.report.check_results
            self.assertTrue(results)
            self.assertTrue(any(result.command.startswith("python -m unittest") for result in results))
            self.assertTrue(all(result.ok for result in results), [r.output for r in results])
        finally:
            run.close()


if __name__ == "__main__":
    unittest.main()
```

## tests/test_rendering.py

`tests/test_rendering.py` - 96 lines

```py
import json
import tempfile
import unittest
from pathlib import Path

from reposcout.agent import AgentOptions, RepoAgent
from reposcout.models import CheckResult
from reposcout.rendering import render_html, render_json, render_markdown, write_report
from support import python_repo


class RenderingTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.repo = python_repo()
        cls.agent = RepoAgent(cls.repo.root, AgentOptions(max_primary=3))
        cls.agent_run = cls.agent.run("configuration file missing should raise ConfigError")

    @classmethod
    def tearDownClass(cls):
        cls.agent_run.close()
        cls.repo.cleanup()

    def test_markdown_separates_provenance(self):
        markdown = render_markdown(self.agent_run.report)
        self.assertIn("VERIFIED", markdown)
        self.assertIn("SUGGESTED", markdown)
        self.assertIn("## Contribution plan", markdown)
        self.assertIn("## Checks", markdown)
        self.assertIn("samplelib/config.py", markdown)

    def test_html_is_self_contained_and_escapes_content(self):
        report = self.agent_run.report
        original = report.task
        report.task = '<script>alert("xss")</script> load_config'
        try:
            document = render_html(report)
        finally:
            report.task = original
        self.assertIn("<!doctype html>", document)
        self.assertIn("<style>", document)
        self.assertNotIn('<script>alert("xss")</script>', document)
        self.assertIn("&lt;script&gt;", document)
        self.assertNotIn("http://", document.split("<style>")[0])

    def test_json_roundtrip(self):
        data = json.loads(render_json(self.agent_run.report))
        self.assertEqual(data["version"], "0.1.0")
        self.assertIn("mapping", data)
        self.assertIn("checks", data)
        self.assertIn("plan", data)
        self.assertIn("check_results", data)
        self.assertTrue(data["checks"])
        for check in data["checks"]:
            self.assertIn(check["status"], {"verified", "suggested"})

    def test_execution_results_are_rendered(self):
        report = self.agent_run.report
        report.check_results = [
            CheckResult(
                check_id="test-1",
                command="python -m unittest discover -s tests -v",
                returncode=1,
                duration_ms=12,
                output="AssertionError: boom",
            )
        ]
        try:
            markdown = render_markdown(report)
            document = render_html(report)
        finally:
            report.check_results = []
        self.assertIn("Execution results", markdown)
        self.assertIn("FAIL", markdown)
        self.assertIn("AssertionError: boom", markdown)
        self.assertIn("Execution results", document)
        self.assertIn("FAIL", document)
        self.assertIn("AssertionError: boom", document)

    def test_write_report_creates_parent_directories(self):
        with tempfile.TemporaryDirectory() as directory:
            target = Path(directory) / "nested" / "brief.html"
            written = write_report(self.agent_run.report, target, "html")
            self.assertTrue(written.exists())
            self.assertGreater(written.stat().st_size, 5000)
            text = written.read_text(encoding="utf-8")
            self.assertIn("RepoScout", text)

    def test_write_report_rejects_unknown_format(self):
        with tempfile.TemporaryDirectory() as directory:
            with self.assertRaises(ValueError):
                write_report(self.agent_run.report, Path(directory) / "x.txt", "txt")


if __name__ == "__main__":
    unittest.main()
```

## tests/test_cli.py

`tests/test_cli.py` - 174 lines

```py
"""End-to-end tests for the command line interface.

These tests spawn ``python -m reposcout`` as a subprocess so the argparse
wiring, exit codes and JSON envelopes are exercised the same way a user would
exercise them.
"""

from __future__ import annotations

import json
import subprocess
import sys
import unittest

from support import PROJECT_ROOT, RepoFixture, python_repo


def run_cli(*arguments: str, timeout: int = 180) -> subprocess.CompletedProcess:
    return subprocess.run(
        [sys.executable, "-m", "reposcout", *arguments],
        cwd=PROJECT_ROOT,
        capture_output=True,
        text=True,
        encoding="utf-8",
        errors="replace",
        timeout=timeout,
    )


class CliTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.repo = python_repo()

    @classmethod
    def tearDownClass(cls):
        cls.repo.cleanup()

    def test_version(self):
        result = run_cli("--version")
        self.assertEqual(result.returncode, 0, result.stderr)
        self.assertIn("RepoScout 0.1.0", result.stdout)

    def test_index_accepts_a_positional_repository(self):
        result = run_cli("index", str(self.repo.root), "--json")
        self.assertEqual(result.returncode, 0, result.stderr)
        data = json.loads(result.stdout)
        self.assertGreater(data["files"], 5)
        self.assertGreater(data["symbols"], 0)
        self.assertEqual(data["languages"][0]["lang"], "python")

    def test_map_json_exposes_evidence(self):
        result = run_cli(
            "map",
            "-C",
            str(self.repo.root),
            "--json",
            "missing configuration file raises ConfigError",
        )
        self.assertEqual(result.returncode, 0, result.stderr)
        data = json.loads(result.stdout)
        mapping = data["mapping"]
        self.assertEqual(data["task"], "missing configuration file raises ConfigError")
        self.assertTrue(mapping["primaries"])
        self.assertEqual(mapping["primaries"][0]["path"], "samplelib/config.py")
        self.assertIn(mapping["confidence_label"], {"medium", "high"})

    def test_plan_json_contains_checks_and_plan(self):
        result = run_cli(
            "plan",
            "-C",
            str(self.repo.root),
            "--format",
            "json",
            "missing configuration file raises ConfigError",
        )
        self.assertEqual(result.returncode, 0, result.stderr)
        data = json.loads(result.stdout)
        self.assertEqual(data["task"], "missing configuration file raises ConfigError")
        self.assertEqual(len(data["plan"]["steps"]), 7)
        commands = {
            check["command"] for check in data["checks"] if check["status"] == "verified"
        }
        self.assertIn("python -m unittest discover -s tests -v", commands)

    def test_plan_writes_html_report(self):
        target = self.repo.root / "brief.html"
        result = run_cli(
            "plan",
            "-C",
            str(self.repo.root),
            "--format",
            "html",
            "-o",
            str(target),
            "missing configuration file raises ConfigError",
        )
        self.assertEqual(result.returncode, 0, result.stderr)
        self.assertTrue(target.exists())
        document = target.read_text(encoding="utf-8")
        self.assertIn("<!doctype html>", document)
        self.assertIn("samplelib/config.py", document)

    def test_checks_json_separates_declared_commands(self):
        result = run_cli("checks", "-C", str(self.repo.root), "--json")
        self.assertEqual(result.returncode, 0, result.stderr)
        data = json.loads(result.stdout)
        verified = {check["command"] for check in data["verified"]}
        self.assertIn("python -m unittest discover -s tests -v", verified)
        for check in data["verified"]:
            self.assertTrue(check["sources"], check["command"])
        for check in data["suggested"]:
            self.assertTrue(check["basis"], check["command"])

    def test_run_requires_confirmation(self):
        result = run_cli("checks", "-C", str(self.repo.root), "--run")
        self.assertNotEqual(result.returncode, 0)
        self.assertIn("--yes", result.stderr)

    def test_missing_task_is_rejected(self):
        result = run_cli("map", "-C", str(self.repo.root))
        self.assertNotEqual(result.returncode, 0)
        self.assertIn("task", result.stderr.lower())

    def test_unknown_directory_is_rejected(self):
        result = run_cli("index", str(self.repo.root / "does-not-exist"))
        self.assertNotEqual(result.returncode, 0)
        self.assertIn("not a directory", result.stderr)


class CliCheckExecutionTests(unittest.TestCase):
    """Executed checks must fail the command, not just print FAIL."""

    def setUp(self):
        self.repo = RepoFixture("reposcout-fail-")
        self.addCleanup(self.repo.cleanup)
        self.repo.write(
            ".github/workflows/tests.yml",
            "name: tests\n"
            "on: [push, pull_request]\n"
            "jobs:\n"
            "  test:\n"
            "    runs-on: ubuntu-latest\n"
            "    steps:\n"
            "      - uses: actions/checkout@v4\n"
            "      - name: Unit tests\n"
            "        run: python -m unittest discover -s tests -v\n",
        )
        self.repo.write(
            "tests/test_regression.py",
            "import unittest\n\n\n"
            "class RegressionTests(unittest.TestCase):\n"
            "    def test_expected_failure(self):\n"
            "        self.assertEqual(1, 2)\n\n\n"
            'if __name__ == "__main__":\n'
            "    unittest.main()\n",
        )

    def test_failing_declared_check_exits_non_zero(self):
        result = run_cli("checks", "-C", str(self.repo.root), "--run", "--yes")
        self.assertEqual(result.returncode, 1, result.stdout + result.stderr)
        self.assertIn("[FAIL]", result.stdout)
        self.assertIn("python -m unittest discover -s tests -v", result.stdout)

    def test_failing_declared_check_is_fail_in_json(self):
        result = run_cli("checks", "-C", str(self.repo.root), "--run", "--yes", "--json")
        self.assertEqual(result.returncode, 1, result.stdout + result.stderr)
        data = json.loads(result.stdout)
        self.assertTrue(data["runs"])
        self.assertFalse(all(run["ok"] for run in data["runs"]))


if __name__ == "__main__":
    unittest.main()
```

## CONTRIBUTING.md

`CONTRIBUTING.md` - 49 lines

````md
# Contributing to RepoScout

Thanks for helping improve RepoScout. The project has no third-party runtime
dependencies, so setup is just a Python 3.11+ interpreter.

## Ground rules

- Keep the deterministic pipeline dependency-free.
- Never blur provenance. New claims, checks and actions must be `verified` with
  a real `Source`, or `suggested` with an explicit `basis`.
- Never execute a command unless the user passed `--run --yes`.
- Add a test for behavior changes. Prefer a focused regression test over a broad
  refactor.

## Development workflow

1. Fork the repository and create a topic branch.
2. Run the test suite before and after your change:

```bash
python -m unittest discover -s tests -v
```

3. If you changed retrieval or parsing, add a fixture repository under `tests/`
   and assert on ranking or extracted symbols, not just on "no exception".
4. Update `README.md` when user-facing behavior changes.

## Pull request checklist

- [ ] Tests cover the new behavior, or the PR explains why a test is impractical.
- [ ] The change is scoped to one behavior and contains no drive-by refactors.
- [ ] New user-facing strings avoid promising certainty that the code cannot prove.
- [ ] Verified output still cites a repository file and line.
- [ ] Suggested output still explains its basis.
- [ ] `python -m unittest discover -s tests -v` passes locally.
- [ ] The PR description lists the commands you ran and their results.

## Adding a language parser

Add a parser function in `reposcout/parsing.py`, register it in `parse_file`,
and make sure every emitted `Symbol` and `ImportRef` carries a real line number.
Heuristics are welcome; silently guessing line numbers is not.

## Reporting a mapping bug

Include the task text, the repository revision, the command you ran, and the
expected vs actual top-ranked files. If possible, attach the `--json` mapping
output; it contains the query terms and per-file reasons needed to reproduce a
ranking problem.
````

## README.md

`README.md` - 186 lines

````md
# RepoScout

RepoScout is a small, dependency-free Python agent that answers three questions
about an unfamiliar open-source repository:

1. **Which code does this task touch?** It ranks files with weighted BM25 over
   file text, symbols and paths, then adds one hop of the resolved import graph.
2. **What is the evidence?** Every ranked file carries reasons and, where
   possible, exact `path:line` snippets.
3. **How do I know a change is acceptable?** It separates commands **declared by
   the repository** from commands it merely **suggests**, and it never runs
   anything without `--yes`.

It also emits a beginner-sized contribution plan with a PR checklist, time
estimate and an explicit list of unknowns.

```
VERIFIED  = a file and line in this repository states it
SUGGESTED = RepoScout inferred it; confirm before acting
```

That distinction is enforced in code: `Claim`, `Check` and `Action` raise
`ValueError` when a verified item has no `Source` or a suggested item has no
`basis`.

## Install

RepoScout targets **Python 3.11+** and uses only the standard library.

```bash
git clone <your-fork>
cd reposcout
python -m reposcout --version
```

There is nothing to install. Add the repository root to `PYTHONPATH` or run the
commands from the checkout. A `pyproject.toml` is included if you prefer
`pip install -e .`.

## Quick start

```bash
# Build .reposcout/index.sqlite3
python -m reposcout index .

# Describe the project and its verification setup
python -m reposcout overview

# Map an issue to files, with evidence
python -m reposcout map "config loader crashes when the file is missing"

# Produce a Markdown brief, or a self-contained HTML report
python -m reposcout plan "add a timeout option to the HTTP client"
python -m reposcout plan "add a timeout option to the HTTP client" --format html
python -m reposcout plan "add a timeout option to the HTTP client" -o brief.html

# List declared vs suggested checks
python -m reposcout checks

# Execute declared+runnable checks only, after explicit confirmation
python -m reposcout checks --run --yes
```

Use `-C path` to point at another checkout, `--json` on `index`, `overview`,
`map` and `checks` for machine-readable output, and `--refresh` to rebuild the
index after editing.

`checks --run --yes` exits non-zero when any executed check fails (or times
out), so it can gate a script. Listing checks, or running them without
`--run`, always exits 0 unless there is an argument or repository error.

## What "verified" means here

| Item | Verified means | It does not mean |
| --- | --- | --- |
| File mapping | The file exists and the cited `path:line` produced the match | The file is the right place for the fix |
| Check | The command is written in a config, CI workflow, Makefile, package script or contributor doc | The command passed, or that its tools are installed |
| Read action | The file/line exists in the index | Reading it will explain the bug |
| Edit action | Never verified | A proposed change site is always a hypothesis |
| Checklist item | A contribution document contains that checkbox | The item applies to your specific PR |

Check discovery covers `pyproject.toml`, `setup.cfg`, `tox.ini`, `noxfile.py`,
`Makefile`, `package.json`, `Cargo.toml`, `go.mod`, `pom.xml`, Gradle,
`composer.json`, common CI providers, and fenced commands in contribution or
development docs. Example blocks and directory trees are ignored, and
ecosystem defaults (for example `cargo test` because a `Cargo.toml` exists) are
always `suggested`.

## Output

`plan` renders Markdown to stdout, JSON, or a **single-file HTML** report with
embedded CSS, light/dark support, evidence blocks, confidence meters, check
tables and the agent trace. HTML mode without `-o` writes to
`.reposcout/reports/<task-slug>.html`.

Pass `--run-checks --yes` to execute declared **and** runnable checks; the report
then includes a real execution table (PASS/FAIL, exit code, duration, truncated
output) alongside the declaration table. Without that flag, the report says
plainly that nothing was executed. The JSON shape includes `check_results` for
the same data.

The JSON schema is the `Report.to_dict()` shape in `reposcout/models.py`, so it
is stable enough to feed into another tool.

## How it works

```
discover ──► parse ──► SQLite index ──► BM25 + boosts ──► graph hop ──► evidence
                                                   │
                                                   ├──► check discovery (verified/suggested)
                                                   └──► contribution plan + report
```

- **Discovery** walks the tree, honours `.gitignore`, skips vendored/generated
  directories, lockfiles and credential files, and profiles manifests, docs,
  contribution guides, CI and tests.
- **Parsing** uses Python's `ast`; other languages use conservative line
  patterns with exact line numbers. Unsupported languages are tokenized but not
  symbol-parsed.
- **Indexing** stores files, symbols, imports, resolved edges and per-field term
  frequencies in SQLite. No server, no embeddings, no network.
- **Retrieval** combines BM25 (`body`, `symbol`, `path` fields), exact
  identifier/path boosts, one-hop import expansion, test association and
  phrase boosts.
- **Planning** turns the mapping and checks into seven steps and records each
  action's provenance.

### Optional language model

The deterministic pipeline is the default. `--llm` enables an OpenAI-compatible
chat endpoint; it can propose additional search terms and a short narrative.
Terms are used only if they already exist in the index, and narrative citations
must reference retrieved files, otherwise the text is discarded.

```bash
export REPOSCOUT_LLM_BASE_URL=https://api.openai.com/v1
export REPOSCOUT_LLM_API_KEY=...
export REPOSCOUT_LLM_MODEL=gpt-4o-mini
python -m reposcout plan "..." --llm
```

No key? RepoScout records a note and continues deterministically.

## Limitations

- Retrieval is lexical, not semantic. Paraphrased tasks can rank poorly; adding a
  filename, symbol, error type or error text helps a lot.
- Symbol extraction outside Python is heuristic. Dynamic imports, generated
  files and anything outside the index are invisible.
- The import graph is one hop and resolves common conventions only. It is
  evidence for investigation, not proof of a dependency.
- The plan is advice. It cannot know product intent, maintainer preferences or
  hidden acceptance criteria; those are listed under unknowns.
- Declared commands can be stale. Runnability is reported separately, and a
  command is never executed without `--run --yes`.

## Development

Run the test suite with the same command the project declares in CI:

```bash
python -m unittest discover -s tests -v
```

Layout:

```
reposcout/
  agent.py       orchestration and optional LLM grounding
  checks.py      command discovery, auditing, execution
  cli.py         argparse interface
  indexing.py    SQLite schema, import resolution, BM25
  models.py      provenance-enforcing dataclasses
  parsing.py     Python AST + heuristic language parsers
  planning.py    contribution plan and PR checklist
  rendering.py   Markdown, JSON, self-contained HTML
  retrieval.py   task-to-code mapping and evidence
  scanning.py    tree walk, ignore rules, repository profile
  tokens.py      tokenizing, stemming, synonyms, task signals
  util.py        small shared helpers
tests/
```

## License

MIT. See [LICENSE](LICENSE).
````

## LICENSE

`LICENSE` - 21 lines

```text
MIT License

Copyright (c) 2026 RepoScout contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

End of bundle.
