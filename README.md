# IKEA BILRESA Scroll Wheel — Home Assistant blueprint (fixed, colour-capable)

A drop-in replacement for jhol-byte's `Ikea_bilresa_scroll_wheel` blueprint that **actually
works on Home Assistant 2026.x**, plus the two helper scripts its colour modes need.

Built and verified against 3× IKEA KAJPLATS E27 bulbs (Matter over Thread) on HA 2026.11.

## The problem with the upstream blueprint

Upstream derives the wheel's entity ids in `trigger_variables` from the selected device:

```yaml
var_button_ch1: >-
  {% for item in device_entities(var_remote) %}
    {% set match = item | regex_findall(find='^event\..+[a-zA-Z]_3.*$') %}
    {% if (match | length) > 0 %}
      {{ match[0] }}
    {% endif %}
  {% endfor %}
```

Folded multi-line Jinja, so every loop iteration emits whitespace and the rendered value is
`"        \n  event.some_remote_button_3\n    "` instead of `event.some_remote_button_3`.

Home Assistant's `event` trigger compares `event_data` with a plain dict comparison
(`event_data.items() >= event_data_items` in `components/homeassistant/triggers/event.py`) —
**no strip, no normalisation**. A padded value never matches, so *every trigger in the
blueprint is dead*: the automation loads, never fires, and logs nothing.

Verified on HA 2026.11 with real and synthetic `state_changed` events: a literal
`" light.x "` in `event_data` never fires, the bare id fires, and a padded
`trigger_variables` value never fires.

This repository renders the ids on a **single line**, whitespace-free.

## What else is different

| | upstream | this fork |
|---|---|---|
| triggers | dead on HA 2026.x | whitespace-free ids, fire correctly |
| scroll latency | `queued` but no default tuning | `queued`, `max_queued_automation_calls: 20`, no tail delay |
| default scroll source | `relaxed` (button counts) | `instant` (hidden switch-position sensors) |
| colour modes | call two scripts that are never published | same calls, scripts shipped in [`scripts/`](scripts) |
| colour defaults | `color_hue_step: 6`, `color_temp_step: 100` | `15` / `200` (both need far fewer detents) |
| metadata | no `author`, no `homeassistant.min_version` | both set |

`mode: queued` with no trailing `delay:` is what makes the wheel feel smooth. A blueprint that
uses `mode: single` plus a `delay:` in its action tail drops detents whenever they arrive
~0.5 s apart — that, not the radio or Thread, is the usual reason one wheel feels laggy while
another (a hand-written `queued` automation) feels instant.

## Install

1. **Enable the scroll sensors** (Settings → Devices & Services → Entities, search `bilresa`,
   show disabled). Each channel has two hidden `..._current_switch_position_N` sensors
   (channel 1 = endpoints 1+2, channel 2 = 4+5, channel 3 = 7+8). `instant` scroll mode needs
   them; `relaxed` mode does not.

2. **Import the blueprint** — one click:

   [![Open your Home Assistant instance and show the blueprint import dialog.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fandrey-skl%2Fha-bilresa-scrollwheel-blueprint%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fandrey-skl%2Fikea_bilresa_scroll_wheel.yaml)

   Or manually: Settings → Automations & Scenes → Blueprints → Import blueprint → paste the
   raw URL of `blueprints/automation/andrey-skl/ikea_bilresa_scroll_wheel.yaml`.

3. **Only if you use a colour mode** (`lights: color temp only`, `lights: color hue only`,
   `lights: color temp and color hue`): create the two scripts first. Settings → Automations &
   Scenes → Scripts → Add Script → ⋮ → **Edit in YAML** → paste
   [`scripts/light_color_hs_helper.yaml`](scripts/light_color_hs_helper.yaml) and
   [`scripts/light_color_temp_helper.yaml`](scripts/light_color_temp_helper.yaml).
   Without them the colour branches match, run and do nothing — silently, with no log entry.

4. Create the automation from the blueprint and pick your wheel.

## Suggested configuration

Three channels on one light group (what this fork was built for):

| Channel | Scroll function | Notes |
|---|---|---|
| 1 | `lights: dim` | `dim_step_pct: 10`, `dim_min_pct: 1`, `dim_turns_lights_on: true` |
| 2 | `lights: color hue only` | `color_hue_step: 15`, `color_saturation: 100` |
| 3 | `lights: color temp only` | `color_temp_step: 200`, `min_color_temp: 2200`, `max_color_temp: 4000` |

plus `scroll_wheel_mode_ext_ch{1,2,3}: instant` and `light_transition_duration: 0.15`.
Clicks can stay on `light.toggle`; a long-click of `100% + color_temp_kelvin: 3000` gives you a
reset to neutral white.

## Gotchas

- **Saturation, not hue, is what you see.** Reusing the light's current saturation makes a 15°
  step invisible (55°/34% → 69°/34% looks identical). `light_color_hs_helper` drives saturation
  from the `color_saturation` input for that reason.
- **Hue and colour temperature fight each other.** Setting `hs_color` puts the bulb in `hs`
  mode; setting `color_temp_kelvin` puts it back in `color_temp` mode. A colour set on the hue
  channel does not survive a colour-temperature channel adjustment. That is the bulb.
- **Entity ids are derived from the device**, via `device_entities()` and a regex on the
  `_1…_9` suffix. Renaming these entities breaks the blueprint for that device.
- **Bulb rounding**: the Kelvin clamp lands 1–2 K off the configured bound.
- **Traces**: only the last 5 runs are kept, and a frequent trigger evicts the interesting ones.
  Use the logbook to count triggers.

## Credits and licence

Fork of **Ikea_bilresa_scroll_wheel** by **jhol-byte** —
[forum thread](https://community.home-assistant.io/t/ikea-bilresa-scroll-wheel-blueprint-matter/965365),
[gist b2731a4d2476f530d76b9ff409f7f3a4](https://gist.github.com/jhol-byte/b2731a4d2476f530d76b9ff409f7f3a4), version 2026-04-06.

MIT for the changes in this repository (the whitespace fix, defaults, metadata, helper scripts and
docs). The upstream gist carries **no licence**, so its portions remain all-rights-reserved by its
author — nothing here claims authorship of them or grants rights over them. Full scope note at the
top of [LICENSE](LICENSE).
