---
name: "Primal Champion"
type: "feature"
source: "Barbarian (PHB 2014)"
level: 20
enabled: true
notes: "Your Strength and Constitution scores each increase by 4, and their maximum scores become 24. The score increases are represented by the enabled bonuses below; do not apply the same +4 manually a second time."
bonuses:
  - type: str
    value: 4
    active_when: enabled
  - type: con
    value: 4
    active_when: enabled
---
