# Species backend – unified Features architecture

Copy the species files to `Public/3 Backend/Species/`.

Species files own basic species data such as ability bonuses, size, speed, languages and choices.
Named racial/species traits are listed under `features:`. The Character Sheet resolves those names
against `Public/3 Backend/Features/`.

If a referenced Feature file exists, it is shown under Features & Traits and its bonuses/actions/resources
are used automatically. If the file does not exist, the reference is safely ignored.

Example:
`Warforged -> Integrated Protection -> Features/Integrated Protection.md -> +1 AC`

This avoids defining the same mechanic twice.
