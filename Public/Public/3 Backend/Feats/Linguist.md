---
name: Linguist
type: feat
source: PHB 2014
choices:
  ability_increase:
    amount: 1
    options:
    - int
  languages:
    count: 3
    source: languages
effects:
- id: linguist_cipher
  type: utility_rule
  value: create_written_ciphers
notes: 2014 feat backend entry for Linguist. Structured fields describe its character-sheet-relevant mechanics.
---
