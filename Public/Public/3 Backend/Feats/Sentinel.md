---
name: Sentinel
type: feat
source: PHB 2014
effects:
- id: sentinel_oa_stop
  type: speed_set
  value: 0
  condition: hit with opportunity attack
- id: sentinel_disengage
  type: opportunity_attack_rule
  value: disengage_does_not_prevent
- id: sentinel_reaction
  type: reaction_attack
  condition: creature within 5 ft attacks target other than you
notes: 2014 feat backend entry for Sentinel. Structured fields describe its character-sheet-relevant mechanics.
---
