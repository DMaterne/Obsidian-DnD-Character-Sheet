---
name: Grappler
type: feat
source: PHB 2014
prerequisites:
  str: 13
effects:
- id: grappler_advantage
  type: advantage
  target: attacks
  condition: target grappled by you
actions:
- name: Pin Grappled Creature
  activation: action
  condition: target grappled by you
  effect: make grapple check; on success both creatures become restrained
notes: 2014 feat backend entry for Grappler. Structured fields describe its character-sheet-relevant mechanics.
---
