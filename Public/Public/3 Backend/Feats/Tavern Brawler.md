---
name: Tavern Brawler
type: feat
source: PHB 2014
proficiencies:
  weapons:
  - improvised
choices:
  ability_increase:
    amount: 1
    options:
    - str
    - con
effects:
- id: tb_unarmed
  type: unarmed_damage
  value: 1d4
actions:
- name: Tavern Brawler Grapple
  activation: bonus_action
  condition: after hitting with unarmed strike or improvised weapon
  effect: attempt to grapple target
notes: 2014 feat backend entry for Tavern Brawler. Structured fields describe its character-sheet-relevant mechanics.
---
