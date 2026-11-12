# URL Shortener

> Source: *System Design Interview*, Vol. 1 — "Design a URL Shortener"

## Summary

A service that turns long URLs into short codes and redirects anyone who opens a short URL to the original address. Traffic is read-heavy: links are created once and opened many times.

This case explores how to generate unique short codes at scale, how to keep redirects fast with caching, and how to store and partition a large, ever-growing set of mappings.

## Design documents

- [Functional requirements](design/01-functional-requirements.md)
- [Non-functional requirements](design/02-non-functional-requirements.md)

## How to run

> 🚧 To be added once the implementation starts.
