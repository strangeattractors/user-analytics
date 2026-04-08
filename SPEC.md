# Stats tab — product logic & implementation spec

**Mockup**: https://strange-attractor.com/user-analytics/
**iOS brand canon**: `YouSquared-ios/.claude/rules/ui-brand.md`
**Reviewers**: Joris (iOS implementation), AdB (data model + AI call prompts), own-call AI team (prompt inputs)

This doc is the source of truth for the product logic behind the Stats tab. The mockup at the URL above shows 3 personas × their sub-views. This doc covers the conditional logic, data model, and prompt inputs that the mockup only *implies* visually.

---

## 1. Hero metric — Lead value handled by your AI

### What it shows
Cumulative **all-time** $ lead value handled by the AI, as a single huge Lora number.

### Duration label
Replaces the v1 "Last 30 days" subtitle with the duration since the user's first handled call:
- `> 90 days`: "Last N years M months" (e.g. "Last 2 years 3 months")
- `≤ 90 days`: "Last N days" (e.g. "Last 47 days")
- `= 0`: hide hero number, render activation hero variant (see §1.5)

### Delta
`↑/↓ X%` vs previous 30 days of lead value accrual.
- `≥ 0`: coral (`#FF8370`), paired with a "Share with your boss →" CTA (invite-to-share)
- `< 0`: **orange** (`#F57C00`), paired with "Tap to see why →" CTA that scrolls down to the diagnosis card

### Cumulative area chart inside the card
Thin coral line + faint coral area fill, showing cumulative lead value over time. Not a sparkline of last-30-days (v1) — full history.

### Cold-start variant (§1.5)
When user has $0 lifetime lead value (no handled calls yet, or pricing field unset), replace the big number with a 3-step activation stepper inside the same dark gradient card:
1. Activate call forwarding
2. Share contacts
3. Confirm your VIPs

Each step is tappable → takes user to the relevant feature. Eyebrow text changes from "LEAD VALUE HANDLED BY YOUR AI" to "GET YOUR AI HANDLING CALLS".

### Data source
New backend endpoint `GET /v3/stats/hero`:
```json
{
  "all_time_lead_value": 342180,
  "first_handled_at": "2024-01-03T00:00:00Z",
  "delta_30d_pct": 0.34,
  "delta_direction": "accelerating" | "slowing" | "flat" | "cold_start",
  "series": [{"t": "...", "cumulative": 12300}, ...]
}
```

Lead value depends on the user's **Pricing dynamic field** (ref: `~/.claude/skills/web-analytics-dashboards/SKILL.md` § Equipment Rental Lead Value Attribution). If the pricing field is unset, the user sees an "Add your pricing to start tracking lead value" inline CTA instead of a number.

---

## 2. The 3 KPIs

Rendered as a 3-card grid directly under the hero. Each card is tappable → drill-down destination. Each card can render in coral/neutral ("healthy") or orange ("needs attention"). At most 1 small (i) info icon per card.

### 2.1 Channels activated

**Definition**: How many communication channels the AI is actively handling.

**Channel taxonomy**:
| Channel | Activation requirement | State |
|---|---|---|
| **Inbound calls** | Call forwarding enabled → Y² phone number active | `inactive` → `activating` → `active` → `degraded` |
| **Voicemail** | Automatically active once inbound calls are active | Mirrors inbound state |
| **Scheduled outbound** | Sub-industry selected → contacts shared → subscribed → scenarios activated | `inactive` → `ready` → `active` → `degraded` |
| **SMS (inbound)** | Future — blocked on text forwarding / number porting research | always `unavailable` for now |

**Card states**:
| State | Condition | Copy | Color |
|---|---|---|---|
| Green | All available channels active and healthy | "3 of 3 active" | neutral card |
| Orange — not all activated | 1+ channel inactive | "1 of 3 active" | **orange** |
| Orange — degraded | At least one active channel showing a health issue | "3 of 3 active • 1 issue" | **orange** |
| Cold start | All 3 inactive | "0 of 3 active" | **orange** |

**Orange triggers** (each links to a specific destination):
- Inbound calls: call forwarding not enabled → deep link to Call Forwarding settings
- Inbound calls active but volume dropped unexpectedly → AI call handling troubleshoot page
- Scheduled outbound: sub-industry not selected OR no contacts shared OR not subscribed → onboarding step
- Scheduled outbound: active but backlog of drafts waiting → link to To-Do tab
- Scheduled outbound: active but low volume → link to scenarios (more could be activated)
- Any channel: AI usage credits reached → upgrade CTA

**Tap destination**: Channels drill-down screen (see §5.3)

### 2.2 Seats activated

**Definition**: How many team members are contributing to the user's AI secretary account.

