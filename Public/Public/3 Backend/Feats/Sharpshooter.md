---
name: Sharpshooter
type: feat
source: PHB 2014
effects:
- id: ss_long_range
  type: ignore_disadvantage
  target: long_range_ranged_weapon_attacks
- id: ss_cover
  type: ignore_cover
  target: half_and_three_quarters_cover
actions:
- name: Sharpshooter Power Shot
  activation: attack_option
  condition: before attack with proficient ranged weapon
  effect: -5 attack roll, +10 damage on hit
notes: 2014 feat backend entry for Sharpshooter. Structured fields describe its character-sheet-relevant mechanics.
---
