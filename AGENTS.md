# AGENTS.md — Adv360 Pro ZMK

Repository for the Kinesis Advantage 360 Pro keyboard running ZMK firmware.

## Repository layout

- `config/adv360.keymap` — primary file to edit; defines all layers and behaviors
- `config/adv360_left.keymap` / `adv360_right.keymap` — both `#include "adv360.keymap"` (do not edit)
- `config/macros.dtsi` — predefined macros (`macro_ver`, `macro_kinesis`, Win/Mac shortcuts, etc.)
- `config/version.dtsi` — generated, may be empty/checked-in empty
- `config/boards/arm/adv360/` — board definition; do not edit unless changing hardware behavior
- `assets/key-positions.md` + `key-positions.png` — authoritative key-position numbering (0–75)
- `keyboardLayout/` — user's source-of-truth Clique layer screenshots (gitignored)
- `settings-reset.uf2` — flashable file that wipes settings flash (Studio keymap, BT bonds)
- `.github/workflows/build.yml` — produces two artifacts:
  - `firmware-no-clique` — plain ZMK, no Studio
  - `firmware-clique` — built with `-DCONFIG_ZMK_STUDIO=y` for the Kinesis Clique web UI

## Hard rules

1. **Each layer's `bindings = <...>` must have exactly 76 entries.** Validate after every edit:
   ```python
   import re
   c = open('config/adv360.keymap').read()
   for n, b in re.findall(r'display-name = "([^"]+)".*?bindings = <(.*?)>;', c, re.DOTALL):
       print(n, b.count('&'))
   ```
