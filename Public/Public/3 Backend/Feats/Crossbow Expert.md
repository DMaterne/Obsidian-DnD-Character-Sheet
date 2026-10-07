---
name: Crossbow Expert
type: feat
source: PHB 2014
effects:
- id: crossbow_loading
  type: ignore_property
  target: loading
  scope: proficient_crossbows
- id: crossbow_melee_range
  type: ignore_disadvantage
  target: ranged_attacks_with_enemy_within_5ft
actions:
- name: Hand Crossbow Shot
  activation: bonus_action
  condition: after attacking with a one-handed weapon
  requires: loaded hand crossbow held
notes: 2014 feat backend entry for Crossbow Expert. Structured fields describe its character-sheet-relevant mechanics.
---
