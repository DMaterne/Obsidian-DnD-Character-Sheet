---
name: Athlete
type: feat
source: PHB 2014
choices:
  ability_increase:
    amount: 1
    options:
    - str
    - dex
effects:
- id: athlete_stand
  type: movement_rule
  value: stand_from_prone_costs_5ft
- id: athlete_climb
  type: movement_rule
  value: climbing_no_extra_movement
- id: athlete_jump
  type: movement_rule
  value: running_jump_requires_5ft
notes: 2014 feat backend entry for Athlete. Structured fields describe its character-sheet-relevant mechanics.
---
