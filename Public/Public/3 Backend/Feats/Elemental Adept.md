---
name: Elemental Adept
type: feat
source: PHB 2014
prerequisites:
  spellcasting: true
choices:
  damage_type:
    count: 1
    options:
    - acid
    - cold
    - fire
    - lightning
    - thunder
effects:
- id: elemental_ignore_resistance
  type: ignore_resistance
  target: chosen_damage_type
  scope: spells
- id: elemental_damage_floor
  type: damage_die_rule
  value: spell_damage_die_1_counts_as_2
  target: chosen_damage_type
notes: 2014 feat backend entry for Elemental Adept. Structured fields describe its character-sheet-relevant mechanics.
---
