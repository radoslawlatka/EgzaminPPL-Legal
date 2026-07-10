# Golden Exemplars — v2

**Frozen artifact.** Supply these as few-shot examples on **every** generation request, alongside the
style contract (`style-contract.md`). The model imitates the tone, length, and structure of examples far
more reliably than it follows rules — these are the strongest consistency lever you have.

A handful of shapes cover almost everything. Each shows the **raw Markdown** the generator should produce
for `Explanation (PL)` and `Explanation (EN)`. Keep them few and hand-approved; do not let them drift.

**What these exemplars demonstrate (v2):** every one **explains *why*** — none merely restates a fact.
Quantitative concepts give the **general formula with a legend first** and a worked line only after.
Regulatory facts **name their source**. Key terms are **bold**. A concrete real-world example grounds a
concept where it helps.

---

## Example 1 — Plain concept with its reason (the default shape)

*One paragraph, no extra blocks — but it gives the **reason**, not just the rule. This is what most
explanations look like.*

**PL**
```
Gdy dwa statki powietrzne zbliżają się na podobnej wysokości, pierwszeństwo ma ten po **prawej** stronie; drugi ustępuje, zmieniając kurs w **prawo** i przechodząc za nim. Reguła jest jedna i stała na całym świecie, dzięki czemu każdy pilot przewiduje manewr drugiego, zamiast oboje zgadywać — i odległość rośnie, zamiast maleć.
```

**EN**
```
When two aircraft converge at a similar altitude, the one on the **right** has priority; the other gives way by altering course to the **right** and passing behind it. The rule is single and globally fixed, so each pilot can predict the other's move instead of both guessing — and the gap opens rather than closes.
```

---

## Example 2 — Quantitative concept (general formula + legend first, worked line after)

*The general law comes **first**, with a legend; a worked example with representative numbers comes
**after**, never instead. The explanation is reused across reworded questions, so the symbolic formula —
not one arithmetic line — is the concept. Note the **datum** caveat: its position is set per aircraft type.*

**PL**
```
**Środek ciężkości** to punkt przyłożenia wypadkowej siły ciężkości statku powietrznego. Wyznacza się go jako sumę momentów wszystkich mas podzieloną przez masę całkowitą; moment pojedynczej masy to jej ciężar razy **ramię** — odległość od punktu odniesienia (**datum**), którego położenie określa producent danego typu statku powietrznego.

```formula
x<sub>CG</sub> = Σ(m<sub>i</sub> · d<sub>i</sub>) / Σm<sub>i</sub>
alt: Położenie środka ciężkości równa się sumie iloczynów każdej masy i jej ramienia, podzielonej przez sumę wszystkich mas.
```

gdzie: x<sub>CG</sub> — położenie środka ciężkości, m<sub>i</sub> — masa elementu, d<sub>i</sub> — ramię (odległość od datum).

Na przykład dla mas 155 kg na ramieniu 0,8 m i 320 kg na ramieniu 2,4 m:

```formula
x<sub>CG</sub> = (155 kg × 0,8 m + 320 kg × 2,4 m) / (155 kg + 320 kg) = 1,88 m
alt: Środek ciężkości równa się: w liczniku 155 kilogramów razy 0,8 metra plus 320 kilogramów razy 2,4 metra, w mianowniku 155 plus 320 kilogramów, co daje 1,88 metra.
```
```

**EN**
```
The **centre of gravity** is the point where an aircraft's resultant weight acts. It is found as the sum of the moments of all masses divided by the total mass; a single mass's moment is its weight times its **arm** — the distance from the reference point (**datum**), whose position the manufacturer fixes for each aircraft type.

```formula
x<sub>CG</sub> = Σ(m<sub>i</sub> · d<sub>i</sub>) / Σm<sub>i</sub>
alt: The centre-of-gravity position equals the sum of each mass times its arm, divided by the sum of all masses.
```

where: x<sub>CG</sub> — centre-of-gravity position, m<sub>i</sub> — item mass, d<sub>i</sub> — arm (distance from the datum).

For instance, with masses of 155 kg at an arm of 0.8 m and 320 kg at 2.4 m:

```formula
x<sub>CG</sub> = (155 kg × 0.8 m + 320 kg × 2.4 m) / (155 kg + 320 kg) = 1.88 m
alt: Centre of gravity equals: in the numerator 155 kilograms times 0.8 meters plus 320 kilograms times 2.4 meters, in the denominator 155 plus 320 kilograms, which gives 1.88 meters.
```
```

---

## Example 3 — Regulatory concept with a rule and a source (paragraph + takeaway + Sources)

*A regulatory fact **names its source**. Use a single `>` takeaway for the one rule worth isolating;
cite the governing instrument by label (a URL only if it is a stable, known link).*

> **Marker is fixed:** the ingester recognises only `## Sources` (English), so keep that literal heading
> even in Polish — the app renders a localized label. Translate only the link text and the note.

