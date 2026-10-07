---
name: Magic Initiate
type: feat
source: PHB 2014
choices:
  spellcasting_class:
    count: 1
    options:
    - bard
    - cleric
    - druid
    - sorcerer
    - warlock
    - wizard
  cantrips:
    count: 2
    source: chosen_class_cantrips
  level_1_spell:
    count: 1
    source: chosen_class_level_1_spells
resources:
  id: magic_initiate_spell
  max: 1
  recharge: long_rest
  display: checkboxes
notes: 2014 feat backend entry for Magic Initiate. Structured fields describe its character-sheet-relevant mechanics.
---
