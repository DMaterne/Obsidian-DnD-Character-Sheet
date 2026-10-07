---
name: Spell Sniper
type: feat
source: PHB 2014
prerequisites:
  spellcasting: true
choices:
  attack_cantrip:
    count: 1
    source: eligible_attack_cantrips
effects:
- id: spell_sniper_range
  type: spell_range_multiplier
  value: 2
  condition: spell requires attack roll
- id: spell_sniper_cover
  type: ignore_cover
  target: half_and_three_quarters_cover
  scope: spell_attacks
notes: 2014 feat backend entry for Spell Sniper. Structured fields describe its character-sheet-relevant mechanics.
---
