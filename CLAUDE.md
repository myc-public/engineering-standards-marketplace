# engineering-standards-marketplace

Standards d'ingénierie du socle et marketplace de plugins Claude Code. Lire d'abord [README.md](README.md).
Règles communes du socle : `platform-handbook`. Décision : ADR 0038.

## Commandes

- Validation : `claude plugin validate --strict .` puis `claude plugin validate --strict plugins/<plugin>`.

## Règles propres à ce dépôt

- Un standard suit `standards/_modele.md` ; une règle = un identifiant `STD-<DOMAINE>-NNN`, jamais réutilisé ni
  renuméroté ; une règle retirée garde son identifiant, marquée « retirée ».
- `standards/` ne mentionne pas Claude Code : il doit rester lisible et applicable sans lui.
- `plugins/*/skills/*/references/` est une copie générée de `standards/` : ne jamais l'éditer à la main.
- Pas de dossier `rules/` ni de `CLAUDE.md` dans un plugin : Claude Code ne les charge pas (ADR 0038).
- Chemins d'un plugin : relatifs, préfixés `./`, jamais hors de la racine du plugin.
- Tout changement de plugin : `version` de `plugin.json` incrémentée, `claude plugin validate --strict` vert.
- Nouveau plugin : une entrée dans `.claude-plugin/marketplace.json` et une ligne dans le README.
