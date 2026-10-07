---
name: Ritual Caster
type: feat
source: PHB 2014
prerequisites:
  any_ability_13:
  - int
  - wis
choices:
  ritual_class:
    count: 1
    options:
    - bard
    - cleric
    - druid
    - sorcerer
    - warlock
    - wizard
  ritual_spells:
    count: 2
    source: chosen_class_level_1_rituals
effects:
- id: ritual_book
  type: spellcasting_rule
  value: cast_recorded_spells_as_rituals_and_copy_eligible_rituals
notes: 2014 feat backend entry for Ritual Caster. Structured fields describe its character-sheet-relevant mechanics.
---
