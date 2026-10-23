# Agent Tax Filing Handoff

An open, vendor-neutral draft for how an AI agent hands a prepared individual tax return to an e-file
provider, while the taxpayer reviews, consents and signs personally on the provider's own page.

- [SPEC.md](SPEC.md): the draft (version 0.1)
- [schema/tax-filing.schema.json](schema/tax-filing.schema.json): JSON Schema for `/.well-known/tax-filing.json`
- [examples/terminal-tax.json](examples/terminal-tax.json): the document the reference implementation serves at
  https://terminal.tax/.well-known/tax-filing.json (from its first production release after 2026-10-23)

In one paragraph: the agent calls `start` (no tax data), reads the provider's packet `schema`, and calls
`submit` with the completed packet. It gets back either correction guidance for the taxpayer or one opaque,
single-use link. Social Security numbers, PINs and one-time codes never pass through the agent; the
taxpayer types them, reviews the complete return and signs on the provider's page. The result goes to the
taxpayer, not to the agent.

Comments, issues and other implementations are welcome. The reference implementation is
[Terminal Tax](https://terminal.tax). Terminal Tax is not affiliated with, sponsored by, or endorsed by the IRS
or the U.S. Treasury.

Text is licensed CC BY 4.0; the schema and examples are MIT. See [LICENSE](LICENSE).
