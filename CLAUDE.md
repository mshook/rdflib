# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RDFLib is a pure Python package for working with RDF (Resource Description Framework). It provides parsers, serializers, a Graph interface, Store implementations, and SPARQL 1.1 query/update support.

**Current version:** 8.0.0a0 (main branch is unstable, version 8 alpha)

**Python support:** 3.9+

## Development Commands

### Environment Setup

```bash
# Install dependencies with poetry (recommended)
poetry install

# Install with all extras (for extensive testing including Berkeley DB)
poetry install --all-extras

# Install in editable mode with pip
pip install -e .
```

### Testing

```bash
# Run all tests
poetry run pytest

# Run specific test file
poetry run pytest test/test_graph/test_graph.py

# Run tests with coverage
poetry run pytest --cov --cov-report term --cov-report html

# Run tests without internet (excluding webtest markers)
poetry run pytest -m "not webtest"

# Run SPARQLStore tests against public endpoints (skipped by default)
poetry run pytest --public-endpoints
poetry run pytest test/test_store/test_store_sparqlstore_public.py --public-endpoints

# View coverage report
python -m http.server --directory=htmlcov
```

### Linting and Formatting

```bash
# Format code with black (required before commits)
poetry run black .

# Check formatting without changes
poetry run black --check --diff .

# Run ruff linter
poetry run ruff check .

# Auto-fix ruff issues
poetry run ruff check --fix .

# Type checking with mypy
poetry run mypy --show-error-context --show-error-codes
```

### Task Runner (Taskfile)

