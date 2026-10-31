# Agent Tax Filing Handoff (ATFH)

Version: 0.1 (draft)
Date: 2026-10-23
Status: Informational draft, open for comment. Not a standard of any government, standards body or the
Model Context Protocol project.
Editor: Terminal Tax LLC
License: text CC BY 4.0; schema and examples MIT (see LICENSE)

## Abstract

People already prepare their individual income tax returns with general-purpose AI agents. This document
describes how such an agent hands a completed return to an e-file provider without becoming the filer:
the agent submits a structured packet, the provider validates it mechanically and returns either
correction guidance or one opaque link, and the taxpayer personally reviews, consents and signs on the
provider's own page. It also defines a discovery document, `/.well-known/tax-filing.json`, so an agent can
find a provider and its handoff endpoint without prior configuration.

The document is vendor-neutral. Terminal Tax (https://terminal.tax) is its first implementation.

## 1. Conventions

The key words MUST, MUST NOT, REQUIRED, SHOULD, SHOULD NOT, RECOMMENDED, MAY and OPTIONAL are to be
interpreted as described in BCP 14 (RFC 2119, RFC 8174) when, and only when, they appear in all capitals.

## 2. Roles

- **Taxpayer**: the person whose return it is, and every person who must sign it.
- **Agent**: the AI assistant the taxpayer is already working with. It gathers facts from the taxpayer
  and the documents the taxpayer chose to share, and it prepares the packet.
- **Provider**: an e-file provider authorized by the taxing authority to transmit the return. For the
  United States federal return, that is an Authorized IRS e-file Provider transmitting through Modernized
  e-File (MeF).
- **Filing page**: a page the provider serves to the taxpayer's own browser. It is the only place the
  taxpayer identifies themself, consents, signs and (where the provider charges) enters payment details.

## 3. Requirements

These requirements are the point of the document. An implementation that meets the message formats in
section 5 but not this section does not implement ATFH.

**R1. The agent prepares; the provider does not infer.** The provider MUST NOT infer, default or choose a
tax fact, election or treatment. A required fact that is absent is returned to the agent as a blocker
(section 5.4), never filled in. The provider MAY perform deterministic calculation from supplied facts and
mechanical validation (type, range, format, consistency, the taxing authority's published rules). An
unanswered question is unknown, not "no", "none", false or 0.

**R2. Protected values never come from the agent.** The provider MUST refuse a packet that carries a
taxpayer identification number (SSN or ITIN), an IRS Identity Protection PIN, a signature PIN, a prior-year
PIN, or a one-time code. The taxpayer types these on the filing page. A provider MAY accept other identity
details (name, date of birth, address, contact details, bank account for a refund or payment) from the agent
in a section kept apart from the return, and MUST show each to the taxpayer to confirm or correct.

**R3. The taxpayer signs personally, on the provider's page.** The signature is made by the taxpayer, with
their own keystrokes, after the complete return is available to review (for the US federal return, the
Self-Select PIN method in IRS Publication 1345). The agent MUST NOT supply, generate or relay a signature
value, and the provider's agent-facing interface MUST NOT accept one.

**R4. Notice and consent happen on the provider's page.** The provider's privacy notice, and every consent or
permission the provider takes, MUST be presented and given on the filing page; the provider MUST NOT accept one
relayed by the agent. Where the law requires a consent (in the United States, a consent under 26 U.S.C. 7216
and 26 CFR 301.7216-3 is needed only for a use or disclosure no exception in 26 CFR 301.7216-2 covers), it is
taken there in the form the law prescribes (Rev. Proc. 2013-14).

**R5. One link, only after a complete packet.** The provider MUST NOT issue a filing link until it holds a
packet that passed complete mechanical admission. The link is an opaque, single-use, short-lived bearer
capability bound to that one packet. It is not a login and MUST NOT become a durable account credential.
The provider SHOULD require a separately delivered possession factor (for example, a one-time code sent to
the taxpayer's own email or phone) before any identity or return content renders.

**R6. A changed packet is a new link.** Once admitted, the packet is bound to its link by hash and cannot be
edited. Before the taxpayer signs, a provider MAY accept a corrected packet on the same handle; it then issues a
new link and the earlier link stops working. Any change voids earlier review and signature evidence.

**R7. Little flows back to the agent.** The provider returns to the agent only the items in section 5: a
continuation handle, the packet schema, blockers and the filing link. It MUST NOT return computed figures,
identity values, signature values or the taxing authority's result to the agent. The provider tells the
taxpayer the outcome directly (for example, by email to the contact verified on the filing page).

**R8. The page asks no tax question.** The filing page MAY present disclosures, take acknowledgements and
collect the identity, authentication, signature, contact and payment values R2 to R5 place there. It MUST NOT
elicit a calculation fact, election or treatment; those belong in the agent conversation.

**R9. Payment (OPTIONAL profile).** Where the provider charges, card details SHOULD be entered only into the
payment processor's own hosted fields on the filing page, never through the agent. A provider that captures
payment only after the taxing authority accepts the return declares `"capture": "after-acceptance"` in its
discovery document.

**R10. No controlled artifacts.** This document references MeF by name only. Packet formats are defined by
each provider's own schema (section 5.2). Implementations MUST NOT publish or redistribute taxing-authority
schemas, business rules or other materials whose terms restrict distribution.

## 4. Discovery

### 4.1 The well-known document

A provider SHOULD serve a JSON document at `https://<provider-host>/.well-known/tax-filing.json`
(RFC 8615), with `Access-Control-Allow-Origin: *`. Its members:

| Member | Required | Meaning |
| --- | --- | --- |
| `specVersion` | yes | The ATFH version, `"0.1"`. |
| `spec` | no | A URL of this document. |
| `provider` | yes | `name`, and URLs for `website`, `support`, `privacyNotice`, `terms`; MAY add `operator`, `about`. |
| `filing` | yes | Array of what the provider files: `country` (ISO 3166-1 alpha-2), `authority`, `channel`, `level` (`federal`, `state`, `local`); MAY add `taxYears`, `returnTypes`, `amended`, `scope` (URL of the full scope). A provider that checks each return's scope when it is submitted (section 5.3, blockers) MAY omit them. |
| `handoff` | yes | `transport` (`mcp-streamable-http` in this version), `endpoint`, `mcpProtocolVersions`, `authentication` (`none` or `oauth`), `directJsonRpc` (optional, section 5), and `operations` mapping `start`, `schema` and `submit` to tool names. |
| `taxpayerPage` | yes | The provider's statements about R2 to R5: `linkIssued`, `privacyNoticeOnPage`, `agentRelayedConsentAccepted` (always false), `possessionCode`, `reviewCompleteReturn`, `signature`, `protectedValuesFromAgent`. |
| `agentReceives` | yes | The list of items the agent can receive (R7). |
| `outcomeDelivery` | yes | `provider-to-taxpayer`. |
| `payment` | no | `cardEntry`, `capture` (R9). |
| `documentation` | no | URLs: `llmsTxt`, `guide`, `mcpServerCard`. |
| `notice` | no | A plain-language notice the provider must show where it names a government body (for example, an IRS non-affiliation statement). |

A JSON Schema is in `schema/tax-filing.schema.json`; `examples/terminal-tax.json` is a live document.

The document is advisory. The tool definitions an agent receives from the live connection are authoritative
when they differ.

### 4.2 llms.txt

A provider that serves `/llms.txt` SHOULD name its `/.well-known/tax-filing.json` URL in it.

### 4.3 MCP registry and server card

A provider that publishes an MCP Server Card or an MCP Registry `server.json` MAY point to the discovery
document from its publisher-provided `_meta` member, with the value `{"discovery": "<url>", "specVersion":
"0.1"}`. This draft does not yet fix the key name; it is an open question for the MCP community, and the
draft will name a neutral key once that is settled.

## 5. The handoff

The three operations are MCP tools (Model Context Protocol, Streamable HTTP transport), reached through the
ordinary MCP lifecycle: `initialize`, then `tools/list` and `tools/call`. That is the interoperable baseline.
Tool names are the provider's; the discovery document maps them.

Optional extension: a provider MAY also answer a bare JSON-RPC `tools/call` sent without a prior `initialize`
(`"directJsonRpc": true`), so an agent with only a shell or an HTTP tool can file in the same conversation. This
departs from the MCP lifecycle, which makes initialization the first interaction, and an agent MUST NOT assume
it unless the discovery document declares it.

### 5.1 start

Takes no arguments and no tax data. Returns `{ "continuationHandle": "<opaque>" }`. The handle identifies
one short-lived submission. It MUST NOT be able to read a stored return, establish identity, sign, pay or
transmit. The agent MUST NOT show it to the taxpayer or place it in a URL.

### 5.2 schema

Takes the handle. Returns the provider's packet schema and its digest (`packetSchemaSha256`). A provider MAY
split a large schema into named sections and MAY offer starter templates in which every taxpayer-specific
value is a placeholder the provider refuses until it is replaced. The call is read-only.

### 5.3 submit

Takes the handle, the completed packet and the schema digest it was prepared against. Returns one of:

- `{ "blockers": [ ... ] }`: nothing was written and no link exists. The agent corrects the packet from
  the taxpayer's answers and submits again with the same handle.
- `{ "status": "filing_session_ready", "filingSessionUrl": "<url>" }`: the packet is held, sealed and unused,
  for the taxpayer. The agent gives the taxpayer the URL exactly as returned, as bare text. Declining on the
  page, or the link expiring, erases the held packet.
- `{ "status": "filing_session_ready" }` with no URL: an exact resubmission after the taxpayer has already
  acted on the page. No link is returned a second time, and the response reveals nothing about how far that
  filing went. To file again, the agent calls `start` for a new handle.

### 5.4 Blockers

A blocker is correction guidance, not a refusal of the person. Each has `path` (the packet field, or `""`
for the whole packet), `rule` (a short machine code such as `missing_required` or `invalid_type`), and
`detail` (what to fix and, where the fact is missing, the question to ask the taxpayer). A blocker MUST NOT
echo a submitted value. A rate-limit blocker carries `retryAfterSeconds`. All blockers for a packet SHOULD
come back in one response. A situation the provider cannot file is said plainly, with the supported next
step, and is never disguised as missing information.

## 6. Security considerations

The filing link is a bearer capability: anyone who has it can open the page. R5's possession factor is what
keeps a leaked link from revealing a return. Providers SHOULD rate-limit the handoff endpoint, cap payload
size, purge unsigned packets on a short timer and apply the taxing authority's own security and bulk-filing
requirements. The continuation handle is a second, narrower capability; keep it out of logs.

## 7. Privacy considerations

The agent already holds the taxpayer's facts; the handoff moves them to one provider only because the taxpayer
asked to file. R2 keeps the most sensitive identifiers out of agent transcripts. R7 keeps the taxing
authority's result and computed figures out of them. The provider's use of return information is governed by
the law that applies to it (in the United States, 26 U.S.C. 7216 and the FTC Safeguards Rule, 16 CFR 314).

## 8. IANA considerations

If this document gains independent implementations, its editors intend to request registration of the
`tax-filing.json` well-known URI suffix under RFC 8615.

## 9. Reference implementation

Terminal Tax implements this draft: discovery at https://terminal.tax/.well-known/tax-filing.json, the endpoint
https://mcp.terminal.tax/v1/hosted-mcp, and tools `create_filing_session` (start), `get_packet_schema`
(schema) and `continue_filing_session` (submit). Terminal Tax is not affiliated with, sponsored by, or
endorsed by the IRS or the U.S. Treasury, and nothing here is published, reviewed or endorsed by the IRS.

## 10. Changes and comments

Open an issue or pull request in this repository. Changes are recorded in its history.
