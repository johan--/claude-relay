# Code Concerns

_Automatically detected code quality issues and technical debt._

## Summary

- **Total Symbols**: 390
- **Total Files**: 27
- **Duplicate Methods**: 0
- **Large Files**: 5
- **God Classes**: 0
- **High-Impact Files**: 2

## Large Files

_Files with high symbol density - may need refactoring._

| File | Symbols | Risk |
|------|---------|------|
| `lib/public/app.js` | 52 | Critical |
| `lib/public/modules/tools.js` | 52 | Critical |
| `bin/cli.js` | 42 | High |
| `lib/public/modules/sidebar.js` | 25 | Medium |
| `lib/public/modules/terminal.js` | 21 | Medium |

## High-Impact Files

_Files imported by many others - changes here have wide effects._

| File | Imported By | Impact |
|------|-------------|--------|
| `lib/public/modules/icons.js` | 9 files | Medium |
| `lib/public/modules/utils.js` | 8 files | Medium |

## Copy-Paste Patterns

_Method prefixes suggesting templated or duplicated code._

| Prefix | Unique Methods | Total Occurrences |
|--------|---------------|-------------------|
| `handle*` | 19 | 19 |
| `get*` | 19 | 19 |
| `update*` | 15 | 15 |
| `create*` | 13 | 13 |
| `init*` | 10 | 10 |
| `set*` | 10 | 10 |
| `load*` | 5 | 5 |
