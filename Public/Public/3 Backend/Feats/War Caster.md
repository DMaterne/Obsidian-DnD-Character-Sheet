---
name: War Caster
type: feat
source: PHB 2014
prerequisites:
  spellcasting: true
effects:
- id: warcaster_concentration
  type: advantage
  target: concentration_saves
  condition: save caused by damage
- id: warcaster_somatic
  type: spellcasting_rule
  value: perform_somatic_components_with_weapons_or_shield_in_hands
actions:
- name: War Caster Spell Opportunity
  activation: reaction
  condition: creature provokes opportunity attack
  effect: cast eligible one-action single-target spell instead
notes: 2014 feat backend entry for War Caster. Structured fields describe its character-sheet-relevant mechanics.
---
