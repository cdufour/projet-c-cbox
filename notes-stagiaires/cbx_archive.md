# `cbx_archive` — explication pas à pas

> Note formateur destinée aux stagiaires qui découvrent le module
> `cbx_archive.c` et le type opaque `CBoxArchive` (Phase 2)

## L'idée en une phrase

`cbx_archive.c` est le **chef d'orchestre de l'archive** : il prend plusieurs
fichiers en entrée, et produit **un seul fichier binaire** `.cbx` qui les
contient tous, avec un sommaire pour s'y retrouver — exactement comme quand on
range plusieurs objets dans un carton et qu'on colle une liste sur le
couvercle : « voici ce qu'il y a dedans, et où ».

## Le fichier `.cbx` vu de l'extérieur

Un fichier `.cbx` final est organisé en trois zones, l'une après l'autre :

```
[ HEADER  ]  ← la petite étiquette : "je suis une archive CBX, version 1,
               il y a 3 fichiers dedans" + un code de contrôle (CRC)
[ TABLE   ]  ← le sommaire : une ligne par fichier
               ("app.bin, taille 65536, rangé à tel endroit, compressé")
[ DATA    ]  ← le contenu réel des fichiers, transformé (compressé/chiffré),
               collés bout à bout
```

Le **header** et la **table** sont la « fiche d'information », les **DATA**
sont le « contenu ». Pour lire une archive, on lit d'abord le sommaire, puis
on va piocher dans les DATA à l'endroit indiqué.

## Le type opaque : une boîte noire

Dans `cbx_archive.h`, on ne met qu'une seule ligne :

```c
typedef struct cbx_archive CBoxArchive;
```

C'est une **promesse sans détails** : « il existe une structure qui s'appelle
CBoxArchive, mais son contenu ne regarde que `cbx_archive.c` ». La définition
complète vit en privé dans le `.c`. L'utilisateur de la bibliothèque ne
manipule jamais la struct directement — il ne passe que des pointeurs aux
quatre fonctions de l'API. Ça s'appelle un **type opaque**, et c'est ce qui
force à utiliser l'API proprement.

## Étape par étape : créer une archive

### 1. `cbox_create("fw.cbx")` — ouvrir le chantier

```c
CBoxArchive *a = cbox_create("fw.cbx");
```

Ce que ça fait :

- `fopen("fw.cbx", "wb")` → crée le fichier (ou le **vide** s'il existait
  déjà) ;
- `malloc` une struct `cbx_archive` avec dedans : le `FILE *`, un tableau
  d'entrées **vide**, et une zone DATA **vide** ;
- retourne le pointeur, ou `NULL` si ça a échoué (fichier non ouvrable,
  malloc raté).

À ce stade : **rien n'est écrit sur le disque**. Tout est en mémoire, prêt à
recevoir.

### 2. `cbox_add_entry(a, &desc)` — ranger un fichier (×N)

Appelée **une fois par fichier** à empaqueter. Pour chaque fichier, elle :

1. **Lit** le contenu du fichier source (via la `cbx_entry_desc`, une
   abstraction qui sait d'où viennent les données — répertoire réel ou
   générateur de test) ;
2. **Transforme** les données : brut → compression RLE → chiffrement XOR (en
   appelant les modules `cbx_rle` et `cbx_crypto`) ;
3. **Calcule** le CRC32 des données **brutes** d'origine — c'est l'empreinte
   qui servira plus tard à vérifier que l'extraction est fidèle ;
4. **Stocke tout en mémoire** :
   - les données transformées → concaténées à la fin de la zone DATA en
     mémoire ;
   - une ligne de sommaire → ajoutée à la table des entrées : nom, taille
     brute, taille stockée, **offset** (= position dans la zone DATA), mode,
     CRC.

```c
while (source.next(source.ctx, &desc) == 1) {
    cbox_add_entry(a, &desc);   /* répété pour chaque fichier */
}
```

À ce stade : toujours **rien sur le disque**. La mémoire ressemble à :

```
struct cbx_archive
├── FILE *fp            (fichier ouvert, vide)
├── entries[0] : "app.bin",     offset 0,     crc ...
├── entries[1] : "config.ini",  offset 18234, crc ...
├── data : [app.bin transformé][config.ini transformé]...
```

### 3. `cbox_close(a)` — tout écrire d'un coup

Le grand final. Maintenant qu'on connaît le nombre d'entrées, on peut enfin
écrire le fichier complet, **dans l'ordre naturel** :

