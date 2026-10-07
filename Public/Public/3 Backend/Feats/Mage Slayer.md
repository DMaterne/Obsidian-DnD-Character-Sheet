---
name: Mage Slayer
type: feat
source: PHB 2014
effects:
- id: mage_slayer_concentration
  type: disadvantage
  target: concentration_save
  condition: caused by your damage
- id: mage_slayer_save
  type: advantage
  target: saving_throws_vs_spells
  condition: spell cast by creature within 5 ft
actions:
- name: Mage Slayer Attack
  activation: reaction
  condition: creature within 5 ft casts a spell
  effect: make melee weapon attack
notes: 2014 feat backend entry for Mage Slayer. Structured fields describe its character-sheet-relevant mechanics.
---
