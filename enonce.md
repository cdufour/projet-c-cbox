# Projet C — « CBox » : conteneur binaire, protocole de transfert et CLI d'empaquetage

**POEI — Linux, Langages & Systèmes (LLS · EXPLEO)** · AJC FORMATION
Module **Projet C** · **3 jours / 21 heures** · Travail en **binôme**, en autonomie encadrée

---

## Contexte général

Vous êtes développeur embarqué dans une société qui produit des firmwares.
Avant d'être transférés vers une cible, un firmware et ses fichiers associés
(configuration, assets, carte mémoire…) doivent être **regroupés en un seul
artefact binaire versionné, vérifiable et reproductible**, puis **transférés
vers la cible sur un lien octet par octet** (série, USB…) — exactement ce que
font `tar`, `cpio` ou les images de partition en industrialisation embarquée.

Votre mission : écrire **CBox**, un outil en ligne de commande qui :

- **empaquette** un ensemble de fichiers dans une archive binaire propriétaire
  au format `.cbx` que vous concevez vous-mêmes,
- **compresse** (RLE) et **chiffre** (chiffrement XOR à flot) les entrées,
  de façon élémentaire mais correcte au niveau octet,
- permet de **lister, extraire, vérifier l'intégrité** et **interroger** cette
  archive,
