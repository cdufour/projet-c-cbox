# L'octet de contrôle du RLE — questions fréquentes des stagiaires

> Note formateur sur la Phase 2 du projet CBox. Compagne de `cbx_rle_crypto.md` (vue d'ensemble) — celle-ci creuse le point qui bloque le plus souvent : **l'octet de contrôle et ses bornes**.
>  Volontairement sans code d'implémentation.

## Rappel du format de paquet

Tout le flux compressé est une suite de paquets. Chaque paquet commence par
un **octet de contrôle** `c`, dont le bit de poids fort sert de drapeau :

```
bit 7 de c = 0  →  paquet LITTÉRAL : recopier tels quels les (c + 1) octets
                   qui suivent l'octet de contrôle
bit 7 de c = 1  →  paquet RUN : l'octet qui suit est répété (c & 0x7F) + 2 fois
```

Exemple complet sur 10 octets d'entrée :

```
entrée   : AA AA AA AA AA 41 62 07 99 1A FF FF FF
            └──── run ×5 ────┘└── littéral ×5 ──┘└─ run ×3 ─┘

compressé : 83 AA   04 41 62 07 99 1A   81 FF
             │        │                  │
             │        │                  └─ run : (1)+2 = 3 fois FF
             │        └─ littéral : 4+1 = 5 octets recopiés
             └─ run : (0x83 & 0x7F)+2 = 5 fois AA
```

Le décodeur n'a jamais rien à deviner : chaque octet de contrôle lui dit
exactement quoi faire, et la seule information nécessaire pour décoder un
paquet est contenue dans son premier octet.

## Q1 — « Que signifie le cas c + 1 octets littéraux ? »

Analogie : on dicte une suite de nombres au téléphone. Les répétitions
(« zéro, zéro, zéro… ») se résument en « combien de zéros » — c'est le run.
Les suites quelconques (« 41, 62, 7, 99 ») ne se compressent pas : le mieux
est d'annoncer la longueur à l'avance — « attention, 5 nombres : 41, 62,
7, 99, 1A ». C'est le littéral : des données recopiées telles quelles,
précédées d'un annonceur de longueur.

Le cas littéral est ce qui rend le RLE **utilisable sur n'importe quelles
données** : sans lui, la première zone sans répétition ferait échouer
l'encodeur.

## Q2 — « Pourquoi +1 (et pourquoi +2 pour le run) ? »

Parce qu'un paquet **vide ne servirait à rien**. Si `c` codait
directement la longueur, `c = 0` annoncerait « zéro octet à recopier » —
un octet de contrôle dépensé pour ne rien décrire. En décalant d'un cran,
les 7 bits couvrent les longueurs **utiles** :

```
valeur de c :      0    1    2   ...  127
littéral     :     1    2    3   ...  128     ← c + 1
run          :     2    3    4   ...  129     ← (c & 0x7F) + 2
```

Le run commence à 2 pour la même raison : un run de 1 octet coûterait
2 octets émis pour 1 décrit (perte sèche). On ne perd aucune longueur
utile, et on gagne le cas limite de 128 en un seul paquet (cf. Q6).

Même idée que les étages d'un immeuble : le rez-de-chaussée existe, donc
le « 1er étage » est le deuxième plancher — un simple décalage d'indices.

## Q3 — « Run : on répète l'octet d'avant c + 2 fois ? »

Deux corrections :

1. c'est l'octet **d'après** le contrôle, pas celui d'avant — chaque paquet
   est autonome (contrôle + données), le décodeur ne regarde jamais en
   arrière ;
2. c'est `(c & 0x7F) + 2`, pas `c + 2` : puisque bit 7 vaut 1, `c` vaut au
   moins `0x80`, et sans le masque la longueur serait fausse de 128. Le
   masque efface le drapeau pour ne garder que les 7 bits de longueur :

```
c        = 0x83 = 1 0000011
                     └──┬──┘ longueur brute = 3
c & 0x7F = 3  →  3 + 2 = 5 répétitions
```

## Q4 — « Quand l'encodeur choisit-il run ou littéral ? »

C'est le cœur de l'encodeur (le décodeur, lui, obéit). À chaque position,
l'encodeur compte les octets identiques qui se suivent :

```
2 octets identiques ou plus (ou 3, selon le seuil choisi)
    → paquet RUN
octet isolé
    → accumuler dans le paquet LITTÉRAL courant
```

Avec deux règles d'évacuation liées aux bornes :

- littéral plein (128 accumulés) → l'émettre, en recommencer un ;
- run plus long que le maximum d'un paquet → le couper en plusieurs runs
  consécutifs (300 octets identiques = plusieurs paquets) ;
- un run qui apparaît **pendant** un littéral en cours → clore le littéral
  **avant** d'émettre le run. C'est une machine à deux états (« j'accumule
  un littéral » / « je compte un run »), pas un if/else isolé.

Subtilité : un run de 2 ne gagne rien (2 émis pour 2 décrits). Certains
encodeurs ne basculent en run qu'à partir de 3 (gain net). Les deux choix
se défendent — **documenter le seuil retenu dans le `FORMAT.md`**.

Garde-fou global : si le résultat compressé dépasse l'original, l'entrée
est stockée brute (bit RLE à 0). Même un encodeur mal réglé ne peut jamais
faire grossir l'archive.

## Q5 — « L'octet de contrôle est-il un délimiteur ? La longueur est-elle indéterminée ? »

Non et non — deux confusions à dissiper :

- **Ce n'est pas un délimiteur, c'est un préfixe de longueur.** Un
  délimiteur encadre la donnée (marque avant / marque après) et oblige à
  interdire ou échapper la marque dans les données. Un préfixe de longueur
  annonce « N octets suivent » : la fin se déduit en **comptant**, et
  aucun octet de donnée ne peut se faire passer pour une frontière — c'est
  pourquoi le format RLE n'a besoin d'aucun échappement, même si les
  données contiennent des valeurs qui ressemblent à un contrôle (`0x00`,
  `0x83`…).
- **La longueur est parfaitement déterminée** : dès la lecture de `c`, le
  décodeur connaît le nombre exact d'octets du paquet (`c + 1`). Rien à
  chercher, aucune fin à deviner.

Contraste utile pour la Phase 3 : le protocole de trame, lui, utilise de
vrais **délimiteurs** (FLAG `0xC0`) — et c'est précisément pourquoi il a
besoin de l'échappement SLIP. Annoncer (compter) vs délimiter (échapper) :
deux stratégies classiques de cadrage, le projet en fait rencontrer les
deux.

## Q6 — « Un littéral de 128 octets : c déborde sur le bit 7 ? »

C'est LE cas limite (celui de la liste de tests de l'énoncé), et la
réponse est contre-intuitive : **128 tient dans un seul paquet**.

