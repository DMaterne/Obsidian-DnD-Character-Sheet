---
name: Medium Armor Master
type: feat
source: PHB 2014
prerequisites:
  armor_proficiency: medium
effects:
- id: mam_stealth
  type: ignore_disadvantage
  target: stealth
  condition: wearing medium armor
- id: mam_dex_cap
  type: armor_rule
  value: medium_armor_dex_cap_3
notes: 2014 feat backend entry for Medium Armor Master. Structured fields describe its character-sheet-relevant mechanics.
---
