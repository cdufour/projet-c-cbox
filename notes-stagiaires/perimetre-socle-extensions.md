# Périmètre du projet : socle obligatoire et extensions

Cette note précise le **périmètre réellement exigé** pour la fin du projet.
Elle ne modifie **pas** l'énoncé (`projet-c-cbox-enonce.md`), qui reste la
référence : elle distingue, parmi ce qu'il décrit, ce qui est **imposé** de ce
qui constitue des **extensions valorisées**. L'objectif est clair : chaque
binôme doit pouvoir mener le projet **à terme et proprement**, quel que soit
son rythme, sans sacrifier la qualité (compilation stricte, Valgrind propre,
CI verte) sur l'autel de la quantité.

**Lisez-la en entier** : elle change la lecture de certains passages de
l'énoncé (exemples avec `--key`, flux corrompus) et sert de base à la grille
d'évaluation.

---

## Le principe

L'énoncé décrit le projet **complet**, c'est-à-dire le niveau maximal. Tout ce
qui y figure reste vrai et réalisable, mais il se répartit désormais en deux
niveaux :

- le **socle** : exigé de tous les binômes, sans exception — c'est le contrat
  minimal pour que le projet soit considéré comme **terminé** ;
- les **extensions** : au choix du binôme, sans obligation de tout traiter —
  elles sont **créditées à la grille d'évaluation** et sont le moyen le plus
  rentable de se distinguer si le socle est déjà propre.

Un projet livré avec un socle **complet et impeccable** (zéro warning,
Valgrind propre, CI verte, `FORMAT.md` cohérent avec le code) vaut mieux
qu'un projet où tout est commencé et rien n'est terminé. **La qualité du socle
prime sur la quantité d'extensions.**

## Socle obligatoire

Tout ce qui suit reste **imposé**, tel quel :

- **Phase 1 en entier** : architecture modulaire, Makefile générique avec
  dépendances automatiques, `FORMAT.md` (format `.cbx` : tailles, bits
  `flags`/`mode`, endianness, ordre des transformations, comportements
  d'erreur), `cbx_format`, `cbx_crc`, `cbx_io`, `cbx_source` (les deux
  implémentations), `build.sh`.
- **Phase 2 sans le chiffrement** : `cbx_rle` complet (bornes des paquets,
  repli brut si gonflement, cas limites testés), type opaque `cbx_archive_t`,
  les cinq sous-commandes `pack`/`list`/`extract`/`verify`/`info` — sans
  `--key` — et le dispatch par pointeurs de fonctions.
- **Phase 3 hors trames** :
  - `RAPPORT-MEMOIRE.md`, harnais `tests/`, `run_tests.sh`, `log_report.sh`,
    `cppcheck` ;
  - pipeline CI/CD complet (4 stages, `main` protégée, MR bloquée par un
    pipeline rouge, tag `v1.0` → artefact), robustesse archive (magic
    erroné, tronquée, CRC invalide), restitution finale.

## Extensions (au choix, valorisées)

Par ordre de rentabilité indicative — chacune se suffit à elle-même :

1. **`cbx_crypto` complet** (Phase 2, Étape 3) : dérivation FNV-1a de la
   passphrase, flot xorshift32/LCG, XOR involutif, option `--key` sur
   `pack`/`extract`, détection d'une mauvaise clé par le CRC avec code
   retour 4. Les exemples de l'énoncé utilisant `--key` (notamment l'annexe
   A.4) décrivent **ce niveau**, pas le socle.
2. **Protocole de trame `cbox send` / `cbox recv`** (Phase 3, Étape 3),
   **en entier** :
   - **niveau 1 — chemin nominal** : délimitation FLAG, échappement à la
     SLIP, CRC16, vérification SEQ, réassemblage **octet pour octet
     identique** à l'original (comparé par `cmp` dans `run_tests.sh`) ;
   - **niveau 2 — canal bruité `channel_corrupt`** : altération/perte
     déterministe d'octets, détection propre (message + code retour 5) d'un
     octet altéré, d'une trame perdue et d'un flux tronqué, sans crash ni
     fuite.
   Tout ce que dit l'énoncé sur les trames (y compris les exemples de
   l'annexe A.6 et la spécification dans `FORMAT.md`) relève de ces
   niveaux, pas du socle. Réaliser le niveau 1 suffit à valoriser
   l'extension ; le niveau 2 la valorise davantage.
3. Les **bonus de l'Étape 6 Phase 3** : ARQ stop-and-wait, Huffman,
   `cbox diff`, `source_manifest`, stage CI `lint`, croisement ARM + QEMU,
   `--stats`.

Un binôme qui choisit `cbx_crypto` ou les trames doit documenter les bits
`mode` concernés, le format des trames et le comportement d'erreur dans son
`FORMAT.md`, comme pour le reste.

## Conséquences pratiques

- **Démo de restitution** : le `send`/`recv` n'y figure que s'il est
  implémenté ; la démo du socle porte sur les cinq sous-commandes, le
  pipeline et l'analyse mémoire.
- **`FORMAT.md`** : sans l'extension trames, votre `FORMAT.md` couvre le
  seul format `.cbx` — c'est acceptable pour le socle ; si vous faites
  l'extension, la spécification des trames y est attendue.
- **Les invariants ne bougent pas**, quel que soit votre niveau d'ambition :
  `-std=c11 -Wall -Wextra` zéro warning, Valgrind propre sur tout ce qui est
  livré, commits + tags par demi-journée, les deux membres commitent.
- **En cas de retard** : sacrifiez dans l'ordre les extensions, jamais le
  socle. La qualité (Valgrind, CI, cohérence `FORMAT.md`/code) ne se négocie
  pas.

> En cas de doute sur ce qui est attendu à un instant donné, demandez au
> formateur — la réponse tient en une phrase, la mauvaise interprétation
> peut coûter une demi-journée.
