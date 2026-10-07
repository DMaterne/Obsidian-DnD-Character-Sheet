---
name: Alert
type: feat
source: PHB 2014
bonuses:
- type: initiative
  value: 5
effects:
- id: alert_surprise
  type: immunity
  target: surprised
  condition: while conscious
- id: alert_unseen
  type: deny_advantage
  target: attacks_from_unseen_attackers
notes: 'Extremely vigilant: improves initiative and protects against surprise and unseen attackers.'
---
