# Boomerang Reading Guide

Boomerang has a lot of moving parts. Read it in layers instead of trying to absorb the whole repository at once.

---

## Table of Contents

- [If you have 10 minutes](#if-you-have-10-minutes)
- [If you have 30 minutes](#if-you-have-30-minutes)
- [If you have 2 hours](#if-you-have-2-hours)
- [If you want to contribute](#if-you-want-to-contribute)
  - [Protocol review](#protocol-review)
  - [Threat modeling](#threat-modeling)
  - [Hardware review](#hardware-review)
  - [UX review](#ux-review)
  - [Technical writing](#technical-writing)
- [What not to do on the first read](#what-not-to-do-on-the-first-read)
- [The main idea to remember](#the-main-idea-to-remember)

## If you have 10 minutes

Goal: understand the idea without drowning in acronyms.

1. Read the root [`README.md`](README.md).
2. Read the glossary entries for:
   - `Boomlet`
   - `mystery`
   - `Watchtower`
   - `SAR`
   - `fallback regime`
3. Look at the simple diagrams in the README.

After this pass, you should be able to say:

> Boomerang is a research design for Bitcoin cold storage where the primary withdrawal path is deliberately bounded-but-unpredictable, enforced by hardware, and includes hidden emergency signaling.

That is enough for a first read.

---

## If you have 30 minutes

Goal: understand the protocol shape.

Read in this order:

1. [`README.md`](README.md) — overview.
2. [`GLOSSARY.md`](GLOSSARY.md) — terminology.
3. [`spec/SPEC.md`](spec/SPEC.md), sections:
   - Status
   - Goals and non-goals
   - System model
   - Security and timing model
   - Descriptor
   - Withdrawal protocol
   - Failure behavior
4. [`withdrawal/README.md`](withdrawal/README.md) — the ceremony in more detail.

Questions to keep in mind:

- What exactly is being made unpredictable?
- What remains bounded?
- Which actors are trusted for custody, coordination, user interface, and real-world response?
- What causes the primary path to fail closed?
- When does the deterministic fallback become available?

---

## If you have 2 hours

Goal: understand the design rationale and the main risks.

Read in this order:

1. [`README.md`](README.md)
2. [`GLOSSARY.md`](GLOSSARY.md)
3. [`spec/SPEC.md`](spec/SPEC.md)
4. [`DEEPDIVE.md`](DEEPDIVE.md)
5. [`security_models/README.md`](security_models/README.md)
6. [`security_models/assumption_register.md`](security_models/assumption_register.md)
7. [`security_models/forced_determinism.md`](security_models/forced_determinism.md)

Questions to keep in mind:

- Which claims depend on Boomlet hardware being honest and non-extractable?
- Which claims depend on Secure Terminal integrity?
- Which claims depend on Watchtower or SAR availability?
- How does the system avoid visible behavior differences between safe and duress cases?
- What happens if a peer disappears?
- What happens if operators wait too long before rollover?
- What is still underspecified before production use?

---

## If you want to contribute

Pick one workstream instead of trying to review everything.

### Protocol review

Start with [`spec/SPEC.md`](spec/SPEC.md). Focus on:

- exact message binding;
- transaction identity continuity;
- freshness and replay resistance;
- timeout and fail-closed behavior;
- fallback interaction.

### Threat modeling

Start with [`security_models/README.md`](security_models/README.md). Focus on:

- assumption quality;
- hidden trust boundaries;
- forced determinism;
- metadata leakage;
- service compromise;
- supply-chain compromise.

### Hardware review

Start with:

- [`secure_terminal/README.md`](secure_terminal/README.md)
- [`duress_protection/README.md`](duress_protection/README.md)
- the Boomlet-related sections of [`spec/SPEC.md`](spec/SPEC.md)

Focus on:

- whether the secure element can really enforce the required state machine;
- whether the Secure Terminal can preserve display/input integrity;
- how lifecycle, backup, activation, and tamper evidence should work.

### UX review

Start with the root README, then read:

- [`setup/README.md`](setup/README.md)
- [`withdrawal/README.md`](withdrawal/README.md)
- [`duress_protection/README.md`](duress_protection/README.md)

Focus on:

- steps a stressed human may misunderstand;
- repeated prompts that could create fatigue;
- terminology that sounds scarier than necessary;
- places where the system needs confirmation, rehearsal, or safer defaults.

### Technical writing

Start anywhere that feels confusing. Useful writing contributions include:

- replacing acronyms with names on first use;
- moving threat details after the mental model;
- adding small examples;
- distinguishing assumptions from guarantees;
- keeping the scary-but-necessary parts accurate without making the first page feel hostile.

---

## What not to do on the first read

Do not start with the largest diagrams. They are useful after the mental model is in place.

Do not judge the design by one alarming phrase. Terms like `duress`, `SAR`, and `rescue information` are part of the threat model, but the project is not asking casual users to run this today.

Do not assume every mechanism is production-ready. The repository repeatedly states that it is a research-stage design with open issues.

Do not skip the limitations. Boomerang’s value is clearest when its assumptions and failure modes are visible.

---

## The main idea to remember

Boomerang is not trying to make Bitcoin magically safe under all pressure.

It is trying to make one attacker advantage less reliable:

> the assumption that a pressured signer can make high-value funds move on a predictable schedule.
