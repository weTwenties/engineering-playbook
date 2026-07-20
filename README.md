# weTwenties Engineering Playbook

> Practical engineering decisions, patterns, and lessons from building real products.

## About

The **weTwenties Engineering Playbook** is a public knowledge base for documenting how we design, build, and evolve software under real product and delivery constraints.

It focuses on the reasoning behind technical decisions—not only the final implementation. Each article aims to capture the original context, constraints, architecture, trade-offs, failure modes, and reusable lessons that can be applied to future projects.

This repository is intended to grow into a shared engineering asset for the weTwenties team, collaborators, and developers interested in practical product engineering.

## Scope

- Frontend architecture and runtime UI
- Backend and system design
- CMS-driven products and content pipelines
- Reusable components, builders, and internal platforms
- Storage, publishing, and delivery workflows
- Testing, automation, and observability
- AI-assisted engineering with explicit guardrails
- Case studies from outsourcing and product development

## Principles

- Context matters more than universal best practices.
- Configuration should not become an uncontrolled programming language.
- Reuse should reduce change cost, not create premature abstraction.
- Faster delivery still requires validation, versioning, rollback, and observability.
- Good architecture should make similar future changes cheaper and safer.

## Articles
- [#001 — JSON Runtime UI trong dự án Mini App: Giảm những lần release không cần thiết"](./articles/001-json-runtime-ui/README.md)
- [#002 — Component Registry trong Runtime UI: Thiết kế một contract ổn định giữa JSON và source code"](./articles/002-component-registry/README.md)

- [#001 — JSON Runtime UI trong dự án Mini App: Giảm những lần release không cần thiết](./articles/001-json-runtime-ui/README.md)
- [#002 — Component Registry trong Runtime UI: Thiết kế một contract ổn định giữa JSON và source code](./articles/002-component-registry/README.md)

## Related

For shorter personal notes and the thinking behind these decisions, see [Build Less by Khanh Vinh Nguyen](https://github.com/khanhvinhnguyen/build-less).

---

Maintained by **weTwenties**.

## License

Unless otherwise noted:

- Written content and diagrams are licensed under
  [Creative Commons Attribution 4.0 International](LICENSE).
- Code snippets and example implementations are licensed under the
  [MIT License](LICENSE-CODE).

The weTwenties name, branding, and trademarks are not included in these licenses.
