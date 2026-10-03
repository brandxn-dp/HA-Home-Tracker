# HA-Home-Tracker

Recurring household chores ("Mow the Lawn", "Water Plants"…) shown as Bubble Card
buttons. Each card's bar fills (or empties) as the chore gets closer to due, can show
a subheading such as "Due in 3 days", and resets when you tap it.

<img src="docs/style-reference.png" alt="Bubble Card slider buttons used as the style reference" width="260">

This repo contains no custom integration. Scheduling is handled by
[Chore Helper](https://github.com/bmcclure/ha-chore-helper), an existing HACS
integration. This repo adds the missing piece: a progress percentage that a Bubble
Card slider can display.

| You want                                         | Provided by                                                   |
| ------------------------------------------------ | ------------------------------------------------------------- |
| Task name, frequency (daily/weekly/monthly/yearly) | Chore Helper                                                 |
| Pick which weekdays it repeats on                | Chore Helper (weekly → *Due day*)                             |
| Repeat on fixed days vs. a time after completion | Chore Helper (*Every [x] weeks* vs. *After [x] weeks*)        |
| Icon per task                                    | Chore Helper icon or the card's `icon:`                       |
| Bar shows progress until due, filling or emptying | `chore_progress` macro + Template Helper + Bubble Card slider |
| Optional "time until due" subheading             | `chore_due_text` macro in the card's `state_content`          |
| Tap to reset                                     | Card tap action → `chore_helper.complete`                     |

## Files

| File | Purpose |
| --- | --- |
| [`custom_templates/chore_tracker.jinja`](custom_templates/chore_tracker.jinja) | Jinja macros: `chore_progress` (0–100 %) and `chore_due_text` ("Due in 3 days") |
| [`examples/bubble-card-chore.yaml`](examples/bubble-card-chore.yaml) | One fully commented chore card |
| [`examples/bubble-card-chore-grid.yaml`](examples/bubble-card-chore-grid.yaml) | Two-column grid of four chores, including the "empty" and colour-by-urgency variants |

## Setup

### 1. Install the prerequisites (HACS)

- **Chore Helper**: HACS → Integrations → search "Chore Helper" (add
  `https://github.com/bmcclure/ha-chore-helper` as a custom repository if it doesn't appear),
  then restart Home Assistant.
- **Bubble Card**: HACS → Frontend → "Bubble Card".

### 2. Create a chore

Settings → Devices & Services → Helpers → **Create helper** → **Chore**.

- **Friendly name**: `Mow the Lawn` (creates `sensor.mow_the_lawn`)
- **Frequency**: one of
  - `Every [x] weeks`: tied to the calendar. A chore due every Saturday stays on Saturdays, even if you do it on Thursday.
  - `After [x] weeks`: tied to when you last did it. The next due date is counted from the completion date.
- **Icon**: optional (e.g. `mdi:mower`)
- Next page: **Due every/after** = `1`, **Due day** = the weekday(s) it repeats on (leave
  empty for "any day"), plus optional first/last month to make it seasonal.

The chore sensor's state is the number of days until it is due (negative = overdue).

### 3. Add the macros

Copy [`custom_templates/chore_tracker.jinja`](custom_templates/chore_tracker.jinja) to
`<config>/custom_templates/chore_tracker.jinja`. Create the `custom_templates` folder if
it doesn't exist. Then go to Developer tools → Actions and run
`homeassistant.reload_custom_templates`. No restart is needed.

### 4. Create a progress sensor per chore

Settings → Devices & Services → Helpers → **Create helper** → **Template** → **Template a sensor**:

- **Name**: `Mow the Lawn Progress` (creates `sensor.mow_the_lawn_progress`)
- **State template**:
  ```jinja
  {% from 'chore_tracker.jinja' import chore_progress %}
  {{ chore_progress('sensor.mow_the_lawn') }}
  ```
- **Unit of measurement**: `%`. Bubble Card reads a `%` sensor directly as the fill level.
- **State class**: `Measurement`

The value goes from `0` right after the chore is done to `100` at the start of the due day,
and it updates every minute. Until a chore has been completed once, there's no start date,
so a 7-day cycle is assumed. Pass a different length if needed:
`chore_progress('sensor.clean_gutters', 90)`.

### 5. Add the card

Dashboard → Edit → Add card → **Manual**, and paste
[`examples/bubble-card-chore.yaml`](examples/bubble-card-chore.yaml) (one card) or
[`examples/bubble-card-chore-grid.yaml`](examples/bubble-card-chore-grid.yaml) (a grid).
Replace the entity IDs with your own.

Card options to adjust:

| Option | Effect |
| --- | --- |
| `name`, `icon` | Card title and icon |
| `invert_slider_value: true` | Bar starts full and **empties** toward the due date instead of filling up |
| `state_content` | The subheading. Delete it to hide the subheading |
| `styles` → `.bubble-range-fill` | Bar colour. The grid example shows a green → amber → red version |
| `tap_action` / `button_action.tap_action` | Tap = `chore_helper.complete` (reset). Uncomment `confirmation:` to ask first |
| `hold_action` | Hold = more-info for the chore (history, last completed, next due) |

## Alternative without a progress sensor

If you don't want a Template Helper for each chore, point the card straight at the Chore
Helper sensor and set the scale yourself:

```yaml
entity: sensor.mow_the_lawn
min_value: 0
max_value: 7          # the chore's period in days
invert_slider_value: true
```

The bar then moves in whole-day steps, and `max_value` has to match each chore's period.
The progress-sensor approach avoids both limitations.
