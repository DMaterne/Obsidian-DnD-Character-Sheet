# Obsidian D&D Character Sheet

An interactive D&D 5e character sheet built entirely inside **Obsidian** using Markdown, YAML and DataviewJS.

## Requirements

### Dataview — Required

This character sheet requires the **Dataview** Community Plugin.

1. Open **Obsidian → Settings → Community Plugins**
2. Install **Dataview**
3. Enable the plugin
4. Make sure **JavaScript Queries** are enabled in the Dataview settings
5. Restart Obsidian

The character sheet will not work correctly without Dataview.

### Editor with Slider — Optional

For a better editing experience and improved UI, the **Editor with Slider** Community Plugin is recommended.

It is **not required** for the character sheet to function.

I use a width of 69 (not because it's a funny number, it just fits the best... ok it is a funny number)

---

## Getting Started

### 1. Download the Vault

Download or clone this repository and open the folder as an **Obsidian Vault**.

After opening it for the first time, install and enable the required plugins described above.

### 2. Open the Character Sheet

Open the main character sheet file in Obsidian.

The sheet automatically loads the character data, inventory, actions, spells and features from the backend files.

### 3. Create Your Character

Character data is stored in:

`Public/3 Backend/EIMER.md`

You can either edit this file or create a copy for your own character.

The character file contains the basic character information, including:

- Name, race, class and level
- Ability scores
- Hit Points and Armor Class
- Saving Throw and Skill proficiencies
- Inventory
- Actions
- Spells
- Features
- Spell Slots
- Attunement
- Notes and Backstory

### 4. Connect Your Character to the Sheet

At the top of the character sheet, change:

`const CHARACTER_PATH = "Public/3 Backend/Characters/EIMER.md";`

to the path of your own character file.

Example:

`const CHARACTER_PATH = "Public/3 Backend/Characters/MyCharacter.md";`

### 5. Adding Content

Additional character content is stored in separate Markdown files in the corresponding folders located in 'Public/3 Backend/'

To add something to your character, create or copy the corresponding Markdown file and reference it inside your character file.

For example:

`features:`
`  - Public/3 Backend/Features/Integrated Protection.md`

`spells:`
`  - Public/3 Backend/Spells/Cure Wounds.md`

`actions:`
`  - Public/3 Backend/Actions/Light Crossbow.md`

Items are added through the inventory:

`inventory:`
`  - item: Public/3 Backend/Items/Light Crossbow.md`
`    quantity: 1`
`    equipped: true`

### 6. Using the Character Sheet

Most common values are calculated automatically from the backend data.

The sheet includes:

- Ability Scores and Modifiers
- Saving Throws and Skills
- Armor Class, Initiative and Speed
- Current, Maximum and Temporary HP
- Damage and Healing controls
- Inventory and Equipment
- Item Attunement
- Actions, Bonus Actions and Reactions
- Spells and Spell Slots
- Features and Traits
- Passive Senses
- Character Notes

Changes such as HP, used Spell Slots, equipped items and attuned items can be managed directly through the character sheet.

---

## Customization

The system is designed around simple Markdown and YAML files, so new items, spells, actions and features can easily be added or modified.

The included **EIMER** character can be used as an example when creating your own character.

> **Note:** This project is still under development. Some features or file structures may change in future versions.

## Conclusion

If you have any feedback or a feature you want just let me know. I am doing this for fun and use the sheet myself.

If you are a D&D player or DM using obsidian, you should check out my other project: https://github.com/DMaterne/obsidian_account_manager

I am a coffeine addict, if you liked my project and want to show a bit of appreciation I won't mind: buymeacoffee.com/DMaterne
