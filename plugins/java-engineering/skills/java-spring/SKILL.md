---
name: java-spring
description: Standards Java / Spring Boot du socle (STD-JAVA). À utiliser pour écrire, modifier ou relire du code Java ou un pom.xml d'un service Spring Boot - controller, service, DTO, mapper, exception, transaction, sécurité, migration Flyway, configuration, logs, tests.
when_to_use: « ajoute un endpoint », « crée un service », « relis ce controller », « écris les tests », « est-ce conforme ? »
paths: ["**/*.java", "**/pom.xml"]
---

# Standards Java / Spring Boot (STD-JAVA)

Source normative : [references/java-spring.md](references/java-spring.md), copie de
`standards/java/java-spring.md` du dépôt `engineering-standards-marketplace`. Ne pas la modifier ici.

## Quel fichier lire

| Tâche | Lire |
|---|---|
| Écrire ou modifier du code Java, un `pom.xml`, une migration | [references/java-spring.md](references/java-spring.md), section de la couche touchée |
| Relire du code | [references/java-spring.md](references/java-spring.md), toutes les règles |

Sections : 3.1 structure et injection (001-002) ; 3.2 API et DTO (003-005) ; 3.3 erreurs (006-007) ;
3.4 transactions (008-010) ; 3.5 sécurité (011-012) ; 3.6 persistance (013-014) ; 3.7 configuration et logs
(015-016) ; 3.8 tests (017-019) ; 3.9 style (020).

## Procédure

1. Avant d'écrire : lire les règles de la section concernée ; les règles du `CLAUDE.md` du dépôt s'y ajoutent et
   priment en cas de conflit.
2. En écrivant : appliquer chaque règle **DOIT** ; une règle **DEVRAIT** non appliquée est signalée avec sa raison.
3. Ne pas corriger un écart existant hors du périmètre de la tâche : le signaler.
4. En rendant le travail : lister les règles concernées par identifiant (`STD-JAVA-NNN`), conformes ou non.

## Format d'une revue

| Règle | Niveau | Fichier:ligne | Constat | Correction proposée |
|---|---|---|---|---|

Une ligne par écart ; terminer par « Conformes : » suivi des identifiants vérifiés sans écart.
