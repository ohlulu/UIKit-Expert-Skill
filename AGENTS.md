# Agent Guidelines for UIKit Expert Skill

This document provides guidance for AI agents working with this skill to ensure consistency and avoid common pitfalls.

## Core Principles

### 1. UIKit Focus Only
**This is a UIKit skill.** Do not include:
- SwiftUI patterns (except when bridging is necessary)
- General Swift language features unrelated to UIKit
- Backend or server-side Swift patterns
- Swift concurrency deep dives (use UIKit-specific async patterns only)

### 2. No Architectural Opinions
**Stick to facts, not architectures.** Avoid:
- Enforcing MVVM, MVC, VIPER, or any specific architecture
- Mandating coordinator patterns
- Requiring specific folder structures
- Dictating dependency injection patterns

**Exception**: Suggest separating business logic for testability without enforcing how.

### 3. No Tool-Specific Instructions
**Agents cannot use external tools.** Do not include:
- Xcode Instruments profiling instructions
- Debugging tool usage
- IDE-specific features

## Content Guidelines

### Suggestions vs Requirements

**Use "suggest" or "consider" for optional optimizations:**
- ✅ "Consider prefetching cells for smoother scrolling"
- ❌ "Always prefetch cells"

**Use "always" or "never" only for correctness issues:**
- ✅ "Never force-unwrap IBOutlets in init"
- ✅ "Always call super in lifecycle methods"

## What to Include

### ✅ Include These Topics:
- View controller lifecycle
- Auto Layout (programmatic and Interface Builder)
- UITableView / UICollectionView patterns
- Diffable data sources and compositional layout
- Navigation patterns (push, present, custom transitions)
- Animation and transition APIs
- Performance patterns (cell reuse, image handling, off-screen rendering)
- Accessibility best practices
- Modern UIKit APIs and deprecations

### ❌ Exclude These Topics:
- Swift concurrency deep dives
- Architectural patterns and mandates
- Tool usage instructions
- Testing frameworks and patterns
- Build system configuration
- SwiftUI (except bridging)

## Updating the Skill

When adding new content:
1. Ask: "Is this UIKit-specific?"
2. Ask: "Is this a fact or an opinion?"
3. Ask: "Can agents actually use this?"
4. Ask: "Is this about correctness or style?"

If unsure, err on the side of excluding content.

## Summary

**Focus**: UIKit APIs, patterns, and correctness
**Avoid**: Architecture, tools, Swift language features
**Tone**: Factual, helpful, non-prescriptive
**Goal**: Make agents UIKit experts without enforcing opinions
