# Keynote Notes — Organized
### Anna's keynote on foundational security

---

## Theme: The core cycle — novelty → heroics → baseline

Every generation of security challenges follows the same arc. A new threat emerges, heroics kick in at the moment of crisis, and over time the post-mortem of lessons learned converts those heroics into a new baseline. Then the cycle repeats.

> Example: Encryption was once a novel, celebrated capability. Today it's table stakes.

### Case study — the "I Love You" worm

Exposed critical gaps that became foundational practice after the fact: lack of network segmentation, no info-sharing mechanisms, no vulnerability disclosure programs. Out of it came threat intel, DNS reputation, threat hunting, models, pentesting.

---

## Lesson: Failures as forcing functions for innovation

**Netflix 2008** — a failed data center didn't just get patched. It pushed the company toward cloud-native architecture entirely. An incident treated as a signal, not just a fix.

**Bug bounty / Casey Ellis (Bugcrowd)** — aligning incentives beats punishing consequences. Collaboration over adversarial relationships unlocks better outcomes than the "catch and punish" model.

> Key framing: when an incident hits, ask not just *"how do we fix this?"* but *"is this a signal that our model needs to change entirely?"*

---

## Lesson: On-prem → cloud — you can't just lift and shift security

On-prem was appealing because it felt like maximum control. But applying on-prem security rules directly to cloud environments (lift and shift) fails — the threat model and architecture are fundamentally different. Cloud-native security required entirely new thinking, not migration of old practices.

> *"Innovate within a new boundary"* — cloud-native was the change, not a continuation of the old model.

**Parallel to now:** AI should not be treated as something that won't crash. The expectation should be that it will follow the same story arc — failures, corrections, eventual new baseline. The history of security gives us that plot line.

---

## Lens: Security as strategy, not afterthought

**Behavioral psychology + security** — humans are anomalous. They will not behave predictably. Security must be designed to anticipate human error, not just detect it. Human risk management as a discipline.

**Security culture shift** — security culture should be a frontline defense, not a retroactive call-out. No "slap on the wrist" model — the goal is building a culture that prevents, not punishes.

**Vendor / industry critique** — too many vendors build products nobody actually needs — appearance over substance. Practitioners are pushing back: holistic risk tolerance assessments, reallocation of focus to what actually matters. Less hype, more pragmatic solutions.

> *"Shipping fast is not going to last."*

---

## Optimism: Why the community should feel capable right now

- Lower barrier to entry
- Secure by default
- Automated legacy upgrades
- Legacy risk is tractable
- Security designed in, not bolted on

Security has never been better positioned — from design through implementation. The "impossible backlog" is becoming a solvable problem. Automated tooling is handling legacy upgrade burdens that used to be manual.

**The connective tissue: nonprofits and community orgs** — the scaffolding that holds all of this together is the nonprofit and community organizations that underpin the ecosystem. Don't overlook them.

> Adversaries are organized and they share — that is what makes them powerful. Knowledge sharing within the defender community is critical. "Early adopter tax" is real, but so is the compounding advantage of sharing early.

---

## Follow-ups & open questions

- Open question from the keynote: *"What actually matters?"* — worth sitting with as a filter for evaluating tools, vendors, and initiatives.