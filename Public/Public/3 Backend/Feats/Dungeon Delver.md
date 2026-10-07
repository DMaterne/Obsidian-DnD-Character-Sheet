---
name: Dungeon Delver
type: feat
source: PHB 2014
effects:
- id: dd_search
  type: advantage
  targets:
  - perception
  - investigation
  condition: detecting secret doors
- id: dd_traps
  type: advantage
  targets:
  - saves_vs_traps
  condition: avoiding or resisting traps
- id: dd_trap_resistance
  type: resistance
  target: trap_damage
- id: dd_fast_travel
  type: passive_perception_rule
  value: no_fast_pace_penalty
notes: 2014 feat backend entry for Dungeon Delver. Structured fields describe its character-sheet-relevant mechanics.
---
