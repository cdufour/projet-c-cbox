# Notes stagiaires — projet CBox

Ce dossier regroupe des **notes d'appui / d'aide** rédigées par le formateur
pour accompagner la réalisation du projet CBox. Elles viennent en complément
de l'énoncé (`projet-c-cbox-enonce.md`) et du `FORMAT.md` de votre binôme,
jamais à la place.

## Table des matières

| Note | Module | Sujet |
| --- | --- | --- |
| [`cbx_crc.md`](cbx_crc.md) | `cbx_crc` | Checksum CRC32 (algorithme, implémentation, tests) |
| [`cbx_archive.md`](cbx_archive.md) | `cbx_archive` | Type opaque `cbx_archive_t`, moment des écritures disque, table dynamique, pointeurs et heap |
| [`cbx_source.md`](cbx_source.md) | `cbx_source` | Abstraction de source par pointeurs de fonctions (`source_dir`, `source_synthetic`) |
| [`cbx_rle_crypto.md`](cbx_rle_crypto.md) | `cbx_rle`, `cbx_crypto` | Compression RLE (format de paquets, bornes, repli brut) et chiffrement XOR à flot (idées, pièges, limites) |
| [`cbx_rle_octet_controle.md`](cbx_rle_octet_controle.md) | `cbx_rle` | L'octet de contrôle en détail : littéral `c + 1`, run `(c & 0x7F) + 2`, règle de choix encodeur, préfixe de longueur vs délimiteur, cas limite 128 |
| [`perimetre-socle-extensions.md`](perimetre-socle-extensions.md) | transverse | Périmètre exigé du projet : socle obligatoire vs extensions valorisées (chiffrement, protocole de trame, bonus) |

## Comment lire ces notes

- **Les exemples et suggestions de code et d'implémentation sont indicatifs.**
  Vous avez toute liberté de faire « à votre manière » — autre découpage,
  autres noms, autre logique — tant que les **objectifs finaux** de l'énoncé
  sont atteints (compilation stricte sans warning, pas de fuite mémoire,
  comportement des sous-commandes, format `.cbx` conforme à votre
  `FORMAT.md`).
- **Certains choix du formateur visent davantage la validation pédagogique
  que l'optimisation ou la performance** : privilégier le buffering complet
  en mémoire plutôt qu'un streaming optimisé, réécrire un PRNG plutôt que
  `rand()`, sérialiser champ par champ plutôt qu'écrire une `struct` telle
  quelle… Ces choix assumés couvrent des notions vues en formation ;
  en contexte réel, on ferait parfois différemment (et c'est un excellent
  sujet de discussion en soutenance).
- Ces notes ne remplacent ni l'énoncé (qui fait foi), ni votre `FORMAT.md`
  (qui est **votre** contrat — c'est lui que le formateur relit en premier).