- **transfère** l'archive sur un canal octet via un **protocole de trame
  maison** (délimitation, séquence, CRC — dans l'esprit de SLIP/HDLC),
- s'intègre dans une chaîne d'industrialisation : **Makefile, dépôt Git,
  pipeline CI/CD GitLab, tests, analyse mémoire**.

Le projet se déroule en **trois phases** :

| Phase    | Contenu                                                             | Jour      |
| -------- | ------------------------------------------------------------------ | --------- |
| Phase 1  | Fondations : architecture modulaire, Makefile, format `.cbx`        | Journée 1 |
| Phase 2  | Bibliothèque d'archive, compression, chiffrement, CLI complet       | Journée 2 |
| Phase 3  | Protocole de trame, tests, analyse mémoire, CI/CD, restitution      | Journée 3 |

**Méthode de travail.** Vous travaillez en binôme, en toute autonomie. Vous êtes
libres d'effectuer les choix adaptés, de développer les parties dont vous jugez
avoir le plus besoin et d'apporter vos propres solutions aux problèmes posés.
Le formateur encadre par sa présence, répond aux questions, peut épauler votre
binôme en difficulté ou faire un point collectif sur une notion non acquise.

**Consignes transverses (non négociables).**

- Compilation stricte : `-std=c11 -Wall -Wextra`, **zéro warning**.
- Aucune fuite mémoire : `valgrind --leak-check=full` doit être propre sur
  toutes les sous-commandes.
- Chaque demi-journée se termine par un **commit + tag** (`v0.1`, `v0.2`, …)
  et un dépôt poussé : ces jalons servent de suivi.
- Les deux membres du binôme commitent, chacun sur sa branche, avec des fusions
  vers `main` par Merge Request dès la Phase 3.
- Le dépôt est hébergé sur **GitLab** : chaque binôme crée son propre projet
  (nommé `cbox`), y pousse son code, et **invite le formateur en Reporter**
  dès la création — le formateur suit l'avancement (tags, pipelines, MR) et
  fait ses reviews directement dans le projet, sans intervenir dessus.

---

## Le format `.cbx` — spécification cadre

Le squelette suivant est **imposé** (il garantit la couverture pédagogique du
projet) ; vous précisez vous-mêmes les tailles exactes, l'ordre des bits des
champs `flags`/`mode`, et vous documentez vos choix dans un `FORMAT.md`.

```
Fichier .cbx
┌───────────────────────────────────────────────────────────┐
│ HEADER (taille fixe)                                       │
│   magic[4]        = "CBX1"                                 │
│   version         : uint16_t                               │
│   flags           : uint16_t   (bit 0 : chiffré, …)        │
│   entry_count     : uint32_t                               │
│   header_crc32    : uint32_t                               │
├───────────────────────────────────────────────────────────┤
│ TABLE DES ENTRÉES (entry_count éléments, taille fixe)      │
│   name[64]        : char                                    │
│   size            : uint32_t   (taille des données stockées│
│                    dans DATA : après compression/chiffrement)│
│   raw_size        : uint32_t  (taille originale du fichier)│
│   offset          : uint32_t  (position dans la zone DATA) │
│   mode            : uint16_t  (bit 0 : RLE, bit 1 : XOR,   │
│                    bits 8-10 : r/w/x)                      │
│   entry_crc32     : uint32_t   (CRC des données BRUTES)    │
├───────────────────────────────────────────────────────────┤
│ DATA (contenu transformé — compressé et/ou chiffré —       │
│      concaténé des fichiers)                               │
└───────────────────────────────────────────────────────────┘
```

Ordre de transformation au `pack` : **brut → compression RLE → chiffrement
XOR** (et l'inverse exact à l'extraction). Le `entry_crc32` porte sur les
données **brutes** : il valide donc l'intégrité de bout en bout, y compris
après décompression/déchiffrement.

> **Piège classique à éviter :** ne jamais écrire une `struct` en mémoire telle
> quelle avec `fwrite` — l'alignement/padding dépend du compilateur et de la
> cible. Toute écriture/lecture passe par des fonctions de (dé)sérialisation
> dédiées, avec une **endianness de stockage fixée par convention**
> (little-endian recommandé) indépendante de la machine hôte.

---

# PHASE 1 — Fondations : architecture modulaire, Makefile, format `.cbx`

> Objectif : poser dès le départ l'architecture d'un vrai projet C — modules,
> headers, Makefile générique, spécification de format. Comme pour la station
> météo en Phase 2, **on démarre directement en modulaire** : l'étape
> monolithique a déjà été vécue, on ne la refait pas.

## Arborescence cible

```
cbox/
├── Makefile
├── .gitlab-ci.yml          ← écrit en Phase 3
├── FORMAT.md               ← format .cbx + protocole de trame
├── RAPPORT-MEMOIRE.md      ← rédigé en Phase 3
├── include/
│   ├── cbx_format.h      ← types/constantes partagés (header, entrée, magic)
│   ├── cbx_crc.h
│   ├── cbx_io.h
│   ├── cbx_source.h      ← abstraction de source (cf. Étape 5)
│   ├── cbx_rle.h         ← compression RLE (Phase 2)
│   ├── cbx_crypto.h      ← chiffrement XOR à flot (Phase 2)
│   ├── cbx_frame.h       ← protocole de trame (Phase 3)
│   ├── cbx_archive.h     ← type opaque cbx_archive_t (Phase 2)
│   └── cbx_cli.h         ← dispatch des sous-commandes (Phase 2)
├── src/
│   ├── main.c            ← orchestration uniquement
│   ├── cbx_crc.c
│   ├── cbx_io.c
│   ├── cbx_source.c
│   ├── cbx_rle.c
│   ├── cbx_crypto.c
│   ├── cbx_frame.c
│   ├── cbx_archive.c
│   └── cbx_cli.c
├── tests/                ← harnais à base d'assert() (Phase 3)
├── scripts/              ← build.sh, run_tests.sh, log_report.sh
└── build/                ← généré par le Makefile, ignoré par Git
```

## Composants et notions mobilisées

Chaque composant du projet est un point d'entrée pour réinvestir une notion
vue pendant les 10 jours de formation — le tableau sert de checklist de
couverture :

| Composant | Rôle | Notions mobilisées |
| --- | --- | --- |
| `cbx_format.h` | Constantes, `struct`/bit-fields du header et des entrées, macros (magic, version, tailles, masques `flags`/`mode`) | Typage strict (`stdint.h`), `const`, bit-fields, préprocesseur *(J1/J2)* |
| `cbx_crc.c/h` | Checksum CRC32 (calcul par décalages/XOR ou table précalculée) | Opérateurs bit à bit, tableaux *(J2, fondamentaux — tableaux)* |
| `cbx_io.c/h` | Sérialisation/désérialisation endianness-safe du header et des entrées, lecture/écriture disque | Bit à bit, pointeurs, fichiers, gestion d'erreurs *(J2, fondamentaux — fichiers)* |
| `cbx_source.c/h` | Abstraction de la **source des fichiers à empaqueter**, par pointeur de fonction : `source_dir` (scan d'un répertoire réel) et `source_synthetic` (génération déterministe, graine) — même pattern que `capteur.h` de la météo | Pointeurs de fonctions, HAL, fichiers, allocation dynamique *(J4)* |
| `cbx_rle.c/h` | Compression RLE par paquets (octet de contrôle, runs bornés, repli brut si gonflement) | Opérateurs bit à bit, bornes et cas limites, tampons *(J2)* |
| `cbx_crypto.c/h` | Chiffrement XOR à flot : graine dérivée par hash FNV-1a, flot xorshift32/LCG, XOR involutif in place | Manipulation bit à bit, déterminisme, algèbre du XOR *(J2)* |
| `cbx_frame.c/h` | Protocole de trame maison : délimitation FLAG, échappement à la SLIP, SEQ, CRC16, canal abstrait (`stdio` + `channel_corrupt`) par pointeurs de fonctions | Protocoles binaires, échappement d'octets, pointeurs de fonctions *(J2/J4)* |
| `cbx_archive.c/h` | **Type opaque** `cbx_archive_t` : `cbx_archive_create/open/add_entry/close`, table d'entrées dynamique | Types opaques, pointeurs de pointeurs, `malloc`/`realloc`/`free`, tableaux de pointeurs *(J4, fondamentaux — allocation dynamique)* |
| `cbx_cli.c/h` + `main.c` | **Table de dispatch** de sous-commandes (`pack`/`list`/`extract`/`verify`/`info`/`send`/`recv`) par pointeurs de fonctions | Pointeurs de fonctions, callbacks, `argv` *(J4)* |
| `scripts/build.sh` | Compilation stricte + smoke test | Shell script |
| `scripts/run_tests.sh` | Corpus déterministe (`source_synthetic`), pack → extract → verify (± RLE/XOR), send/recv, comparaison `diff -r` | Shell script, boucles, codes de retour |
| `scripts/log_report.sh` | Résumé des logs verbeux du programme (`grep`/`sed`/`awk`) | Shell script, filtres Unix |
| `tests/` | Mini-harnais de tests par `assert()` (CRC, RLE, XOR, sérialisation, trames) — pas de framework externe | Fonctions, préprocesseur (`NDEBUG`) |
| `.gitlab-ci.yml` | Pipeline `build → test → quality → package`, protection de `main`, artefact déclenché par tag | CI/CD *(projet1)* |
| Dépôt Git | Branches par membre, tags par livrable, au moins une fusion avec conflit résolu, une Merge Request bloquée par un pipeline rouge | Git |

## Étape 1 — Mise en place du projet et du dépôt Git

1. Créez le dépôt Git du binôme sur **GitLab** (projet nommé `cbox`), et
   **invitez le formateur en Reporter** (Manage → Members → Invite members) :
   il doit pouvoir lire code, MR, pipelines et tags dès le premier jalon.
   Chaque membre travaille ensuite sur une branche `dev/<prénom>` ; `main`
   reste la branche d'intégration.
2. Écrivez un `.gitignore` qui exclut au minimum `build/`.
3. Rédigez un `README.md` minimal (objectif du projet, usage prévu).
4. Créez l'arborescence modulaire ci-dessus (dossiers vides ou squelettes).

## Étape 2 — `FORMAT.md` : spécification des formats

Rédigez la spécification complète **du format `.cbx` et du protocole de
trame** :

- taille exacte du header et d'une entrée, signification et position de
  **chaque bit** des champs `flags` et `mode` (notamment bits RLE/XOR),
  endianness de stockage choisie et justification ;
- l'ordre exact des transformations (brut → RLE → XOR) et la règle de
  repli : si la version compressée est **plus grande** que l'original
  (données incompressibles), l'entrée est stockée brute avec le bit RLE à 0 ;
- le format des trames du protocole de transfert (cf. Phase 3, Étape 3) :
  délimiteurs, échappement, champs, CRC ;
- le comportement attendu en cas d'archive invalide (magic erroné, version
  inconnue, checksum faux).

> Ce document est votre **contrat** : les modules des Phases 2 et 3 s'y
> réfèrent, et le formateur s'en sert pour relire votre code. Un `FORMAT.md`
> vague se paie cash en Phase 2.

## Étape 3 — Makefile générique

Reprenez les acquis de la station météo et écrivez un Makefile avec :

- variables `CC`, `CFLAGS` (`-std=c11 -Wall -Wextra -I include`), `SRCDIR`,
  `BUILDDIR`, `TARGET` ;
- listing automatique des sources (`wildcard`) et conversion en objets
  (`patsubst`) — un nouveau `.c` dans `src/` doit être pris en compte sans
  toucher au Makefile ;
- règle générique `$(BUILDDIR)/%.o : $(SRCDIR)/%.c` avec création automatique
  de `build/` (cible-order `| $(BUILDDIR)`) ;
- **gestion automatique des dépendances** : `-MMD -MP` + `-include $(DEPS)`
  (modifier un `.h` doit recompiler ce qui dépend de lui) ;
- cibles `all`, `clean`, `re`, déclarées `.PHONY`.

**Vérifications attendues** : `make` complet ; toucher un seul `.c` ne
recompile qu'un `.o` ; toucher `cbx_format.h` recompile tous les modules qui
l'incluent ; `make clean && make` repart de zéro.

## Étape 4 — `cbx_format.h` et `cbx_crc`

1. `include/cbx_format.h` : gardes d'inclusion, macros (`CBX_MAGIC`,
   `CBX_VERSION`, tailles, masques des bits `flags`/`mode`), définitions
   `struct`/bit-fields du header et des entrées. Utilisez des types à largeur
   garantie (`stdint.h` : `uint16_t`, `uint32_t`…) — jamais `int` dans le
   format fichier.
