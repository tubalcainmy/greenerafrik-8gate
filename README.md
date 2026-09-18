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

# Retry leads: delivered, flagged, not billed

**Already applied to this file.** A visitor who was disqualified, used
their one correction, and then passed is delivered to the installer like
any other lead, but flagged and never billed.

- the result CTA points at the installer's line
- the payload carries `deliverable: true` and `billable: false`
- `priorDQ` records which gate rejected them, the answer they gave, and
  when. `priorDQAnswerChanged` says whether the answer to that gate
  actually changed, which separates a mis-click from an edit
- the Meta `Lead` conversion does **not** fire, so the campaign is never
  optimised toward people who game the gates. A custom
  `RetryLeadFlagged` event fires instead

To stop sending them entirely instead, set `deliverable` back to
`!isRetry` in the DELIVERY ROUTING block.

### The Make.com change this needs

The lead arrives on the normal installer route, so nothing has to change
for it to be delivered. What you need is a branch that stops a flagged
lead being counted as billable.

```
Route 1  (new, put it first)
  Filter: billable = false
  Actions: append to the billing sheet with a "flagged, not billed"
           column, notify yourself, and still forward to the installer

Route 2..N  (your existing per-installer routes)
  Filter: billable = true  AND  installer_id = <the installer>
```

If you would rather keep one route per installer, add `billable` as a
column in the sheet and exclude it in your invoice formula instead.
What matters is that a flagged lead never lands in the count you invoice
against, and never counts toward the 30-in-30 guarantee.

Leads sent before this change carry no `billable` field, so treat a
missing `billable` as `true`.

# Branding

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

## Contrast is corrected for you, but not composition

Two values are fixed at load so a palette cannot make text unreadable:

- **the button label** flips between white and near-black by whichever
  actually reads on your `primary`, measured, not guessed
- **`primaryDark`** is darkened until it clears 4.5:1 as text, because it
  is used for the correction link and the pale badge as well as for
  button hover

Open the page with **`?brandcheck=1`** and the console prints a contrast
table for every pairing, naming the value to change. Do that before you
ship a palette.

What it cannot fix is composition. **`navy` and `navy2` should share a
hue.** Changing one and leaving the other is the most common mistake and
puts cards of one colour family on a background of another. `wash` and
`blueWash` should carry a faint tint of the same hue for the same reason.

`navy` also has to be genuinely dark, because it carries white text
across the hero and the whole result page. If `?brandcheck=1` says white
on navy is under 4.5:1, darken it.

## The browser tab icon

```js
faviconPath: "",
```

Left empty, `rep.photo` is used, so a client instance gets a branded tab
without a second thing to configure. The extension does not have to be
exact: if the file is a `.png` where the config says `.jpg`, both the rep
photo and the favicon fall back to the other extension by themselves.

A photo is a poor favicon at 16px and costs every visitor the full image
download, which on Nigerian mobile data is worth avoiding. For a live
page, save a small square crop as `assets/favicon.png` at 180x180 and
point `faviconPath` at it.

## The report number

```js
reportPrefix: "GA",
```

Appears on the result page as `#GA-20260918-4567`. Left empty, the
business name's initials are used. Set it per instance so a duplicated
page does not hand out another installer's report numbers.

# Duplicating this for a new installer

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
| `brand` | The colour scheme, see Branding |
| `installerId` | Routes the lead in the shared Make.com scenario. Must be unique |

**Start from the master, not from this file.** The master carries no
installer identity, so nothing of GreenerAfrik's can travel into another
installer's page by accident. It also runs in template mode until you set
a real `installerId`, which means it cannot post a lead or fire a pixel
event while you are setting it up.

Steps:

1. Create the new repository and copy the master `index.html`,
   `README.md` and `assets/` into it.
2. Replace `assets/rep.jpg` with the new installer's photo. Square crop,
   face centred, 400x400 or larger.
3. Edit `SITE_CONFIG` down the table above.
4. **Set a unique `installerId`.** It namespaces the lockout key as well
   as routing the lead, so two installers sharing an id would share a
   lockout.
5. Add the matching route in the Make.com scenario.
6. Test the whole assessment locally before you enable Pages.

---

# Other things you will want to edit

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
