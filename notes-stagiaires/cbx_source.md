# `cbx_source` — explication pas à pas (niveau débutant)

> Note formateur pour la Phase 1 du projet CBox (étape 5). Destinée aux
> stagiaires qui découvrent le module `cbx_source.c` et l'abstraction de
> source par pointeurs de fonctions. Compagne de `notes-stagiaires/cbx_archive.md`.

## Le problème que `cbx_source` résout

Pour faire un `cbox pack`, il faut bien obtenir les fichiers à empaqueter
*d'une manière ou d'une autre*. Or il y a au moins deux façons :

- lire un **vrai répertoire** sur disque (`firmware/` avec `app.bin`,
  `config.ini`…) — c'est l'usage réel ;
- **générer** un corpus de fichiers déterministe à partir d'une graine —
  indispensable pour les tests et la CI : deux exécutions avec la même graine
  doivent produire des archives identiques, sinon impossible de vérifier quoi
  que ce soit.

On pourrait écrire deux `pack` différents… mais tout le reste de la chaîne
(transformation, table, écriture) serait **dupliqué**. La bonne question à se
poser est :

> « Est-ce que le reste du programme a besoin de savoir D'OÙ viennent les
> fichiers ? »

Non. Qu'un fichier vienne d'un répertoire réel ou d'un générateur, une fois
qu'on a son nom et son contenu, c'est pareil. `cbx_source` est la brique qui
formalise cette idée : **une seule interface, plusieurs implémentations**.

## L'analogie du tapis roulant

Imagine un chef (`cbox pack`) devant un tapis roulant. Il ne sait pas — et il
n'a pas à savoir — qui alimente le tapis : une équipe qui dépose des colis
réels, ou une machine qui fabrique des colis de test. Le chef fait une seule
chose : « donne-moi le colis suivant » jusqu'à ce qu'on lui réponde « c'est
vide ».

C'est exactement l'interface `cbx_source` :

```
         ┌─────────────────────────────────┐
         │  cbox pack (le chef)            │
         │  "donne-moi l'entrée suivante"  │
         └────────────┬────────────────────┘
                      │  next()
        ┌─────────────┴─────────────┐
        │                           │
  source_dir                 source_synthetic
  (lit un vrai répertoire)   (génère à partir d'une graine)
```

C'est le **même pattern que `capteur.h` de la station météo** : un capteur
réel ou un capteur simulé, derrière la même interface. Ici on l'applique côté
*entrée de données* — et on le retrouvera une troisième fois en Phase 3 avec
le canal de transfert (`cbx_frame`).

## L'interface, ligne par ligne

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

### `cbx_entry_desc` — « la fiche du colis »

La description d'UN fichier à empaqueter : son nom, sa taille, et de quoi
lire son contenu. C'est ce que `cbox_add_entry` recevra (cf. la note
`cbx_archive`). Le `cbx_archive` ne sait pas d'où vient cette fiche — et
c'est tout l'intérêt.

### `fn_source_next` — « donne-moi le suivant »

C'est un **pointeur de fonction** : une variable qui contient l'adresse d'une
fonction, qu'on peut appeler. Décomposons la signature :

```c
int (*fn_source_next)(void *ctx, cbx_entry_desc *out);
 └┬┘  └────┬────┘┘──────────┬─────────────────┘
 type   nom du       paramètres
        pointeur
```

- `void *ctx` — le **contexte** : un pointeur vers les données privées de
  l'implémentation (le répertoire ouvert pour `source_dir`, l'état du
  générateur pour `source_synthetic`). `void *` = « pointeur sur n'importe
  quoi » ; chaque implémentation sait ce qu'elle y a rangé et le reconvertit
  avec un cast.
- `cbx_entry_desc *out` — paramètre **de sortie** : la fonction remplit la
  fiche du colis à cette adresse (on passe un pointeur pour que l'appelant
  voie la fiche remplie).
- **valeur de retour** — un code à trois états, à respecter à la lettre :
  - `1` : une entrée a été remplie dans `*out`, traite-la ;
  - `0` : fini, plus rien à donner ;
  - `< 0` : erreur (répertoire illisible, éclaircie disque…).

### `cbx_source` — le couple fonction + contexte

```c
typedef struct {
    fn_source_next next;
    void *ctx;
} cbx_source;
```

La struct ne contient que **deux pointeurs** : la fonction à appeler et les
données privées qu'il faut lui passer. C'est la façon C de faire un « objet
avec une méthode » : `src->next(src->ctx, &desc)` se lit
« appelle ta fonction sur ton contexte » — l'équivalent objet de
`source.getNext()`, sans objet ni classe.

## Comment le chef consomme le tapis

Côté `cbox pack`, le code est **identique quelle que soit la source** :

```c
cbx_source src;              /* initialisée par source_dir ou source_synthetic */
cbx_entry_desc desc;
int r;

while ((r = src.next(src.ctx, &desc)) == 1) {
    cbox_add_entry(a, &desc);   /* range ce colis dans l'archive */
}
if (r < 0) {
    /* erreur de la source : message + code retour dédié */
}
```

Aucun `if` sur « est-ce un répertoire ou un générateur ? » : le choix a été
fait **une fois**, au moment de remplir `src`. C'est ça, la puissance du
pointeur de fonction : le décideur choisit l'implémentation, le consommateur
n'en dépend plus.