2. `src/cbx_crc.c` + `include/cbx_crc.h` : un **CRC32** (algorithme
   polynômial, par décalages et XOR bit à bit **ou** par table précalculée —
   à votre charge de choisir et de justifier dans `FORMAT.md`).
   Prototype attendu : `uint32_t cbx_crc32(const uint8_t *data, size_t len);`
   Le CRC d'une donnée vide doit valoir `0`.

## Étape 5 — `cbx_io` et abstraction de source

1. `cbx_io.c/h` : sérialisation/désérialisation **endianness-safe** du header
   et d'une entrée — `cbx_write_header(FILE *, const CbxHeader *)`,
   `cbx_read_header(FILE *, CbxHeader *)`, et l'équivalent pour une entrée.
   Chaque fonction retourne un code d'erreur homogène (enum documenté).
2. `cbx_source.c/h` : abstraction de la **source des fichiers à empaqueter**,
   par pointeur de fonction — le même pattern que le `capteur.h` de la météo :

   ```c
   typedef struct {
       const char *name;      /* nom de l'entrée */
       uint32_t    size;
       /* … moyen de lire le contenu … */
   } cbx_entry_desc;

   typedef int (*fn_source_next)(void *ctx, cbx_entry_desc *out);
   typedef struct {
       fn_source_next next;   /* renvoie 1 = entrée, 0 = fini, <0 = erreur */
       void *ctx;
   } cbx_source;
   ```

   Écrivez **deux implémentations** :
   - `source_dir` : parcourt un vrai répertoire (`opendir`/`readdir`), en se
     limitant à son premier niveau — pas de descente récursive dans les
     sous-dossiers (le format d'archive est à plat, les noms d'entrée ne sont
     pas des chemins) ;
   - `source_synthetic` : génère un corpus **déterministe** de N fichiers à
     partir d'une graine — indispensable pour des tests reproductibles en CI.