**Plan-aware copy**:
| Plan | State | Copy |
|---|---|---|
| Solo | 0 seats | "0 team members added" |
| Team (N max) | 0 added | "0 of N team members" |
| Team | M of N added, all active | "M of N active" |
| Team | M added but some pending | "M of N — 2 pending" |

**Invite lifecycle** (stored per-invite):
1. `invited` — email/SMS sent
2. `account_created` — invitee signed up
3. `subscribed` — invitee on a paid plan (or plan is funded by inviter)
4. `contributed` — invitee shared X contacts (count tracked)

**Solo → why team matters** (shown in Team drill-down):
- Team members can jump in faster on high-value time-sensitive situations (inbox access, notifications, live call transfer)
- Team members contribute their work contacts & contact events → expands reach of scheduled outbound
- Watch who opened invite → created account → subscribed → contributed N contacts

**Fund invitee's plan**: Inviter can pay for the invitee's subscription. Surfaced as a CTA on any row where `status === 'account_created'` and `subscribed === false`.

**Orange triggers**:
- Solo user: always orange if they're on a plan tier that could use team (not Free, not the cheapest tier)
- Team plan: 0 active members
- Team plan: invitees stuck at `account_created` for > 7 days without subscribing

**Tap destination**: Team drill-down (see §5.2)

### 2.3 Caller satisfaction

**Definition**: Weighted satisfaction score across all AI-handled calls.

**Rename**: Was "Satisfaction" → now "Caller satisfaction" everywhere.

**Weighting**: Each call's satisfaction score is weighted by:
1. Whether the caller is a VIP contact
2. The all-time $ lead value attributed to that contact

So 1 unhappy call from a $50k VIP contact weighs more than 10 happy calls from one-off contacts.

**Display**: Percentage (e.g. "94%") + small subtitle ("112 rated great"). The 4-segment distribution bar (Great / OK / Needs work / Unhappy) is shown on the drill-down, not on the card.

**Orange triggers**:
- VIPs not confirmed yet
- Contacts not shared
- AI personalization fields not filled in
- Raw weighted satisfaction below threshold (e.g. < 85%)

Each trigger maps to a specific activation step, surfaced in the drill-down.

**Tap destination**: Satisfaction drill-down (see §5.1)

---

## 3. Highlight calls — 2 cards under the KPIs

Two side-by-side cards sitting directly under the KPI row, above the fold on most devices:

1. **✨ AI Fascination call** — the most entertaining/novel call the AI has handled recently
2. **★ High-$ lead call** — a recent high-$ value lead the AI captured

Each card shows:
- Category eyebrow (AI FASCINATION / HIGH-$ LEAD)
- Contact name + avatar
- Short summary (1 line)
- Duration + ▶ play indicator
- Thumbnail (dark gradient with coral glow, same visual language as hero)

**Tap behavior**:
- Tap card body → scrolls the page down to the **"Recent AI Fascination"** or **"Recent Qualified Leads"** section (depending on which card). These sections show 3-4 calls each with **fully visible** summaries (current prototype truncates — fix this).
- Within those sections, each row has a ▶ Watch button that opens the Highlight Call Player (§5.4).

**Watched-call tracking**: The "featured" card on the main Stats screen is always the next unwatched call in that category. Once watched, next one slots in. Mechanics:
```
ranked_fascination_calls - watched_fascination_calls = queue
main_card = queue[0]
```

**Boosting AI fascination supply**: When the fascination queue is empty or sparse, show a small "Want more fascination calls?" nudge suggesting the user customize contact notes for specific contacts (those whose notes are empty but whose call volume is high) — this unlocks richer AI responses.

**Boosting high-$ lead supply**: Not a separate product mechanic — happens naturally as the AI handles more calls, but can also link to "Activate more channels" if the queue is empty.

**Cold start**: No highlight cards. Replaced with a single "activation nudge" card: "Your AI hasn't made waves yet. Here's how to get there →" linking to onboarding.

---

## 4. Communication handled by your AI (chart)

**Rename**: Was "Calls per cycle" → now "Communication handled by your AI".

**What it shows**: Stacked bar chart, 1 bar per billing cycle (or week/month depending on data volume), 6 bars visible with horizontal scroll for more (matches PR #144 `UsageHistoryView` pattern).

**Stack segments**:
- Inbound AI calls (bottom, primary coral)
- Voicemails (middle, lighter coral)
- Inbound texts (top, lightest — shows as "coming soon" / empty until SMS inbound launches)

**Missed calls — creative handling**:
Missed calls should NOT be a stacked segment because volume can be disproportionately large early on (e.g. 50 calls handled vs 200 missed because forwarding is misconfigured). Instead:
- Render missed calls as a **thin red line overlay** on top of the bars, scaled to the same y-axis
- Small legend dot: "— missed" alongside the stack legend
- If missed > handled, show a diagnostic banner above the chart: "Missed calls are outpacing handled — check your call forwarding →"

**Problem banners** (shown above the chart when applicable):
- "AI usage limit reached — upgrade for more" (if user hit plan credit cap)
- "Low conversion rate — your greeting sounds like a voicemail" (link to greeting optimization)
- "Missed calls are outpacing handled" (link to call forwarding troubleshoot)

**Data source**: Existing `UsageHistoryResponse` endpoint (PR #144) + new fields:
```json
{
  "history": [
    {
      "cycle_starts_at": "...",
      "cycle_ends_at": "...",
      "inbound_calls": 87,
      "voicemails": 12,
      "inbound_texts": 0,
      "missed_calls": 8,
      "entitlement": "..."
    }
  ]
}
```

---

## 5. Drill-down destinations

### 5.1 Satisfaction drill-down

- Header: "Caller satisfaction"
- Top: same 4-segment distribution bar (Great / OK / Needs work / Unhappy) in hero card size
- VIP-weighting callout: "Satisfaction is weighted by VIP contacts and their all-time lead value"
- Below: 4 expandable rows, one per category
  - Tapping "Unhappy" expands to show a list of unhappy-reason categories ("Wrong info given" / "Transfer failed" / "Didn't transfer" / "Caller frustrated" etc.)
  - Tapping a reason expands to show individual unhappy calls (avatar, date, 1-line summary, ▶ Listen button)

If the user has orange triggers on the satisfaction KPI, surface them above the distribution bar:
- "Confirm your VIPs → Satisfaction uses VIP weighting. Until you confirm VIPs, it's approximate."
- "Share your contacts → Your AI handles known contacts better."
- "Fill in AI personalization fields → Context boosts satisfaction."

Each is a coral CTA linking to the relevant feature.

### 5.2 Team drill-down

**Solo variant**:
- Big dark hero card (same gradient) with "Bring your team in" eyebrow + "Invite a teammate" primary CTA
- 3 explainer cards (warm surface, linear Solar icons):
  1. 🏃 **Faster on hot leads** — "Team members get instant notifications for high-value calls and can jump in live"
  2. 📇 **Richer contact intelligence** — "Every new teammate contributes their contacts, expanding your AI's reach"
  3. 📞 **Live call transfer** — "Route complex calls to the right person with one tap"
- Empty state: "No team members yet. Invite your first teammate →"

**Mature variant**:
- Member list, each row:
  - Avatar (curated palette)
  - Name
  - Status pill: `Invited` | `Account created` | `Subscribed` | `Contributed N contacts`
  - Relative timestamp ("2 days ago")
  - Action button (varies by status):
    - `Invited` + > 3 days → "Resend invite"
    - `Account created` → "Fund their plan" (coral CTA)
    - `Subscribed` → no CTA, success state
    - `Contributed N` → "View their contacts" (link)
- Footer: "Invite another teammate" CTA

### 5.3 Channels drill-down

3 channel cards stacked, plus a 4th "coming soon" card:

Each card:
- Channel name + activation status dot (green / orange / grey)
- Volume trend (tiny 7-day sparkline)
- One-line health summary
- Primary CTA (varies by state):
  - Inactive: "Activate →"
  - Active + healthy: "View details"
  - Degraded: descriptive action e.g. "14 drafts waiting — Go to To-Do"

**Inbound calls card**:
- Inactive: "Enable call forwarding →"
- Active healthy: "View call handling settings"
- Degraded (volume dropped): "Volume down 40% — troubleshoot"
- Degraded (usage cap): "Usage limit reached — upgrade"

**Voicemail card**: mirrors inbound call state (no independent activation)

**Scheduled outbound card**:
- Inactive (sub-industry missing): "Select your sub-industry →"
- Inactive (contacts missing): "Share your contacts →"
- Inactive (not subscribed): "Upgrade →"
- Ready (scenarios inactive): "Activate your first scenarios →"
- Active healthy: "View scenarios"
- Degraded (drafts waiting): "14 drafts waiting in To-Do →"
- Degraded (low volume): "Volume down — activate more scenarios →"

**SMS inbound card**:
- Always shows "Coming soon" with small "Text forwarding research in progress" subtitle

### 5.4 Highlight call player

- Dark gradient hero card at top with:
  - Category eyebrow (AI FASCINATION / HIGH-$ LEAD)
  - Contact name + ▶ play button (large, centered)
  - Duration + call date
  - 1-line summary
- Transcript preview scrollable below the player
- Action row: Download audio · Share · Post to social (Twitter / LinkedIn / TikTok)
- **Next up** section at bottom: 2 more highlight calls queued, horizontal-scroll cards
- Watched state: when user closes this screen, that call is marked watched → next call slots into the hero card on the Stats screen

---

## 6. Next milestone (optional section)

**Status**: Unclear whether to keep this. If kept:
- User can set their target ("What's your next goal?")
- Given target + current state, compute:
  - Projected calls/month needed at current rate
  - Team members / pooled contacts / channels that would help hit it faster
- "Book 30 min with an account manager" CTA (first session free, subsequent require an upgraded plan with white-glove support)

If removed, the space between KPIs/highlights and the Communication chart tightens.

---

## 7. Captured vs missed

**Status**: Needs research. Displaying "captured leads" accurately requires confirming the lead converted to revenue, which requires connecting to the user's CRM and/or payment processor (Stripe, Square, QuickBooks, etc.).

For v2, render the donut + stats with mock data and a footer note:
> "Connect your CRM or payment processor to confirm lead conversion →"

Make the footer tappable → placeholder "Coming soon" screen.

---

## 8. Own-call AI prompt inputs

These are the Stats-tab-specific values the own-call AI needs in its system prompt to guide the user conversationally. The voice AI currently uses the `base_own` prompt (`clone-backend/api-service/app/v3/prompts/base_own.py`) — these inputs would be injected into a new `stats_tab_context` block.

```python
stats_tab_context = {
    # Persona classification (derived)
    "persona": "mature_accelerating" | "mature_slowing" | "orange_mid" | "solo_cold_start",

    # Hero metric state
    "all_time_lead_value": 342180,
    "first_handled_at": "2024-01-03",
    "delta_30d_pct": 0.34,
    "delta_direction": "accelerating" | "slowing" | "flat",
    "hero_cta_state": "share_invite" | "diagnose_slowdown" | "activation_stepper",

    # KPI states — each is "green" or "orange" with a list of actionable reasons
    "channels_kpi": {
        "state": "green" | "orange",
        "active_count": 3,
        "total_count": 3,
        "orange_reasons": [],  # e.g. ["call_forwarding_off", "usage_cap_reached"]
    },
    "seats_kpi": {
        "state": "green" | "orange",
        "plan": "solo" | "team",
        "active_count": 4,
        "max_seats": 8,
        "orange_reasons": [],  # e.g. ["solo_no_invites", "invitees_stuck_at_account_created"]
    },
    "satisfaction_kpi": {
        "state": "green" | "orange",
        "weighted_pct": 0.94,
        "vips_confirmed": True,
        "contacts_shared": True,
        "ai_personalization_filled": True,
        "orange_reasons": [],
    },

    # Highlight calls
    "highlight_fascination_available": True,
    "highlight_fascination_next": {"contact_name": "...", "duration_s": 134, "summary": "..."},
    "highlight_lead_available": True,
    "highlight_lead_next": {...},

    # Channel-specific activation status
    "call_forwarding_enabled": True,
    "sub_industry_selected": True,
    "contacts_shared": True,
    "subscribed": True,
    "scenarios_activated": True,
    "outbound_backlog_count": 0,

    # Next milestone (if user has set one)
    "next_milestone_target": 500000,
    "next_milestone_progress_pct": 0.68,
}
```

### Example prompt snippets the AI can use

For `persona = solo_cold_start`:
> "You just opened the Stats tab and everything's at zero — that's expected, you're just getting started. The fastest way to see real numbers here is to activate call forwarding so your AI can start handling calls. Want me to walk you through it?"

For `persona = mature_slowing`:
> "Your lead value is down 8% this month — that's why the Stats tab is showing orange. Looks like it's because scheduled outbound volume dropped. You have 14 drafts waiting in To-Do that haven't gone out. Want to go through them together?"

For `persona = mature_accelerating`:
> "You're up 34% this month — $342k in lead value handled since you started. That's a big deal. The Stats tab has a 'Share with your boss' button right there. Want me to help you draft a quick update for them?"

The voice AI should proactively surface orange-state reasons with specific next actions, not generic encouragement.

---

## 9. Out of scope for v2 mockup

- iOS native Theme.swift additions (`.heroSerif()`, `.heroGradientDark()`) — tracked in ui-brand.md §7
- Backend data model changes — Joris + AdB to design once this mockup is approved
- `u2-issues-request` GitHub issue filing — will happen after mockup + spec review
- Captured vs missed CRM integration — research phase
- iOS simulator implementation — Joris owns after spec sign-off
- Own-call AI prompt migration — AI team owns after spec sign-off

---

## 10. File map

| File | Purpose |
|---|---|
| `index.html` | Visual reference (3 personas × sub-views) |
| `SPEC.md` | This file — product logic, data model, AI prompt inputs |
| `README.md` | One-paragraph pointer |