```
longueur 128  →  c = 128 − 1 = 127 = 0x7F  →  bit 7 = 0  ✓ littéral
```

`0x7F`, c'est sept 1 et un bit 7 à zéro : toujours dans le domaine
littéral. Le débordement n'arrive qu'à **129** :

```
longueur 129  →  c = 128 = 0x80  →  bit 7 = 1  ✗ drapeau RUN ! collision
              →  il faut couper : littéral de 128 (c = 0x7F) + paquet pour
                  le reste
```

**128 est exactement la frontière** : le maximum encodable en un seul
paquet — et c'est le bénéfice direct du décalage `+1` (sans lui, le
maximum serait 127, et le cas très naturel « pile 128 » exigerait déjà deux
paquets). L'octet qui suit les 128 octets littéraux est forcément un nouvel
octet de contrôle.

## Lien encodeur / décodeur (le pont entre les deux questions)

L'encodeur écrit `longueur − 1` dans `c` ; le décodeur lit `c + 1`. Le
`−1` côté écriture et le `+1` côté lecture sont les deux faces du même
décalage — celui qui garantit que la valeur 0 de `c` n'est jamais un paquet
vide mais un littéral d'un octet.

## Liste de contrôle des tests (rappel)

- entrée vide ;
- un seul octet ;
- 300 octets identiques (plusieurs runs) ;
- données aléatoires (incompressibles → repli brut) ;
- littéral de **exactement 128** octets (un seul paquet, `c = 0x7F`) ;
- littéraux de 127 et 129 octets (bornes de part et d'autre) ;
- un run qui commence ou finit pile sur une borne de paquet ;
- aller-retour `decode(encode(x)) == x` sur tous les cas ci-dessus.
