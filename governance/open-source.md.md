# Open Source Standards

> **Status:** Active
> **Applies to:** All public GitHub repositories
> **Last Updated:** March 2026

## Purpose

This standard establishes mandatory requirements for publishing and maintaining open source projects on GitHub. Every public repository MUST meet these requirements before being made public and MUST maintain them throughout the project's lifetime.

This standard uses RFC 2119 terminology: MUST, MUST NOT, SHOULD, SHOULD NOT, MAY.

---

## 1. Required Repository Files

Every public repository MUST include the following files at the repository root:

### 1.1 File Checklist

| File | Purpose | Required |
|------|---------|----------|
| `LICENSE` | Legal terms for use, modification, and distribution | MUST |
| `README.md` | Project overview, setup instructions, usage examples | MUST |
| `.gitignore` | Prevent committing sensitive or unnecessary files | MUST |
| `CONTRIBUTING.md` | How to report bugs, suggest features, and submit PRs | SHOULD |
| `CODE_OF_CONDUCT.md` | Community behaviour expectations | SHOULD |
| `SECURITY.md` | How to report security vulnerabilities | SHOULD |
| `CHANGELOG.md` | Version history and notable changes | MAY |

### 1.2 GitHub Community Health Files

GitHub recognises certain files placed in a `.github/` directory. These MAY be used as an alternative to root-level placement for `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, and `SECURITY.md`.

---

## 2. Licensing

### 2.1 License Selection

Every public repository MUST include a `LICENSE` file at the repository root. The chosen license MUST be an [OSI-approved license](https://opensource.org/licenses).

| License | When to Use |
|---------|-------------|
| **MIT** | Default choice. Simple, permissive, widely understood |
| **Apache 2.0** | When patent protection is needed |
| **GPL 3.0** | When copyleft (derivative works must remain open) is desired |
| **ISC** | Simplified MIT equivalent |

When no specific requirement dictates otherwise, prefer **MIT**.

### 2.2 License File Requirements

The `LICENSE` file MUST:

- Contain the full license text (not a summary or link)
- Include the correct copyright year(s)
- Include the correct copyright holder name
- Match the license identifier declared in `package.json`, `pyproject.toml`, or equivalent

### 2.3 License Header in Source Files

Source files MAY include a license header comment, but this is NOT required for MIT-licensed projects. If headers are used, they MUST match the root `LICENSE` file.

### 2.4 Third-Party License Compliance

Projects MUST NOT include dependencies with licenses that conflict with the project's own license. CI pipelines SHOULD include license compatibility checking for dependencies.

---

## 3. README Requirements

### 3.1 Mandatory Sections

Every `README.md` MUST include:

| Section | Content |
|---------|---------|
| **Title and description** | What the project does in 1-2 sentences |
| **Installation / Setup** | How to get it running locally |
| **Usage** | Basic usage examples with code snippets |
| **Requirements** | Runtime dependencies, language versions, OS support |
| **License** | License name with link to `LICENSE` file |

### 3.2 Recommended Sections

A `README.md` SHOULD also include:

| Section | Content |
|---------|---------|
| **Badges** | License, build status, version |
| **Disclaimer** | Limitations, liability, professional advice warnings |
| **Contributing** | How to contribute (or link to `CONTRIBUTING.md`) |
| **Architecture** | High-level overview for complex projects |
| **Acknowledgements** | Credits for inspiration, dependencies, or contributors |

### 3.3 Disclaimer Requirements

Projects that handle sensitive data or could be used in legal, medical, financial, or compliance contexts MUST include a disclaimer that:

- States the software is provided "as is" with no warranty
- Clarifies it is not a substitute for professional advice
- Notes the user's responsibility for legal compliance
- Warns that AI-generated outputs (if applicable) may contain errors

---

## 4. Security Policy

### 4.1 SECURITY.md

Public repositories SHOULD include a `SECURITY.md` file that describes:

- How to report security vulnerabilities (email, not public issue)
- Expected response timeline
- Supported versions receiving security updates
- Scope of the security policy

### 4.2 Template

```markdown
# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability, please report it responsibly.

**Do NOT open a public GitHub issue for security vulnerabilities.**

Instead, email: [security contact email]

### What to include

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

### Response timeline

- **Acknowledgement**: Within 48 hours
- **Initial assessment**: Within 1 week
- **Fix or mitigation**: Depends on severity

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest  | Yes       |

## Scope

This policy applies to the latest release of this project.
Vulnerabilities in dependencies should be reported to the
upstream maintainer.
```

---

## 5. Sensitive Data Protection

### 5.1 Pre-Publication Audit

Before making any repository public, the following audit MUST be completed:

- [ ] No API keys, tokens, or secrets in source code or commit history
- [ ] No passwords or credentials in configuration files
- [ ] No private file paths or internal hostnames
- [ ] No PII (names, emails, addresses) in test data or examples
- [ ] No proprietary business logic that should remain private
- [ ] `.gitignore` covers all sensitive file patterns
- [ ] Git history does not contain previously committed secrets

### 5.2 Gitignore Requirements

Every repository MUST have a `.gitignore` that excludes at minimum:

```gitignore
# Environment and secrets
.env
.env.*
*.key
*.pem
config.yaml        # If contains local paths or credentials

