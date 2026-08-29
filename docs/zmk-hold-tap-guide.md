# ZMK hold-tap guide

One key. Two jobs.

- Tap action when key tapped.
- Hold action when key held.
- Decision controlled by timer, other keys, positions, and event order.

Official docs: [ZMK Hold-Tap Behavior](https://zmk.dev/docs/keymaps/behaviors/hold-tap)

## Built-in behaviors

### Mod-tap: `&mt`

Hold sends modifier. Tap sends key.

```dts
&mt LSHIFT A
```

Meaning:

- Hold: `LSHIFT`
- Tap: `A`

Default flavor: `hold-preferred`.
Default tapping term: 200 ms.

### Layer-tap: `&lt`

Hold activates layer. Tap sends key.

```dts
&lt 2 SPACE
```

Meaning:

- Hold: layer 2
- Tap: Space

Default flavor: `tap-preferred`.
Default tapping term: 200 ms.

## Core event model

Hold-tap starts undecided.
ZMK captures later key-position events while decision pending.
After decision, ZMK runs chosen action, then replays captured events.

Fast chord with `balanced`:

```text
hold-tap down -> target down -> target up -> hold-tap up
                                  ^ hold chosen
```

Works as modifier chord. Target press and release replay under modifier.

Opposite release order:

```text
hold-tap down -> target down -> hold-tap up -> target up
                                  ^ tap chosen
```

Produces tap action plus unmodified target.

No minimum delay needed between presses with `hold-preferred`.
Electrical event order still matters. Finger moves first does not guarantee switch actuates first.

## Flavor options

Set with:

```dts
&mt {
    flavor = "balanced";
};
```

### `hold-preferred`

Hold wins when:

- Tapping term expires.
- Another key goes down.

Tap wins when hold-tap releases first.

Fastest normal modifier chords.
Also easiest accidental modifier during rolling typing.

QMK equivalent: roughly `HOLD_ON_OTHER_KEY_PRESS`.

### `balanced`

Hold wins when:

- Tapping term expires.
- Another key goes down and back up while hold-tap stays down.

Tap wins when hold-tap releases before interrupting key.

Good roll protection.
Release order becomes important.
Adds decision latency, but captured keyboard events preserve final chord when order correct.

QMK equivalent: roughly `PERMISSIVE_HOLD`.

### `tap-preferred`

Hold wins when tapping term expires.
Other key press does not force hold.

Tap wins when hold-tap releases before timer.

Strong tap bias.
Poor fit for very fast modifier chords unless user holds past timer.

QMK equivalent: roughly default QMK hold-tap policy.

### `tap-unless-interrupted`

Hold wins only when another key goes down before timer.
Timer expiry chooses tap.

Fast chords. No duration-only hold.
Useful when hold action always accompanies another keyboard key.
Bad when hold action needed alone or with event type not treated as interrupt.

## Timing options

### `tapping-term-ms`

Time before duration can decide hold.

```dts
&mt {
    tapping-term-ms = <200>;
};
```

Built-in `&mt` and `&lt` use 200 ms by default.
Custom hold-tap should set value explicitly.

Lower value:

- Faster timer-based hold.
- More accidental holds.
- Harder clean taps.

Higher value:

- Easier taps.
- Slower duration-only hold.
- More dependence on interrupt flavor.

Term measures hold-tap press duration. Not delay between hold-tap and target key.

### `quick-tap-ms`

Recent tap followed by same hold-tap press inside window forces tap.
Second tap stays pressed until physical release, allowing normal key repeat.

```dts
&mt {
    quick-tap-ms = <175>;
};
```

Useful for Backspace, Space, and repeated letters.
Can block intended modifier on quick second use.
Disabled by default. `0` gives no practical quick-tap window.

ZMK measures first press to second press.
QMK `QUICK_TAP_TERM` measures first release to second press.
Same numeric value does not mean same behavior.

### `require-prior-idle-ms`

Recent non-modifier key press can force immediate tap.

```dts
&mt {
    require-prior-idle-ms = <125>;
};
```

Example:

```text
A down -> 80 ms -> home-row mod down
                   ^ forced tap when prior-idle term = 125 ms
```

Good defense against fast typing rolls.
Also can reject intentional modifier immediately after typing.
Higher value means stronger tap bias.
Disabled by default.

Only prior non-modifier keycode matters. Modifier events do not trigger rule.

## Positional hold-tap

### `hold-trigger-key-positions`

List key positions allowed to trigger hold.
First relevant key outside list forces tap before tapping term expires.

```dts
hml: home_row_mod_left {
    compatible = "zmk,behavior-hold-tap";
    #binding-cells = <2>;
    bindings = <&kp>, <&kp>;
    flavor = "balanced";
    tapping-term-ms = <280>;
    hold-trigger-key-positions = <6 7 8 9 10 11>;
};
```

Common home-row setup:

- Left-hand behavior lists right-hand positions.
- Right-hand behavior lists left-hand positions.
- Cross-hand chord becomes modifier.
- Same-hand roll becomes tap.

Position numbers start at zero.
Order follows active matrix transform/keymap binding order.
Do not copy position list from another keyboard blindly.

Position rule only changes decisions before tapping term expires.
Holding past term can still produce hold with `hold-preferred`, `balanced`, or `tap-preferred`.

### `hold-trigger-on-release`

Delay positional evaluation until interrupting key release.

```dts
hml: home_row_mod_left {
    /* Other hold-tap properties... */
    hold-trigger-key-positions = <6 7 8 9 10 11>;
    hold-trigger-on-release;
};
```

Useful for stacking multiple home-row modifiers.
Held second modifier does not force early positional tap.
First released interrupt drives position decision.

No useful effect without `hold-trigger-key-positions`.

## Early-hold options

### `hold-while-undecided`

Press hold action immediately while decision still pending.
If final result becomes tap, ZMK releases hold before sending tap.

```dts
&mt {
    hold-while-undecided;
};
```

Useful for:

- Shift-click.
- Modifier plus mouse action.
- Behavior needing immediate hold state.

Risks:

- Provisional Alt can activate menu.
- Provisional GUI can open launcher.
- Layer can briefly activate before tap correction.

Combo caveat: if hold-tap position belongs to combo, eager hold waits for combo timeout.

### `hold-while-undecided-linger`

Keep provisional hold active until after tap behavior releases.

```dts
&mt {
    hold-while-undecided;
    hold-while-undecided-linger;
};
```

Special case: hold and tap both manipulate same modifier.
Example: hold `LGUI`, tap sticky `LGUI`.
Prevents host seeing release-press gap.

Usually meaningless without `hold-while-undecided`.

## Retro tap

### `retro-tap`

Timer may classify key as hold, but no other key used.
Release then emits tap action.

```dts
&mt {
    retro-tap;
};
```

Example: hold `&mt LSHIFT A` past term, press nothing else, release. Output becomes `a`.

Useful when accidental long solo press should remain letter.
Not useful for Shift-click: hold action waits until another keyboard position event.

## Custom hold-tap behavior

Use custom node when different keys need different policy.
One global `&mt` policy often too blunt.

```dts
/ {
    behaviors {
        hrm: home_row_mod {
            compatible = "zmk,behavior-hold-tap";
            #binding-cells = <2>;
            bindings = <&kp>, <&kp>;
            flavor = "balanced";
            tapping-term-ms = <200>;
            quick-tap-ms = <0>;
        };
    };
};
```

Use:

```dts
&hrm LCTRL D
```

First parameter goes to hold behavior.
Second parameter goes to tap behavior.

Policy belongs to behavior node, not ordinary binding occurrence.
Need different policy? Define another named behavior.

## Underlying behavior limits

Hold-tap forwards one parameter to each child behavior.

Works directly:

```dts
bindings = <&kp>, <&kp>;
bindings = <&mo>, <&kp>;
bindings = <&kp>, <&caps_word>;
```

Behavior taking zero parameters still needs dummy binding argument:

```dts
caps: caps_hold_tap {
    compatible = "zmk,behavior-hold-tap";
    #binding-cells = <2>;
    bindings = <&kp>, <&caps_word>;
    flavor = "balanced";
    tapping-term-ms = <200>;
};
```

Use:

```dts
&caps CAPS 0
```

Behavior requiring more than one parameter cannot fit directly.
Wrap it in macro, then use macro as child behavior.

## Configuration scope

### Change all mod-taps

```dts
&mt {
    flavor = "hold-preferred";
    tapping-term-ms = <180>;
};
```

Every `&mt` binding receives policy.

### Change all layer-taps

```dts
&lt {
    flavor = "balanced";
    tapping-term-ms = <200>;
};
```

Every `&lt` binding receives policy.

### Change selected keys

Define custom behaviors:

- Home-row modifier behavior.
- Thumb modifier behavior.
- Layer-tap behavior.
- Per-hand positional behavior.
- Per-finger timing behavior.

No clean normal per-binding property override. Binding only supplies child parameters.

## Official timeless home-row recipe

Timing matters less. Hand position and release order matter more.

```dts
/ {
    behaviors {
        hml: home_row_mod_left {
            compatible = "zmk,behavior-hold-tap";
            #binding-cells = <2>;
            bindings = <&kp>, <&kp>;
            flavor = "balanced";
            require-prior-idle-ms = <150>;
            tapping-term-ms = <280>;
            quick-tap-ms = <175>;
            hold-trigger-key-positions = < /* right-hand positions */ >;
            hold-trigger-on-release;
        };

        hmr: home_row_mod_right {
            compatible = "zmk,behavior-hold-tap";
            #binding-cells = <2>;
            bindings = <&kp>, <&kp>;
            flavor = "balanced";
            require-prior-idle-ms = <150>;
            tapping-term-ms = <280>;
            quick-tap-ms = <175>;
            hold-trigger-key-positions = < /* left-hand positions */ >;
            hold-trigger-on-release;
        };
    };
};
```

Official docs currently show 150 ms in code but mention 125 ms in prose. Tune from measured typing, not prose typo.

Tradeoffs:

- Strong same-hand roll protection.
- Cross-hand modifiers preferred.
- Same-hand shortcut may require waiting past tapping term.
- Prior-idle rule can reject rapid intentional modifier.
- Quick-tap rule can reject rapid repeated modifier use.

## Practical profiles

### Shortcut-first

```dts
flavor = "hold-preferred";
tapping-term-ms = <150>;
quick-tap-ms = <0>;
```

Fast modifier on target key-down.
More accidental holds during typing rolls.

### Roll-safe

```dts
flavor = "balanced";
tapping-term-ms = <200>;
require-prior-idle-ms = <100>;
quick-tap-ms = <0>;
```

Release order matters.
Recent typing favors tap.

### Cross-hand timeless

```dts
flavor = "balanced";
tapping-term-ms = <280>;
require-prior-idle-ms = <125>;
quick-tap-ms = <175>;
hold-trigger-key-positions = < /* opposite hand */ >;
hold-trigger-on-release;
```

Best roll defense.
Requires mirrored modifier habits.

### Interrupt-only

```dts
flavor = "tap-unless-interrupted";
tapping-term-ms = <200>;
quick-tap-ms = <0>;
```

Fast chord.
Solo long hold becomes tap.

### Mouse-ready modifier

```dts
flavor = "balanced";
tapping-term-ms = <200>;
hold-while-undecided;
```

Immediate modifier for pointer action.
Possible provisional modifier side effects.

## Kconfig capacity settings

Normal users rarely touch these.

```conf
CONFIG_ZMK_BEHAVIOR_HOLD_TAP_MAX_HELD=10
CONFIG_ZMK_BEHAVIOR_HOLD_TAP_MAX_CAPTURED_EVENTS=40
```

- `MAX_HELD`: simultaneous active hold-taps.
- `MAX_CAPTURED_EVENTS`: events buffered while decision pending.

Defaults: 10 and 40.
Increase only after logs show exhaustion.
`CONFIG_ZMK_BEHAVIORS_QUEUE_SIZE` controls behavior/macro queue, not hold-tap captured-event array.

## Deprecated spellings and options

Avoid:

```dts
tapping_term_ms = <200>;
quick_tap_ms = <175>;
global-quick-tap;
```

Use:

```dts
tapping-term-ms = <200>;
quick-tap-ms = <175>;
require-prior-idle-ms = <175>;
```

`global-quick-tap` exists for old compatibility. Modern config should state `require-prior-idle-ms` directly.

## Alternatives to hold-tap

Hold-tap not only path.

- Sticky keys: one-shot modifier. No need hold chord.
- Combos: two-key modifier chord. No dual-role timer.
- Caps Word: avoid repeated Shift holds.
- Dedicated modifier layer: predictable, more layer switching.
- Dedicated physical modifier key: zero ambiguity.

## Tuning method

Change one axis at time.

1. Start `hold-preferred`. Confirm fast chords reliable.
2. If rolling letters cause modifiers, try `balanced`.
3. Add low `require-prior-idle-ms`, perhaps 80-125 ms.
4. Add positional lists only if mirrored cross-hand mods acceptable.
5. Add `quick-tap-ms` only for keys needing repeat.
6. Add eager hold only for mouse or proven early-state need.
7. Test press order and release order separately.

Test sequences:

```text
mod down -> target down -> target up -> mod up
mod down -> target down -> mod up -> target up
previous letter -> mod down -> target tap -> mod up
mod tap -> quick second mod hold -> target tap
same-hand chord
cross-hand chord
modifier + mouse click
```

## Debugging

If correct nested sequence still fails:

- Confirm flashed build matches source.
- Check switch actuation order, not finger contact order.
- Check debounce settings.
- Check split-half transport behavior.
- Enable ZMK debug logging.
- Look for hold-tap capacity or captured-event errors.

Older boards can enable USB logging with:

```conf
CONFIG_ZMK_USB_LOGGING=y
```

Use logging firmware for diagnosis. Disable noisy debug setup afterward.

## Sources

- [Official hold-tap guide](https://zmk.dev/docs/keymaps/behaviors/hold-tap)
- [Hold-tap devicetree schema](https://github.com/zmkfirmware/zmk/blob/main/app/dts/bindings/behaviors/zmk%2Cbehavior-hold-tap.yaml)
- [Hold-tap state machine](https://github.com/zmkfirmware/zmk/blob/main/app/src/behaviors/behavior_hold_tap.c)
- [Built-in behaviors](https://github.com/zmkfirmware/zmk/blob/main/app/dts/behaviors.dtsi)
- [Hold-tap tests](https://github.com/zmkfirmware/zmk/tree/main/app/tests/hold-tap)
- [Sticky keys](https://zmk.dev/docs/keymaps/behaviors/sticky-key)
- [Combos](https://zmk.dev/docs/keymaps/combos)
- [Caps Word](https://zmk.dev/docs/keymaps/behaviors/caps-word)
