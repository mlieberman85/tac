## Application for darnit at Sandbox stage

### List of project maintainers

The project has 3 maintainers across 2 organizations:

* Michael Lieberman, Kusari, @mlieberman85
* Ben Cotton, Kusari, @funnelfiasco
* Marco De Vincenzi, New York University, @Marc-cn

### Sponsor

Darnit reports to the OpenSSF **Supply Chain Integrity Working Group** and
commits to providing quarterly updates.

* [Supply Chain Integrity WG](https://github.com/ossf/wg-supply-chain-integrity)

### Mission of the project

Darnit is a framework for auditing software repositories against compliance
and security baselines, collecting the project context those audits need,
remediating gaps, and producing signed attestations of the result. Its
reference module implements the [OpenSSF Baseline](https://baseline.openssf.org/);
additional baselines and organization-specific postures are added as
modules via Python entry points and TOML configuration.

Darnit intentionally integrates with existing tooling as much as possible
rather than overlapping. It does not
replace OpenSSF Scorecard, Allstar, Minder, SLSA, gittuf, or similar tools,
it is the layer that knows how to drive them for a given project. Almost
every real project is unique enough that "just install Scorecard" or "just
enable SLSA tooling" requires non-trivial integration work (understanding
the build, the governance, who has write access, where docs live). That
integration tax is what darnit exists to absorb so the underlying ecosystem
tooling actually gets adopted, and adopted correctly.

#### How darnit is used

Darnit's primary deployment shape is as an agentic Skill or agent that
drives darnit's suite of MCP tools. A user asks a goal-shaped question, "is this
repository Baseline-compliant?" or "what do we need to change to meet SLSA
Build L3?" and the agent calls darnit's MCP tools to:

1. **Run checks** against a selected module.
2. **Collect data** the checks or remediations require (project metadata,
   maintainers, build configuration, etc.).
3. **Remediate** any gaps, with the maintainer in the loop for changes
   outside the repository.
4. **Attest** that the project meets the bar at a given level, so
   downstream users can rely on the assertion without re-deriving it.

Darnit is also available as a CLI (for CI use) and a Python library, but
the MCP surface is the contract; the others are wrappers.

#### Non-Goals

* Replacing OpenSSF Scorecard, Allstar, Minder, SLSA, gittuf, or similar
  tools. Darnit drives them, it does not duplicate them.
* Defining a new attestation or compliance-report format. Darnit emits
  existing in-toto and (planned) Gemara shapes.
* Gating CI by itself. Producing SARIF and other consumable outputs is in
  scope; the integration points that act on them are the consumer's.
* Automated remediation of changes that touch state outside the repository
  without explicit human-in-the-loop confirmation.

#### How it works

**Modules.** Each capability darnit ships, the OpenSSF Baseline,
SLSA-derived checks, an organization's internal hardening profile, is a
**module**. Modules are defined in TOML, and the same TOML surface serves
both end users (declaring which modules to apply and how to combine them,
e.g. *Baseline + gittuf + an internal company profile*, with no forking)
and module authors (defining new modules; Python is only needed for
genuinely novel logic).

**The deterministic -> pattern -> LLM -> manual cascade.** Every phase
(checks, data collection, remediation) follows the same four-pass cascade:
try the most deterministic answer first (Scorecard, an API call, a file
check), fall back to pattern matching, then to an
LLM-with-human-confirmation, and finally to a manual prompt when even the
LLM is uncertain. The fail-to-manual fallback is what backs darnit's "WARN
means we don't know" stance: an inconclusive answer is never silently
upgraded to PASS, and remediation darnit isn't confident in is emitted as
a checklist rather than an automated PR. Concretely:

* **Checks.** If the repository has no `SECURITY.md` and pattern matching
  finds no obvious policy, an LLM pass may notice that the README links
  to a hosted security policy page, surfaced to the user for confirmation.
  If even that is inconclusive, the control resolves to WARN.
* **Data collection.** When `MAINTAINERS.md` is absent, darnit looks at
  commit-access lists and contributor signals; if a confident answer is
  still unavailable, it surfaces a manual question rather than guessing.
* **Remediation.** Deterministic API calls and template fills (e.g.,
  enabling branch protection, generating a `SECURITY.md`) come first; the
  result can then be enriched with collected `.project/` context and
  optional LLM input so the artifact describes *this* project rather than
  a generic placeholder. Every remediation goes through dry-run and is
  emitted as a PR for human review. Potentially destructive actions like
  API calls are gated via human approval in the system. 

**Project metadata in CNCF `.project/`.** Context collected during the
data-collection phase is normalized into the CNCF `.project/` format
(link TBD) so the metadata is portable across tools and survives if a
project stops using darnit.

**Outputs and attestations.** Reports are emitted as Markdown, SARIF
2.1.0, or JSON depending on the consumer. Audit attestations are opt-in:
**in-toto attestations are available today**, signed with
[sigstore](https://www.sigstore.dev/); **[Gemara](https://github.com/revanite-io/sci)-format
compliance reports are planned** as a second opt-in output behind the same
module-selection surface.

### Project development model: AI-assisted, human-guided

To be transparent: darnit heavily uses coding agents to supplement human
work, but humans are always the decision-makers. We lean into this
because darnit itself is an AI-harness framework using AI tools to build
an AI-tooling framework gives us a tight feedback loop on the MCP surface
and the failure modes real users will hit.

The workflow is spec-driven via
[GitHub Spec Kit (`speckit`)](https://github.com/github/spec-kit): every
non-trivial change starts as a written spec, plan, and task breakdown
before implementation. Implementation is done in collaboration with AI
tooling, primarily coding agents with
custom [Skills](https://docs.claude.com/en/docs/claude-code/skills) and
[MCP servers](https://modelcontextprotocol.io/), so the agent operates
against a reviewed spec for guardrail purposes.

No PR merges without human review. CI runs the full test suite, lint,
type-check, spec-implementation sync validator, generated-docs freshness
check, and security scans; at least one maintainer reviews every PR for
correctness, scope, and adherence to spec. AI-suggested code is treated
the same as any other contribution: it must be understood by the human
merging it. Design and scope are owned by the maintainers.

### Alignment with the OpenSSF and the SCI WG

Darnit aligns with the [OpenSSF's mission](https://openssf.org/about/) of
making it easier to sustainably secure the development, maintenance, and
consumption of open source software: it lowers the cost of *adopting* the
security tooling the ecosystem already produces.

Within the SCI WG, darnit is integrative rather than overlapping. The
in-toto attestation output is signed via sigstore and is designed as a
first-class input to SLSA-style provenance pipelines, gittuf policy
evaluation, and GUAC-style aggregation. Planned Gemara output covers the
same audit data in a compliance-reporting shape.

### IP policy and licensing due diligence

Darnit is licensed under Apache License 2.0. License and IP due diligence
has not yet been performed by the Linux Foundation; an IP-policy tracking
issue will be opened in the TAC repository before this PR is merged.

* **Status**: TBD - LF IP/license due diligence pending.
* **Tracking issue**: TBD.

The repository currently lives at
[kusari-oss/darnit](https://github.com/kusari-oss/darnit); the maintainers
intend to transfer it to a neutral GitHub organization as part of sandbox
onboarding so the project is community-stewarded rather than
vendor-stewarded.

### Project References

| Reference          | URL |
|--------------------|-----|
| Repo               | https://github.com/kusari-oss/darnit |
| OpenSSF Baseline   | https://baseline.openssf.org/ |
| CNCF `.project/`   | https://github.com/cncf/automation/tree/c62a3a34a4a5e1317baab1b343ef1c0b33ee09ec/utilities/dot-project |
| License            | https://github.com/kusari-oss/darnit/blob/main/LICENSE (Apache-2.0) |
| Contributing guide | https://github.com/kusari-oss/darnit/blob/main/CONTRIBUTING.md |
| Security.md        | https://github.com/kusari-oss/darnit/blob/main/SECURITY.md |
| Spec-driven dev    | https://github.com/kusari-oss/darnit/tree/main/specs |
| Demos              | https://www.youtube.com/watch?v=ObOaexecUMQ |
| Roadmap            | See github issues |