## Implémentation 1 : `source_dir` — les colis réels

Elle parcourt un **vrai répertoire** avec `opendir`/`readdir` :

```c
/* Aperçu du squelette (simplifié) */
typedef struct {
    DIR           *dir;
    const char    *base;    /* chemin du répertoire scanné */
} dir_ctx;

static int dir_next(void *ctx, cbx_entry_desc *out)
{
    dir_ctx *c = ctx;    /* on retrouve notre contexte — c'est le réflexe clé */
    /* À toi : boucler sur readdir, filtrer "." / ".." / cachés,
       remplir *out (name, size via stat, moyen de lire le contenu),
       renvoyer 1 tant qu'il reste un fichier, 0 quand c'est fini */
}

cbx_source source_dir(const char *path)
{
    /* opendir, allouer le contexte, retourner { dir_next, ctx } */
}
```

Deux règles de l'énoncé à ne pas oublier :

- **premier niveau uniquement** : pas de descente récursive dans les
  sous-dossiers — le format `.cbx` est à plat, les noms d'entrée ne sont pas
  des chemins ;
- **`readdir` ne garantit pas l'ordre** : si on veut des archives
  reproductibles octet par octet, trier les noms avant de les émettre
  (sinon deux `pack` du même répertoire peuvent donner des fichiers
  différents).

## Implémentation 2 : `source_synthetic` — les colis de test

Elle **fabrique** N fichiers à partir d'une graine (`--seed 42 --count 100`) :

```c
typedef struct {
    uint32_t seed;      /* graine du PRNG (xorshift/LCG maison) */
    uint32_t count;     /* nombre de fichiers à générer */
    uint32_t emitted;   /* combien déjà envoyés */
} synth_ctx;

static int synth_next(void *ctx, cbx_entry_desc *out)
{
    /* À toi : gérer le quota (emitted/count) et générer nom + contenu
       déterministes depuis (seed, emitted) — même graine, mêmes fichiers */
}
```

Le point crucial est le **déterminisme** : même graine → mêmes noms, mêmes
tailles, mêmes contenus, dans le même ordre. C'est ce qui permet à
`scripts/run_tests.sh` (Phase 3) de faire pack → extract → `diff -r` et de
comparer avec un CRC attendu : sans corpus reproductible, pas de tests
fiables ni de CI possible.

## Pourquoi c'est le « bon » design (à retenir pour la soutenance)

1. **Un seul consommateur** : `cbox pack` est écrit une fois, testé une fois,
   et marche avec toutes les sources — y compris celles qui n'existent pas
   encore.
2. **Extension sans modification** : le bonus `source_manifest` (liste de
   chemins lue dans un fichier texte) s'ajoute en écrivant une troisième
   fonction `manifest_next` + son contexte — **zéro changement** dans le
   reste du code. C'est le principe ouvert/fermé, version C.
3. **Testabilité** : on peut tester toute la chaîne d'empaquetage sans
   toucher au disque réel, avec des données dont on connaît tout.
4. **Même pattern partout** : `capteur.h` (météo), `cbx_source` (entrée des
   données), `cbx_frame` (sortie/canal, Phase 3) — une fois compris, le
   pattern se reconnaît et se réécrit de tête.

## Pièges classiques

- **Le contexte par paramètre, pas global** : résister à la tentation d'une
  variable globale pour l'état de la source — le `ctx` dans la struct, passé
  en premier paramètre de la callback, permet d'avoir plusieurs sources
  vivantes en même temps et rend le code réentrant.
- **Le cast du contexte** : la callback reçoit un `void *` ; la première
  ligne est toujours de le reconvertir en son vrai type
  (`dir_ctx *c = ctx;`). Si tu te trompes de type, ça compile parfois quand
  même et ça explose au runtime — c'est le prix du `void *`, à connaître.
- **`"."` et `".."`** : `readdir` les retourne ; les filtrer sinon tu essaies
  d'empaqueter le répertoire lui-même (récursion infinie ou erreur).
- **Libérer le contexte** : `opendir` → `closedir`, `malloc` du contexte →
  `free`. Prévoir une fonction de nettoyage (ou documenter qui libère quoi) —
  valgrind le vérifiera.
- **Fini ≠ erreur** : `0` (fin normale du tapis) et `< 0` (panne) ne se
  traitent pas pareil côté appelant. Ne pas fusionner les deux.

## Résumé en une image

```
cbx_source = un tapis roulant à deux cases
┌───────────────────────┐
│ next : « suivant ! »  │ ← le pointeur de fonction (le moteur)
│ ctx  : état privé     │ ← les données du fournisseur (le stock)
└───────────────────────┘
     alimenté par source_dir (vrais fichiers)
     ou source_synthetic (fichiers générés, déterministes)

Le chef (cbox pack) : « next… next… next… » jusqu'à 0, erreur si < 0.
```

L'interface est minuscule (deux pointeurs, une fonction, trois codes
retour), mais c'est elle qui découple le *producteur* des données du
*consommateur* — exactement ce qu'un HAL (couche d'abstraction matérielle)
fait en embarqué entre le driver générique et le capteur réel.
