---
name: "Unarmored Defense"
type: "feature"
source: "Barbarian (PHB 2014)"
level: 1
enabled: true
notes: "While not wearing armor, your Armor Class equals 10 + your Dexterity modifier + your Constitution modifier. You can still use a shield."
bonuses:  
- type: ac  
  formula: "10 + dex_mod + con_mod"  
  active_when: unarmored
  calculation: base
---