## Étape 6 — Premier jalon

`scripts/build.sh` : wrapper autour de `make` qui fait un smoke test
(l'exécutable se lance, `cbox` sans argument affiche l'usage et retourne un
code d'erreur non nul). L'usage complet donne d'emblée l'idée finale de ce que
devra permettre le programme :

```
$ cbox
cbox — conteneur binaire .cbx
Usage : cbox <sous-commande> [options] <archive>

Sous-commandes :
  pack     Empaqueter des fichiers dans une archive
  list     Lister les entrées d'une archive
  extract  Extraire les entrées vers un répertoire
  verify   Vérifier l'intégrité d'une archive
  info     Afficher un résumé de l'archive
  send     Envoyer une archive sur un flux de trames (stdout)
  recv     Réassembler une archive depuis un flux de trames

Options communes : --rle, --key <passphrase>, -v (verbeux)
$ echo $?
1
```

Puis **commit + tag `v0.1`** (fin de matinée) puis
**tag `v0.2`** (fin de journée) sur les éléments livrés.

**Livrable de la Phase 1** — dépôt Git propre, `FORMAT.md`, Makefile générique
avec dépendances automatiques, `cbx_format.h`, `cbx_crc`, `cbx_io` et
`cbx_source` (2 implémentations) compilant sans warning, `build.sh`, Valgrind
propre sur le smoke test.

---

# PHASE 2 — Bibliothèque d'archive, compression, chiffrement, CLI complet

> Objectif : le cœur métier. Un **type opaque** `cbx_archive_t` caché derrière
> une API, les **transformations binaires** des données (RLE, XOR), les cinq
> sous-commandes, et un dispatch par pointeurs de fonctions.

## Étape 1 — Type opaque `cbx_archive_t`

Dans `cbx_archive.h`, ne déclarez qu'un **type incomplet** :

```c
typedef struct cbx_archive cbx_archive_t;
```

La définition complète vit dans `cbx_archive.c`. API minimale :

```c
cbx_archive_t *cbx_archive_create(const char *path);      /* crée/écrase l'archive */
cbx_archive_t *cbx_archive_open(const char *path);        /* ouvre + valide header  */
int  cbx_archive_add_entry(cbx_archive_t *, const cbx_entry_desc *);
int  cbx_archive_close(cbx_archive_t *);                  /* écrit header+table, libère */
```

La table des entrées est un **tableau dynamique de pointeurs**
(`malloc`/`realloc`/`free`), grandissant par blocs — pas de limite fixe.
Toutes les erreurs d'allocation sont traitées ; `cbx_archive_close` libère tout.

## Étape 2 — Compression RLE au niveau octet

`cbx_rle.c/h` — un codage **RLE (run-length encoding)** maison, sur le modèle
« paquets de longueur » :

```
# Format de paquet proposé (à préciser dans FORMAT.md)
octet de contrôle c :
  si bit 7 = 0  → (c + 1) octets littéraux suivent (copie)
  si bit 7 = 1  → l'octet suivant est répété (c & 0x7F) + 2 fois
```

Prototypes :

```c
size_t cbx_rle_encode(const uint8_t *in,  size_t n,  uint8_t *out, size_t cap);
size_t cbx_rle_decode(const uint8_t *in,  size_t n,  uint8_t *out, size_t cap);
/* retour : taille produite, ou (size_t)-1 si tampon de sortie trop petit */
```

Contraintes et pièges à traiter :

- la taille d'un run et d'un paquet littéral est **bornée par l'octet de
  contrôle** (127/128) — les longues répétitions donnent plusieurs paquets ;
- **règle de repli** : si `rle_encode` produit plus que l'original, l'entrée
  est stockée brute (bit RLE à 0) — l'archive ne doit jamais grossir à cause
  de la compression ;
- testez les cas limites : entrée vide, un seul octet, 300 octets identiques,
  données aléatoires (incompressibles), paquet littéral de exactement 128
  octets.

## Étape 3 — Chiffrement XOR à flot élémentaire

`cbx_crypto.c/h` — un **chiffre à flot XOR** avec flot de clés généré par un
PRNG déterministe maison :

```c
void cbx_crypto_init(cbx_cipher *c, const char *passphrase);
void cbx_crypto_stream(cbx_cipher *c, uint8_t *data, size_t n); /* XOR in place */
```

- la passphrase est dérivée en graine 32 bits par un **hash FNV-1a** écrit par
  vos soins ;
- le flot de clés vient d'un **xorshift32** ou LCG implanté à la main — pas de
  `rand()` (ni reproductibilité garantie, ni qualité) ;
- le même appel `cbx_crypto_stream` chiffre et déchiffre (XOR involutif) ;
- le bit « chiffré » est positionné dans `mode` ; `extract` demande la
  passphrase (`--key`) et échoue proprement si le CRC final ne correspond pas
  (mauvaise clé détectée par `entry_crc32`).

> **Lucidité exigée** : un XOR à flot maison n'est **pas** de la cryptographie
> sûre (flot réutilisé si deux entrées partagent la même passphrase sans
> nonce, PRNG prévisible…). Documentez honnêtement ces limites dans
> `FORMAT.md` : en embarqué réel, on utilise mbedTLS/libsodium — ici
> l'objectif est de maîtriser la manipulation octet par octet et le
> raisonnement clé/flot/CRC.

## Étape 4 — `cbox pack`

Empaquette une archive depuis une source au choix, en appliquant la chaîne de
transformation du `FORMAT.md` :

```
$ cbox pack --dir mon_repertoire [--rle] [--key secret] sortie.cbx
$ cbox pack --synth --seed 42 --count 100 [--rle] [--key secret] sortie.cbx
```

Exemple de session (mode verbeux) :

```
$ cbox pack --dir firmware/ --rle --key s3cret firmware.cbx -v
[pack] source     : repertoire "firmware/"
[pack] app.bin    : 65536 o -> 18234 o (RLE, 27.8%)
[pack] config.ini : 412 o   -> 412 o   (incompressible, stockage brut)
[pack] boot.img   : 20480 o -> 20480 o (RLE desactive : gonfle a 20512 o)
[pack] 3 entrees, 86428 o bruts -> 39126 o stockes (45.3%)
$ echo $?
0
```

Le binaire écrit respecte le `FORMAT.md` : magic, version, header CRC32 sur
les champs du header, CRC32 par entrée (sur les données brutes), zone DATA en
concaténation des données transformées.

## Étape 5 — `list`, `extract`, `verify`, `info`

- `cbox list archive.cbx` : table des entrées (nom, taille brute, taille
  stockée, drapeaux RLE/XOR, mode) alignée ;
- `cbox extract archive.cbx dest/ [--key secret]` : restaure les fichiers —
  déchiffre puis décompresse, et vérifie chaque `entry_crc32` sur les données
  brutes ;
- `cbox verify archive.cbx` : revalide l'intégralité (magic, version, header
  CRC, CRC de chaque entrée après transformation inverse) et retourne 0 si et
  seulement si tout est sain ;
