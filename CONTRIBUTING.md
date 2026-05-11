# Contributing

Thanks for helping grow the cheatsheet. The goal is **accurate, skimmable entries** that still include enough context and **small examples** to be useful in interviews and day-to-day work.

## Entry shape

Each topic file groups related terms. Every term uses:

```markdown
## Term name

One short paragraph: what it is and why it matters.

Optional bullets for nuances, pitfalls, or relations to other terms.

### Example

Short snippet (Python, SQL, shell, TypeScript, Redis CLI, etc.)—only if it clarifies usage, not full apps.

### See also

Links to other headings in this repo: [JWT](./03-auth-and-security.md#jwt-json-web-token)
```

Use `##` for the term (anchors for the README index). Use `###` for subsections inside a term.

## Style

- Prefer **precise** over buzzwordy; call out overloaded words (e.g. “consistency” in CAP vs ACID).
- Keep examples **minimal** (roughly 5–20 lines); no fictional brand names in security examples.
- When behavior differs by language or product, say so (“In CPython…”, “In Postgres…”).
- Cross-link to other entries in this repo instead of duplicating long explanations.

## When to add a new file vs extend an existing one

- **Extend** the closest domain file (e.g. OIDC lives next to OAuth in `03-auth-and-security.md`).
- **Add a new numbered file** only when a domain is large and mostly independent (e.g. splitting “Security” from “Auth” later). Renumber files in one PR and update `README.md` links.

## README index

When you add a term with a new `##` heading, add a link in `README.md` under **Alphabetical index** (GitHub slug: lowercase, spaces → `-`, drop most punctuation—verify in preview).

## License

By contributing, you agree your contributions are under the same license as this repository ([MIT](./LICENSE)).