2. **Property names use hyphens, not underscores.** ZMK silently ignores unknown properties.
   - ✅ `quick-tap-ms`, `tapping-term-ms`, `require-prior-idle-ms`
   - ❌ `quick_tap_ms` (was a real bug in this repo's history)
3. **Do not commit Clique screenshots.** They live in `keyboardLayout/` (gitignored).
4. **Both halves run independent firmware.** Always flash left **and** right after a keymap change.

## Key positions (memorize the tricky ones)

```
Row 1:   0  1  2  3  4  5  6        |  &mo3  7  8  9 10 11 12 13
Row 2:  14 15 16 17 18 19 20        |        21 22 23 24 25 26 27
Row 3:  28 29 30 31 32 33 34 |center cluster 35 36 37 38|         39 40 41 42 43 44 45
Row 4:  46 47 48 49 50 51   |52 (L thumb small)| |53 (R thumb small)|  54 55 56 57 58 59
Row 5:  60 61 62 63 64       |65 66 67 (L thumb)| |68 69 70 (R thumb)|       71 72 73 74 75
```

Left-hand homerow: **A=29, S=30, D=31, F=32**.
Right-hand homerow: **J=41, K=42, L=43, ;=44**.

**Thumb cluster gotcha** (verified against `key-positions.png`, mistakes happened twice in this repo):
- 52 = small upper-LEFT thumb (Home)
- 53 = small upper-RIGHT thumb (PgUp)
- 65 = **big leftmost** left-thumb (Space)  ← easy to confuse with 67
- 66 = big-center left-thumb (Tab)
- 67 = small lower-LEFT thumb (End)
- 68 = small lower-RIGHT thumb (PgDn)
- 69 = big-center right-thumb (Enter)
- 70 = big rightmost right-thumb (Backspace)

## Layer indices (current keymap)

| Idx | Name    | Activation |
|-----|---------|------------|
| 0   | Base    | default |
| 1   | Fn/Num  | `&tog 1` at pos 6 (T1), `&mo 1` at pos 60 (M1 thumb), `&lt 1 RIGHT` at pos 64 |
| 2   | FKeys   | `&mo 2` at pos 75 |
| 3   | Mod     | `&mo 3` at pos 7 (Bluetooth, bootloader, RGB, backlight, battery) |
| 4   | Sym     | `&lt 4 G` at pos 33 and `&lt 4 H` at pos 40, plus `&mo 4` at pos 34 |
| 5   | Nav     | `&lt 5 SQT` at pos 45 (apostrophe key, tap = `'`) |
| 6   | Mouse   | `&lt 6 DOWN` at pos 72 |

**Reading Clique screenshots:** the badges `M1`, `M2` … `M6` next to a key
mean *that key is the layer activator*. If the badge is on a letter (e.g.
"G M4"), the binding must be `&lt LAYER LETTER`, not a bare `&mo` on the
neighbouring inner-cluster key. Putting `&mo 5` on the apostrophe key
*works* but eats the `'` character — use `&lt 5 SQT`.

## Homerow mods — battle-tested settings

```dts
hm: homerow_mods {
    compatible = "zmk,behavior-hold-tap";
    label = "HOMEROW_MODS";
    #binding-cells = <2>;
    require-prior-idle-ms = <250>;
    tapping-term-ms = <400>;
    quick-tap-ms = <175>;
    flavor = "tap-preferred";
    bindings = <&kp>, <&kp>;
};
```

**What each setting actually does:**
- `tapping-term-ms` — minimum hold duration before the modifier triggers. **Increase if a single letter goes missing during fast typing** (e.g. "last" → nothing because L was held > term and acted as RAlt).
- `require-prior-idle-ms` — if the previous keypress was within X ms, this key is forced to a tap. **Protects letters typed AFTER a homerow-mod letter, not the homerow-mod letter itself.**
- `quick-tap-ms` — re-pressing the same key within X ms is always a tap (useful for repeat-delete).
- `flavor = "tap-preferred"` — never trigger hold just because another key is pressed; only trigger if `tapping-term-ms` is exceeded.

**What does NOT work for cross-hand misfires:**
- ❌ Bilateral / positional homerow mods (`hold-trigger-key-positions`). They only fix *same-hand* misfires. If you set `L` to only trigger as RAlt when followed by a left-hand key, then "last" produces `RAlt+a, RAlt+s` (silent dead-keys). Verified to break typing in this repo.

## ZMK Studio — critical gotcha

The `firmware-clique` artifact has `CONFIG_ZMK_STUDIO=y`. ZMK Studio stores a runtime keymap in **settings flash** that **overrides the compiled-in `.keymap` at boot**. Symptoms:
- "I rebuilt and flashed but my changes aren't showing up"
- Old Clique GUI keymap keeps coming back

**Fix:** flash `settings-reset.uf2` once to wipe settings flash, then immediately flash your normal `.uf2`. Or switch to `firmware-no-clique`, which has no Studio and ignores settings flash for keymap.

You only need `settings-reset.uf2` when:
- Switching between `firmware-clique` and `firmware-no-clique` variants
- Studio overrides feel "stuck"
- BT pairing is broken

You do **not** need it for normal re-flashes of the same variant.

## Flashing workflow

1. Push branch → GitHub Actions builds both artifacts
2. Download `firmware-no-clique` (recommended for hand-edited keymaps)
3. Left half: `Mod + macro1` (\\ top-right) → mounts as USB → copy `*-left.uf2`
4. Right half: `Mod + macro3` → mounts as USB → copy `*-right.uf2`
5. Power-cycle if needed

Note: `Mod` here means the layer-3 momentary key (`&mo 3`), not a physical "Mod" label. In this keymap, that's the inner-top-right key on the right half.

## Common pitfalls

1. **Do not redefine `&mt` or `&lt`** in your behaviors block — overwriting predefined behaviors silently breaks them. Define new ones with different names (e.g. `hm`, `hml`).
2. **Hold-tap parameter count.** `&hm HOLD TAP` (2 params), `&lt LAYER TAP` (2 params), `&mo LAYER` (1 param). A hold-tap cannot wrap behaviors that take >1 parameter (e.g. `&bt BT_SEL 0`); use a macro instead.
3. **Layer key on its own physical key (e.g. position 40, the H key) needs `&lt 4 H`, not `&mo 4`** — otherwise you lose the letter `H` entirely. (Bug found by Opus review in this repo.)
4. **Layer references must point to existing layer indices.** `&mo 7` with only 7 layers (0–6) silently fails to compile.
5. **`PRCNT` not `PERCENT`, `SEMI` not `SEMICOLON`, `BSLH` not `BACKSLASH`, `DQT` not `DQUOTE`, `FSLH` not `SLASH`** — see `dt-bindings/zmk/keys.h`.
6. **Macro keys.** In Kinesis docs, "macro1/macro2/macro3/macro4" refer to the **physical** keys on the corners of each half (the inner-top keys), not to ZMK software macros. In this keymap, the bootloader is bound on the Mod layer at the `\` position (row 2 outer-right).
7. **Behavior references need the `&kp` prefix.** A bare `&UP` (instead of `&kp UP`) in a binding list causes a devicetree parse error: *"expected number or parenthesized expression"*. The leading `&` is the *behavior* (`&kp`, `&trans`, `&mo`, `&lt`, `&hm`, …); the keycode is its parameter.
8. **Reading shift-pair labels in Clique screenshots.** Each cell shows the shifted character above and the base character below. A cell that shows only one symbol is the base output. Two stacked horizontal marks in row 1 of the Sym layer that *look* like `=` are usually `_` over `-` (i.e. `&kp UNDER` or `&kp MINUS`); a real `=` is rendered as two lines very close together with the `+` cross above it (compare to pos 0 of the Base layer for the canonical look).

## Validation script (run before every commit)

```bash
python3 -c "
import re
c = open('config/adv360.keymap').read()
ok = True
for n, b in re.findall(r'display-name = \"([^\"]+)\".*?bindings = <(.*?)>;', c, re.DOTALL):
    cnt = b.count('&')
    print(('OK' if cnt==76 else 'FAIL'), n, cnt)
    if cnt!=76: ok=False
exit(0 if ok else 1)
"
```

## Useful external references

- ZMK hold-tap docs: https://zmk.dev/docs/keymaps/behaviors/hold-tap
- ZMK keycodes: https://zmk.dev/docs/keymaps/list-of-keycodes
- Nick Coutsos's online keymap editor (GUI alternative to hand-editing): https://nickcoutsos.github.io/keymap-editor/
- Kinesis Adv360 manual (physical key labels, reset buttons): https://kinesis-ergo.com/wp-content/uploads/Advantage360-ZMK-KB360-PRO-Users-Manual-v3-10-23.pdf
