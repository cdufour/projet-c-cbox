# cbx_crc — à quoi sert ce module ?

Module de checksum **CRC32** (IEEE 802.3, polynôme réfléchi 0xEDB88320).
Son intérêt est double :

## 1. Intérêt pédagogique (le vrai but du module)

Premier module de la Phase 1, positionné comme exercice de révision des acquis J2 :

- **opérateurs bit à bit** : décalages, XOR ;
- **tableaux** : implémentation possible par table précalculée de 256 entrées ;
- **typage strict** : `stdint.h`, `size_t`.

C'est un exercice autonome, facile à tester, qui ne dépend d'aucune autre brique
du projet — idéal pour se remettre le pied à l'étrier avant les modules plus
complexes (`cbx_io`, `cbx_rle`, `cbx_crypto`…).

## 2. Intérêt fonctionnel dans le format CBox

Le CRC32 est utilisé à deux endroits du format d'archive :

- `header_crc32` : protège l'en-tête (magic, version, flags, nombre d'entrées)
  contre la corruption ;
- `entry_crc32` : calculé sur les données **brutes** (avant compression RLE et
  chiffrement XOR). Il valide donc l'intégrité **de bout en bout** : à
  l'extraction, après décompression et déchiffrement, on peut détecter qu'une
  archive est corrompue ou que le mot de passe XOR était erroné — le XOR avec
  une mauvaise clé donne des données « valides » en apparence, et c'est le CRC
  qui permet de le détecter.

## 3. Intérêt « en général » d'un checksum type CRC32

Au-delà de CBox, les checksums (CRC, Adler-32, hash légers…) sont omniprésents
en système et réseau :

- **détection de corruption** : un support de stockage (disque, flash), un
  transfert (réseau, câble série) ou un process peut altérer des octets
  (coupure, rayure, bruit électrique, bug logiciel). Le checksum stocké à côté
  des données permet de détecter l'altération au moment de la relire ;
- **validation sans lire tout le contenu** : quelques octets suffisent pour
  vérifier l'intégrité d'un bloc, sans comparaison coûteuse avec l'original
  (qu'on n'a d'ailleurs souvent plus) ;
- **légèreté et déterminisme** : contrairement à un hash cryptographique
  (SHA-256…), un CRC se calcule très vite (quelques cycles/octet avec une
  table), avec un résultat identique sur toute machine — choix délibéré ici :
  on veut détecter des erreurs *accidentelles*, pas un attaquant ;
- **limites à connaître** : un CRC détecte les corruptions accidentelles
  (erreurs en rafale jusqu'à 32 bits, toute erreur sur un nombre impair de
  bits) mais n'est **pas** une preuve d'authenticité : il est trivial de
  forger des données avec un CRC valide. D'où son usage pour l'intégrité,
  pas la sécurité.

Exemples concrets : trames Ethernet (FCS CRC32), ZIP/GZIP/PNG (CRC32),
Zmodem/XMODEM, SATA/USB, en-têtes de firmware (uImage), etc.
