# Rebuilt Class Backends

Copy the class `.md` files to `Public/3 Backend/Classes/`.

Schema:
- `proficiencies`: automatically granted armor, weapon, tool and saving-throw proficiencies
- `proficiency_choices`: Builder dropdown choices
- `asi_levels`: levels at which the Builder offers ASI/Feat
- `subclass`: label/unlock level; add your existing subclass files to `options`
- `features`: level-gated references resolved against `Public/3 Backend/Features/`
- `progression`: proficiency bonus / attunement scaffold

The 12 PHB 2014 classes are included. Artificer is included as the Eberron 2019 class.
Feature references only become active/displayed when matching files exist in the Features folder.
