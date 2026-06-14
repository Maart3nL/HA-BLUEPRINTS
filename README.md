# IKEA STYRBAR (ZHA) — Smooth Dimming & Adaptive Colour

A Home Assistant **blueprint** that turns an IKEA **STYRBAR** remote (E2001 / E2002,
ZHA model *Remote Control N2*) into a full light controller: smooth brightness and
colour‑temperature ramps, double‑press groups, and a **time‑of‑day adaptive colour
temperature** that keeps your lights warm at night and cool by day — automatically.

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FMaart3nL%2FHA-BLUEPRINTS%2Fblob%2Fmain%2Fml_ikea_styrbar_zha_light_control_v2.yaml)

> **ZHA only.** This blueprint reads raw `zha_event` button codes and does not work
> with Zigbee2MQTT or deCONZ.

---

## What it does

- **Smooth ramps** — holding a button ramps brightness or colour temp gradually
  instead of jumping. Each light's starting value is snapshotted once and stepped
  predictably, and the loop stops early once every light reaches its bound.
- **Adaptive colour temperature** — on turn‑on *and* continuously over time, lights
  follow a day/night colour‑temp curve with a smooth sunrise/sunset blend. *(New in
  v3.2 — see [below](#adaptive-colour-temperature-v32).)*
- **Mixed‑light safe** — each light only receives the parameters it supports, so you
  can mix Hue, IKEA and plain on/off bulbs in one group.
- **Double‑press groups** — Up/Down double‑press control a second light group
  (optional, needs one helper).
- **Shared settings** — day/night temps, steps, floors, etc. live in helper entities,
  so changing one helper updates every STYRBAR automation at once. Each can be
  overridden per remote.

---

## Requirements

- Home Assistant **2024.10** or newer.
- The **ZHA** integration with a paired IKEA STYRBAR remote.
- The helper entities from [`styrbar_helpers.yaml`](styrbar_helpers.yaml) (created once,
  shared by all your STYRBAR automations).

---

## Button / action map

The STYRBAR has four buttons: **Up** (top), **Down** (bottom), **Left** and **Right**
(the side arrows).

| Button | Short press | Long press (hold) | Double press |
|---|---|---|---|
| **Up** | Turn **on** the *up‑short* lights | Brightness **up** (smooth ramp) | Turn **on** the *up‑double* lights |
| **Down** | Turn **off** the *down‑short* lights | Brightness **down** (ramp; floors, never off) | Turn **off** the *down‑double* lights |
| **Left** | Colour temp **warmer**, one step | Colour temp **warmer** (smooth ramp) | — |
| **Right** | Colour temp **cooler**, one step | Colour temp **cooler** (smooth ramp) | — |

**Which lights each action uses**

- Up/Down short & double each use their own light selector.
- Colour temp (Left/Right) acts on the **union of the Up‑short + Up‑double** lights.
- Brightness up uses the Up‑short + Up‑double union; brightness down uses the
  Down‑short + Down‑double union.

---

## Setup

### 1. Create the helpers

Add the contents of [`styrbar_helpers.yaml`](styrbar_helpers.yaml) to your
`configuration.yaml` (or a [package](https://www.home-assistant.io/docs/configuration/packages/))
and restart Home Assistant. These hold the shared "(Synced)" settings.

### 2. Import the blueprint

Click the **My Home Assistant** badge at the top, or go to
**Settings → Automations & Scenes → Blueprints → Import Blueprint** and paste:

```
https://github.com/Maart3nL/HA-BLUEPRINTS/blob/main/ml_ikea_styrbar_zha_light_control_v2.yaml
```

### 3. Create an automation from it

Pick your STYRBAR device, assign the light groups, and (optionally) select the
`input_text` helper to enable double‑press and release events.

---

## Settings reference

### Per‑automation options (set in the automation)

| Option | Default | What it does |
|---|---|---|
| Turn‑on brightness | 100 % | Brightness applied when lights are switched on. |
| Transition time | 0.3 s | Fade for ramps, colour‑temp changes and turn‑off. |
| Turn‑on transition time | 0 s | Fade used only when switching **on** (some bulbs misbehave from off — keep 0 if so). |
| Long press – max iterations | 15 | Safety cap on ramp repeats while held. |
| Long press – loop delay | 150 ms | Delay between ramp steps. |
| Double‑press delay | 500 ms | Max gap between the two presses of a double‑press. |
| Debounce delay | 0 ms | Increase if you see duplicate events. |

### Synced settings (helper entities)

Read **live** at run time. Missing/unavailable helper → built‑in fallback. Point an
automation's "(Synced) … helper" override at a different entity to give one remote its
own values.

| Setting | Default helper | Fallback |
|---|---|---|
| Daytime colour temp | `input_number.styrbar_day_color_temp` | 3400 K |
| Night‑time colour temp | `input_number.styrbar_night_color_temp` | 2200 K |
| Daytime window start | `input_datetime.styrbar_day_start` | 06:00 |
| Daytime window end | `input_datetime.styrbar_day_end` | 19:00 |
| Colour temp step – short | `input_number.styrbar_color_temp_step_short` | 500 K |
| Colour temp step – long | `input_number.styrbar_color_temp_step_long` | 400 K |
| Brightness step (%) | `input_number.styrbar_brightness_step` | 8 % |
| Brightness floor (%) | `input_number.styrbar_brightness_floor` | 2 % |
| Adaptive only when off | `input_boolean.styrbar_adaptive_only_when_off` | off |
| **Adaptive colour temp follows time** | `input_boolean.styrbar_adaptive_ct_follow` | off |
| **Adaptive transition window (min)** | `input_number.styrbar_adaptive_transition_min` | 90 |

---

## Adaptive colour temperature (v3.2)

Set the daytime and night‑time temps and the start/end of your "day" window, and the
blueprint computes the colour temperature for the current time:

- **Day plateau** between *start + window* and *end − window* → daytime temp.
- **Night plateau** outside the day window → night‑time temp.
- **Smooth blend** across the transition window at each edge (a sunrise/sunset ramp).

With the defaults (06:00–19:00, 3400 K / 2200 K, 90‑minute window):

```
04:00 → 2200 K   06:45 → 2800 K   07:30 → 3400 K   12:00 → 3400 K
17:30 → 3400 K   18:15 → 2800 K   19:00 → 2200 K   22:00 → 2200 K
```

Set the transition window to **0** for a hard day/night switch (the pre‑v3.2 behaviour).

Turn **`input_boolean.styrbar_adaptive_ct_follow` on** to also apply this curve to
already‑on lights continuously (re‑checked every 5 minutes), not just at turn‑on. It
only touches lights that are on, support colour temp, and aren't already on target.

> **Note:** while *follow* is on, the timed update will pull a colour temp you set by
> hand with Left/Right back onto the curve within a few minutes. Turn the toggle off if
> you want manual colour‑temp changes to stick.

---

## Behaviour rules

- Brightness / colour‑temp adjustments only affect lights that are **already on** (and
  support the feature). Off or unsupported lights are skipped.
- Brightness floors at the configured floor (**never** turns a light off) and caps at 100 %.
- Colour temp is clamped to each light's supported min/max kelvin.
- The blueprint runs in `restart` mode: a new button press cancels an in‑progress ramp
  (this is how releasing a held button stops the ramp).

---

## Optional helper — double‑press & release events

Double‑press (Up/Down) and `ahb_controller_event` release events need an `input_text`
helper (`styrbar_last_event`, max length ≥ 100) to track state across restarts. Leave
the blueprint's "(Optional) Helper – Last Controller Event" field empty to skip them;
everything else still works.

---

## Changelog

**v3.2**
- Continuous adaptive colour temp: on‑lights follow the day/night curve over time, not
  only at turn‑on (`styrbar_adaptive_ct_follow`).
- Smooth sunrise/sunset blend across a configurable window
  (`styrbar_adaptive_transition_min`, default 90 min; 0 = hard switch). Turn‑on uses the
  same curve.

**v3.1**
- Mixed‑light safe turn‑on (each light gets only the params it supports).
- Less duplication; no redundant Zigbee traffic during ramps.
- Brightness step & floor are percentages.
- Per‑remote helper overrides.

**v3.0**
- Smooth brightness & colour‑temp ramps with early exit.
- Configurable transition; "apply turn‑on settings only when off" toggle.