- `cbox info archive.cbx` : résumé (version, flags, nombre d'entrées, taille
  totale brute vs stockée → **taux de compression global**).

Exemple de sorties attendues (largeurs et textes libres, l'information est
imposée) :

```
$ cbox list firmware.cbx
NAME                RAW       STORED   FLAGS  MODE
app.bin             65536      18234   RLE    -rwx
config.ini            412        412   -      -rw-
boot.img            20480      20480   -      -r-x
3 entrees

$ cbox info firmware.cbx
Format    : CBX1 version 1
Entrees   : 3
Taille    : 86428 o bruts, 39126 o stockes (45.3%)
Fichier   : firmware.cbx (39286 o)

$ cbox verify firmware.cbx
OK : magic, header CRC32, 3/3 entrees valides
$ echo $?
0

$ cbox extract firmware.cbx out/ --key mauvaise
ERREUR : crc invalide pour "app.bin" (attendu 1a2b3c4d, obtenu 9f8e7d6c)
         passphrase incorrecte ?
$ echo $?
4
```

## Étape 6 — Dispatch par pointeurs de fonctions

Le `main.c` ne contient **aucun `if`/`switch` en cascade** sur la
sous-commande : une **table de dispatch** `{ "pack", do_pack }, …` parcourue
et appelée par pointeur de fonction (écho direct du bonus « menu météo »).
Ajoutez des macros de journalisation conditionnelle (`DEBUG`/`VERBOSE` au
compilateur via `-D`) et des codes retour homogènes documentés dans le
`README.md`.

## Étape 7 — Hygiène mémoire et Git

1. Passez `valgrind --leak-check=full` sur **les cinq sous-commandes** avec
   des archives de tailles variées (1, 100, 1000 entrées, compressées ou non,
   chiffrées ou non) ; corrigez toute fuite.
2. Provoquez un **merge volontairement conflictuel** entre vos deux branches
   (même fonction modifiée des deux côtés) et résolvez-le proprement.
3. Faites une **revue croisée** : chacun relit le module écrit par l'autre et
   laisse ses remarques dans la Merge Request.

**Livrables de la Phase 2** — tags `v0.3` (pack + RLE + XOR opérationnels,
merge propre) et `v0.4` (CLI complet, memory-clean).

---

# PHASE 3 — Protocole de trame, qualité, CI/CD et restitution

> Objectif : industrialiser. Transfert de l'archive sur un canal octet par un
> protocole de trame maison, tests, scripts shell, analyse du binaire produit,
> pipeline GitLab, et démonstration finale.

## Étape 1 — Analyse mémoire du binaire : `RAPPORT-MEMOIRE.md`

Analysez votre propre binaire et rédigez un court rapport :

- `size`, `readelf -S`, `objdump -h` : identifiez et expliquez le rôle des
  sections `.text`, `.rodata`, `.data`, `.bss` **telles qu'elles se matérialisent
  dans cbox** (où atterrit la table CRC si vous en avez une ? les chaînes de
  format du `printf` ? les variables globales initialisées vs non initialisées ?) ;
- `nm` : quelques symboles remarquables (visibilité de vos fonctions, `static`) ;
- `valgrind` : résumé des blocs alloués au pic, toute anomalie restante.

## Étape 2 — Tests et scripts

