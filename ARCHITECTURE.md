# Virtual Chemistry Lab: Architecture

A chemical mixing engine for Android, built with Flutter and the Flame game engine, aimed at public-school STEM students. This document describes how the pieces fit together so a team of beginners can build and extend it in parallel.

## 1. Goals and non-goals

**Goals**
- Mixing substances produces chemically correct results: right products, right ratios, right leftovers.
- Teach core stoichiometry: balanced equations, mole ratios, limiting and excess reactants, conservation of mass.
- Add new chemicals and reactions by editing data, not code.
- A working demo on Android quickly, with a codebase beginners can understand.

**Non-goals (for the demo)**
- Equilibrium and reversible reactions.
- Full thermodynamics or real kinetics. Rates are simplified.
- Automatic prediction of unknown reactions. Only reactions in the data file exist.
- Student-authored reactions (a different product; see Open Questions).

## 2. Core principles

1. **Chemistry is data, behavior is code.** Reactions live in JSON. The engine applies them. Nobody edits engine code to add a reaction.
2. **The engine knows nothing about Flame or Flutter.** It is a pure Dart package, easy to unit test without a device.
3. **Amounts are moles internally.** Convert to grams or milliliters only for display.
4. **Mixtures are sets of species, not labeled liquids.** "Vinegar" is water plus acetic acid, not a single substance. This is what makes potassium-in-vinegar and baking-soda-in-vinegar both come out right.
5. **No reaction is the default.** Only pairs listed in the data react.

## 3. Layers

```
┌──────────────────────────────────────────────┐
│ UI layer (Flutter widgets)                   │  menus, shelf, amounts, info panel
├──────────────────────────────────────────────┤
│ Game layer (Flame)                           │  beaker, particles, bubbles, colors
├──────────────────────────────────────────────┤
│ Engine layer (pure Dart)                     │  mixture state, reaction logic
├──────────────────────────────────────────────┤
│ Data layer (JSON assets + loader/validator)  │  species, reactions, presets
└──────────────────────────────────────────────┘
```

Dependencies point downward only. The engine never imports Flame, and the data layer never imports the engine.

### 3.1 Data layer

Two JSON files under `assets/data/`, loaded at startup.

**`species.json`**: every substance that can exist in a beaker.

```json
{
  "id": "nahco3",
  "name": "Baking soda",
  "formula": "NaHCO3",
  "atoms": { "Na": 1, "H": 1, "C": 1, "O": 3 },
  "molarMass": 84.01,
  "state": "solid",
  "color": "#FFFFFF"
}
```

`atoms` is stored explicitly so the validator doesn't need a formula parser (formulas like `Ca(CH3COO)2` with parentheses are a common source of bugs). A parser can replace this later.

**`reactions.json`**: every reaction the engine knows.

```json
{
  "id": "bicarb_vinegar",
  "type": "double_replacement",
  "reactants": { "nahco3": 1, "ch3cooh": 1 },
  "products": { "ch3coona": 1, "h2o": 1, "co2": 1 },
  "rate": 0.5,
  "conditions": [],
  "effects": [
    { "kind": "gas", "species": "co2" },
    { "kind": "temperature", "delta": -2 }
  ]
}
```

- `reactants` / `products` map species id to coefficient.
- `rate` is a simple speed constant (fraction of the available extent consumed per second) used for animation pacing.
- `conditions` are optional (for example `{"kind": "minTemperature", "value": 100}`). Empty means it reacts on contact.
- `effects` tell the game layer what to show. The engine only reports them; it does not draw anything.
- `type` is a teaching label only. The engine ignores it.

**`presets.json`** defines mixtures offered on the shelf, like vinegar:

```json
{
  "id": "vinegar",
  "name": "Vinegar",
  "contents": { "h2o": 53.0, "ch3cooh": 0.83 },
  "volumeMl": 1000
}
```

Contents are moles per preset volume, so vinegar is mostly water with a small amount of reactive acetic acid (about 5% w/v, roughly 0.83 mol per liter).

**Validator** (runs at load time, and as a unit test in CI):
- Every species id referenced in a reaction exists.
- Atoms balance on both sides of every reaction.
- No two reactions have identical reactant sets.
- Molar masses and states are present.

A bad entry should fail loudly with the reaction id in the message.

### 3.2 Engine layer

Pure Dart, no Flutter imports.

