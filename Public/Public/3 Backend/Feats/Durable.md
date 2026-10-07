---
name: Durable
type: feat
source: PHB 2014
choices:
  ability_increase:
    amount: 1
    options:
    - con
effects:
- id: durable_hit_dice
  type: healing_rule
  target: hit_die
  value: minimum_twice_con_modifier
notes: 2014 feat backend entry for Durable. Structured fields describe its character-sheet-relevant mechanics.
---