1. `tests/` : un mini-harnais **sans framework externe**, à base d'assert()`
   (vecteurs connus du CRC32, aller-retour sérialisation/désérialisation,
   cas limites du RLE — vide, run long, incompressible —, aller-retour
   XOR avec bonne et mauvaise clé, round-trip création + relecture d'une
   archive synthétique). Compilé avec `-DNDEBUG`, le harnais doit rester muet.
2. `scripts/run_tests.sh` : génère un corpus via `source_synthetic` (graine
   fixe), enchaîne pack → extract → verify (avec et sans `--rle`/`--key`),
   **compare** les fichiers extraits à l'original (`diff -r`), et échoue
   (code retour non nul) au premier problème. Ce script est la base du stage
   `test` de la CI.
3. `scripts/log_report.sh` : synthétise les logs verbeux de cbox avec
   `grep`/`sed`/`awk` (nombre d'entrées, taux de compression, erreurs).
4. `cppcheck --enable=all` sur `src/` : corrigez toute remarque.

## Étape 3 — Protocole de trame maison : `cbox send` / `cbox recv`

Le transfert d'un firmware vers une cible se fait sur un **canal octet sans
structure** (liaison série). Concevez dans `cbx_frame.c/h` un protocole de
trame **dans l'esprit SLIP/HDLC**, spécifié dès la Phase 1 dans `FORMAT.md` :

```
# Squelette imposé (à préciser/justifier dans FORMAT.md)
Trame :
  [FLAG][TYPE][SEQ][LEN (2 octets, LE)][PAYLOAD (LEN octets)][CRC16]
Octets réservés (FLAG, ESC) dans le payload : échappement à la SLIP
  FLAG = 0xC0, ESC = 0xDB → ESC 0xDC pour FLAG, ESC 0xDD pour ESC
Types de trame : DATA, FIN (fin de flux), éventuellement ACK/NACK (bonus)
SEQ : numéro de trame sur 8 bits, incrémenté — détection de perte/réordonnancement
CRC16 : checksum du contenu de trame (polynôme CCITT, implémentation maison)
```

L'abstraction du canal d'entrée/sortie se fait **par pointeurs de fonctions**
(même pattern que `cbx_source`) : `fn_write`/`fn_read` sur `void *ctx` — avec
deux implémentations : `stdio` (fichier/stdout) et un `channel_corrupt`
de test qui altère ou perd des octets de façon déterministe (graine).

Sous-commandes :

```
$ cbox send archive.cbx > flux.bin      # découpe l'archive en trames
$ cbox recv flux.bin archive.cbx        # réassemble, valide SEQ + CRC16
```

Comportements exigés :

- réassemblage **octet pour octet** identique à l'original (vérifié par le
  CRC32 du fichier complet dans `run_tests.sh`) ;
- détection propre (message + code retour dédié) d'un octet altéré, d'une
  trame perdue (trous dans `SEQ`) et d'un flux tronqué ;
- `recv` sur un flux bruité par `channel_corrupt` échoue **sans crash ni
  fuite**.

Exemple de session (mode verbeux) :

```
$ cbox send firmware.cbx -v > flux.bin
[send] 39286 o en 8 trames de 4096 octets max (payload echappe)
[send] trame FIN emise, 8/8 acquittees localement

$ cbox recv flux.bin firmware_reçu.cbx -v
[recv] trame 0 : 4096 o, crc16 ok
[recv] trame 1 : 4096 o, crc16 ok
...
[recv] trame 7 : 614 o, crc16 ok
[recv] FIN recu : 8 trames, sequence complete 0-7
[recv] crc32 du fichier reassemble : 5f3c9a1b (identique a l'original)

$ cbox recv flux_corrompu.bin out.cbx -v
[recv] trame 0 : 4096 o, crc16 ok
[recv] trame 1 : crc16 invalide (attendu e41f, obtenu 77a2)
ERREUR : donnees alterees dans la trame 1
$ echo $?
5
```

## Étape 4 — Pipeline CI/CD GitLab

Écrivez `.gitlab-ci.yml` avec **quatre stages** :

| Stage     | Job minimal                                        |
| --------- | -------------------------------------------------- |
| `build`   | `make` (échec si warning avec `-Werror` en CI)     |
| `test`    | `scripts/run_tests.sh` + harnais `tests/`          |
| `quality` | `cppcheck` strict                                  |
| `package` | archive `.tar.gz` (binaire + README), **sur tag uniquement** |

Puis mettez en place la discipline d'équipe :

1. **Protégez `main`** (« pipelines must succeed », fusion par MR uniquement) ;
2. ouvrez une MR depuis une branche `feature/` contenant **un test cassé
   exprès** et vérifiez que la fusion est **bloquée** par le pipeline rouge ;
3. corrigez, faites repasser le pipeline au vert, fusionnez ;
4. taguez `v1.0` et poussez : le stage `package` doit se déclencher et
   produire un artefact téléchargeable.

## Étape 5 — Robustesse

Votre outil doit **échouer proprement**, sans crash ni fuite, sur : archive
inexistante, magic erroné, version inconnue, table des entrées incohérente
(offset/size hors zone DATA), archive tronquée, checksum invalide, mauvaise
passphrase, trame corrompue ou perdue. Chaque cas produit un message d'erreur
explicite sur `stderr` et un code retour dédié. Exemple :