**PL**
```
**Dowódca statku powietrznego** odpowiada za wykonanie i bezpieczeństwo lotu oraz za osoby i mienie na pokładzie i ma ostateczną władzę nad statkiem powietrznym, dopóki nim dowodzi. Odpowiedzialność przypisano jednej osobie, aby w locie nigdy nie była rozmyta.

> W sytuacji niebezpiecznej dowódca może odstąpić od przepisów w zakresie niezbędnym dla bezpieczeństwa.

## Sources
- [Ustawa — Prawo lotnicze]() — odpowiedzialność dowódcy statku powietrznego
- [SERA.2010]() — responsibility of the pilot-in-command
```

**EN**
```
The **pilot-in-command** is responsible for the conduct and safety of the flight and for the persons and property on board, and holds final authority over the aircraft while in command. The duty rests with one person so that responsibility is never ambiguous in flight.

> In an emergency the pilot-in-command may deviate from the rules to the extent required for safety.

## Sources
- [Ustawa — Prawo lotnicze]() — responsibility of the pilot-in-command
- [SERA.2010]() — responsibility of the pilot-in-command
```

---

## Example 4 — Emphasis and indices (bold term + sub/sup in a formula + legend)

*Bold the key term; use `<sub>`/`<sup>` (or Unicode exponents) inside the formula; follow a symbolic
formula with a one-line legend (`gdzie:` / `where:`) defining each variable. A conceptual law needs no
worked line.*

**PL**
```
Siła nośna zależy od **ciśnienia dynamicznego**, które rośnie z kwadratem prędkości: przy stałym kącie natarcia podwojenie prędkości czterokrotnie zwiększa siłę nośną.

```formula
L = ½ · ρ · V² · S · C<sub>L</sub>
alt: Siła nośna L równa się: jedna druga razy gęstość powietrza ro, razy kwadrat prędkości, razy powierzchnia skrzydła, razy współczynnik siły nośnej C L.
```

gdzie: ρ — gęstość powietrza, V — prędkość rzeczywista, S — powierzchnia skrzydła, C<sub>L</sub> — współczynnik siły nośnej.
```

**EN**
```
Lift depends on **dynamic pressure**, which rises with the square of airspeed: at a constant angle of attack, doubling the airspeed quadruples the lift.

```formula
L = ½ · ρ · V² · S · C<sub>L</sub>
alt: Lift L equals one half times air density rho times airspeed squared times wing area times the lift coefficient C L.
```

where: ρ — air density, V — true airspeed, S — wing area, C<sub>L</sub> — lift coefficient.
```

---

## Example 5 — Concept grounded by a real-world example (mechanism + "for instance")

*Explain the **mechanism**, then anchor it with a concrete, real-world instance. No formula is needed
when the concept is qualitative — the example carries the understanding.*

**PL**
```
Wysokościomierz jest kalibrowany według **atmosfery wzorcowej (ISA)**, która zakłada stały pionowy gradient temperatury. Gdy powietrze jest zimniejsze od wzorcowego, jest gęstsze, ciśnienie spada z wysokością szybciej, a przyrząd — który mierzy tylko ciśnienie — **zawyża wskazanie**: statek powietrzny jest niżej, niż pokazuje wysokościomierz. Na przykład w zimowy dzień, lecąc ku masie zimnego powietrza na stałej wysokości wskazywanej, rzeczywista wysokość nad terenem stale maleje — stąd zasada „z ciepła w zimno, uważaj nisko".
```

**EN**
```
An altimeter is calibrated to the **International Standard Atmosphere (ISA)**, which assumes a fixed vertical temperature lapse rate. When the air is colder than standard it is denser, pressure falls off faster with height, and the instrument — which senses only pressure — **over-reads**: the aircraft is lower than indicated. For instance, flying on a winter day toward a cold air mass at a constant indicated altitude, the true height above terrain steadily decreases — the origin of the maxim *"from high to low, look out below"*.
```

---

## Example 6 — Visual concept with an image (paragraph + pending image)

*A **visual/spatial** concept (here, a component layout) gets one picture. Request it in place as an
`image` block with `key` / `alt` / `depicts` and **no URL**; the picture is attached later and the app
shows nothing until then. Use the **same `key`** in both languages. (Text/abstract concepts get no image.)*

**PL**
```
Usterzenie ogonowe zapewnia statkowi powietrznemu stateczność i sterowność wokół osi pionowej i poprzecznej. Statecznik poziomy ze sterem wysokości odpowiada za pochylanie, a statecznik pionowy ze sterem kierunku — za odchylanie.

```image
key: empennage-components
alt: Usterzenie ogonowe samolotu — statecznik poziomy i pionowy, ster wysokości, ster kierunku
depicts: części usterzenia ogonowego i ich rola w stateczności oraz sterowności
```
```

**EN**
```
The empennage gives an aircraft stability and control about the vertical and lateral axes. The horizontal stabilizer with the elevator controls pitch, and the vertical stabilizer with the rudder controls yaw.

```image
key: empennage-components
alt: Aircraft empennage — horizontal and vertical stabilizers, elevator, rudder
depicts: the empennage parts and their role in stability and control
```
```
