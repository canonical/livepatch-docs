# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.0] - 2026-06-18

### Added

- Added introductory context and visible `{toctree}` navigation blocks (`:titlesonly:`) across index pages.
- Added a redirect for the live patching architecture document.

### Changed

- Refactored documentation content for consistency, clarity, and alignment with the documentation gold standard.
- Standardized heading style to declarative noun-phrase headings and enforced sentence case.
- Replaced manual index link lists with structured toctree-based navigation and removed hidden toctrees.
- Restructured release notes pages by replacing collapsible sections with plain headings and simplifying landing-page navigation.
- Revised `README.md` and the documentation landing page to follow the Diátaxis model and present Livepatch with product-focused context.
- Improved server explanation and support documentation structure, including clearer diagnostic guidance.

### Fixed

- Corrected capitalization, proper nouns, hyphenation, and redirect target typos.
- Replaced non-compliant style patterns, including em dashes, contractions, latinisms, weak phrasing, and non-descriptive link text.
- Enforced Livepatch terminology rules (Livepatch as a proper noun, never a verb) and corrected all `on-premise` usage to `on-premises`.
- Normalized formatting for commands, file paths, and configuration values using inline code where appropriate.
- Preserved MyST frontmatter, Sphinx directives, and code blocks while applying structural and style cleanups.
- Added blank lines after MyST anchors, fixed heading levels in patch management docs, and updated documentation URLs and the custom wordlist.
- Excluded `_dev/*` from docs processing.
- Improved code highlighting contrast to resolve pa11y accessibility findings.

## [0.3.0] - 2026-06-15

### Added

- Introduced top-level documentation sections for:
    - `client`
    - `server`
    - `support`
    - `release-notes`
    - `contribute`
- Added release notes pages for client and server documentation.
- Added contribution and AI usage policy guidance in `docs/contribute/index.md`.

### Changed

- Refactored the documentation structure from legacy `livepatch*` paths to `client/` and `server/` spaces.
- Introduced a third-level Diátaxis-style organization in key areas (for example: `architecture`, `security`, `troubleshooting`, `configuration`, `installation`, `operations`, and `patch-management`).
- Standardized MyST heading IDs to the new path-based naming convention and updated in-page/cross-page anchor usages accordingly.
- Normalized toctree entries to explicit titled form (for example: `Installation <installation/index.md>`).

### Fixed

- Expanded `docs/redirects.txt` to preserve old URLs across the restructure, including legacy `livepatch/`, `livepatch_on_prem/`, and `livepatch_server_on_public_clouds/` paths.
- Added missing redirects from moved client explanation pages to `support/` pages.
- Removed shell prompt prefixes (`$`) from fenced command examples for consistency and copy/paste usability.

## [0.2.0] - 2026-06-11

### Added

- Added skills to the repository under `.github/skills/`.

### Changed

- Standardized terminology and naming:
    - `PostgreSQL` preferred in prose (instead of mixed `Postgres` usage).
    - `airgapped` preferred in prose and tutorial naming.
    - `bug tracker` preferred (instead of `bugtracker`).
    - `Snap Store` preferred (instead of `Snapstore`).
    - `patch store` wording clarified where applicable.
- Renamed tutorial files to match standard naming
- Updated links, toctree entries, and anchors to use renamed files.
- Normalized heading structure across affected markdown files (single H1 and valid heading hierarchy).

## [0.1.0] - 2026-06-10

### Added

- Initial commit for the Livepatch documentation set in this repository.
- Adopted Sphinx documentation structure (`docs/`, toctrees, Sphinx build/check tooling).
- Imported and organized baseline documentation content from Discourse.
- Structured content by Diataxis documentation types (tutorial, how-to, reference, explanation).

### Fixed

- Corrected spelling/typing issues
- Fixed multiple MyST cross-reference targets and stale/empty links in docs.
- Updated `docs/.custom_wordlist.txt`.
