---
name: Half-Elf
type: species
source: PHB 2014
size: Medium
speed: 30
bonuses:
- type: cha
  value: 2
languages:
- Common
- Elvish
choices:
  ability_increases:
    count: 2
    amount: 1
    distinct: true
    exclude:
    - cha
    options:
    - str
    - dex
    - con
    - int
    - wis
  skill_proficiencies:
    count: 2
    source: skills
  extra_language:
    count: 1
    source: languages
notes: Two chosen +1 abilities and two chosen skills.
features:
- Darkvision
- Fey Ancestry
- Skill Versatility
---
