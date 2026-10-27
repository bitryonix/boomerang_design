# Boomerang: coercion-aware Bitcoin cold storage

> [!WARNING]
> **Design status: not production-ready.** Boomerang is an unfinished protocol
> design. Its hardware assumptions, operating procedures, parameter choices,
> and complete implementation have not been validated for real funds.

> **Cooperation is not completion.** Boomerang's design goal is to prevent
> forced human cooperation from reliably producing a prompt, verifiable payout.

Boomerang is a Bitcoin cold-storage protocol design for an attacker who shows
up in person and forces the people who control the keys to cooperate.

Five people, called custodians, protect the funds, and the earliest way to spend
requires all five to participate. Each custodian has a trusted signing device
that withholds its part of the Bitcoin signature until a required withdrawal
procedure finishes. During that procedure, every device independently draws a
fresh private requirement for how much valid progress it must observe. The
users and their computers cannot inspect that requirement in advance, choose
it, lower it, or make the device count invalid progress.

> [!IMPORTANT]
> **The central coupling.** The messages required to complete a withdrawal also
> carry each custodian's encrypted "safe" or "under duress" answer to a rescue
> service arranged in advance. Complete cooperation therefore does not reveal
> a precise finish time, and continuing the forced withdrawal may deliver a
> duress signal while signing remains unavailable. Whether the resulting
> interval is operationally long enough to matter depends on production
> parameters and response capability that have not yet been established.

## Contents

