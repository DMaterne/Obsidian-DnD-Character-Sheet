---
name: Shield Master
type: feat

prerequisites:
  shield: wielded_for_benefits
effects:
- id: shield_master_save
  type: save_bonus
  target: dex_save
  value: shield_ac_bonus
  condition: effect targets only you
actions:
- name: Shield Master Shove
  activation: bonus_action
  condition: after taking Attack action; wielding shield
  effect: attempt to shove a creature within 5 ft
- name: Shield Master Evasion
  activation: reaction
  condition: DEX save for half damage; wielding shield; save succeeds
  effect: take no damage

notes: >-
  You use shields not just for protection but also for offense. You gain the
  following benefits while you are wielding a shield: 
  
  - If you take the Attack
  action on your turn, you can use a bonus action to try to shove a creature
  within 5 feet of you with your shield. 
  
  - If you aren't incapacitated, you can
  add your shield's AC bonus to any Dexterity saving throw you make against a
  spell or other harmful effect that targets only you. 
  
  - If you are subjected to
  an effect that allows you to make a Dexterity saving throw to take only half
  damage, you can use your reaction to take no damage if you succeed on the
  saving throw, interposing your shield between yourself and the source of the
  effect.
  
---