# Data and databases
*.db
*.sqlite
data/

# IDE and OS
.idea/
.vscode/
.DS_Store
Thumbs.db

# Dependencies
node_modules/
.venv/
__pycache__/

# Build output
dist/
build/
*.egg-info/
```

### 5.3 Secret Scanning

Repositories SHOULD enable GitHub's secret scanning feature. If secrets are found in commit history, the repository MUST NOT be made public until history is cleaned using `git filter-repo` or equivalent.

---

## 6. Configuration and Examples

### 6.1 Example Configuration

If the project requires a configuration file that is gitignored (e.g. `config.yaml`), the repository MUST include a corresponding example file (e.g. `config.example.yaml`) that:

- Contains placeholder values, not real data
- Documents all available configuration options
- Uses clearly fake paths and credentials (e.g. `/path/to/your/data`, `your-api-key-here`)
- Is referenced in the README setup instructions

### 6.2 Example Data

If the project processes data files, the repository MAY include small example/sample files for testing. These MUST NOT contain real PII or sensitive content.

---

## 7. Versioning and Releases

### 7.1 Semantic Versioning

Projects SHOULD follow [Semantic Versioning](https://semver.org/):

- **MAJOR**: Breaking changes to public API or CLI
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, backward compatible

### 7.2 GitHub Releases

Projects with users beyond the author SHOULD create GitHub Releases with:

- A tag matching the version (e.g. `v1.2.0`)
- Release notes summarising changes
- Migration notes for breaking changes

### 7.3 Changelog

Projects MAY maintain a `CHANGELOG.md` following [Keep a Changelog](https://keepachangelog.com/) format. If no changelog is maintained, GitHub Releases serve as the version history.

---

## 8. CI/CD for Open Source

### 8.1 GitHub Actions

Public repositories SHOULD include CI workflows for:

| Workflow | Purpose |
|----------|---------|
| **Lint** | Code style and formatting checks |
| **Test** | Unit and integration tests |
| **Build** | Verify the project builds successfully |
| **Security** | Dependency vulnerability scanning |

### 8.2 Branch Protection

The default branch SHOULD have branch protection rules:

- Require PR reviews before merging
- Require status checks to pass
- Prevent force pushes

### 8.3 Dependency Updates

Projects SHOULD enable automated dependency updates via Dependabot or Renovate. Security updates SHOULD be applied promptly.

---

## 9. Community and Contributions

### 9.1 Issue Templates

Repositories SHOULD include issue templates in `.github/ISSUE_TEMPLATE/`:

- **Bug report**: Steps to reproduce, expected vs actual behaviour, environment
- **Feature request**: Problem statement, proposed solution, alternatives

### 9.2 Pull Request Template

Repositories SHOULD include a `.github/pull_request_template.md` with:

- Summary of changes
- Type of change (bug fix, feature, breaking change)
- Testing checklist
- Documentation update confirmation

### 9.3 Code of Conduct

Projects that accept community contributions SHOULD adopt a Code of Conduct. The [Contributor Covenant](https://www.contributor-covenant.org/) is the recommended default.

---

## 10. Archiving and End of Life

### 10.1 Unmaintained Projects

If a project is no longer actively maintained:

- The repository SHOULD be archived via GitHub's archive feature
- The README MUST be updated with an unmaintained notice at the top
- Open issues and PRs SHOULD be closed with an explanation

### 10.2 Archive Notice Template

```markdown
> **This project is no longer actively maintained.**
> It remains available as-is under the existing license.
> No further updates, bug fixes, or security patches will be provided.
```

---

## 11. Pre-Publication Checklist

Before making a repository public, verify all of the following:

- [ ] `LICENSE` file present with correct copyright holder and year
- [ ] `README.md` has all mandatory sections (title, setup, usage, requirements, license)
- [ ] `.gitignore` excludes secrets, data, and environment files
- [ ] No secrets, API keys, or credentials in code or git history
- [ ] No PII in test data, examples, or commit messages
- [ ] Example configuration file provided if config is gitignored
- [ ] Disclaimer present if project handles sensitive data or AI outputs
- [ ] `SECURITY.md` present if project accepts external contributions
- [ ] License identifier matches in `LICENSE` file and package manifest
- [ ] All dependencies have compatible licenses

---

## References

- [OSI Approved Licenses](https://opensource.org/licenses)
- [Choose a License](https://choosealicense.com/)
- [GitHub Community Health Files](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions)
- [Contributor Covenant](https://www.contributor-covenant.org/)
- [Keep a Changelog](https://keepachangelog.com/)
- [Semantic Versioning](https://semver.org/)
- [GitHub Secret Scanning](https://docs.github.com/en/code-security/secret-scanning)
