# Contributing to the Unofficial GPUI Cheat Book

Thank you for your interest in contributing! This is a community-maintained resource, and your contributions help everyone learning GPUI.

## Quick Start

1. Find a section marked with **TODO**
2. Write concise, practical content with code examples
3. Submit a pull request

## Content Guidelines

### Writing Style

- **Concise over comprehensive** - Get to the point quickly
- **Code-first** - Show working examples, then explain
- **Practical over theoretical** - Focus on "how to do X"
- **Avoid overlap** - Don't duplicate what's already on gpui.rs

### Code Examples

- Should be complete and runnable where possible
- Include relevant imports
- Show real-world patterns, not just toy examples
- Reference Zed source code when applicable

### Page Structure

A typical page should include:

````markdown
# Title

Brief introduction (1-2 sentences)

## Example

\```rust
// Working code example
\```

## Explanation

Explain what the code does and why

## Common Patterns

Show variations and typical use cases

## Pitfalls

What NOT to do, common mistakes
````

### Finding Examples

The best examples come from:

1. **Zed repository** - `~/git/zed/crates/gpui/examples/`
2. **Zed source code** - Real-world patterns in the editor
3. **GPUI tests** - `~/git/zed/crates/gpui/tests/`
4. **Your own experience** - Patterns you've discovered

## Priority Areas

If you're not sure where to start, these sections are high priority:

1. Fundamentals (entities, contexts, views)
2. State Management patterns
3. UI Components (buttons, inputs)
4. Cookbook examples (counter, todo list)
5. Common Pitfalls

## Questions?

Feel free to open an issue to discuss:

- Proposed new sections
- Unclear existing content
- Structure suggestions

Thank you for helping make GPUI more accessible! 🎉
