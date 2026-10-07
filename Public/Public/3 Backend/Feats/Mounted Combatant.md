---
name: Mounted Combatant
type: feat
source: PHB 2014
effects:
- id: mounted_advantage
  type: advantage
  target: melee_attacks
  condition: mounted; target smaller than mount
- id: mounted_redirect
  type: attack_redirect
  target: mount
- id: mounted_evasion
  type: mount_save_rule
  value: dex_half_to_zero_success_half_failure
notes: 2014 feat backend entry for Mounted Combatant. Structured fields describe its character-sheet-relevant mechanics.
---
