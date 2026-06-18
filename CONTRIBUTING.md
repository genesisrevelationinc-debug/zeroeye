# Contributing to ZeroEye

Thank you for your interest in contributing! This guide covers how to set up a local development environment, build the project, and submit a pull request.

## Prerequisites

- **Python** 3.x (for the build system)
- **Git**
- Module-specific toolchains as needed (see [README.md](README.md) for details)

Required tools by module:

| Module | Tools |
|--------|-------|
| `backend` | Rust (cargo) |
| `frontend` | Node.js 22.x, npm |
| `market` | Go |
| `frailbox` | C toolchain (gcc, make) |
| `engine` | C++ toolchain (g++, cmake ≥3.28) |
| `compliance` | OpenJDK 21 |
| `v2` | Ruby, Redis |
| `scans` | Lua 5.4, luarocks |
| `openapi` | GHC, cabal |

## Local Setup

1. **Fork** the repository on GitHub, then clone your fork:

