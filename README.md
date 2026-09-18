[Uploading README.md…]()
# greenerafrik-8gate

GreenerAfrik Energies instance of the Tubalcain 8-Gate qualification system.

One self-contained `index.html`. No build step, no framework, no external
JavaScript. Everything a client instance normally needs is in the
`SITE_CONFIG` block at the top of the file.

---

# How to work on this file

**Test before you push.** Download `index.html`, open it in your browser
by double-clicking it, and run the whole assessment. It works from the
local file, nothing needs a server. GitHub Pages caches hard, so a change
that looks broken live is often just a stale cache. Hard refresh with
`Ctrl+Shift+R`, or `Cmd+Shift+R` on a Mac.

**Search, do not scroll.** Line numbers in this guide will drift as you
edit. Use `Ctrl+F` on the quoted code instead, and copy the old text
exactly, spaces included.

**The console commands you will need while testing.** Open the browser
console with `F12`, then Console.

```js
// Clear the lockout so you can run the assessment again
localStorage.removeItem('solar_gate_lock_v1__greenerafrik-ibadan');

// Clear the shuffled option order so you get a fresh order
sessionStorage.clear();

// See the lockout record currently held
localStorage.getItem('solar_gate_lock_v1__greenerafrik-ibadan');

// Fake a past disqualification, to test the return-visit screen.
// Reload after running this.
writeLock({ kind:'dq', dq:'dq-timeline', gate:'q4',
            value:'funds-building', at:new Date().toISOString(), attempts:1 });
```

Swap `greenerafrik-ibadan` for whatever `installerId` you set when you
duplicate this for another installer.

**Where things live in the file.**

| Roughly at | What it is |
| --- | --- |
| Line 11 | `SITE_CONFIG`, the only block you edit per installer |
| Line 101 | `brand`, the colour scheme |
| Line 1450 | `LOCK_DAYS`, the lockout window |
| Line 1566 | How many corrections a returning visitor is offered |
| Line 1649 | Which gates get their options shuffled |
| Line 2034 | `SPEND_MIN` and `SPEND_MAX`, the input sanity limits |
| Line 2129 | The Gate 2 spend threshold, currently ₦30,000 |
| Line 2486 | Delivery routing on the outgoing lead |

---

# Change 1: Deliver retry leads to the installer, flagged and unbilled

**Do this one.** Right now a visitor who was disqualified, used their one
correction, and then passed is not sent to GreenerAfrik at all. The
decision was to send them instead, flagged, and not charge for them.

Three edits, all inside `index.html`.

### Edit A: the outgoing lead

Search for `DELIVERY ROUTING`. Replace this whole block:

```js
    // ── DELIVERY ROUTING ──
    // deliverable false means this lead must NOT reach the installer and
    // must not be billed. It is set only when a previously disqualified
    // visitor used their one correction and then passed. The Make.com
    // router needs a filter on this ahead of the installer routes;  see
    // the README. Without that filter these leads still get through.
    deliverable: !isRetry,
    route: isRetry ? 'tubalcain-nurture' : 'installer',
    priorDQ: isRetry ? state.retryOfDQ : null,
    priorDQAnswerChanged: isRetry ? answerChanged : null,
```

with this:

```js
    // ── DELIVERY ROUTING ──
    // A retry lead IS delivered to the installer, but flagged and never
    // billed. billable false plus the priorDQ block is what the Make.com
    // router and the billing sheet read. To stop sending them entirely
    // instead, set deliverable back to !isRetry.
    deliverable: true,
    billable: !isRetry,
    route: 'installer',
    priorDQ: isRetry ? state.retryOfDQ : null,
    priorDQAnswerChanged: isRetry ? answerChanged : null,
```

### Edit B: the WhatsApp button on the result page

Search for `function bindWhatsAppCTA`. Replace these seven lines:

```js
  // A retry visitor still sees their real result, because the ROI and the
  // package are computed from their own figures and there is nothing
  // dishonest about showing them. The conversation goes to Tubalcain for
  // review rather than to the installer, who is neither sent this lead
  // nor billed for it.
  const retry = !!state.retryOfDQ;
  const num = retry ? SITE_CONFIG.installer.unqualifiedWhatsapp : SITE_CONFIG.installer.whatsapp;
```

with these three:

```js
  // A retry lead goes to the installer like any other, because it passed
  // every gate. The flag rides in the lead data, not in the buyer's own
  // message, and it is never billed.
  const retry = !!state.retryOfDQ;
  const num = SITE_CONFIG.installer.whatsapp;
```

Then, a few lines below in the same function, replace:

```js
  btn.addEventListener('click', () => {
    if (retry) {
      trackEvent('retry_whatsapp_clicked', { event_label: 'Retry lead opened nurture WhatsApp', content_id: resultType, content_type: 'gate_lockout' });
    } else {
      trackWhatsAppClick(resultType);
    }
  }, { once: true });
```

with:

```js
  btn.addEventListener('click', () => {
    trackWhatsAppClick(resultType);
    if (retry) {
      trackEvent('retry_whatsapp_clicked', { event_label: 'Flagged retry lead opened installer WhatsApp', content_id: resultType, content_type: 'gate_lockout' });
    }
  }, { once: true });
```

A flagged lead now counts in your normal WhatsApp click number, because
it is a real click on a real delivered lead, and it also fires its own
event so you can still separate them in GA4.

### Edit C: the event name, which currently says the wrong thing

Search for `RetryLeadSuppressed`. Replace:

```js
    fbq('trackCustom', 'RetryLeadSuppressed', { gate: state.retryOfDQ.gate, dq: state.retryOfDQ.dq });
    trackEvent('retry_lead_suppressed', {
      event_label: 'Passed after ' + (DQ_LABELS[state.retryOfDQ.dq] || state.retryOfDQ.dq) + ', not delivered',
```

with:

```js
    fbq('trackCustom', 'RetryLeadFlagged', { gate: state.retryOfDQ.gate, dq: state.retryOfDQ.dq });
    trackEvent('retry_lead_flagged', {
      event_label: 'Passed after ' + (DQ_LABELS[state.retryOfDQ.dq] || state.retryOfDQ.dq) + ', delivered flagged and not billed',
```

Safe to rename both because nothing has fired yet. Once the campaign is
live, renaming an event splits your reporting in two, so do it now or not
at all.

### Do NOT change this line

```js
  if (sent && !isAmber && !isRetry) fbq('track', 'Lead', { content_name: 'Solar 9-Gate Qualifier', lead_status: 'green', value: systemCost });
```

The `!isRetry` stays. The Meta `Lead` conversion must not fire on a retry
lead even though you are now delivering it. Let it fire and the algorithm
starts looking for more people who come back and change their answers.

### How to test Edit 1

1. Run the assessment and fail it deliberately at Gate 1 by picking
   "still building" for funds.
2. Reload. You should land on the same rejection screen with the
   correction link.
3. Click the link, answer everything so you pass.
4. On the result page, right-click the green WhatsApp button, copy the
   link, and check the number is **2349034308140** and not 2348029234994.
5. In the console, check the lead went out flagged:

```js
// Look in the Network tab for the hook.us2.make.com request and open
// its payload. You want to see:
//   deliverable: true
//   billable: false
//   priorDQ: { dq: "dq-timeline", gate: "q4", ... }
```

### The matching Make.com change

The lead now arrives on the normal installer route, so nothing has to
change for it to be delivered. What you need is a branch that stops a
flagged lead being counted as billable.

```
Route 1  (new, put it first)
  Filter: billable = false
  Actions: append to the billing sheet with a "flagged, not billed"
           column, notify yourself, and still forward to GreenerAfrik

Route 2..N  (your existing per-installer routes)
  Filter: billable = true  AND  installer_id = <the installer>
```

If you would rather keep one route per installer and not split it, add
`billable` as a column in the sheet and exclude it in your invoice
formula instead. Either works. What matters is that a flagged lead never
lands in the count you invoice against, and never counts toward the
30-in-30 guarantee.

Old leads sent before this change carry `deliverable: false` and no
`billable` field, so treat a missing `billable` as `true`.

---

# Change 2: Branding, for a new installer

`SITE_CONFIG.brand`, around line 101:

```js
brand: {
  navy:        "#001B3D",  // darkest surface: hero, result page
  navy2:       "#062A52",  // cards on the result page
  primary:     "#0081CC",  // buttons, chips, selected states
  primaryDark: "#00629B",  // hover, and text on pale washes
  wash:        "#EEF3F8",  // page background
  blueWash:    "#E4F0FA",  // pale accent panels
},
```

**How to pick the six values from one brand colour.** Say the installer's
brand green is `#2FA36B`.

| Value | How to get it | Example |
| --- | --- | --- |
| `primary` | The brand colour itself | `#2FA36B` |
| `primaryDark` | The same colour about 25% darker | `#1F7A4E` |
| `navy` | Very dark, same hue. Near black with a tint of the brand | `#14281F` |
| `navy2` | A step lighter than `navy` | `#1D3A2C` |
| `wash` | Near white with the faintest tint | `#EFF5F1` |
| `blueWash` | A pale but visible tint | `#E3F2E9` |

Two more values are worked out for you and you normally leave them alone:

- `--primary-light`, the accent on the dark result page, is `primary`
  lightened until it is readable against `navy`. A dark brand colour
  will not disappear.
- `--on-dark-muted`, the supporting text on dark surfaces, is a light
  tint of `navy2`, so it sits in the brand's own hue.

Override them only if the brand has exact values, by uncommenting:

```js
    // primaryLight: "#7FC4F0",
    // onDarkMuted:  "#C4D3E3",
```

Button label colour is decided at load. Hand it a pale brand colour like
a yellow and the label flips to near black by itself.

**Do not brand the result colours.** Green means passed, amber means one
item to confirm on site, red means rejected. They are hardcoded on
purpose. An installer whose brand is red would otherwise make a passing
result look like an error.

**How to test branding.** Open the file, then in the console:

```js
SITE_CONFIG.brand = { navy:"#14281F", navy2:"#1D3A2C", primary:"#2FA36B",
                      primaryDark:"#1F7A4E", wash:"#EFF5F1", blueWash:"#E3F2E9" };
applyBrand();
```

The page repaints instantly. Try it before you commit the values, and
check the hero, a question card, and the dark result page.

---

# Change 3: Duplicating this for a new installer

Only `SITE_CONFIG` changes. Everything else stays identical, which is the
whole point of the master.

| Field | What it does |
| --- | --- |
| `installer.businessName` | Appears throughout the page and in the consent line |
| `installer.whatsapp` | Where qualified buyers land. Format `234...`, no plus, no leading zero |
| `installer.unqualifiedWhatsapp` | Where every disqualified visitor goes. **Keep this as your own line** |
| `installer.cityLabel` | Long form, e.g. `Ibadan, Oyo State` |
| `installer.cityShortName` | Short form, used in option labels, e.g. `Ibadan` |
| `installer.locations` | The four named areas at Gate 2 |
| `installer.outOfAreaTerms` | Cities that disqualify if typed into the city field |
| `rep` | Name, title and photo of the person who appears to ask the questions |
| `tier1.packages` / `tier2.packages` | The installer's own price list |
| `warrantyMonths` | Badge on the result card |
| `financingLine` | Badge shown to part-payment and financing buyers |
| `packageInclusions` | What every package includes, shown as badges |
| `brand` | The colour scheme, see Change 2 |
| `installerId` | Routes the lead in the shared Make.com scenario. Must be unique |

Steps:

1. Create the new repository and copy `index.html`, `README.md` and
   `assets/` into it.
2. Replace `assets/rep.jpg` with the new installer's photo. Square crop,
   face centred, 400x400 or larger.
3. Edit `SITE_CONFIG` down the table above.
4. **Set a unique `installerId`.** It namespaces the lockout key as well
   as routing the lead, so two installers sharing an id would share a
   lockout.
5. Add the matching route in the Make.com scenario.
6. Test the whole assessment locally before you enable Pages.

---

# Change 4: Other things you will want to edit

### The named service areas

`installer.locations`, around line 26. Four is the right number. More and
the list stops being scannable on a phone, which is what makes people
click without reading. Anywhere else in the city is still covered by the
"Elsewhere in {city}" option, which reveals a typed field.

### The minimum monthly spend

Two places, and they must agree.

```js
  if (ans.spend < 30000) { trackDisqualification('dq-spend','q1',ans.val); showDQ('dq-spend'); return; }
```

That is the real gate, around line 2129. If you change the number, also
check the wording on the `dq-spend` screen so it does not contradict it.
`SPEND_MIN` and `SPEND_MAX` around line 2034 are only input sanity
checks, catching a fat-finger extra zero. They are not the gate.

### Packages

`tier1.packages` and `tier2.packages`. Keep them in **ascending price
order**, because the list is rendered in the order you write it and a
buyer scanning prices out of sequence reads the page as broken.

`belowMinimum: true` marks a package under the installer's real entry
system. `recoverable: true` sends that visitor to the instalment recovery
step instead of rejecting them. Every option renders identically on
screen, with no tell, so a rejected visitor cannot work out the rule and
retry with a different pick.

`tier2.packages` needs `kva` as a number, because the appliance load is
matched against it.

### Warranty, inclusions and financing line

```js
  warrantyMonths: 18,
  financingLine: "Go Solar. Pay Small Small.",
  packageInclusions: [
    "Free periodic system check",
    "24/7 after-sales support",
  ],
```

Only put things in `packageInclusions` the installer will actually
honour. This is the page the buyer screenshots and quotes back later.
Whatever you put here should match the wording in the Quotation Template
and the Authority Close Kit, or a buyer who reads all three will ask
which one is true.

### The lockout window

```js
const LOCK_DAYS = 30;
```

How long a disqualification is remembered. Thirty days outlives a single
ad flight without permanently burning someone whose circumstances change.

### How many corrections a returning visitor gets

```js
  if ((lock.attempts || 1) < 2) {
```

`< 2` gives one correction. `< 1` gives none, a hard block. `< 3` gives
two, which is too generous.

### Which gates shuffle their options

```js
  shuffleQuestionOptions('q3');   // whose permission
  shuffleQuestionOptions('q7');   // progress with other installers
  shuffleQuestionOptions('q8');   // payment method
  shuffleQuestionOptions('q5r');  // instalment recovery
  shuffleQuestionOptions('q4');
  shuffleQuestionOptions('q6', ['none']);
```

Delete a line to stop that gate shuffling. The second argument pins a
`data-val` to the bottom.

Two are deliberately absent and should stay absent. **The package ladder
is not shuffled**, for the price-order reason above. **The home check
pins "None of these" last**, because a fast clicker landing on it first
would flag their own roof for no reason and turn a clean lead into an
amber one.

---

# Reference

## Gate 2, location

Four named areas shuffled, then **Elsewhere in {city}**, then **Somewhere
else**.

Selecting *Elsewhere in {city}* reveals a text field. The entry is
validated for format and checked against `installer.outOfAreaTerms`, so
someone typing *Lagos* into a field labelled Ibadan is routed out rather
than delivered. `territory` in the lead reads as `Akobo, Ibadan`, with
the raw entry kept at `answers.city.typedArea`.

*Somewhere else* disqualifies.

## Option order

Shuffled per session, deterministically against a seed in
`sessionStorage`. That means going back through the gates shows the same
order rather than reshuffling under the visitor's hands, a reload inside
the same session keeps the order, a new session gets a new one, and
session resume still works because selection is bound to `data-val`
rather than to position.

The point is that no position is ever the safe click, and a rejected
visitor who reloads learns nothing from the layout.

Stop referring to options by number anywhere, including in the close
kit. "Option 3" no longer means anything. Use the wording or the
`data-val`.

## Gate lockout

A record in `localStorage` under
`solar_gate_lock_v1__<installerId>`.

**First return:** the visitor lands back on the rejection screen they
saw, with one "answered something by mistake?" link. Using it reopens the
flow and marks the attempt.

**Second return:** the same screen, no link.

**After `LOCK_DAYS`:** the record expires and they start clean.

**A visitor who already submitted** lands on an "already completed"
screen pointing at the installer's WhatsApp, rather than back at question
one. This stops a second lead for the same person, which is the usual
pay-per-lead billing dispute.

The key must stay namespaced by `installerId`. Every instance shares one
origin on GitHub Pages, so an unnamespaced key would lock a visitor out
of every other installer's page after one rejection here.

**What it stops, and what it does not.** It survives reloads, tab closes
and browser restarts, which covers the ordinary retry and most traffic
arriving through the in-app browser on a Facebook click. A private
window, cleared site data, another browser or another phone all defeat
it. Nothing client side can change that: the page posts nothing for a
disqualified visitor, so there is no server record to check against, and
the WhatsApp number that would identify a person is only collected at
Question 9, which a rejected visitor never reaches. That is the argument
for flagging as well as blocking.

## Lead payload, the fields that matter

| Field | What it holds |
| --- | --- |
| `installer_id` | Routes the lead |
| `active_tier` | Which product delivered this buyer |
| `firstName`, `whatsapp`, `location` | As typed |
| `territory`, `city` | Area inside the service zone |
| `contactConsent` + text + timestamp | Your consent record |
| `budgetRange` | The package selected or matched. **Never call this a confirmed budget** |
| `packageRange` | System size band |
| `matchedPackage` | Tier 2 only. Name, kVA, price, what it powers, undersized flag |
| `applianceLoad` | Tier 2 only. Appliances selected, total watts, required kVA |
| `packageBracketTag` | `instalment-recovered`, or empty |
| `timeline` | When funds become available, not install preference |
| `paymentMethod` | `full`, `part`, `financing` or `unsure` |
| `competitorStatus` | How far along they are with anyone else |
| `homeFlags` | Roof, shading or electrical items reported |
| `leadStatus` | `green`, or `yellow` when a home flag is present |
| `roi` | Payback years, month one saving, twenty year total, system cost, monthly spend |
| `deliverable` | False means do not send to the installer |
| `billable` | False means do not charge for it |
| `priorDQ` | Which gate rejected them, the answer, and when |
| `priorDQAnswerChanged` | Whether that answer changed, separating a mis-click from an edit |

## GA4 and Meta events

| Event | Fires when |
| --- | --- |
| `qualifier_started` | Page loaded, Question 1 visible |
| `qualifier_resumed` | Session restored mid-assessment |
| `qualifier_disqualified` | Rejected at a gate |
| `qualifier_completed` / `generate_lead` | Passed and submitted |
| `unqualified_whatsapp_clicked` | A rejected visitor opened your nurture line |
| `dq_lockout_shown` | A rejected visitor returned. `value` is the attempt number |
| `dq_correction_used` | They used their one correction |
| `retry_lead_flagged` | They passed on the retry. Named `retry_lead_suppressed` until you make Edit C |
| `retry_whatsapp_clicked` | A flagged retry lead opened WhatsApp |
| `duplicate_submission_blocked` | A visitor who already submitted returned |

Watch `dq_lockout_shown` against `qualifier_disqualified` in week one.
That ratio is how many people are trying the gates a second time, and it
is the number a hard block would have hidden from you.

## Tracking ownership

The campaign pixel stays Tubalcain's per SLA clause 12. Do not swap in a
GreenerAfrik pixel. Disqualified WhatsApp clicks fire their own events so
they can never be counted as delivered leads.