```dart
class Species { final String id, name, formula; final Map<String,int> atoms;
                final double molarMass; final SpeciesState state; }

class Reaction { final String id; final Map<String,int> reactants, products;
                 final double rate; final List<Condition> conditions;
                 final List<Effect> effects; }

class Mixture {
  final Map<String,double> moles;   // species id -> moles
  double temperature;               // degrees C
  void add(String id, double mol);
}

class ReactionEngine {
  ReactionEngine(this.reactions);
  StepResult step(Mixture m, double dt);   // advance by dt seconds
}

class StepResult { final List<ReactionEvent> events; final bool settled; }
```

**The step algorithm**

For each tick:

1. Find **active reactions**: every reactant present with moles > 0 and all conditions met.
2. Sort active reactions by a fixed order (rate descending, then id) so results are deterministic.
3. For each active reaction, in order:
   1. Compute the **maximum extent** (how many "reaction units" are possible): `min(moles[r] / coeff[r])` over its reactants, using the **current** moles after earlier reactions this tick.
   2. Compute the **step extent**: `min(maxExtent, rate * dt * maxExtent)`, or `maxExtent` if rate is instantaneous.
   3. Subtract `coeff * extent` from each reactant, add `coeff * extent` to each product.
   4. Record an event with the reaction id and extent.
4. If no reaction had extent above a small epsilon, mark the mixture **settled**.

Why this order matters: reactants are consumed from one shared pool, so a reactant used by two reactions can't be consumed twice (the potassium-in-vinegar problem). The first reaction in the order takes what it can; the next uses what's left.

**Worked check** (limiting reactant): 5 mol H₂ + 1 mol O₂ with 2H₂ + O₂ → 2H₂O. Max extent = min(5/2, 1/1) = 1. After: H₂ = 3, O₂ = 0, H₂O = 2. O₂ is limiting.

**Invariants to test**
- Atom counts across all species are identical before and after any step.
- No species goes below zero (clamp float error to zero).
- The same input always gives the same output.

**Display helpers** (also in the engine package): grams from moles, molarity from moles and volume, which reactant is currently limiting for a given reaction.

### 3.3 Game layer (Flame)

A `FlameGame` subclass that owns a `Mixture`, a `ReactionEngine`, and the visual components.

Components:
- `BeakerComponent`: the container, liquid level, and liquid color (a blend of species colors weighted by moles).
- `ParticleSpawner`: reads effects from `StepResult` and spawns visuals:
  - `gas` effect → bubble particles, spawn rate proportional to the extent this tick.
  - `precipitate` effect → solid particles that sink and settle.
  - `colorChange` → tween the liquid color.
  - `temperature` → thermometer reading, optional heat shimmer.
  - `flame` / `light` → for energetic reactions.
- `SolidComponent`: solids such as chalk or magnesium whose size shrinks as their moles drop.

Rules:
- Game loop: in `update(dt)`, call `engine.step(mixture, dt)`, then pass events to the visuals. The game layer reads engine state; it never changes chemistry itself.
- Visual intensity is derived from engine output (extent per tick), so the fizz fades naturally as a reactant runs out.
- Performance: cap particle counts (for example 200 bubbles) since this runs on low-end Android phones.

### 3.4 UI layer (Flutter widgets)

Flame widgets embedded in a normal Flutter screen:
- **Shelf**: scrollable list of species and presets; tap or drag to add to the beaker.
- **Amount control**: slider or stepper in grams or milliliters (converted to moles by the engine helpers).
- **Info panel**: shows the active equation, balanced, with coefficients, the limiting reactant, and what's left over.
- **Mass tracker**: total mass in the beaker plus mass escaped as gas, to demonstrate conservation of mass.
- **Reset** button.

State management: keep it simple. A single `ChangeNotifier` (or Riverpod provider) exposes the current `Mixture` snapshot to widgets. Avoid heavy architecture patterns for a beginner team.

## 4. Project structure

```
chem_lab/
├─ assets/data/
│  ├─ species.json
│  ├─ reactions.json
│  └─ presets.json
├─ packages/chem_engine/        # pure Dart, no Flutter
│  ├─ lib/
│  │  ├─ models/                # Species, Reaction, Mixture, Effect
│  │  ├─ engine.dart            # ReactionEngine
│  │  ├─ validator.dart
│  │  └─ units.dart             # moles <-> grams <-> molarity
│  └─ test/                     # unit tests (run without a device)
├─ lib/
│  ├─ game/                     # Flame: LabGame, components, effects
│  ├─ ui/                       # Flutter screens and widgets
│  ├─ state/                    # ChangeNotifier / providers
│  └─ main.dart
└─ ARCHITECTURE.md
```

