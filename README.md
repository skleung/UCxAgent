# UCx AI Agent — RN Culture Review Workflow

A browser-based, AI-assisted prototype that helps nurses catch urine culture discrepancies and take protocol-driven action — no server, no build step, just open `index.html`.

---

## Background

Presented as part of the SAEM 2026 Didactic:
**"AI in the Loop: Nursing-Led Urine Culture Stewardship"**

📊 **Slides:** [View presentation](https://docs.google.com/presentation/d/1ubtqu35hBLdYa7Kn1aez0YIQ8LQMBNHL/edit?slide=id.g3d94b645db6_0_94#slide=id.g3d94b645db6_0_94)

The prototype demonstrates a practical RN-in-the-loop workflow: when a urine culture finalizes, an AI agent checks whether the empiric antibiotic is still appropriate, surfaces the discrepancy, and guides the nurse through a structured review-and-act loop — either narrowing the antibiotic per protocol or escalating complex cases to an MD.

---

## Quick Start

```bash
# Clone
git clone https://github.com/skleung/UCxAgent.git
cd UCxAgent

# Open in browser — no server needed
open index.html       # macOS
start index.html      # Windows
xdg-open index.html   # Linux
```

Tested in Chrome, Safari, and Firefox. Works fully offline after first load (CDN assets cached by browser).

---

## Use Cases & Core Assumptions

### Target Users
- **Primary:** RNs working in ED, inpatient wards, or outpatient clinic review queues
- **Secondary:** Pharmacists, antimicrobial stewardship teams, and clinical informaticists evaluating AI-assisted workflows

### Key Use Cases

| Case Type | Distribution | Workflow |
|---|---|---|
| **Happy Path** — culture confirms empiric Rx | ~50% | RN marks reviewed; no change needed |
| **Discrepancy** — organism resistant to prescribed drug | ~30% | RN reviews AI recommendation, confirms oral switch per protocol |
| **Escalate to MD** — MDRO, pyelonephritis, no oral options | ~20% | RN sends pre-filled SBAR to MD; callback initiated if patient discharged |

### Clinical Assumptions
- **Scope:** Uncomplicated UTIs and lower UTIs where oral antibiotic step-down is safe per institutional protocol
- **Antibiogram sources:** Stanford Health Care and Harvard MGH institutional antibiograms (hardcoded; togglable in UI)
- **Escalation triggers:** MDRO organisms (MRSA, VRE, ESBL, CRE, MDR Pseudomonas), pyelonephritis indicators (fever, flank pain, CVA tenderness), IV-only organism profiles, allergy conflicts
- **RN authority:** Protocol-driven oral antibiotic changes (narrower spectrum, same indication) are within RN scope at many institutions — this prototype assumes an existing standing order or protocol framework
- **Not a replacement** for clinical judgment or pharmacy review; designed to surface information and reduce the cognitive load of cross-referencing culture results manually

### Antibiograms Included
- **Stanford Health Care** (13 organisms, oral + IV options)
- **Harvard MGH** (same 13 organisms, slightly different susceptibility profiles)

Organisms covered: *E. coli*, *Klebsiella pneumoniae*, *Pseudomonas aeruginosa*, *Proteus mirabilis*, *Enterococcus faecalis*, *Staphylococcus saprophyticus*, *Enterobacter cloacae*, MRSA, VRE (*E. faecium*), ESBL *E. coli*, ESBL *Klebsiella*, CRE *Klebsiella*, MDR *Pseudomonas*

---

## Architecture

### Tech Stack (CDN-only, zero build step)

| Concern | Solution |
|---|---|
| UI framework | React 18 via `unpkg.com` UMD build |
| JSX transform | Babel Standalone (`@babel/standalone`) |
| Styling | Tailwind CSS Play CDN |
| Icons | Heroicons SVG (inline) |
| State | `useState`, `useMemo`, `useCallback`, `useEffect` only |
| Data | Synthetic FHIR-inspired generator (no API calls) |

### Component Tree

```
App
├── Header
│   ├── InstitutionToggle  (Stanford ↔ Harvard MGH)
│   └── QueueStats         (Action Required / Cleared / Pending counts)
├── MainLayout
│   ├── PatientQueue       (left pane / mobile "Patients" tab)
│   │   └── PatientCard × N
│   └── WorkspacePane      (right pane / mobile "Review" tab)
│       ├── EmptyState
│       └── PatientWorkspace
│           ├── PatientHeader    (name, MRN, allergies, encounter context)
│           ├── AccordionSection: Epic Meds + Clinical Context
│           ├── AccordionSection: Urinalysis
│           ├── AccordionSection: Culture Results  (MIC table, color-coded S/I/R)
│           └── AI Decision Panel
│               ├── DiscrepancyAlert / AdmissionAlert
│               ├── RecommendationCards  (top 3, recalculated per antibiogram)
│               └── ActionRow
│                   ├── ChangeRxButton  → ChangeRxModal
│                   ├── PharmacyButton  → PharmacyModal
│                   └── EscalateButton  → EscalateModal (SBAR)
└── MobileBottomTabBar  (fixed, md:hidden)
```

### Data Flow

```
generatePatients(120)
  └── Patient objects (FHIR-inspired) stored in useState
        └── analyzePatient(patient, institution)
              ├── Looks up organism in ANTIBIOGRAMS[institution]
              ├── Checks active drug susceptibility
              ├── Detects allergy conflicts (with cross-reactivity)
              ├── Flags MDRO / pyelonephritis / IV-only profiles
              └── Returns { status, alert, recommendations, requiresAdmission }
                    └── Drives badge colors, button availability, SBAR content
```

### Key Design Decisions

**Single-file deliverable** — The entire app lives in one `index.html`. No build pipeline, no Node.js, no deployment infrastructure. Open the file in any modern browser and it runs. CDN assets load once and are cached.

**Synthetic data, not real EHR** — Patient data is procedurally generated at runtime using a seeded pool of realistic names, organisms, susceptibilities, and clinical contexts. No PHI; safe to demo anywhere.

**Immutable patient status** — Each patient's `status` and `escalateReasons` are set once at generation time and never mutated by the AI engine. `analyzePatient()` is a pure function used only for UI rendering (recommendations, alert copy). This prevents the "Change Rx button disappearing" class of bug where a dynamic re-analysis could silently re-classify a case.

**Batch loading** — 15 patients load per batch. When all Action Required cases in a batch are resolved, a banner prompts the RN to load the next 15. Simulates a realistic paced queue without overwhelming the demo screen.

**Institution toggle via `useMemo`** — Switching between Stanford and MGH antibiograms instantly recalculates all recommendations across the entire patient list via a single `useMemo` dependency, with no extra state.

**Mobile-first layout** — A single `mobilePane` state variable ("queue" | "workspace") drives full-screen pane switching below the `md` breakpoint. Above `md`, both panes are always visible side-by-side. A fixed bottom tab bar (iOS/Android pattern) provides navigation; tapping a patient auto-navigates to the workspace.

---

## Patient Data Model

```javascript
{
  id, mrn,
  name: { first, last },
  dob, age, sex,
  encounterDate,     // date of ED/clinic visit
  encounterMD,       // attending who saw patient
  isDischarge,       // true for 72% of patients
  unit,              // null if discharged; "3N-408" if inpatient
  phone,             // callback number for discharge follow-up
  allergies: [{ substance, severity, reaction }],
  activeMedication: { drug, dose, frequency, route, prescribedAt },
  urinalysis: { wbc, rbc, bacteria, nitrite, leukocyteEsterase, ... },
  culture: {
    organism, colonyCount, gramStain,
    susceptibilities: [{ drug, mic, interpretation }],  // S / I / R
    isMDR
  },
  clinicalContext,   // free-text paragraph
  status,            // "cleared" | "discrepancy" | "escalate" | "pending"
  escalateReasons,   // ["MDRO organism", "Pyelonephritis indicators", ...]
  requiresAdmission, // true when all susceptible options are IV-only
  actionTaken        // null | { type, drug, note, timestamp }
}
```

---

## Workflows

### Flow A — Discrepancy (oral Rx change)
1. RN sees red **Action Required** badge on patient card
2. Opens workspace → red "⚠ DISCREPANCY" banner identifies the mismatch
3. Culture Results accordion shows MIC table with current drug highlighted **R** (red)
4. AI panel surfaces top 3 oral alternatives ranked by susceptibility %
5. RN clicks **Change Rx** → modal shows current drug crossed out, recommends narrowest appropriate option, previews SBAR note
6. Confirm → patient badge turns green, chart note logged

### Flow B — Escalation (MDRO / complex)
1. Red badge patient, **Change Rx** button disabled ("Requires MD review")
2. RN clicks **Escalate to MD** → SBAR modal pre-filled with organism, resistance flags, clinical context
3. Confirm → patient marked **Escalated** (purple badge)

### Flow B2 — Escalation (IV-only / likely admission)
1. Amber **Admission Likely** badge in queue
2. Workspace shows amber banner: "⚠ No oral options — IV therapy required"
3. **Escalate** button highlighted amber; SBAR auto-includes admission recommendation and patient callback note
4. Confirm → patient marked **Escalated** (amber badge)

### Flow C — Ask Pharmacy
1. Available on any discrepancy case alongside **Change Rx**
2. Opens pre-filled consult request with organism, susceptibilities, and draft question
3. Sending sets patient to **Pharmacy Review** (teal badge) without resolving the alert

### Flow D — Happy Path
1. Green badge — culture confirms empiric antibiotic is appropriate
2. RN reviews data, clicks **Mark Reviewed**
3. No antibiotic change needed; note logged

---

## Limitations & Disclaimer

This is a **demo prototype** for educational and research purposes only.

- All patient data is entirely synthetic and fictional
- Antibiogram data is approximated from publicly available institutional reports; not suitable for clinical decision-making
- Does not connect to any EHR, pharmacy system, or clinical database
- RN scope-of-practice for antibiotic changes varies by institution and state — this prototype assumes a protocol/standing order framework exists
- Not FDA-cleared or intended for clinical use

---

## License

MIT
