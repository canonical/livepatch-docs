# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
