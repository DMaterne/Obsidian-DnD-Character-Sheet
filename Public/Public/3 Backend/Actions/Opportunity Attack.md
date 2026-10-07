---
name: Opportunity Attack
category: action
action_type: reaction
mode: attack
shape: melee_single
range: 1
targets: 1
attacks: 1

dice_count: 1
dice_size: 8
damage_bonus_stat: int
damage_bonus: 0
damage_type: thunder

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

notes: Armorer weapon attack. On hit, target has disadvantage on attacks against others until your next turn.
---
# Opportunity Attack