```
$ cbox list bidon.cbx
ERREUR : magic invalide (attendu "CBX1", lu "\x89PNG") — pas une archive cbox
$ echo $?
2

$ xxd -s 0 -l 16 bidon.cbx   # l'etudiant doit savoir verifier par lui-meme
00000000: 8950 4e47 0d0a 1a0a ...
```

## Étape 6 — Bonus (binômes en avance)

Au choix, sans obligation :

- **ARQ stop-and-wait** : trames ACK/NACK avec réémission, `send`/`recv`
  reliés par deux tubes (`mkfifo`) pour un vrai dialogue bidirectionnel ;
- **Huffman** : remplacer/compléter le RLE par un codage de Huffman
  (construction de l'arbre, table canonique stockée dans l'archive) ;
- **`cbox diff`** : comparer deux archives (entrées ajoutées/supprimées/modifiées) ;
- **troisième source `source_manifest`** : liste de chemins lue depuis un
  fichier texte (commentaires `#`) ;
- **stage CI `lint`** avec `clang-format --dry-run` ;
- **croisement éclair** pour ARM + test sous QEMU (clin d'œil au module
  Linux embarqué à venir) ;
- **`--stats`** : temps d'empaquetage, taux de compression par entrée et
  débit via `clock_gettime`.

## Étape 7 — Restitution

