---
name: Heavy Armor Master
type: feat
source: PHB 2014
prerequisites:
  armor_proficiency: heavy
choices:
  ability_increase:
    amount: 1
    options:
    - str
effects:
- id: ham_reduction
  type: damage_reduction
  value: 3
  damage:
  - bludgeoning
  - piercing
  - slashing
  condition: wearing heavy armor; nonmagical attack
notes: 2014 feat backend entry for Heavy Armor Master. Structured fields describe its character-sheet-relevant mechanics.
---
