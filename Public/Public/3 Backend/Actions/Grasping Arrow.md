---
name: Grasping Arrow
category: action
action_type: action
mode: attack
shape: ranged_single
range: 6
targets: 1
attacks: 2

dice_count: 2
dice_size: 6
damage_bonus_stat: 
damage_bonus: dex
damage_type: poison

hit_chance: 0.65
crit_chance: 0.05
ranged_attack: false
friendly_fire: false
requires_los: false

utility_type: disadvantage_vs_others
utility_value: 0
utility_formula: "enemy_avg_damage * (enemy_hit_chance_vs_ally - enemy_hit_chance_vs_ally_disadvantage) * proc_chance"

resource_cost: 0
resource_formula: "0"

notes: Momement reduced by 10 ft. 2d6 damage on movement.
---
# Thunder Gauntlets