# Juju engineering harness core design

Working draft. The harness coordinates engineering work so that humans can
understand, verify, and accept its results. Humans own the objective, acceptable
risk, and final acceptance. Agents investigate, implement, test, and review
within explicit assignments.

The intended foundation is the Juju-managed Hermes and OpenViking stack in
[hermes-operator](https://github.com/manadart/hermes-operator/tree/8e0808023345bea6b5bd1fe0d81fb748735f0908).
The design adds an engineering contract and workflow to those components.

## Core contract

1. Work has a stated outcome, scope, and acceptance criteria. Investigation
   can produce findings and unresolved questions; implementation produces a
   change to assess against those criteria.
2. Claims of correctness identify supporting evidence and its limits. Passing
   checks support specific claims; reviewers assess whether those checks cover
   the behaviour and failure modes that matter.
3. An author can complete an assignment and submit its result for review.
   Acceptance of that result requires independent review; authors cannot
   approve their own work. Review recommendations and human acceptance are
   separate decisions.
4. A handoff identifies the result, its rationale, the evidence, and remaining
   uncertainty. Another agent or engineer must be able to continue without
   inheriting the author's conversation.
5. Evidence and review refer to a specific revision of the work. Changes after
   review require reassessment before acceptance. Decisions remain traceable
   to their owner, scope, and supporting evidence.
6. Process scales with the consequences of error. Keep work comprehensible,
   make blockers explicit, and distinguish an unavailable check from a failed
   check. Repository facts and decisions are referenced at their source;
   remembered explanations must be checked for applicability.

## Components and responsibilities

| Component | Role in the harness |
|---|---|
| Juju and the machine charms | Deploy and reconcile the services, configuration, secrets, and context relation |
| Hermes | Persistent agent runtime, sessions, tools, local knowledge, and delegation; the proposed coordinator of engineering assignments |
| OpenRouter | Model access for Hermes and the context service's extraction and embedding work |
| OpenViking | Persistent context, extraction, and retrieval to inform future work |
| Git and GitHub | Versioned source, engineering decisions, changes, and review artifacts; optional GitHub App authentication is supplied by the Hermes charm |

The initial operator supports one persistent Hermes identity and its subagents
inside one LXD instance. Its configuration caps concurrent children at two and
disables gateway Kanban dispatch. This is the starting topology for the design.
See the [rendered workload configuration](https://github.com/manadart/hermes-operator/blob/8e0808023345bea6b5bd1fe0d81fb748735f0908/charms/hermes/src/workload.py#L66).

## What becomes an instruction artifact

The core above is a shared engineering contract. Keep its authoritative text
versioned in Git and deliver an identified version to every participant. The
artifact's filename can follow once its scope and delivery are settled.

| Artifact | Purpose |
|---|---|
| Shared contract | Obligations common to author, reviewer, and coordinator |
| Repository `AGENTS.md` | Durable conventions and constraints for work in that repository |
| Role skill | A reusable procedure, with any supporting resources, for one engineering activity |
| Task brief | The current objective, role, scope, criteria, input revision, permitted actions, and expected result |
| Handoff and review record | The result, rationale, evidence, uncertainty, and decision about that revision |

`AGENTS.md` can carry the repository-facing portion of the contract. Skills can
carry the procedures. The entire harness spans these artifacts and the runtime
behaviour that connects them. Installing a skill does not itself enforce a
review gate.

## Delivery to cooperating agents

The operator pins Hermes at `345cd2b057a452236de401d3534b8502a7465e8d`.
At that revision, delegated agents receive explicit `goal` and `context` fields
and project instructions from the resolved workspace. They start without the
parent's conversation or automatic memory loading; the child prompt deliberately
excludes `SOUL.md`. A contract stored only in the parent's persona therefore
does not reach all participants. See [child construction](https://github.com/NousResearch/hermes-agent/blob/345cd2b057a452236de401d3534b8502a7465e8d/tools/delegate_tool.py#L203)
and [child prompt assembly](https://github.com/NousResearch/hermes-agent/blob/345cd2b057a452236de401d3534b8502a7465e8d/tools/delegate_tool_progress.py#L149).

Project instruction discovery prefers `.hermes.md` or `HERMES.md` over the
`AGENTS.md` chain. It depends on the resolved workspace. Delivery must account
for that precedence and ensure the intended repository guidance is present.
See [context discovery](https://github.com/NousResearch/hermes-agent/blob/345cd2b057a452236de401d3534b8502a7465e8d/agent/prompt_builder.py#L1584).

Proposed delivery:

1. Give the coordinator the contract and task brief when starting the work.
2. Supply each delegate with the contract, its bounded assignment, the exact
   repository and revision, and relevant artifact paths through its delegation
   context. Identify the required role procedure explicitly.
3. Make procedure files and their resources available locally. Hermes supports
   skills under its home and configured external directories; the charm uses
   `/var/lib/hermes` as its home. The installation/update mechanism and enabled
   skill tools remain to be specified. See [skill directory resolution](https://github.com/NousResearch/hermes-agent/blob/345cd2b057a452236de401d3534b8502a7465e8d/agent/skill_utils.py#L337).
4. Record the delivered versions and collect the result against the assignment.
   Check required evidence before advancing the workflow. Preserve blocked and
   incomplete outcomes explicitly.

## Knowledge and workflow state

OpenViking supplies remembered context to help agents investigate. The proposal
keeps authoritative instructions, task status, acceptance, and review evidence
in explicit records. Semantic recall must not be the only way to recover which
revision was reviewed or what a human authorized.

The current relation provisions an isolated OpenViking account per related
application, with one active relation per context identity. It does not yet
establish shared institutional knowledge across independently deployed agents,
or a private identity for every delegated child. Shared knowledge and its
promotion rules need a deliberate design. See the [relation contract](https://github.com/manadart/hermes-operator/blob/8e0808023345bea6b5bd1fe0d81fb748735f0908/interfaces/openviking/README.md).

## First workflow to exercise

Start with one bounded change: task and acceptance criteria, implementation
and evidence, independent review, then human acceptance or redirection.

The reviewer receives the original task, criteria, exact change, and evidence.
It develops its own assessment from those artifacts. The author's rationale
is an input to examine, rather than a conclusion to inherit. Findings return
to the author as a new assignment; a revised result returns for review.

A human can operate this flow through Hermes while its contract is refined.
The current local terminal backend runs agent commands as the same service
user without a separate sandbox for each task. Separate reviewer conversations
provide a fresh assessment, but do not establish separate permissions. The
implementation still needs an enforceable boundary for accepting and merging
work. See the operator's [runtime description](https://github.com/manadart/hermes-operator/blob/8e0808023345bea6b5bd1fe0d81fb748735f0908/charms/hermes/README.md#runtime-and-state).

The next increment is one concrete author-to-reviewer handoff on this stack:
its inputs, output, revision identity, and acceptance conditions. Durable task
storage, recovery of interrupted work, artifact distribution, and shared
knowledge policy remain open. These are proposed harness behaviours; source
inspection establishes the available components, not a validated engineering
flow.
