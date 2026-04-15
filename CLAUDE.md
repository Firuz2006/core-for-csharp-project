# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Plugin/template system for Claude Code — defines standard structure, tooling, and conventions that every new C# project should follow. Not a runnable application itself.

## Language

- Repo language: Russian + English technical terms
- Comments in code: English
- Docs (`agent_docs/`, this file): Russian allowed

## Structure

This repo contains templates, rules, and configuration that get applied when scaffolding new C# projects. Core artifacts:

- Project templates and conventions for .NET 10 / C# 13+
- Standard `agent_docs/` structure (product.md, architecture.md, constraints.md, decisions.md)
- Shared analyzer configs, EditorConfig, Directory.Build.props patterns

## C# Project Conventions (enforced by this plugin)

- **Target**: .NET 10, nullable reference types enabled, warnings as errors
- **Style**: records for data, classes for services, minimal API over controllers
- **No `var`** where type isn't obvious from the right side
- **Testing**: xUnit + FluentAssertions, test project mirrors src structure
- **Analyzers**: Roslynator, StyleCop (subset), nullable warnings = errors
- **Structure**: `src/`, `tests/`, `agent_docs/` at root of every generated project