- [The attacker's job](#the-attackers-job)
- [Who is involved](#who-is-involved)
- [Boomerang in 60 seconds](#boomerang-in-60-seconds)
- [A concrete coercion scenario](#a-concrete-coercion-scenario)
- [A coerced withdrawal becomes a race](#a-coerced-withdrawal-becomes-a-race)
- [One coupled mechanism](#one-coupled-mechanism)
- [Attack economics](#attack-economics)
- [Spending paths](#spending-paths)
- [Claims and limits](#claims-and-limits)
- [Intended use](#intended-use)
- [Q&A](#qa)
- [Read next](#read-next)

## The attacker's job

Boomerang assumes an attacker strong enough to identify all five custodians,
take physical control of them and the equipment needed for a withdrawal, and
force them to make a real payment to an attacker-chosen address. Every
custodian follows the attacker's instructions correctly. No one withholds a
password, substitutes a decoy, or merely pretends to cooperate.

To steal the funds, the attacker must complete every step below.

1. Prepare a payment to an address the attacker controls and force all five
   custodians to review and confirm that exact payment.
2. Keep all five custodians and their devices available while the required
   withdrawal procedure runs.
3. Obtain all five final Bitcoin signatures.
4. Verify the payout, move the bitcoin beyond recovery, and escape before a
   responder can intervene.

Multisig, geographic separation, and isolated keys raise the cost of finding
and controlling every required participant. Once the attacker has assembled
all five people and their devices, an ordinary withdrawal can become a
schedulable checklist. Boomerang makes the third step wait on requirements
chosen privately by the devices. The messages needed to satisfy those
requirements also deliver encrypted answers to rescue services. The attacker
must therefore sustain control without knowing the precise signing time, while
a duress answer may already have started a response.

## Who is involved

Each custodian uses a small trusted signing device called a Boomlet. A
coordination service called the Watchtower passes withdrawal messages among
the five custodians. Each custodian also arranges a Search and Rescue service
(`SAR`) in advance to receive encrypted indications that the user is safe or
under duress and begin a prepared response when appropriate. The
[glossary](GLOSSARY.md) provides a concise index of these and other protocol
terms.

## Boomerang in 60 seconds

| What the attacker must do | What Boomerang forces | Why it matters |
| --- | --- | --- |
| Compel all five users to review and confirm the same exact unsigned transaction | Each Boomlet records an approval for the withdrawal procedure; Bitcoin signing remains a later step | Full human cooperation does not skip the device-enforced gates |
| Keep the withdrawal advancing | Every Boomlet independently draws a fresh private progress requirement, and only valid exchanges with the other devices count toward it | The users cannot disclose or accelerate a precise finish |
| Reach signing, obtain a verifiable payout, and escape | Required messages carry fresh encrypted indications of safety or duress, and the protocol waits for exact SAR acknowledgments | A duress answer can activate a prepared response while signing remains unavailable |

Boomerang therefore turns a forced withdrawal into a race. The attacker must
maintain control through an unpredictable wait, while the required withdrawal
traffic can start a prepared response before signing becomes available.

> [!IMPORTANT]
> **The glossary is required from this point onward.** The remaining sections
> use protocol names, states, keys, and message types at technical density. Read
> [`GLOSSARY.md`](GLOSSARY.md) before continuing and keep it available as a
> reference.

## A concrete coercion scenario

Suppose five custodians protect a high-value treasury in a Taproot output. The
output's earliest script branch is the five-of-five Boomerang branch. Bitcoin
consensus makes it available no earlier than block height `milestone_block_0`,
and satisfying it requires a signature under every custodian's Boomerang
public key. Each of those signatures is produced jointly by the custodian's
recoverable normal key and a share held in a small trusted device called a
Boomlet. A Boomlet will not use its share until the withdrawal procedure
specified by the protocol has run to completion. An attacker identifies every
custodian, controls them and their equipment, dictates a destination, and
forces everyone through every step for the same attacker-chosen unsigned
transaction. No one is withholding a password or pretending to cooperate.

In a conventional five-of-five ceremony, that much cooperation may be enough
to produce Bitcoin transaction signatures. In Boomerang, each user
independently confirms the identifier (`tx_id`) of the exact unsigned
transaction on an air-gapped display-and-input device called the Secure
Terminal during the withdrawal. That confirmation tells the user's Boomlet
which transaction is meant. The Boomlet then signs a protocol message called
`TxApproval`, which authorizes progress for this withdrawal and is bound to its
`withdrawal_id`. Bitcoin transaction signing becomes available only after the
later device-enforced progress requirements have been met.

A coordination service called the Watchtower collects one valid `TxApproval`
from each of the five Boomlets. Every Boomlet verifies the ordered set of
exactly five `TxApproval` messages and independently computes the same
`approved_withdrawal_id`, the identifier that binds every later step to this
withdrawal and this approval set. One peer starts each withdrawal as its
initiator; the other four are that ceremony's non-initiators, and their
Boomlets send attestations proving that they received and verified the
complete set and agree on that identifier. These attestations serve as receipts
of that agreement.

The initial duress and commitment phase can overlap with final attestation
collection. The initiator may enter its consent response and submit its signed
`TxCommit` early. This message commits its Boomlet to the approved withdrawal
and travels with the encrypted answer intended for its SAR; the Watchtower may
stage both. The Watchtower must verify all four attestations before relaying the
initiator's encrypted answer to its SAR, acknowledging the initiator commit to
the other peers, or accepting their signed `TxCommit` messages. Each
non-initiator commits only after verifying the Watchtower-signed initiator
commit.

At its prescribed point, each user selects five countries from independently
shuffled columns derived from the protocol's fixed 193-entry consent
vocabulary. The memorized five-country set means safe; any other valid
selection means duress. Those physical selections are distinguishable to
someone watching the interaction. In either case, the required protocol
messages carry the same kind of encrypted placeholder to the Search and Rescue
service (`SAR`) that this user bound during setup. Each SAR must durably process
and acknowledge the exact placeholder it received; the full ordering rules are
in
[DESIGN §10](DESIGN.md#10-withdrawal-in-detail). A Boomlet may enter the
withdrawal state called `DIGGING` only after it verifies the complete signed
`TxCommit` collection and its own exact SAR acknowledgment.

Entering `DIGGING` is the only moment a Boomlet draws its `mystery`. This fresh
private threshold is sampled from bounds fixed by the protocol profile and
sets how many successful local `counter` increments this device requires before
it will sign. The users cannot read or lower the thresholds before they are
reached and cannot make the `counter` values advance early. An increment
requires a valid `pong` tied to the active withdrawal, advancing local chain
progress, and fresh-enough current messages from the other peers; an otherwise
valid catch-up round does not increment.
Signing begins only after every Boomlet reports that its threshold has been
reached.

Full cooperation therefore gives the attacker no shortcut and no precise
finish time in advance. Waiting requires continuing control and coordination
while increasing exposure to a response.

## A coerced withdrawal becomes a race

The attacker still needs signing, a verifiable payout, exfiltration, and a
safe escape. A primary-branch withdrawal makes signing wait on acknowledged
duress traffic and five private thresholds, while a duress answer from any
user starts a response on a parallel track. The high-level map shows those
tracks. The expandable map beneath it follows coercion that begins before the
withdrawal and shows more of the protocol ordering.

```mermaid
flowchart TB
    subgraph ATT["ATTACKER — reach a usable payout and escape"]
        direction LR
        A1["Control all five users"] --> A2["Compel the real<br/>withdrawal"]
        A2 --> A3["Sustain control<br/>and coordination"]
        A3 --> A4["Obtain five Bitcoin<br/>transaction signatures"]
        A4 --> A5["Verify payout,<br/>exfiltrate, escape"]
    end

    subgraph PRO["BOOMERANG — gates the path to signing"]
        direction LR
        B1["Exact transaction review<br/>and device-recorded approvals"] --> B2["Encrypted answers delivered<br/>and acknowledged by each SAR"]
        B2 --> B3["Five private progress requirements<br/>enforced by five devices"]
        B3 --> B4["All five Boomlets<br/>report reached"]
        B4 --> B5["Final signing<br/>becomes available"]
    end

    subgraph RES["RESPONSE — proceeds asynchronously"]
        direction LR
        R1["Duress answer<br/>activates duress response"] --> R2["Prepared responder<br/>assesses and acts"]
    end

    A2 -. "forces" .-> B1
    A3 -. "must keep this moving" .-> B3
    B5 --> A4
    B2 -. "if duress" .-> R1
    B3 -. "required progress carries<br/>fresh encrypted answers" .-> R1
    R2 -. "possible interruption before<br/>the attacker finishes" .-> A5

    classDef attacker fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef protocol fill:#e0f2fe,stroke:#0369a1,color:#082f49
    classDef uncertainty fill:#fef3c7,stroke:#b45309,color:#451a03
    classDef response fill:#dcfce7,stroke:#15803d,color:#052e16
    class A1,A2,A3,A4,A5 attacker
    class B1,B2,B4,B5 protocol
    class B3 uncertainty
    class R1,R2 response
```

**Visual key:** red is the attacker's path, blue is enforced protocol progress,
amber is private completion uncertainty, and green is the response path. Solid
arrows show required sequencing; dashed arrows show influence or a conditional
real-world effect.

<details>
<summary><strong>Expand the detailed protocol-gate map</strong></summary>

This map is explanatory rather than normative. Exact message checks and failure
behavior remain in the [specification](spec/SPEC.md).

```mermaid
flowchart TD
    A["Attacker controls all five users and forces<br/>a withdrawal to the attacker's address"] --> T["During withdrawal, every user verifies<br/>the same unsigned transaction ID"]
    T --> A1["Each Boomlet signs TxApproval<br/>Protocol approval for this withdrawal"]
    A1 --> A2["All Boomlets verify the ordered five-TxApproval set<br/>and derive the same approved_withdrawal_id"]
    A2 --> G["Four non-initiators attest receipt and agreement<br/>The initiator TxCommit may be staged in parallel"]
    G --> I0["Watchtower verifies all four attestations,<br/>then sends the initiator placeholder to its SAR"]
    I0 --> D["SAR durably processes and exactly acknowledges it<br/>Watchtower then signs and relays the initiator commit"]
    D --> N["Each non-initiator verifies that signed commit,<br/>then sends its TxCommit and placeholder"]
    N --> K["Each remaining SAR durably processes, then<br/>acknowledges, its peer's exact placeholder"]
    K --> C["Each Boomlet verifies the complete TxCommit<br/>collection and its own exact SAR acknowledgment"]
    C --> M["Each Boomlet enters DIGGING here<br/>and draws a fresh private mystery"]
    M --> R["Each progress-round ping<br/>carries a newly encrypted placeholder"]
    R --> P["SAR must acknowledge each placeholder<br/>before its ping can enter a pong reply"]
    P --> Q{"All five thresholds reached?"}
    Q -- "No" --> R
    Q -- "Yes" --> S["Signing can begin<br/>All five peers must sign"]
    S --> V["Attacker must still verify payout,<br/>exfiltrate, and escape"]

    I0 -. "if the initiator<br/>signaled duress" .-> X["The prepared response starts on its own track<br/>while the ceremony continues"]
    K -. "if a non-initiator<br/>signaled duress" .-> X
    P -. "if any later placeholder<br/>signals duress" .-> X
    X --> I["Possible interruption or rescue<br/>before the attacker finishes"]

    classDef attacker fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef protocol fill:#e0f2fe,stroke:#0369a1,color:#082f49
    classDef decision fill:#fef3c7,stroke:#b45309,color:#451a03
    classDef response fill:#dcfce7,stroke:#15803d,color:#052e16
    class A attacker
    class T,A1,A2,D,G,I0,N,K,C,M,R,P,S protocol
    class Q decision
    class X,I response
    class V attacker
```

</details>

> [!CAUTION]
> **Physical observation can defeat the duress check.** The Secure Terminal is
> air-gapped, but a nearby observer can see which countries the user selects.
> The design assumes the attacker cannot observe closely enough to learn or
> dictate the user's safe selection. An attacker who learns it can force later
> checks to evaluate as safe. Boomerang conceals the classification in protocol
> messages; physical privacy remains an operational requirement.

> [!NOTE]
> **A SAR acknowledgment is not a rescue guarantee.** It proves exact protocol
> delivery and durable activation when duress was signaled. It does not prove
> timely, lawful, effective, correctly directed, or safe intervention. The
> responder may act while the attacker is still waiting, but the protocol
> cannot guarantee the result.

> [!NOTE]
> **Boomerang defines no special cancellation path.** In a withdrawal approved
> in good faith, the peers might discover during `DIGGING` that the destination
> is controlled by an attacker. The practical opportunity is simply that the
> transaction remains unsigned and unbroadcast. Any required peer can stop
> participating or explicitly abandon the ceremony, after which the group can
> start a fresh withdrawal to a correct destination. This cannot reverse a
> transaction that has already been signed and broadcast.

## One coupled mechanism

> [!IMPORTANT]
> **Required progress dependency.** Initial placeholder → exact SAR
> acknowledgment → `DIGGING`; then signed `ping` + freshly encrypted placeholder
> → exact SAR acknowledgment → eligibility for `pong` → possible local `counter`
> increment. Removing the duress-delivery path also removes required withdrawal
> progress.

A delay alone merely tells an attacker how long to wait. A silent alarm alone
may be left out of the very path the attacker needs. Boomerang couples them so
that neither can be separated from required progress. A Boomlet must verify
its exact initial SAR acknowledgment before entering `DIGGING`. From then on,
each Boomlet repeatedly sends a signed progress message called a `ping`, and
the Watchtower replies with a `pong` for each round; a device's `counter` can
advance only on a valid `pong`. Every `ping` carries a freshly encrypted
placeholder, and the Watchtower must obtain SAR's exact acknowledgment of
that placeholder before the `ping` may be used in a `pong`. Some rounds also
present the user with a fresh duress challenge. The attacker cannot reach
signing without sustaining the same traffic that carries and confirms
concealed duress state.

Each SAR deployment uses a fixed acknowledgment delay. When SAR receives a
placeholder, it records the receipt time and schedules the acknowledgment for
that time plus the fixed delay. SAR releases the acknowledgment at the
scheduled time for both valid safe and duress placeholders. If processing
misses that time, SAR sends no late acknowledgment and presents the same
failure in either case. Response shape, routing, durable-write path, and retry
behavior are also the same, so protocol traffic conceals whether the answer was
safe or duress. This concealment applies only to protocol traffic; physical
observation of the user and compromised equipment are separate threats.

## Attack economics

### Evidence and its limits

Physical attacks on bitcoin holders are documented. A
[2024 peer-reviewed AFT study](https://drops.dagstuhl.de/entities/document/10.4230/LIPIcs.AFT.2024.24)
provides the selected, underreported media sample used below. Its missing
custody and response details limit what can be inferred about Boomerang.

The diagram separates reported observations, the inference they support, and
the case-level facts the reports do not provide.

```mermaid
flowchart TB
    subgraph OBS["OBSERVED IN THE AFT MEDIA SAMPLE"]
        A["105 retained physical-attack reports"] --> B["66 recorded demands for a cryptocurrency transfer<br/>Closest observable category to a compelled withdrawal"]
        B --> C["2 documented cases:<br/>coerced transfers were initiated,<br/>but a 24-hour exchange delay and verification<br/>let victims flag and stop the transfers<br/>before attackers received all the funds"]
    end

    C --> D["Supported observation:<br/>withholding final payout can change an outcome<br/>when a usable response channel exists"]
    D -. "motivates; does not validate" .-> E["Relevance to Boomerang:<br/>keep primary-branch signing unavailable while<br/>required progress can activate a prepared response"]

    subgraph LIMIT["COUNTERFACTUAL BOUNDARY"]
        U["The reports do not establish each case's<br/>custody policy, required participants and devices,<br/>withdrawal path and timing, or responder readiness"]
        U --> N["Therefore the dataset cannot tell us<br/>how many cases Boomerang would have addressed,<br/>prevented, or turned into a rescue"]
    end

    A -. "case-level facts are missing" .-> U

    classDef evidence fill:#e0f2fe,stroke:#0369a1,color:#082f49
    classDef supported fill:#dcfce7,stroke:#15803d,color:#052e16
    classDef inference fill:#fef3c7,stroke:#b45309,color:#451a03
    classDef boundary fill:#f3f4f6,stroke:#4b5563,color:#111827
    class A,B,C evidence
    class D supported
    class E inference
    class U,N boundary
```

**Visual key:** blue boxes are reported observations, green is the limited
empirical conclusion, amber is the design relevance inferred from it, and gray
marks facts the reports do not supply. The dotted connectors are not measured
effects.

| Exact AFT evidence snapshot | Reported cases |
| --- | ---: |
| Retained incidents | **105** |
| Demanded cryptocurrency transfer | **66** |
| Demanded keys or a device | **30** |
| Demand unspecified | **9** |
| Reported successful | **70** |
| Reported failed | **29** |
| Outcome unstated | **6** |

> [!NOTE]
> **Evidence boundary:** These are counts in a selected, underreported media
> sample. They are not population rates, an estimate of personal victimization
> risk, or a Boomerang effectiveness rate.

A later
[2026 TRM Labs/Metropolitan Police review](https://www.trmlabs.com/reports-and-whitepapers/wrench-attacks-crypto-enabled-violent-targeting)
describes 17 reported London offences and an approximate mean cryptoasset loss
of £660,000 per offence; it is operational and commercial context, not official
population statistics or protocol evidence.

> [!IMPORTANT]
> **Observed pre-payout interruption:** in two AFT cases, attackers coerced
> victims into initiating transfers but failed to receive all the funds because
> an exchange's 24-hour delay and verification feature let the victims flag and
> stop them. Those cases did not test Boomerang, and stopping a transfer is not
> the same as rescuing a person. The cases show only that withholding final
> payout while a response channel remains usable can change an outcome.

The datasets and their limits are in the
[coercion-economics analysis](security_models/coercion_economics.md).

Published cases do not provide enough information to estimate what would have
happened if Boomerang had been used. They rarely describe the custody setup or
whether a prepared responder could have acted. Some reported events also do not
match the conditions Boomerang is designed to address, such as street robberies
involving hot wallets, incidents involving compromised hardware, or attacks
motivated primarily by harm.

### Readiness model

The chart below illustrates how the five private thresholds specified by the
protocol combine. When a device enters `DIGGING`, it draws a private threshold
from a range fixed by the protocol profile. That threshold determines how many
successful local `counter` increments the device requires. For simplicity, the
chart examines a hypothetical point at which all five independently maintained
`counter` values equal `k`. The equal values form only an analytical snapshot,
and each device maintains its own `counter`.

> [!IMPORTANT]
> **How to read the readiness curve**
>
> - **Horizontal axis (`x`).** It shows the percentage of one Boomlet's allowed
>   `mystery` values at or below hypothetical local `counter` value `k`.
> - **Vertical axis.** It shows the probability that all five independently
>   drawn thresholds have been reached in the special slice where all five
>   local `counter` values equal `k`.
> - **Excluded meanings.** The chart does not show elapsed time, percent of the
>   withdrawal completed, or synchronized real `counter` values.

```mermaid
xychart-beta
    title "All-five readiness in the synchronized slice k1 = ... = k5 = k"
    x-axis "One Boomlet's allowed mystery values at or below k, x (%)" 0 --> 100
    y-axis "Probability all five thresholds are reached (%)" 0 --> 100
    line [0, 0.00003, 0.001, 0.0076, 0.032, 0.0977, 0.243, 0.5252, 1.024, 1.8453, 3.125, 5.0328, 7.776, 11.6029, 16.807, 23.7305, 32.768, 44.3705, 59.049, 77.3781, 100]
```

| `x`: one Boomlet's allowed values at or below `k` | One Boomlet ready | All five ready (`x^5`) |
| ---: | ---: | ---: |
| 0% | 0% | 0% |
| 25% | 25% | 0.10% |
| 50% | 50% | 3.13% |
| 75% | 75% | 23.73% |
| 90% | 90% | 59.05% |
| 95% | 95% | 77.38% |
| 100% | 100% | 100% |

At `x = 50%`, one Boomlet has a 50% readiness probability, while all five are
ready with probability `0.5^5 = 3.125%`, about 1 in 32. The smooth line is a
normalized guide sampled every five percentage points; a concrete profile has
integer thresholds and therefore a discrete staircase. When actual local
`counter` values differ, all-five readiness is the product of the five
cumulative probabilities at their respective `counter` values. The formal
definition, derivation, and caveats are in
[DESIGN §12](DESIGN.md#12-attack-economics-and-security-argument) and
[coercion economics §4](security_models/coercion_economics.md#4-protocol-derived-completion-distribution).

### The attacker's decision

Expected payout must exceed the cost of sustained control plus the expected
loss from disruption. Boomerang pushes completion toward the far end of the
range and makes required progress carry a response opportunity. After every
incomplete round, the attacker decides whether to continue.

```mermaid
flowchart LR
    I["Incomplete round<br/>Not all five are ready"] --> Q{"Continue coercion?"}
    Q -- "No" --> A["Abandon<br/>Sunk costs remain"]
    Q -- "Yes" --> C["Pay more control cost<br/>Bear more response exposure"]
    C --> R{"All five thresholds<br/>reached?"}
    R -- "No" --> I
    R -- "Yes" --> S["Attempt final signing,<br/>verification, exfiltration, escape"]
    C -. "effective response may arrive" .-> X["Payout interruption<br/>is possible"]
    S -. "response may still arrive<br/>before usable payout and escape" .-> X

    classDef decision fill:#fef3c7,stroke:#b45309,color:#451a03
    classDef attacker fill:#fee2e2,stroke:#b91c1c,color:#450a0a
    classDef response fill:#dcfce7,stroke:#15803d,color:#052e16
    class Q,R decision
    class I,C,S attacker
    class A,X response
```

No public dataset currently supplies defensible universal values for attacker
cost or SAR effectiveness, so the project cannot estimate the minimum expected
proceeds needed to offset sustained-control costs and expected disruption
losses. The formulas, observed data, and sensitivity boundaries are in the
[detailed game-theoretic economics analysis](security_models/coercion_economics.md).

## Spending paths

The current profile has exactly five peers and two on-chain regimes. The full
construction is in [DESIGN §8](DESIGN.md#8-on-chain-construction).

- The primary five-of-five Boomerang branch becomes available at
  `milestone_block_0`. "Primary" identifies the earliest branch in the script
  tree. It does not indicate a preference among paths. Each peer's signing
  key combines a recoverable normal key with a Boomlet-held share. The
  Boomlet's material is host-inaccessible, although the authorized setup flow
  can export it in an authenticated, target-bound envelope to a designated
  backup device, the Boomletwo.
- Deterministic fallback begins at `milestone_block_1` with a five-of-five
  normal-key branch. Later milestones reduce that threshold from four to one.
  This preserves recoverability but restores a timetable an attacker can plan
  around.

```mermaid
flowchart LR
    M0["milestone_block_0<br/>5-of-5 Boomerang keys<br/>Device-enforced withdrawal"]
    M1["milestone_block_1<br/>5-of-5 normal keys"]
    M2["milestone_block_2<br/>4-of-5 normal keys"]
    M3["milestone_block_3<br/>3-of-5 normal keys"]
    M4["milestone_block_4<br/>2-of-5 normal keys"]
    M5["milestone_block_5<br/>1-of-5 normal keys"]
    M0 --> M1 --> M2 --> M3 --> M4 --> M5

    classDef boomerang fill:#e0f2fe,stroke:#0369a1,color:#082f49
    classDef fallback fill:#fef3c7,stroke:#b45309,color:#451a03
    class M0 boomerang
    class M1,M2,M3,M4,M5 fallback
```

The arrows show increasing block-height milestones, not equal time intervals.
Blue is the Boomerang branch; amber marks deterministic normal-key fallback.

Bitcoin consensus enforces the Taproot policy and its absolute timelocks.
Trusted hardware and the off-chain state machine enforce `mystery` generation,
progress, and duress acknowledgments. Operators must roll funds into a fresh
setup before fallback becomes an attractive predictable target.

## Claims and limits

Boomerang targets a cost-sensitive, payout-seeking attacker who needs a valid,
verifiable transfer and a viable exit.

| The design argues | The design does **not** claim |
| --- | --- |
| Compelled cooperation can be made insufficient for a reliably prompt payout | People or bitcoin become “duress-proof” |
| Continuing can impose additional cost and response exposure | Every attacker will be deterred or abandon the attack |
| Required progress can create a response opportunity | Rescue, recovery, timely intervention, or human safety is guaranteed |
| The mechanism can matter against a cost-sensitive, payout-seeking attacker | The argument covers harm-focused, state, ideological, exceptionally resourced, or indefinitely patient attackers |

> [!CAUTION]
> **More time can mean more harm.** Longer coercion can increase injury,
> retaliation, trauma, and danger to victims or responders. Time has value only
> when a credible, prepared response can use it without making the situation
> worse.

Five-of-five prevents a bypassing subset from completing the primary branch,
but any one peer or dependency can stall it. Waiting for fallback, starting too
late, losing devices, or deliberate non-cooperation can force deterministic
recovery. Those are central limitations.

## Intended use

Boomerang is intended for high-value, low-velocity bitcoin under a threat model
that explicitly includes planned physical coercion. Its ceremony is
disproportionate for routine spending. Any eventual deployment would require
hardened devices, trained participants, tested recovery procedures, trustworthy
services, jurisdiction-specific response planning, and independent review.

| Suitable conditions | Unsuitable conditions |
| --- | --- |
| High-value, low-velocity treasury funds | Routine or high-frequency spending |
| A custody policy designed around planned physical coercion | Immediately spendable hot-wallet funds |
| Five participants able to sustain a long ceremony | Operations that cannot tolerate one participant or service stalling progress |
| A credible, rehearsed, jurisdiction-aware response plan | A deployment with no prepared responder able to use the interval |

## Q&A

Quick answers for a first read.

<details>
<summary><strong>Protocol mechanics</strong> (attacker objective, duress delivery, and SAR acknowledgment)</summary>

**What must a payout-seeking attacker actually complete?**
The attacker needs a valid, verifiable transfer plus a viable exit. This
requires compelling every user through transaction review and the pre-signing
protocol steps, keeping the withdrawal ceremony progressing to completion,
obtaining the final Bitcoin signatures, verifying the payment, moving the
bitcoin beyond recovery, and escaping. Learning a seed phrase alone does not
finish the job against the primary branch.

**How do the required withdrawal messages carry duress state?**
Each peer's signed `TxCommit` travels with an encrypted placeholder produced
from that user's duress answer, and every later `ping` carries a freshly
encrypted placeholder. SAR must acknowledge each peer's initial placeholder
before that peer's Boomlet may enter `DIGGING`, and must acknowledge every
`ping`'s placeholder before that `ping` may be used in a `pong`. This coupling
prevents the alarm channel from being dropped without halting required
progress.

**What does a SAR acknowledgment prove, and what remains unproven?**
It proves exact delivery and durable processing of the placeholder, including
durable activation of response state when duress is signaled. It does
not prove timely, lawful, effective, correctly directed, or safe
intervention.

</details>

<details>
<summary><strong>Economics and spending paths</strong> (attacker cost, graph axes, protocol stages, and fallback)</summary>

**Why can uncertainty raise the attacker's cost and exposure?**
The attacker must decide, round after round, whether to keep paying for
control without knowing when completion becomes possible. Readiness requires
the maximum of five private draws, which concentrates completion toward the
far end of the allowed range, and every continued round extends exposure to a
response. The quantitative treatment is in the
[coercion-economics analysis](security_models/coercion_economics.md).

**What separates transaction review, `TxApproval`, `TxCommit`, `DIGGING`, and
final signing?**
Transaction review is a user independently confirming the unsigned
transaction's `tx_id` on the Secure Terminal during a withdrawal.
`TxApproval` is the Boomlet-signed pre-signing protocol authorization that
follows, bound to the withdrawal's `withdrawal_id`. `TxCommit` is the
Boomlet-signed commitment to the unanimously approved withdrawal, carried
with the initial duress placeholder. `DIGGING` is the progress state entered
only after the complete signed `TxCommit` collection and the device's own
exact SAR acknowledgment verify; the `mystery` is drawn there. Final Bitcoin
signing happens after all five devices report their thresholds reached and is
the only step that produces Bitcoin transaction signatures.

**What do the axes of the readiness graph show?**
The graph is a simplified slice in which all five independently maintained
local `counter` values happen to equal `k`. The x-axis is the share of one
Boomlet's allowed `mystery` values at or below `k`; the y-axis is the
probability that all five devices are ready, `x^5`, under independent uniform
draws. A concrete integer profile produces a staircase, not the smooth
normalized guide. It is a distribution over `counter` states, not elapsed time
or percent complete. The formal definition and derivation are in
[DESIGN §12](DESIGN.md#12-attack-economics-and-security-argument).

**Can a subset of peers take the funds?**
Not through the primary Boomerang branch, which requires signatures under all
five Boomerang keys. The fallback branches change the required set on the
agreed schedule. They require five normal keys from `milestone_block_1`, four
from `milestone_block_2`, and so on down to one from `milestone_block_5`. Once
a fallback block height passes, that branch needs only its stated number of normal
keys. This is the recoverability trade the design makes, and it is why
operators are expected to roll funds into a fresh setup before fallback
becomes an attractive predictable target; see the
[forced-determinism analysis](security_models/forced_determinism.md).

</details>

<details>
<summary><strong>Limits and deployment</strong> (excluded attackers, failure boundaries, SAR reality, and design status)</summary>

**Does the argument cover state-level or harm-motivated attackers?**
No. Boomerang targets a cost-sensitive, payout-seeking attacker who needs a
valid, verifiable transfer and a viable exit. Attackers primarily motivated
by harm, state or ideological actors willing to absorb exceptional cost, and
indefinitely patient attackers fall outside the deterrence claim.

**What are the main technical and human-safety limits?**
Five-of-five prevents a bypassing subset but lets any one peer or required
dependency stall the primary ceremony, and WT or SAR unavailability stalls it
too. Failure does not authorize fallback early. A compromised Boomlet can
defeat that device's off-chain enforcement. A compromised Secure Terminal can
misdisplay a transaction identifier or alter duress input, but cannot by itself
advance a Boomlet `counter` or create the Boomlet signing share. Protocol
traffic conceals the safe or duress classification. Physical observation,
learned consent responses, or a responder revealing the signal can expose it.
Longer coercion can increase human harm; time has value only when a credible,
prepared response can use it. The full boundary catalog is in
[DESIGN §14](DESIGN.md#14-failure-and-human-safety-boundaries).

**Do Search and Rescue services exist today?**
Not as an established service category. The specification defines the message
contract a SAR must satisfy and deliberately does not define its legal
authority or physical-response procedures. Whether a real institution can
operate the role lawfully, competently, and in each jurisdiction is an open
deployment question.
[coercion economics §7](security_models/coercion_economics.md#7-calibration-and-evaluation-requirements)
lists the response-exercise evidence a deployment would need.

**Is any of this production-ready?**
No. Hardware assumptions, parameter values, wire vectors, service failover,
device-lifecycle procedures, and response operations remain open, and the
assumption that they can all be supplied without changing the security model
is itself unproven. The [specification](spec/SPEC.md) is a draft;
[DESIGN §16](DESIGN.md#16-design-status-and-verification-path) describes the
verification path.

</details>

## Read next

> [!IMPORTANT]
> **The glossary is a prerequisite for the technical documents below.** Read
> [`GLOSSARY.md`](GLOSSARY.md) first; these documents assume its protocol
> vocabulary.

| Resource | Use it for |
| --- | --- |
| [`GLOSSARY.md`](GLOSSARY.md) | Concise lookup index for actors, keys, states, and protocol terms |
| [`DESIGN.md`](DESIGN.md) | The complete conceptual, economic, and security argument |
| [`spec/SPEC.md`](spec/SPEC.md) | Normative protocol behavior: actors, states, messages, cryptography, and failure rules |
| [`wip/wire_catalog.txt`](wip/wire_catalog.txt) | Generated ASCII packet layouts, field IDs, size formulas, and nesting paths |
| [`security_models/`](security_models/README.md) | Threat model, assumptions, attack trees, risks, and unresolved gaps |
| [`security_models/coercion_economics.md`](security_models/coercion_economics.md) | Detailed quantitative model, observed evidence, and calibration boundaries |
| [`adr/`](adr/README.md) | Accepted design decisions and their rationale |
| [Setup procedure](setup/README.md) and [setup sequence SVG](setup/setup_diagram_without_states.svg) | Setup ceremony and its message sequence |
| [Withdrawal procedure](withdrawal/README.md), [initiator SVG](withdrawal/initiator_withdrawal_diagram_without_states.svg), and [non-initiator SVG](withdrawal/non_initiator_withdrawal_diagram_without_states.svg) | Withdrawal steps and role-specific message sequences |
| [Duress protection](duress_protection/README.md) | Consent enrollment and duress signaling |
| [Secure Terminal](secure_terminal/README.md) | Trusted display, input, QR transport, and hardware expectations |

The subsystem documents and listed SVGs are explanatory. The specification
controls where they differ.

### Suggested reading paths

| Path | Suggested route |
| --- | --- |
| **Orientation · 20–25 minutes** | Read this README through [Boomerang in 60 seconds](#boomerang-in-60-seconds), then read [`GLOSSARY.md`](GLOSSARY.md) in full; return to [A concrete coercion scenario](#a-concrete-coercion-scenario) and continue through the README |
| **Technical overview · about 1 hour** | [`DESIGN.md`](DESIGN.md) end to end, then the specification's protocol profile, goals and non-goals, architecture, descriptor, withdrawal protocol, and failure-behavior sections |
| **Deep review** | [`spec/SPEC.md`](spec/SPEC.md) in full, then the [threat model](security_models/README.md), [assumption register](security_models/assumption_register.md), [forced-determinism analysis](security_models/forced_determinism.md), [coercion economics](security_models/coercion_economics.md), and the [ADRs](adr/README.md) |
| **Visual protocol review** | Compare the setup and withdrawal-role SVGs in the resource table with their subsystem procedures; resolve discrepancies against the specification |
| **By contribution angle** | Protocol review uses [`spec/SPEC.md`](spec/SPEC.md). Threat modeling uses [`security_models/`](security_models/README.md). Hardware review uses [Secure Terminal](secure_terminal/README.md), [duress protection](duress_protection/README.md), and the specification's Boomlet sections. Usability review uses the [setup](setup/README.md) and [withdrawal](withdrawal/README.md) ceremonies. |