RDFLib uses [Task](https://taskfile.dev) as a task runner. Common tasks:

```bash
# Run validation (lint + mypy + tests)
task validate

# Run tests
task test

# Lint code
task lint

# Fix auto-fixable linting errors
task lint:fix

# Build documentation
task docs

# Clean all build artifacts
task clean
```

### Documentation

```bash
# Build docs
poetry run mkdocs build

# Serve docs locally with live reload
poetry run mkdocs serve
```

### Pre-commit Hooks

```bash
# Install pre-commit hooks
pre-commit install

# Run pre-commit on all files
pre-commit run --all-files
```

## Architecture

### Core Components

- **`rdflib/graph.py`**: Defines `Graph`, `ConjunctiveGraph`, `QuotedGraph`, and `Dataset` classes. The `Graph` class is the primary interface for working with RDF triples.

- **`rdflib/term.py`**: Core RDF term types: `URIRef`, `BNode`, `Literal`, `Variable`, `IdentifiedNode`, and base `Node` class.

- **`rdflib/namespace/`**: Namespace definitions for common vocabularies (RDF, RDFS, OWL, FOAF, SKOS, etc.). Namespaces can be imported from `rdflib.namespace`.

- **`rdflib/store.py`**: Base `Store` interface that backends must implement.

- **`rdflib/plugin.py`**: Plugin system for registering parsers, serializers, stores, and query processors. Plugins can be registered via entry points or directly.

### Plugin Architecture

RDFLib uses a plugin system with the following plugin types:

- **Parsers** (`rdflib/plugins/parsers/`): RDF/XML, N3, Turtle, NTriples, N-Quads, TriG, TriX, JSON-LD, HexTuples
- **Serializers** (`rdflib/plugins/serializers/`): Same formats as parsers
- **Stores** (`rdflib/plugins/stores/`):
  - `memory.py`: In-memory triple store (default)
  - `berkeleydb.py`: Persistent storage using Berkeley DB
  - `sparqlstore.py`: Remote SPARQL endpoint interface
  - `auditable.py`: Store wrapper for auditing changes
  - `concurrent.py`: Thread-safe store wrapper
  - `regexmatching.py`: Regex-based matching store
- **Query Processors** (`rdflib/plugins/sparql/`): SPARQL 1.1 implementation
  - `parser.py`: SPARQL query parser
  - `algebra.py`: SPARQL algebra operations
  - `evaluate.py`: Query evaluation
  - `processor.py`: Query processor
  - `update.py`: SPARQL Update support

### SPARQL Implementation

The SPARQL engine (`rdflib/plugins/sparql/`) is a significant subsystem:
- **Parser** converts SPARQL queries to algebra representation
- **Algebra** module defines SPARQL operations (Join, Union, Filter, etc.)
- **Evaluate** executes algebra operations against graphs
- **Operators** implements SPARQL operators and functions
- **Aggregates** handles GROUP BY and aggregate functions

Note: SPARQL code has relaxed naming conventions (see `pyproject.toml` per-file-ignores for `rdflib/plugins/sparql/*`).

### Key Design Patterns

1. **Triple Pattern**: RDF data is stored as (subject, predicate, object) triples. Graphs are collections of triples.

2. **Plugin Discovery**: Parsers, serializers, and stores are discovered via:
   - Entry points in package metadata
   - Direct registration with `rdflib.plugin.register()`

3. **Store Abstraction**: The `Store` interface allows different backend implementations (in-memory, persistent, remote SPARQL).

4. **Namespace Management**: Namespaces bind prefixes to URIs for compact representation. Common namespaces are pre-defined.

5. **Literal Normalization**: `NORMALIZE_LITERALS` flag (default: True) controls whether literal lexical forms are normalized according to their datatype when created.

## Development Guidelines

### Code Style

- Follow PEP 8
- Format with Black (version 24.10.0, config in `pyproject.toml`)
- Line length: 88 characters
- Target Python version: 3.9+

### Testing Requirements

- New functionality **must** have unit tests
- Tests should demonstrate the feature works (removing the feature should break the test)
- Use pytest framework for new tests
- Add both simple tests and edge case tests
- Existing `unittest`-based tests work but should be converted to pytest style when touched
- Tests are in `test/` directory, mirroring the `rdflib/` structure

### Type Checking

- RDFLib uses type hints
- Run mypy to check types: `poetry run mypy`
- Configuration in `[tool.mypy]` section of `pyproject.toml`
- Type stubs for lxml are included via `lxml-stubs`

### Pull Request Guidelines

- For new features or API changes, open an issue first to discuss
- PRs require review and approval from at least two people
- Changes with runtime impact must have tests
- Update documentation if behavior changes
- CI checks (type checks, tests, linting) must pass
- Keep PRs small and focused for faster review

### Breaking Changes

- RDFLib follows semantic versioning and trunk-based development
- Breaking changes should be discussed in an issue first
- Changes are made incrementally to the main branch
- Major releases are triggered by breaking changes
- Deprecation warnings are preferred but not required before removal

### Special Considerations

- **Parser/Serializer Development**: Parsers in `rdflib/plugins/parsers/` must implement the `Parser` interface. Serializers implement `Serializer`.

- **Store Development**: Custom stores must inherit from `Store` and implement the required methods.

- **SPARQL Extensions**: SPARQL functions can be extended via the plugin mechanism. See existing operators in `rdflib/plugins/sparql/operators.py`.

- **Namespace Files**: Auto-generated namespace files in `rdflib/namespace/` have relaxed naming rules (see `pyproject.toml` per-file-ignores).

## Testing Data

- Test data is in `test/data/`
- W3C test suites are in `test/test_w3c_spec/`
- Fetch test data: `poetry run python test/data/fetcher.py`

## Docker

RDFLib provides Docker images for testing:

```bash
# Build unstable image
task docker:unstable

# Build latest stable image
task docker:latest
```

## Command-Line Tools

RDFLib installs several CLI tools (defined in `pyproject.toml` `[project.scripts]`):

- `rdfpipe`: Convert between RDF formats
- `csv2rdf`: Convert CSV to RDF
- `rdf2dot`: Convert RDF to GraphViz DOT
- `rdfs2dot`: Convert RDFS to GraphViz DOT
- `rdfgraphisomorphism`: Test graph isomorphism

Run via: `poetry run rdfpipe <args>` or install RDFLib and use directly.

## Documentation Structure

- Documentation source: `docs/`
- Built with MkDocs (Material theme)
- Configuration: `mkdocs.yml`
- API documentation generated from docstrings using mkdocstrings
- Published to ReadTheDocs: https://rdflib.readthedocs.io

## Common Patterns

### Creating and Loading a Graph

```python
from rdflib import Graph, URIRef, Literal, Namespace
from rdflib.namespace import RDF, RDFS, FOAF

g = Graph()
g.parse("data.ttl", format="turtle")  # or parse from URL
```

### Querying with SPARQL

```python
results = g.query("""
    SELECT ?s ?p ?o
    WHERE { ?s ?p ?o }
    LIMIT 10
""")
```

### Adding Triples

```python
from rdflib.namespace import FOAF, XSD

g.add((
    URIRef("http://example.com/person/alice"),
    FOAF.name,
    Literal("Alice", datatype=XSD.string)
))
```

### Serializing

```python
print(g.serialize(format="turtle"))
g.serialize(destination="output.ttl", format="turtle")
```

## Version Information

- Main branch: Version 8 alpha (unstable)
- Stable: Version 7.4.0
- Python requirement: 3.9+
- Dependency management: Poetry
