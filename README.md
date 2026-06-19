# Couples "Invite Your Partner" Validation Probe

**Purpose:** validate the **single riskiest assumption** behind the couples anger-management wedge *before* writing any couples code: **when partner #1 invites partner #2, does partner #2 actually JOIN?** If they don't, there is no two-sided lock-in, no moat, no product.

**This is a standalone web probe** (single `index.html`, no backend). It is NOT the app. It exists only to produce one honest GO/NO-GO number from real couples, cheaply.

> ⚠️ **A web GO is a CEILING, not a green light to build.** A web tap costs a fraction of a real install + onboarding + exposing private logs with your partner watching. So treat the result asymmetrically:
> - **Web NO-GO → trustworthy KILL.** If partners won't even tap "I'm in" on a free link, they won't install an app.
> - **Web GO → only unlocks the *next* gate** (a real in-app invite→install test). It never authorizes building the couples app on its own.

---

## The two-sided funnel

```
P1 (worried partner) lands  →  describes the "same fight" loop  →  creates a link
   →  shares it (SMS/copy)  →  P2 (other partner) OPENS the link  →  P2 confirms it's
   from their partner  →  P2 JOINS (leaves contact + taps "I'm in")
```

Everything is stitched with one `invite_id` carried in the URL (`?invite=<id>&from=<name>&a=<arm>`). P2 is a different person on a different device — `invite_id` is the only join key (no shared identity).

### Control arm (the most important design choice)
P1 devices are split **50/50**:
- **`couples`** — the invite-your-partner flow above.
- **`solo`** — "start it yourself," P1 just joins a list alone (no partner).

Without this, a high join rate could just be generic early-access interest. **If `solo` converts as well as `couples`, the dyadic mechanic adds no pull and the moat is dead** — regardless of how good the join rate looks.

---

## The gates (adversarially revised — these are the real ones)

The first-draft thresholds were tightened by a validity critic that found the probe, as first specced, leaned toward a **false GO** (a founder + 10 friendly couples could fabricate a pass in a weekend). The gates below close that.

