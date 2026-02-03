# GitHub Copilot Instructions Template

> **Purpose**: Use this template to generate a `copilot-instructions.md` file for any project.
> Place the generated file in `.github/copilot-instructions.md` in your repository.

---

## Instructions for AI: How to Use This Template

When asked to create Copilot instructions for a project, analyze the codebase and fill in each section below. Replace all `{{PLACEHOLDER}}` values with project-specific information.

---

# GitHub Copilot Instructions - {{PROJECT_NAME}}

## Project Description

{{Brief description of what the project does, its main purpose, and key features}}

---

## Tech Stack

- **Language(s)**: {{Primary programming languages}}
- **Framework(s)**: {{Main frameworks used}}
- **Package Manager**: {{npm, pip, cargo, etc.}}
- **Build Tools**: {{webpack, vite, setuptools, etc.}}

---

## Code Conventions

### Language and Style

- **Code language**: {{Language for code - typically English}}
- **Documentation language**: {{Language for docs and comments}}
- **Style Guide**: {{PEP 8, Airbnb, Google, etc.}}
- **Formatting**: {{Prettier, Black, rustfmt, etc.}}

### Naming Conventions

- **Variables**: {{camelCase, snake_case, etc.}}
- **Functions/Methods**: {{camelCase, snake_case, etc.}}
- **Classes**: {{PascalCase}}
- **Constants**: {{SCREAMING_SNAKE_CASE}}
- **Files**: {{kebab-case, snake_case, etc.}}

### Type System

- **Type Annotations**: {{Always use / Optional / Not applicable}}
- **Strict Mode**: {{Yes / No}}

---

## Project Structure

```
{{PROJECT_ROOT}}/
├── {{folder1}}/           # {{Description}}
│   ├── {{subfolder}}/     # {{Description}}
│   └── {{file}}           # {{Description}}
├── {{folder2}}/           # {{Description}}
├── {{config_files}}       # {{Description}}
└── {{other_files}}        # {{Description}}
```

---

## Design Patterns

### Architecture

{{Describe the architectural pattern: MVC, Clean Architecture, Microservices, etc.}}

### Key Patterns Used

- **{{Pattern 1}}**: {{Where and why it's used}}
- **{{Pattern 2}}**: {{Where and why it's used}}
- **{{Pattern 3}}**: {{Where and why it's used}}

### Base Classes / Interfaces

- `{{BaseClass1}}`: {{Purpose}}
- `{{Interface1}}`: {{Purpose}}

---

## Key Abstractions

### Core Classes

| Class/Module | Purpose |
| ------------ | ------- |
| `{{Class1}}` | {{Description}} |
| `{{Class2}}` | {{Description}} |
| `{{Class3}}` | {{Description}} |

### Important Functions

| Function | Location | Purpose |
| -------- | -------- | ------- |
| `{{func1}}` | `{{file}}` | {{Description}} |
| `{{func2}}` | `{{file}}` | {{Description}} |

---

## Configuration

### Environment Variables

| Variable | Purpose | Required |
| -------- | ------- | -------- |
| `{{VAR1}}` | {{Description}} | Yes/No |
| `{{VAR2}}` | {{Description}} | Yes/No |

### Configuration Files

- `{{config_file1}}`: {{Purpose}}
- `{{config_file2}}`: {{Purpose}}

---

## Security Guidelines

- **Secrets Management**: {{How secrets are handled}}
- **Authentication**: {{Auth method if applicable}}
- **NEVER**: {{List things to never do}}

---

## Error Handling

### Logging

{{Describe the logging approach and tools used}}

```{{language}}
// Example of proper logging
{{code_example}}
```

### Exception Handling

{{Describe how exceptions should be handled}}

---

## Testing

### Framework

- **Test Framework**: {{pytest, jest, mocha, etc.}}
- **Coverage Tool**: {{coverage, istanbul, etc.}}

### Conventions

- Test files location: `{{tests/}}`
- Naming pattern: `{{test_*.py, *.test.ts, etc.}}`
- Mocking strategy: {{Describe approach}}

---

## Dependencies

### Main Dependencies

| Package | Purpose |
| ------- | ------- |
| `{{package1}}` | {{Description}} |
| `{{package2}}` | {{Description}} |
| `{{package3}}` | {{Description}} |

### Dev Dependencies

| Package | Purpose |
| ------- | ------- |
| `{{dev_package1}}` | {{Description}} |
| `{{dev_package2}}` | {{Description}} |

---

## Best Practices for Copilot

1. **When creating new {{component_type}}**: {{Guidance}}
2. **When modifying {{area}}**: {{Guidance}}
3. **When adding tests**: {{Guidance}}
4. **When handling errors**: {{Guidance}}
5. **When writing documentation**: {{Guidance}}

---

## Common Patterns

### {{Pattern Name 1}}

```{{language}}
{{code_example_showing_pattern}}
```

### {{Pattern Name 2}}

```{{language}}
{{code_example_showing_pattern}}
```

---

## API Conventions (if applicable)

### Endpoints

- **Base URL**: `{{/api/v1}}`
- **Authentication**: {{Bearer token, API key, etc.}}
- **Response Format**: {{JSON, XML, etc.}}

### HTTP Methods

| Method | Usage |
| ------ | ----- |
| GET | {{Description}} |
| POST | {{Description}} |
| PUT/PATCH | {{Description}} |
| DELETE | {{Description}} |

---

## Git Workflow

- **Branch naming**: `{{feature/*, bugfix/*, etc.}}`
- **Commit format**: `{{Conventional Commits, etc.}}`
- **PR requirements**: {{Tests, reviews, etc.}}

---

## Author

**{{Author/Organization}}** - {{License}}

---

## Quick Reference Checklist

When generating code for this project, ensure:

- [ ] Code follows the style guide
- [ ] Type annotations are included (if applicable)
- [ ] Error handling follows project patterns
- [ ] Logging is implemented correctly
- [ ] Tests are included for new functionality
- [ ] Documentation is updated
- [ ] No secrets or sensitive data in code
- [ ] Dependencies are properly declared
