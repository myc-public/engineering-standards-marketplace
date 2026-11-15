# engineering-standards-marketplace

Standards d'ingénierie du socle myc-public et marketplace de plugins Claude Code qui les appliquent dans chaque
dépôt. Décision : [ADR 0038](https://github.com/myc-public/platform-handbook/blob/develop/adr/0038-harness-claude-code-marketplace.md) ;
avancement : [roadmap harness](https://github.com/myc-public/platform-handbook/blob/develop/roadmap/harness.md).

## Structure

| Dossier | Rôle |
|---|---|
| `standards/` | Source normative, indépendante de Claude Code ; squelette commun : [`standards/_modele.md`](standards/_modele.md) |
| `plugins/<plugin>/` | Plugins Claude Code (skills, agents, hooks) au format officiel |
| `plugins/<plugin>/skills/<skill>/references/` | Copie générée des standards (ESM4), jamais éditée à la main |
| `.claude-plugin/marketplace.json` | Catalogue des plugins distribués |
| `governance/` | Versioning, exceptions, matrice de conformité |

Plugins disponibles : `java-engineering` (pilote : `inner-donation-api`, en construction).

## Démarrer en local

Valider le marketplace et le plugin :

```powershell
$repo = 'D:\workspace\public\engineering-standards-marketplace'
claude plugin validate --strict $repo
claude plugin validate --strict (Join-Path -Path $repo -ChildPath 'plugins\java-engineering')
```

Ajouter le marketplace local dans une session Claude Code, puis installer le plugin :

```text
/plugin marketplace add D:\workspace\public\engineering-standards-marketplace
/plugin install java-engineering@engineering-standards-marketplace
```

## Écrire un standard

- Partir de [`standards/_modele.md`](standards/_modele.md) ; une règle = un identifiant `STD-<DOMAINE>-NNN`,
  jamais réutilisé.
- Niveaux DOIT / DEVRAIT / PEUT (RFC 2119) ; chaque règle dit comment elle est contrôlée.

## Liens

- Règles communes du socle : [platform-handbook](https://github.com/myc-public/platform-handbook)
- Format des plugins : documentation Claude Code, « Plugin manifest reference »
