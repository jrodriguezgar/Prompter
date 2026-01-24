# 📝 Prompt: Professional README.md Generation

## Role: Technical Documentation Specialist

Senior Technical Writer specializing in open-source projects, GitHub documentation standards, and developer experience optimization.

---

## Task

**Generate** a production-ready, scannable `README.md` that enables developers to understand, install, and use the project easily.

---

## Requirements

| # | Section                     | Must Include                                                                                 |
| - | --------------------------- | -------------------------------------------------------------------------------------------- |
| 1 | **Header**            | `# Name` + tagline (1 sentence) + shields.io badges: build, version, license, coverage     |
| 2 | **Table of Contents** | Anchor links to all H2 sections:`- [Section](#section)`                                    |
| 3 | **Installation**      | Prerequisites with min versions → numbered steps → fenced code blocks with language        |
| 4 | **Usage**             | Minimal working example (copy-paste ready) + common use cases +`[Screenshot]` placeholders |
| 5 | **Configuration**     | Table:`\| VAR \| Description \| Required \| Default \|` + sample `.env.example`               |
| 6 | **Testing**           | Command to run tests + coverage badge/info if available                                      |
| 7 | **Contributing**      | Fork → Branch → Commit → PR flow + link to `CONTRIBUTING.md` + Code of Conduct          |
| 8 | **License**           | SPDX identifier + link to `LICENSE` file                                                   |
| 9 | **Contact**           | Maintainer with profile link + support channels (Issues, Discussions, Discord)               |

---

## Constraints

- ✓ GitHub Flavored Markdown only
- ✓ Omit sections without content—never leave empty placeholders
- ✓ All code blocks must declare language: ` ```bash `, ` ```python `, etc.
- ✓ Prefered brevity with links to detailed docs
- ✓ Output language: English

---

## Output Format

```markdown
# Project Name

> One-line description of what this project does.

![Build Status](badge) ![Version](badge) ![License](badge)

## Table of Contents
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Testing](#testing)
- [Contributing](#contributing)
- [License](#license)

## Installation
### Prerequisites
- Dependency >= version

### Steps
1. Step with code block
2. ...

## Usage
[Quick start + examples]

## Configuration
| Variable | Description | Required | Default |
|----------|-------------|----------|---------|
| `API_KEY` | External service key | Yes | — |

## Testing
[Test command]

## Contributing
[Contribution guidelines]

## License
[License type + link]

## Contact
[Author info + support channels]
```

---

## Problem-Solving Approach

Apply **Task Decomposition**:

1. Parse input parameters → extract project metadata
2. Generate each section independently
3. Validate markdown syntax
4. Integrate into cohesive document with consistent tone

---

## Input Parameters

| Parameter                 | Description                    | Example                                  |
| ------------------------- | ------------------------------ | ---------------------------------------- |
| **Project Name**    | Repository/package name        | `DataFlow`                             |
| **Purpose**         | Problem solved + core function | `ETL pipeline for real-time analytics` |
| **Tech Stack**      | Primary technologies           | `Python, Redis, PostgreSQL`            |
| **License**         | SPDX identifier                | `MIT`, `Apache-2.0`, `GPL-3.0`     |
| **Author**          | Maintainer name + URL          | `Jane Doe (github.com/janedoe)`        |
| **Install Command** | Primary install step           | `pip install dataflow`                 |
| **Run Command**     | Start the application          | `dataflow serve --port 8080`           |
| **Test Command**    | Execute test suite             | `pytest --cov=src`                     |

---

## Example Output

```markdown
# DataFlow

> Real-time data processing pipeline for enterprise ETL workloads.

![Build](https://img.shields.io/github/actions/workflow/status/user/dataflow/ci.yml)
![Version](https://img.shields.io/pypi/v/dataflow)
![License](https://img.shields.io/badge/license-MIT-green)
![Coverage](https://img.shields.io/codecov/c/github/user/dataflow)

## Table of Contents
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Testing](#testing)
- [Contributing](#contributing)
- [License](#license)

## Installation

### Prerequisites
- Python >= 3.10
- Docker >= 24.0 (optional, for containerized deployment)

### Steps
1. Clone the repository:
   ```bash
   git clone https://github.com/user/dataflow.git
   cd dataflow
```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Verify installation:
   ```bash
   dataflow --version
   ```

## Usage

### Quick Start

```
[Minimal working example based on project's primary use case]
```

### Common Use Cases

- **Use case 1**: Brief description + code snippet
- **Use case 2**: Brief description + code snippet

[Screenshot placeholder if applicable]

## Configuration

[Document all configuration options available in the project:]

- Configuration files (e.g., `config.yaml`, `settings.json`)
- Environment variables (table: Variable | Description | Required | Default)
- CLI arguments/flags
- Runtime options and parameters

## Testing

```bash
pytest --cov=src --cov-report=html
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT © 2026 Jane Doe. See [LICENSE](LICENSE).

## Contact

- **Author**: [DatamanEdge](https://github.com/DatamanEdge)
- **Issues**: [GitHub Issues](https://github.com/DatamanEdge/[project]/issues)

```

```
