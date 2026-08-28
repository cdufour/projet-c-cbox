# `cbx_rle` et `cbx_crypto` — explication pas à pas (niveau débutant)

> Note formateur pour la Phase 2 du projet CBox (étapes 2 et 3). Compagne de
> `notes-stagiaires/cbx_archive.md` et `cbx_source.md`. Volontairement **sans
> code d'implémentation** : les algorithmes sont à écrire par les binômes —
> cette note explique les idées, les pièges et le raisonnement.

## Le rôle de ces deux modules dans la chaîne

Au moment du `pack`, chaque fichier subit deux transformations l'une après
l'autre, **dans cet ordre imposé** :

```
brut ──→ compression RLE ──→ chiffrement XOR ──→ stocké dans la zone DATA
```

et l'inverse **exact** à l'extraction :

```
stocké ──→ déchiffrement XOR ──→ décompression RLE ──→ brut
```

Deux idées à bien capter avant d'écrire la moindre ligne :

1. **Ce sont des transformations d'octets, pures et sans état de fichier** :
   elles reçoivent un tampon, rendent un tampon. Elles ne savent rien des
   archives, des CRC, des fichiers — c'est `cbox_archive` qui les appelle au
   bon moment. Un module = un métier.
2. **L'ordre est important et contre-intuitif** : on compresse **avant** de
   chiffrer. Dans l'autre sens, le flot pseudo-aléatoire du chiffre rendrait
   les données incompressibles (le RLE ne trouverait aucune répétition à
   exploiter) — et l'archive ne gagnerait rien. C'est un vrai piège
   d'ingénierie, à comprendre plutôt que subir.

---

# Partie 1 — `cbx_rle` : la compression

## L'idée en une phrase

Le RLE (run-length encoding) remplace les **répétitions** par une description
compacte : au lieu d'écrire `AAAAAAA` (7 octets), on écrit en substance
« 7 fois A » (2 octets). C'est la compression la plus simple qui existe — et
elle est très efficace sur les données d'embrayage typiques (firmware avec
des zones de zéros, images avec des aplat de couleur).

## Le format de paquet proposé par l'énoncé

Tout le flux compressé est une suite de **paquets**, chacun introduit par un
octet de contrôle `c` :

- si le **bit 7 de `c` vaut 0** → `c` décrit un paquet **littéral** :
  `(c + 1)` octets qui suivent sont à recopier tels quels ;
- si le **bit 7 vaut 1** → `c` décrit un paquet de **run** : l'octet qui
  suit est à répéter `(c & 0x7F) + 2` fois.

Le `& 0x7F` masque le bit de type pour ne garder que les 7 bits de longueur.
Pourquoi le `+1` et le `+2` ? Parce qu'un paquet de longueur **zéro** n'a
aucun sens : décaler les bornes d'au moins 1 (littéral) ou 2 (un run de 1
n'a pas d'intérêt, il serait plus court en littéral) permet de couvrir
davantage de tailles utiles avec les 7 bits disponibles. Ce sont des choix
d'encodage classiques — à justifier dans votre `FORMAT.md`.

## Le raisonnement de l'encodeur, en français

Parcourez le tampon d'entrée avec deux compteurs : la longueur du run courant
(octets identiques qui se suivent) et le littéral courant (octets tous
différents). À chaque position, posez-vous une seule question :

> « Suis-je en train de voir une répétition worth un paquet run, ou des
> octets sans rapport worth un paquet littéral ? »

Quand vous rencontrez 2 octets identiques ou plus, émettez (ou terminez) un
paquet run ; sinon, accumulez en littéral. Les subtilités sont toutes dans
les **bornes** :

- un run ne peut pas dépasser la capacité d'un octet de contrôle (bornes
  autour de 127/128) : une répétition de 300 octets identiques donnera
  **plusieurs paquets run** à la suite — il faut couper ;
- un littéral non plus : un bloc de 200 octets tous différents donnera
  plusieurs paquets littéraux ;
- **le cas exactement 128** est le piège énoncé : un littéral de pile 128
  octets doit être encodable. Vérifiez mentalement vos bornes avec `n = 127`,
  `128`, `129` avant de coder.

## Le raisonnement du décodeur

C'est une boucle très simple, et c'est **le seul endroit du module où l'on
lit l'octet de contrôle** : lire `c`, regarder son bit 7, en déduire « copier
les N suivants » ou « répéter l'octet suivant N fois », avancer. Le décodeur
est toujours plus simple que l'encodeur — si le vôtre est compliqué,
réfléchissez à deux fois.

## La règle de repli : ne jamais gonfler l'archive

Point **imposé** par l'énoncé : si `rle_encode` produit **plus** que
l'original (données incompressibles — aléatoires, déjà compressées…), alors
l'entrée est stockée **brute**, avec le bit RLE à 0 dans son `mode`. La
compression ne doit jamais faire perdre de place. Notez que cette décision
n'appartient pas à `rle_encode` lui-même (il ne fait que transformer) mais à
l'appelant, qui compare les tailles et positionne le drapeau.

Conséquence importante pour la suite : à l'extraction, c'est ce **bit RLE du
`mode`** qui dit s'il faut décompresser ou pas. Le décodeur n'a jamais à
« deviner ».

## Les cas limites à tester (liste de contrôle)

- entrée **vide** ;
- **un seul** octet ;
- **300 octets identiques** (plusieurs paquets run) ;
- données **aléatoires** (incompressibles → repli brut) ;
- paquet littéral de **exactement 128** octets ;
- un run qui **commence** ou **finit** exactement sur une borne de paquet ;
- aller-retour : `decode(encode(x)) == x` pour tous les cas ci-dessus.