## 5. Starter content

Water is the solvent for aqueous mixtures. Initial species (10):

| Species | Formula | Role |
|---|---|---|
| Hydrochloric acid | HCl | Strong acid hub |
| Acetic acid (vinegar) | CH₃COOH | Weak acid hub |
| Sodium hydroxide | NaOH | Strong base |
| Baking soda | NaHCO₃ | Carbonate, gas source |
| Washing soda | Na₂CO₃ | Carbonate, precipitates with metal salts |
| Chalk | CaCO₃ | Solid carbonate |
| Calcium chloride | CaCl₂ | Precipitate partner |
| Magnesium | Mg | Reactive metal |
| Zinc | Zn | Less reactive metal |
| Copper(II) sulfate | CuSO₄ | Displacement and precipitation |

Initial reactions (13, all balanced):

1. HCl + NaOH → NaCl + H₂O
2. HCl + NaHCO₃ → NaCl + H₂O + CO₂
3. 2HCl + CaCO₃ → CaCl₂ + H₂O + CO₂
4. Na₂CO₃ + 2HCl → 2NaCl + H₂O + CO₂
5. 2HCl + Mg → MgCl₂ + H₂
6. 2HCl + Zn → ZnCl₂ + H₂
7. CH₃COOH + NaOH → CH₃COONa + H₂O
8. CH₃COOH + NaHCO₃ → CH₃COONa + H₂O + CO₂
9. 2CH₃COOH + CaCO₃ → Ca(CH₃COO)₂ + H₂O + CO₂
10. CuSO₄ + Zn → ZnSO₄ + Cu
11. CuSO₄ + Mg → MgSO₄ + Cu
12. CuSO₄ + 2NaOH → Cu(OH)₂ + Na₂SO₄
13. CaCl₂ + Na₂CO₃ → CaCO₃ + 2NaCl

Note: reactions 3, 4, 5, 6, 9, and 12 have non-1:1 ratios. These are the cases a naive engine gets wrong, so they are the first test cases.

## 6. Testing strategy

- **Engine unit tests** (most important, no device needed):
  - Atom conservation for every reaction in the data file.
  - Limiting reactant cases (excess A, excess B, exact match).
  - Shared-reactant case: one reactant feeding two reactions is never over-consumed.
  - Determinism: same inputs, same outputs.
- **Data validation test**: runs the validator over all JSON files in CI.
- **Manual checks on a real Android device** for particle performance and touch input.

## 7. Division of work (suggested)

| Track | Skills needed | Tasks |
|---|---|---|
| Content | Chemistry, careful reading | Write and verify species/reactions JSON; check against a textbook |
| Engine | Basic Dart | Models, `step`, unit conversions, tests |
| Game | Flutter, Flame | Beaker, bubbles, color blending, solids |
| UI | Flutter | Shelf, sliders, info panel |

The content track needs no coding, and the validator gives them immediate feedback. The engine track is the best place to teach testing, since the invariants are easy to state.

## 8. Roadmap

1. **Milestone 1: engine only.** Data loader, validator, `step`, tests passing. A console script mixes two species and prints results.
2. **Milestone 2: one reaction on screen.** Beaker plus baking soda and vinegar with bubbles that fade as reactants run out.
3. **Milestone 3: full starter set.** All 13 reactions, shelf UI, info panel with limiting reactant.
4. **Milestone 4: teaching features.** Mass tracker, equation display, "why did this happen?" explanations.
5. **Later:** in-app content editor, class-based reaction rules, more chemicals.

## 9. Open questions

- **Curriculum fit.** Which topics and chemicals do the target schools actually cover? The set should be trimmed or changed to match lesson plans.
- **Who authors content?** If only the team, a spreadsheet or JSON is enough. If students invent reactions, the app needs a clear distinction between "real chemistry" and "made up."
- **Rate realism.** Single `rate` constants are a simplification. Is that acceptable for the target grade level?
- **Safety framing.** Reactions like alkali metals in water are dramatic but dangerous in real life. Decide whether to include them and how to label them.
- **Offline use.** Public-school devices may have poor connectivity; the design assumes everything ships in the app bundle.