En fin de journée 3 : **démonstration de 15-20 minutes par binôme** — build
depuis zéro, pipeline vert sur GitLab, démo live des sous-commandes (dont un
`send`/`recv` sur flux corrompu), analyse mémoire, et défense de vos choix de
conception (format, transformations, protocole, gestion d'erreurs).
Finalisez le `README.md` (usage, architecture, limites connues) avant le tag
`v1.0`.

---

## Checklist finale du projet

- [ ] Dépôt Git : historique lisible, branches par membre, au moins un conflit
      résolu, tags `v0.1` → `v1.0`
- [ ] `main` protégée ; une MR a été bloquée par un pipeline rouge puis passée
- [ ] `.gitlab-ci.yml` : `build`/`test`/`quality`/`package`, pipeline vert ;
      le tag `v1.0` a produit un artefact
- [ ] `FORMAT.md` complet et cohérent avec le code (`.cbx` **et** trames)
- [ ] Compilation `-std=c11 -Wall -Wextra` sans warning
- [ ] Les 5 sous-commandes + `send`/`recv` fonctionnent, Valgrind propre
- [ ] RLE avec repli brut, XOR avec détection de mauvaise clé par CRC
- [ ] `scripts/build.sh`, `run_tests.sh`, `log_report.sh` opérationnels
- [ ] `RAPPORT-MEMOIRE.md` (sections, symboles, valgrind)
- [ ] `README.md` (usage, architecture, choix, limites)
- [ ] Démonstration réalisée

---

## Critères d'évaluation

La grille d'évaluation sera communiquée ultérieurement.

---

## Rappels et pièges

- **Sérialisation** : jamais de `fwrite` de `struct` brute (padding,
  endianness). Encoder champ par champ.
- **Ordre des transformations** : brut → RLE → XOR au pack, XOR → RLE → brut
  à l'extraction. Chiffrer *avant* de compresser détruit la compressibilité
  (le flot pseudo-aléatoire est incompressible) — à comprendre, pas à subir.
- **RLE** : taille de run bornée par l'octet de contrôle ; toujours prévoir
  le repli brut ; tester le paquet littéral de exactement 128 octets.
- **XOR** : le chiffre n'est inviolable que si le flot n'est jamais réutilisé ;
  une mauvaise clé doit être détectée par le CRC, pas par un crash.
- **Trames** : l'échappement SLIP garantit que `FLAG` n'apparaît jamais dans
  les données — sans lui, un payload contenant `0xC0` ferait croire à une fin
  de trame. Testez explicitement un payload contenant `0xC0` et `0xDB`.
- **`readdir`** ne garantit pas l'ordre : triez la table des entrées avant
  écriture si vous voulez des archives reproductibles byte pour byte.
- **CRC et `const`** : une fonction de checksum ne modifie pas ses entrées —
  signature `const uint8_t *`.
- **`static`** : tout ce qui n'a pas à être visible hors de son module est
  `static` (vérifiable avec `nm`).
- **CI** : le runner part d'un répertoire vide — n'importez jamais un état
  local ; tout doit être reconstruit par `make` et les scripts.

---

# Annexe — Exemples d'utilisation et de sorties

> Cette annexe présente un **scénario complet de bout en bout** : elle donne
> un aperçu concret du résultat attendu. Les textes et largeurs de colonnes
> sont libres — **l'information affichée est imposée**. Les codes retour sont
> normalisés : `0` succès, `1` usage, `2` format invalide, `3` E/S, `4`
> CRC/passphrase, `5` protocole de trame.

## A.1 — Aide et usage

```
$ cbox
cbox — conteneur binaire .cbx
Usage : cbox <sous-commande> [options] <archive>

Sous-commandes :
  pack     Empaqueter des fichiers dans une archive
  list     Lister les entrées d'une archive
  extract  Extraire les entrées vers un répertoire
  verify   Vérifier l'intégrité d'une archive
  info     Afficher un résumé de l'archive
  send     Envoyer une archive sur un flux de trames (stdout)
  recv     Réassembler une archive depuis un flux de trames

Options communes : --rle, --key <passphrase>, -v (verbeux)
$ echo $?
1

$ cbox pack
ERREUR : arguments insuffisants
Usage : cbox pack (--dir <repertoire> | --synth --seed <n> --count <n>)
                [--rle] [--key <passphrase>] <sortie.cbx>
$ echo $?
1
```

## A.2 — `pack` : création d'archives

Depuis un répertoire réel, avec compression et chiffrement (mode verbeux) :

```
$ cbox pack --dir firmware/ --rle --key s3cret firmware.cbx -v
[pack] source   : repertoire "firmware/"
[pack] app.bin    : 65536 o -> 18234 o (RLE, 27.8%)
[pack] config.ini :   412 o ->   412 o (incompressible, stockage brut)
[pack] boot.img   : 20480 o -> 20480 o (RLE desactive : gonfle a 20512 o)
[pack] 3 entrees, 86428 o bruts -> 39126 o stockes (45.3%)
$ echo $?
0
```

Depuis le générateur synthétique (corpus déterministe pour les tests et la
CI — deux exécutions avec la même graine produisent des archives
identiques) :

```
$ cbox pack --synth --seed 42 --count 100 --rle corpus.cbx
[pack] source   : generateur synthetique (graine 42)
[pack] 100 entrees, 512000 o bruts -> 210444 o stockes (41.1%)
```

## A.3 — `list` et `info` : inspection

```
$ cbox list firmware.cbx
NAME                RAW       STORED   FLAGS  MODE
app.bin             65536      18234   RLE    -rwx
config.ini            412        412   -      -rw-
boot.img            20480      20480   -      -r-x
3 entrees

$ cbox info firmware.cbx
Format    : CBX1 version 1
Entrees   : 3
Taille    : 86428 o bruts, 39126 o stockes (45.3%)
Fichier   : firmware.cbx (39286 o)
```

Une entrée chiffrée affiche le drapeau `XOR` ; une entrée à la fois
compressée et chiffrée affiche `RLE,XOR`.

## A.4 — `extract` : extraction et erreurs de clé

```
$ cbox extract firmware.cbx out/ --key s3cret
out/app.bin     65536 o  ok
out/config.ini    412 o  ok
out/boot.img    20480 o  ok
3 entrees extraites

$ cbox extract firmware.cbx out/ --key mauvaise
ERREUR : crc invalide pour "app.bin" (attendu 1a2b3c4d, obtenu 9f8e7d6c)
         passphrase incorrecte ?
$ echo $?
4
```

## A.5 — `verify` : validation d'intégrité

```
$ cbox verify firmware.cbx
OK : magic, header CRC32, 3/3 entrees valides
$ echo $?
0

$ cbox verify firmware_tronque.cbx
ERREUR : archive tronquee (614 o attendus dans la zone DATA, 512 o lus)
$ echo $?
4

$ cbox list bidon.cbx
ERREUR : magic invalide (attendu "CBX1", lu "\x89PNG") — pas une archive cbox
$ echo $?
2

$ xxd -s 0 -l 16 bidon.cbx   # verifier par soi-meme
00000000: 8950 4e47 0d0a 1a0a ...
```

## A.6 — `send` / `recv` : transfert sur flux trammé

Transfert nominal :

```
$ cbox send firmware.cbx -v > flux.bin
[send] 39286 o en 8 trames de 4096 octets max (payload echappe)
[send] trame FIN emise

$ cbox recv flux.bin firmware_recu.cbx -v
[recv] trame 0 : 4096 o, crc16 ok
[recv] trame 1 : 4096 o, crc16 ok
...
[recv] trame 7 :  614 o, crc16 ok
[recv] FIN recu : 8 trames, sequence complete 0-7
[recv] crc32 du fichier reassemble : 5f3c9a1b (identique a l'original)

$ cmp firmware.cbx firmware_recu.cbx && echo "octet pour octet identique"
octet pour octet identique
```

Flux corrompu, trame perdue et flux tronqué :

```
$ cbox recv flux_corrompu.bin out.cbx -v
[recv] trame 0 : 4096 o, crc16 ok
[recv] trame 1 : crc16 invalide (attendu e41f, obtenu 77a2)
ERREUR : donnees alterees dans la trame 1
$ echo $?
5

$ cbox recv flux_trou.bin out.cbx -v
[recv] trame 0 : 4096 o, crc16 ok
[recv] trame 1 : 4096 o, crc16 ok
[recv] trame 3 : 4096 o, crc16 ok
ERREUR : trame 2 manquante (sequence 1 puis 3)
$ echo $?
5

$ cbox recv flux_tronque.bin out.cbx
ERREUR : fin de flux sans trame FIN (7 trames recues)
$ echo $?
5
```

## A.7 — Cas d'usage complet type

La séquence qu'un ingénieur d'intégration rejouerait à chaque livraison :

```
$ cbox pack --dir livraison/ --rle --key "$PASS" fw.cbx
$ cbox verify fw.cbx
$ cbox send fw.cbx > /dev/ttyUSB0          # vers la cible (ici stdout)
$ cbox recv /dev/ttyUSB0 fw_recu.cbox      # cote cible
$ cbox recv --check-only /dev/ttyUSB0      # validation sans ecriture (optionnel)
```