**Primary metric = JOIN-GIVEN-OPEN** = `p2_joined / p2_invite_opened`, restricted to **cold strangers** (not your audience/friends), **confirmed-partner** taps only, surviving a 20% seed haircut and removal of your single biggest P1. (Created-to-joined is demoted — it's polluted by SMS/link delivery loss.)

| Outcome | Condition |
|---|---|
| **PASS → unlock the in-app install test** (NOT "build") | join-given-open **≥ 50%** point estimate **AND** Wilson 95% CI lower bound **≥ 30%**, **AND** couples-vs-solo **lift ≥ 15 absolute points** |
| **KILL — thesis falsified** | join-given-open **< 25%** among cold strangers who *opened and read the ask* (every excuse removed: they saw it, the frame was non-shaming, they still said no) |
| **DEAD ZONE — iterate once, re-run** | 25–50%, or CI straddling 30%, or lift < 15 pts. Change the P2 ask copy once and re-run. A second inconclusive read = treat as KILL. |
| **META-KILL — no P1 demand at all** | **< 30 invites created** in the recruiting window despite the TikTok scripts + any ASA test running → the worried-partner top-of-funnel doesn't exist at needed volume |

**Sample size:** **≥ 60 cold-stranger invites** created (target 90–120), from **≥ 20 distinct P1s**, **no single P1 > 3 invites**, **≥ 30 invites per arm**. Plan **600–1,200 P1 landers** (organic converts lander→invite in low single digits). **Budget 4–6 weeks** or a **~$300–500 paid push**; under that, accept "inconclusive" rather than reading noise off n=8.

**Window:** 21-day recruitment cutoff, 30-day measurement (slow "calm-moment" joins are real).

### What counts as a JOIN (strict)
ALL must be true for an `invite_id`:
1. `p2_invite_opened` fired with an `invite_id` that exists in `invite_created` (real invite, not a hand-typed/crawler URL);
2. P2 left a syntactically valid email/phone **not identical to P1's**;
3. P2 tapped the explicit "I'm in" commit (affirmative action, not a scroll);
4. Anti-fraud passes: honeypot empty (`honeypot_tripped=false`), dwell ≥ 4s (`dwell_ok=true`), and **not** P1 opening their own link (`is_creator_self_open=false`), and **confirmed-partner** (`p2_confirmed_partner=true`).

### Honest caveats (don't overclaim a GO)
- **Joiner identity isn't provable.** A friend/sibling/therapist — or P1 on a second phone over cellular — can tap join. The confirmed-partner step + cold-stranger sourcing reduce this, but web join is an **upper bound** on real partner activation.
- **Same-IP is a FLAG, not a reject.** Cohabiting couples share an IP — do *not* exclude shared-IP joins; flag them for inspection. (The old "reject same IP" rule was backwards.)
- **Seed/cluster inflates and the Wilson CI doesn't widen for it** — hence the 20%-haircut + drop-biggest-P1 sensitivity check baked into the gate.

---

## Setup (5 minutes)

1. **Analytics is already wired.** PostHog's free plan caps at 1 project (a dedicated one needs a paid plan), so the probe points at the **Healthy Anger app's own project** (key `phc_ym3Erc…`, already in `index.html`) — **not** HealthyOne's. Isolation is by tag: every event carries `probe_version='anger-couples-v1'`, and **every analysis query below filters on it**. The probe's event names (`invite_created`, `p2_joined`, …) don't collide with the app's, so the app's funnels are unaffected. *(If you ever upgrade PostHog, swap in a dedicated project key for total separation.)*
3. **Deploy** (static, free): drag the `couples-probe/` folder onto **[Netlify Drop](https://app.netlify.com/drop)** → you get a public URL. (Same path used for the Recipella probe and the JCC site.)
4. **Smoke-test before driving traffic** (see below) — a zero join rate from a wiring bug looks identical to a real NO-GO.

### QA / smoke test
- `…/?debug=1` shows an on-screen event log + the active arm.
- Force an arm: `…/?arm=couples` or `…/?arm=solo`.
- Walk P1 couples → create link → open the link in a *different browser/incognito* (so it's not flagged `is_creator_self_open`) → confirm → join. Verify in PostHog Activity that **`p2_joined`** appears. **If `p2_joined` never fires for your own test, the probe is silently dead** — fix before launch (it's the load-bearing event; `p2_invite_opened` and `probe_pageview` are the secondary tripwires).

---

## Instrumentation (events)

Every event carries `role`, `arm`, `invite_id` (where applicable), `session_id`, `probe_version`, `product`.

| Event | Fires when | Key props |
|---|---|---|
| `probe_pageview` | every load, right after init (the "script ran" heartbeat) | `page`, `referrer_source` |
| `p1_situation_started` | P1 focuses the "what's the fight" field | — |
| `p1_situation_submitted` | P1 advances past the situation step | `situation_length` (bucketed; never raw text) |
| `invite_created` | P1 mints the link (**DENOMINATOR**) | `invite_id`, `invite_url` |
| `invite_shared` | P1 taps SMS / Copy | `share_method` |
| `solo_joined` | *(solo arm)* P1 joins the list alone | `join_method` |
| `p2_invite_opened` | P2 opens the link | `is_creator_self_open`, `is_repeat_open`, `open_index` |
| `p2_confirmed_partner` | P2 taps "Yes, that's my partner" | `is_creator_self_open` |
| `p2_join_intent` | P2 focuses the contact field | — |
| `p2_joined` | P2 submits contact + "I'm in" (**NUMERATOR**) | `p2_confirmed_partner`, `join_method`, `honeypot_tripped`, `dwell_ms`, `dwell_ok`, `is_creator_self_open` |

---

## Analysis (HogQL — scope by `probe_version`, in the dedicated project)

**Primary metric — join-given-open, clean (excludes self-open / honeypot / fast-bot):**
```sql
SELECT
  arm,
  countDistinct(if(event='p2_invite_opened' AND properties.is_creator_self_open!=true,
                   properties.invite_id, NULL)) AS opened,
  countDistinct(if(event='p2_joined'
                   AND properties.is_creator_self_open!=true
                   AND properties.honeypot_tripped!=true
                   AND properties.dwell_ok=true
                   AND properties.p2_confirmed_partner=true,
                   properties.invite_id, NULL)) AS joined,
  round(joined / nullIf(opened,0), 4) AS join_given_open
FROM events
WHERE properties.probe_version='anger-couples-v1'
  AND timestamp > now() - INTERVAL 30 DAY
GROUP BY arm;
```

**Secondary — partner-2 join rate (created→joined), and funnel health:**
```sql
SELECT
  countDistinct(if(event='invite_created', properties.invite_id, NULL)) AS invites_created,
  countDistinct(if(event='invite_shared',  properties.invite_id, NULL)) AS invites_shared,
  countDistinct(if(event='p2_invite_opened' AND properties.is_creator_self_open!=true, properties.invite_id, NULL)) AS opened,
  countDistinct(if(event='p2_joined' AND properties.is_creator_self_open!=true, properties.invite_id, NULL)) AS joined
FROM events
WHERE properties.probe_version='anger-couples-v1' AND properties.arm='couples'
  AND properties.invite_id IN (SELECT properties.invite_id FROM events WHERE event='invite_created')
  AND timestamp > now() - INTERVAL 30 DAY;
-- share-through = invites_shared/invites_created ; join_rate = joined/invites_created
```

**Control-arm lift** = `join_given_open(couples)` vs the solo arm's equivalent P1→commit conversion (`solo_joined / probe_pageview[role=p1,arm=solo]`). Lift must be **≥ 15 absolute points**.

**Wilson 95% CI + sensitivity (run on the cleaned numbers):**
```python
# pip-free: stdlib only
from math import sqrt
def wilson(k, n, z=1.96):
    if n == 0: return (0.0, 0.0)
    p = k/n; d = 1 + z*z/n
    c = (p + z*z/(2*n))/d
    h = z*sqrt(p*(1-p)/n + z*z/(4*n*n))/d
    return (round(c-h,3), round(c+h,3))
# example: 18 joins of 32 opens -> point .563, check lower bound >= .30
print(wilson(18, 32))
# sensitivity: re-run after removing your single biggest P1's invites, and after a 20% seed haircut
```

---

## Distribution kit (drive the WORRIED partner — P1 — to the link)

**Creative rule:** frame the conflict as a **shared loop** ("we keep having the same fight"), never a defect in one person. Non-shaming = safe for P1 to send to P2 *and* safe for P1 to be seen engaging publicly. Shame kills reach; "this is just us, and it's fixable" spreads. Every CTA points at the probe link and asks P1 to **send it to their partner** (not just sign up alone).

### TikTok / Reels — Script 1 · "The same fight on a loop" (talking-to-camera)
- **HOOK (0–3s):** "We weren't fighting about the dishes. We've *never* been fighting about the dishes."
- **PROBLEM (3–10s):** "It's the same fight in a different costume. Voices go up, someone walks out, two days of silence, then we pretend it didn't happen. Reset. Repeat."
- **DEMO (10–20s):** "So I tried something dumb-simple. *I* describe the loop we keep getting stuck in — not 'who's the problem,' just the pattern — and it makes one calm thing I can send him so we're both looking at the same map instead of at each other."
- **CTA (20–28s):** "Free thing I'm testing, link in bio. If you and your person have a Fight On Repeat, go look. You don't have to be the 'calm one' to start it."
- **Caption:** "The Dishes Fight is never about the dishes 😭 #marriagetok #couplestherapy #relationshipadvice #samefight"

### TikTok / Reels — Script 2 · "POV: you're the one who Googles it" (text-over-broll, no face)
- **HOOK (0–3s):** "POV: it's midnight and you're the one secretly googling 'how to stop fighting with my husband.'"
- **PROBLEM (3–9s):** "He's asleep. You're awake doing the emotional admin of the whole relationship. Again."
- **DEMO (9–18s):** "Found a tiny free thing. Answer a couple questions about the *fight pattern*, then it gives you a link to send him. Not a lecture. Just: 'this is the loop — want to get out of it with me?'"
- **CTA (18–27s):** "If you've ever been the midnight-googler, it's in my bio. Send it to your person and see if they tap 'I'm in.'"

### Script 3 · "You don't have to be the calm one" + **Reddit**
- Script 3: short authority/empathy angle reinforcing that the recruiter doesn't have to be the patient one. (Full beat sheet in the workflow output.)
- **Reddit** (authentic, not spammy): a genuine first-person post in r/relationships / r/Marriage — "the one who always brings up the hard stuff" — ending with the probe link as "a free thing I'm testing, would love eyes." Read each sub's self-promo rules first.

**Riskiest distribution assumption:** that organic relationship-content can produce **cold strangers** (not your own audience) at the volume the sample needs inside the window. If it can't, the META-KILL fires for a *distribution* reason, not a demand reason — so a small paid push or a longer window is the honest backstop, and **cold-stranger sourcing is what keeps the join rate trustworthy.**

---

*Built from a 5-lens design workflow (experiment · copy · instrumentation · distribution · adversarial validity). Gates reflect the validity critic's revisions; do not loosen them — they're what stop a fabricated GO.*