---

# Partie 2 — `cbx_crypto` : le chiffrement XOR à flot

## L'idée en une phrase

On fabrique un très long flux d'octets pseudo-aléatoires (le **flot de
clés**) à partir d'une passphrase, et on XOR chaque octet de la donnée avec
l'octet correspondant du flot. Comme XOR est son propre inverse
(`a ^ k ^ k == a`), **la même opération chiffre et déchiffre**.

## Les trois étages de la machine

L'énoncé impose une chaîne précise, chaque étage réinvestissant des acquis :

1. **Dérivation de clé (FNV-1a)** : la passphrase (chaîne de longueur
   quelconque) est transformée en une graine de 32 bits par un hash simple
   que vous écrivez vous-même. L'idée du FNV-1a : partir d'une constante,
   et pour chaque octet, XOR puis multiplication par une constante — deux
   lignes, à la portée de tous, et réellement utilisé en vrai.
2. **Générateur pseudo-aléatoire maison** : la graine alimente un xorshift32
   ou un LCG — un PRNG déterministe que vous implémentez à la main. **Pas de
   `rand()`** : ni reproductible d'une plateforme à l'autre, ni d'une
   qualité suffisante. Le PRNG produit la suite d'octets du flot de clés.
3. **XOR in place** : chaque octet de donnée est XORé avec l'octet de flot
   courant, directement dans le tampon (aucune allocation nécessaire — d'où
   le « in place » du prototype).

Le mot clé de tout cet étage est **déterminisme** : même passphrase → même
flot → même chiffré. C'est ce qui permet de déchiffrer plus tard, et c'est
aussi ce qui rend les tests possibles (aller-retour avec bonne clé, échec
CRC avec mauvaise clé).

## Pourquoi « la même fonction chiffre et déchiffre »

C'est la propriété d'**involution** du XOR : appliquer deux fois la même
opération avec le même flot annule la première application. Concrètement,
`cbx_crypto_stream` ne sait même pas s'il chiffre ou déchiffre — il XOR,
point. C'est le fait de **repartir du même état du PRNG** (donc de refaire
`cbx_crypto_init` avec la même passphrase) qui fait que le second appel
restaure l'original.

Piège associé : si vous chiffrez deux tampons avec le **même contexte**
sans le réinitialiser entre les deux, le flot **continue** au lieu de
repartir du début — le second déchiffrement échouera. Un contexte neuf
(ou réinitialisé) par entrée et par sens.

## Comment détecte-t-on une mauvaise passphrase ?

Le module crypto, lui, **ne le peut pas** — et c'est voulu : un flot XOR
n'a aucune redondance qui permette de dire « c'est faux ». La détection se
fait **à l'étage du dessus** : le `entry_crc32` stocké dans l'archive porte
sur les données **brutes**. À l'extraction, on déchiffre, on décompresse,
on recalcule le CRC — s'il ne correspond pas, la passphrase était mauvaise
(ou les données corrompues). Message d'erreur propre et code retour dédié
(le 4), jamais un crash. Retenez la leçon d'architecture : **l'intégrité se
vérifie au niveau applicatif, pas dans le chiffre**.

## Honnêteté intellectuelle exigée

Un XOR à flot maison n'est **pas** de la cryptographie sûre, et l'énoncé
demande de le documenter honnêtement dans le `FORMAT.md` :

- si deux entrées sont chiffrées avec la même passphrase **sans nonce**, le
  flot est **réutilisé** — et XOR de deux chiffrés donne XOR des deux clairs
  (la plus grosse faiblesse possible pour ce type de chiffre) ;
- un PRNG maison est prévisible : qui connaît quelques octets du flot peut
  reconstruire la suite ;
- en embarqué réel, on utilise mbedTLS ou libsodium.

L'objectif pédagogique ici est la **manipulation octet par octet** et le
raisonnement clé/flot/CRC — pas la sécurité. Écrire « ce chiffre n'est pas
sûr, voici pourquoi, voici ce qu'on utiliserait en production » dans votre
`FORMAT.md` vaut mieux qu'un silence gênant en soutenance.

---

# Pièges transversaux aux deux modules

- **Les bornes, toujours les bornes** : RLE (127/128, run multiples) comme
  crypto (taille du flot = taille des données) explosent d'abord sur les cas
  limites. La liste de contrôle de test ci-dessus est votre filet.
- **Tampons de sortie** : les prototypes prennent `out` + capacité `cap` et
  rendent `(size_t)-1` si c'est trop petit — jamais de `malloc` caché dans
  la transformation. Qui alloue, c'est l'appelant.
- **`const` sur les entrées** : une transformation ne modifie pas ce qu'elle
  lit. Signature `const uint8_t *in` pour l'encodeur — sauf pour le XOR
  *in place*, qui modifie explicitement son tampon (et c'est son contrat).
- **Taille produite inconnue à l'avance** : au `pack`, la taille stockée
  dépend de la compression (ou du repli brut) — c'est pour ça que
  `size`/`raw_size` sont deux champs distincts dans la table des entrées.

## Résumé en une image

```
Au pack :    brut ──RLE──> plus petit (si possible) ──XOR──> illisible
À l'extract : stocké ──XOR──> compressé ──RLE──> brut ──CRC──> "identique ?"

RLE : "AAAAAAA"  →  "7 fois A"            (littéral vs run, bornes 127/128)
XOR : donnée ^ flot(passphrase)            (involutif, déterministe,
                                            mauvaise clé vue par le CRC)
```

Deux modules minuscules, mais c'est eux que la Phase 3 testera le plus
finement (harnais `assert()`, cas limites, aller-retours) — soignez-les
tôt.