1. **Header** : magic `"CBX1"`, version, nombre d'entrées, CRC du header —
   via `cbx_write_header` qui encode champ par champ (jamais de `fwrite` de
   struct brute, à cause du padding et de l'endianness) ;
2. **Table** : chaque ligne du sommaire, via `cbx_write_entry` ;
3. **DATA** : tout le bloc accumulé en mémoire, d'un seul `fwrite` ;
4. `fclose` le fichier ;
5. **Libère toute la mémoire** (`free` des entrées, des DATA, de la struct).

Après ça, le fichier sur disque est complet et valide — et le pointeur `a` ne
doit **plus jamais être utilisé**.

## Et pour lire une archive existante ?

`cbox_open("fw.cbx")` fait le chemin inverse : ouvre en `"rb"`, lit le header
via `cbx_read_header`, et **valide** : est-ce vraiment `"CBX1"` ? Version
connue ? CRC du header correct ? Si non → `NULL` et un message d'erreur. Les
sous-commandes `list`, `extract`, `verify`, `info` partent de là : elles
lisent la table, puis vont chercher dans les DATA uniquement ce dont elles ont
besoin.

## Résumé en une image

```
cbox_create    →  j'ouvre un carton vide (en mémoire)
cbox_add_entry →  je plie chaque objet et le pose dans le carton (en mémoire)
cbox_close     →  j'écris la liste sur le couvercle, je scotte, j'expédie
                   (= j'écris TOUT le fichier, puis je libère la mémoire)
cbox_open      →  je lis le couvercle pour vérifier que c'est bien un carton CBX
```

Le point contre-intuitif à bien intégrer : **le disque n'est touché que par
`cbox_close`** (dans ce design), alors que le fichier final commence par le
header — qu'on ne pouvait pas écrire avant de savoir combien d'entrées il y
aurait. Tout buffer en mémoire jusqu'au bout résout élégamment ce problème.

## Annexes : questions posées par les stagiaires

### « On décale tout le contenu écrit à la fin ? »

Non — un `fseek` ne déplace pas d'octets, il déplace seulement la position
d'écriture. Trois designs possibles pour gérer le header écrit en dernier :

1. **Réserver la place dès `cbox_create`** : header provisoire (taille fixe,
   connue) + zone réservée pour la table (capacité du tableau dynamique, ou
   un maximum généreux). Les DATA sont écrites après la zone réservée, à
   position fixe ; `close` repose header + table dans la place libre. Aucun
   octet ne bouge.
2. **Tout buffer en mémoire (recommandé pour ce projet)** : `add_entry` ne
   touche pas le disque ; `close` écrit header → table → DATA d'un coup.
3. **Fichier temporaire** : les DATA partent dans un `.tmp` pendant les
   `add_entry` ; `close` assemble le fichier final. Bon compromis pour de
   grosses archives.

La table, elle, reste **contiguë et en un seul bloc** : c'est le contrat du
format (`cbox_open` lit `entry_count` puis `entry_count` entrées qui se
suivent). Pas de table fragmentée.

### « La struct doit contenir un pointeur sur des données très conséquentes dans le heap ? »

Oui — c'est le coût assumé du design bufferisé :

```c
struct cbx_archive {
    FILE        *fp;
    char        *path;
    cbx_entry   *entries;      /* table dynamique (malloc/realloc) */
    size_t       count;
    size_t       capacity;
    uint8_t     *data;         /* zone DATA accumulée, potentiellement Mo */
    size_t       data_len;
    size_t       data_cap;     /* elle aussi grandit par realloc */
};
```

- le pic mémoire est borné par la taille de l'archive (on ne garde que la
  version transformée, souvent plus petite avec RLE) ;
- si la RAM devient un critère, les designs 1 et 3 font retomber le pic à
  « une entrée à la fois » ;
- corollaire en lecture : `extract`/`verify` ne chargent **jamais** toute la
  zone DATA — `fseek` à `offset`, lecture de `size` octets seulement ;
- avec `data` dans le heap, l'hygiène de `cbox_close` (tout libérer, y compris
  sur les chemins d'erreur) est vérifiée par valgrind sur un pack de 1000
  entrées.

### « Pourquoi `data` est-il de type `uint8_t *` ? »

- le champ `uint8_t *data;` est un pointeur : toujours la même taille (8 octets
  en 64 bits) — la struct reste de taille fixe ;
- ce qu'il désigne est un bloc heap de taille choisie au `malloc`, variable au
  `realloc`. `uint8_t` décrit la taille **d'un élément** (1 octet), pas celle
  du bloc ;

```c
uint8_t *data = malloc(4096);   /* 4096 octets dans le heap       */
data[0] = 0xC0;                 /* chaque case : 1 octet          */
data = realloc(data, 8192);     /* le bloc grandit : 8192 octets  */
/* sizeof(data) == 8 (le pointeur) — pas 8192 ! */
```

Analogie : le pointeur est une **adresse postale écrite sur un papier de
taille fixe** ; la maison désignée peut être un studio ou un château.
`uint8_t *` dit juste « à cette adresse, les choses sont rangées par caisses
de 1 octet ». Même mécanisme que `cbx_entry *entries` — seule la taille d'un
élément change (`sizeof(cbx_entry)` vs 1).

C'est aussi pourquoi la struct doit suivre `data_len`/`data_cap` à côté du
pointeur : le pointeur seul ne « sait pas » la taille du bloc qu'il désigne.
