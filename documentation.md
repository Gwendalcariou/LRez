# Description du projet LRez

LRez est un projet écrit en C++ qui fournit à la fois :

- un outil en ligne de commande
- une bibliothèque réutilisable dans d'autres projets.

Son objectif est de manipuler des données de séquençage de type linked-reads, c'est-à-dire des lectures associées à un barcode. Ce barcode permet de regrouper des reads qui proviennent probablement d'une même molécule d'ADN.

Concrètement, LRez permet de :

- comparer des régions génomiques ou des extrémités de contigs à partir de leurs barcodes communs
- extraire les barcodes présents dans une région d'un fichier BAM
- calculer des statistiques générales sur un fichier BAM
- indexer des fichiers BAM ou FASTQ par barcode
- interroger ces index pour retrouver rapidement les reads ou alignements associés à un barcode ou à une liste de barcodes.

## Rappels & définitions de biologie

**Read :** C'est une petite portion de la séquence d'ADN (ex : ACTGATAGCT...ATCGGGA)
**Contig :** Quand plusieurs reads se chevauchent, alors on peut les assembler et c'est cet assemblage qui forme un contig. Mais ça ne permet pas de reformer la séquence d'ADN en entier et donc on obtient plusieurs contigs séparés.
**Scaffolding :** C'est le fait de relier plusieurs contigs entre eux même si on ne connaît pas la séquence entre ces contigs (ex : contig A --- (zone inconnue) --- contig B, on obtient un scaffold)

**Région génomique :** C'est un intervalle dans un génome (ex : chromosome X, position 1000 à 2000). Pour vulgariser, c'est simplement une zone dans l'ADN.

Quand on séquence de l'ADN, on casse une longue molécule en plein de petits reads mais on perd une information essentielle : quels reads viennent du même petit morceau d'ADN ?

**Barcode :** Pour résoudre ce problème, avant de casser l'ADN, on va ajouter une étiquette (barcode) à chaque molécule. Ensuite on fragmente la molécule et chaque read qu'on y trouve a toujours l'étiquette attachée en amont. On retrouve donc facilement les mêmes molécules à travers les différents reads qui ont tous leur barcode.

Sans les barcodes on ne voit réellement que des positions locales alors qu'avec eux, on obtient une information longue distance et même si deux contigs, avec beaucoup de barcodes partagés, sont éloignés, on peut facilement estimer s'ils sont issus de la même région de la séquence ou pas.

**Linked-reads :** Les linked-reads consistent à associer un identifiant (barcode) à des fragments issus d'une même molécule d'ADN, permettant de regrouper des reads courts et de reconstruire des relations à longue distance dans le génome.

### Types de variation

#### Variations simples à petite échelle

**Polymorphisme :** Un seul nucléotide change (ex : AGTA devient AGTC)
**Insertion :** Ajout de nucléotide(s) (ex : ATG devient ATGGA)
**Délétion :** Suppression de nucléotide(s) (ex : ATGT devient ATG)
On parle aussi d'indel en bio-informatique pour désigner une insertion ou une délétion sans précisément dire quoi.

##### Variations structurelles

**Duplication :** Copie d'une portion d'ADN (ex : ATTGA devient ATTGAATTGA)
**Inversion :** Une partie de la séquence est inversée. Prenons un exemple : ATCG. L'inversion de ATCG est le complément de la séquence inversée. On prend son complément qui est TAGC, son inverse c'est GCTA puis le complément de la séquence inversé, noté comp(GCTA), est CGAT.
**Translocation :** Une portion de la séquence change de position soit dans le même chromosome, soit dans un autre.

### Séquencage ADN

L’ADN est une très longue séquence (jusqu’à des milliards de nucléotides) et donc les machines ne peuvent pas le lire directement. C'est pourquoi on va fragmenter l’ADN en petits morceaux appelés reads et chaque read est séquencé individuellement. Ensuite on obtient des millions de reads (fichier FASTQ).
Enfin on peut procéder à deux étapes finales :

- Soit on assemble les reads
- Soit on les aligne sur le génome de référence (Notion de position)

**Position :** Quand on parle de position, on parle d'un endroit spécifique dans le génome qu'on peut qualifier comme : chr1:1000 et qui signifie base 1000 du chromosome 1, ce qui veut dire que la séquence commence à la position 1000 du chromosome 1.
Après alignement, chaque read possède une position (fichier BAM). Quand on parle d'alignement c'est quand on confronte le read actuel avec la séquence d'origine pour connaître sa position de départ (modulo quelques variations j'imagine).

### Types de fichier

**Fichier FASTQ :** C’est le format brut du séquençage, chaque read contient son identifiant ainsi que sa séquence. Exemple :

```
>read1
ATCGTAGTAGTCGATCGTAGCGACTT
>read2
ATCGTGCTGCCCCAAATTGTGGCTGT
[...]
```

Concrètement, les fichiers FASTQ contiennent les reads non alignés.

**Fichier BAM :**
C'est "l'étape suivante" de FASTQ, on va y retrouver les reads alignés sur le génome de référence ce qui inclut dorénavant la position en plus d'un identifiant ainsi que d'une séquence.

### Léviathan

Léviathan est un outil en bioinformatique assez proche de LRez. Il permet de détecter des liens entre régions génomiques à partir des barcodes. Concrètement Léviathan répond à la question suivante : "Est-ce que deux régions partagent beaucoup de barcodes ?".

Comment fonctionne-t'il ? De ce que j'ai compris il récupère les données liées à un fichier BAM, ensuite il associe des régions à des ensembles de barcodes, puis il procède à l'intersection des différents ensembles et enfin il en déduit si oui ou non les régions sont liées en fonction des résultats.

Il permet de faire beaucoup de choses en lien avec le séquencage d'ADN :

- Relier les contigs, par exemple si deux régions partagent des barcodes alors elles sont probablement voisines
- Détecter les variations structurelles, par exemple si deux régions très éloignées partagent des barcodes alors il y a une possiblie translocation
- Analyser la cohérence du génome en vérifiant si l'assemblage est "logique"

L'outil ne regarde pas directement les séquences mais se base uniquement sur les barcodes et leurs relations.

### Léviathan & LRez

Les deux outils reposent sur le même principe mais ne font pas exactement la même chose.
Ils utilisent tous les deux les linked-reads, les barcodes, les fichiers BAM et des comparaisons d’ensembles de barcodes. En effet ils partagent une même logique scientifique.

Cependant Léviathan est un outil spécifique conçu pour une tâche précise : trouver des liens entre régions ou détecter des variations structurelles. On a un pipeline déjà défini et on l'utilise pour une analyse donnée.
En comparaison, LRez est un outil plus générique qui est plutôt une bibliothèque en CLI qui permet d'extraire des barcodes, indexer des données, faire des requêtes et calculer des stats.

## Organisation générale du projet

Le dépôt est organisé autour de plusieurs dossiers qui n'ont pas tous le même rôle. Certains contiennent le code développé pour LRez, tandis que d'autres correspondent à des dépendances externes ou à de la documentation générée.

## Description détaillée des dossiers

### `src`

C'est le dossier principal du projet. Il contient le code source C++ de LRez, c'est-à-dire l'implémentation du comportement du programme.

On y trouve :

- le point d'entrée principal du programme, avec `main.cpp`
- les fichiers `.cpp` qui implémentent les fonctionnalités métier
- un sous-dossier `subcommands` pour les commandes accessibles dans le terminal
- un sous-dossier `include` pour les fichiers d'en-tête.

Autrement dit, si on veut comprendre comment fonctionne réellement LRez, c'est le dossier le plus important à lire.

### `src/include`

Ce dossier contient les fichiers d'en-tête `.h`.

Leur rôle est de déclarer :

- les fonctions
- les structures ou types utilisés
- les interfaces entre les différentes parties du programme

En C++, on sépare souvent les déclarations dans les fichiers `.h` et l'implémentation dans les fichiers `.cpp`. Ce dossier sert donc à définir l'"architecture visible" du projet.

Par exemple, on y trouve les déclarations des modules qui gèrent :

- l'extraction de barcodes
- la comparaison de barcodes
- l'indexation des BAM et des FASTQ
- les statistiques
- la récupération de reads ou d'alignements

### `src/subcommands`

Ce dossier contient les sous-commandes de l'outil en ligne de commande `LRez`.

Le programme fonctionne avec une logique de commandes, par exemple :

- `LRez compare`
- `LRez extract`
- `LRez stats`
- `LRez index bam`
- `LRez query bam`
- `LRez index fastq`
- `LRez query fastq`

Chaque fichier de ce dossier correspond donc à une commande ou sous-commande spécifique. Son rôle est de :

- lire les arguments passés par l'utilisateur
- vérifier qu'ils sont corrects
- appeler les fonctions du cœur du programme situées dans les autres fichiers de `src`
- formater les résultats en sortie, soit à l'écran, soit dans un fichier.

On peut dire que `src/subcommands` sert d'interface entre l'utilisateur et la logique interne de LRez.

#### `Commande : LRez compare`

Cette commande sert à mesurer combien de barcodes sont partagés entre plusieurs régions ou entre des extrémités de contigs.

Elle peut être utilisée de trois façons :

- avec `--regions`, pour comparer toutes les régions listées dans un fichier entre elles
- avec `--contig`, pour comparer les extrémités d'un contig donné avec les extrémités des autres contigs
- avec `--contigs`, pour faire ce même type de comparaison sur une liste de contigs.

Dans le cas des contigs, elle s'appuie sur un index BAM construit au préalable avec `LRez index bam --offsets`.

Utilité :

- repérer des régions génomiques qui partagent beaucoup de molécules
- identifier des contigs potentiellement voisins ou reliés
- aider des tâches de scaffolding ou d'analyse structurale.

Le résultat produit des lignes du type `élément1 élément2 nombre_de_barcodes_communs`.

#### `Commande : LRez extract`

Cette commande extrait les barcodes présents dans un fichier BAM.

Elle peut fonctionner :

- sur une région précise avec `--region`
- sur l'ensemble du BAM avec `--all`

Par défaut, les barcodes sont dédupliqués : un barcode n'apparaît qu'une seule fois dans la sortie. Avec `--duplicates`, on conserve au contraire toutes les occurrences observées.

Utilité :

- récupérer les barcodes associés à une zone d'intérêt
- construire des listes de barcodes à réutiliser dans d'autres étapes
- explorer manuellement le contenu barcodé d'un BAM

La sortie est simplement une liste de barcodes, un par ligne.

#### `Commande : LRez stats`

Cette commande calcule des statistiques globales sur un fichier BAM barcodé.

Elle produit notamment :

- le nombre total de barcodes distincts
- le nombre de reads alignés
- la distribution du nombre de reads par barcode
- la distribution du nombre de barcodes par région
- la distribution du nombre de barcodes communs entre régions adjacentes

Les options `--regions` et `--size` permettent de définir combien de régions sont échantillonnées et quelle taille de région est utilisée pour ces statistiques locales.

Utilité :

- avoir une vue d'ensemble de la qualité ou de la structure d'un jeu de données linked-reads
- estimer la densité de barcodes dans le BAM
- vérifier si les barcodes apportent bien une information de voisinage entre régions proches

#### `Commande : LRez index bam`

Cette commande construit un index à partir d'un fichier BAM afin d'accélérer les recherches futures par barcode.

Deux types d'index peuvent être créés :

- `--offsets` : index des offsets des alignements dans le BAM
- `--positions` : index des positions d'occurrence des barcodes sous forme `(chromosome, position de début)`

Elle peut aussi filtrer les données indexées :

- avec `--primary`, pour ne garder que les alignements primaires
- avec `--quality`, pour imposer une qualité minimale d'alignement.

Utilité :

- préparer les requêtes rapides sur BAM
- éviter de rescanner tout le fichier à chaque recherche
- fournir l'index nécessaire à `query bam` et à certaines comparaisons de contigs

L'index produit est enregistré dans un fichier de sortie fourni par l'utilisateur.

#### `Commande : LRez query bam`

Cette commande interroge un index BAM déjà construit pour retrouver les alignements associés à un barcode donné ou à une liste de barcodes.

Elle nécessite :

- le fichier BAM
- un index généré par `LRez index bam --offsets`.

La requête peut se faire :

- avec `--query`, pour un seul barcode
- avec `--list`, pour un fichier contenant plusieurs barcodes.

Les alignements retrouvés sont renvoyés au format SAM. Avec `--header`, on peut aussi inclure l'en-tête SAM.

Utilité :

- récupérer rapidement les reads alignés associés à une molécule ou à un groupe de molécules
- extraire les alignements pertinents pour une analyse ciblée
- réutiliser un sous-ensemble d'alignements sans relire tout le BAM

#### `Commande : LRez index fastq`

Cette commande construit un index des barcodes présents dans un fichier FASTQ afin de retrouver plus tard rapidement les reads qui leur sont associés.

Elle prend en charge :

- les FASTQ non compressés
- les FASTQ gzippés avec l'option `--gzip`

Dans le cas d'un FASTQ compressé, le programme construit aussi un index gzip intermédiaire pour pouvoir accéder efficacement aux reads sans décompresser entièrement le fichier à chaque requête.

Utilité :

- préparer des recherches rapides dans de gros FASTQ
- relier directement un barcode à la position de ses reads
- servir de base à `query fastq`

#### `Commande : LRez query fastq`

Cette commande interroge l'index d'un FASTQ pour récupérer les reads associés à un barcode ou à un ensemble de barcodes.

Elle peut fonctionner de trois façons :

- avec `--query`, pour un seul barcode
- avec `--list`, pour un fichier contenant une liste de barcodes
- avec `--collectionOfLists`, pour un fichier qui contient plusieurs noms de fichiers de listes de barcodes

Dans le dernier cas, LRez génère un fichier de sortie FASTQ pour chaque liste, avec le suffixe `.fastq`.

L'option `--gzip` permet d'interroger un FASTQ compressé, à condition qu'il ait été indexé correctement au préalable.

Utilité :

- récupérer les reads bruts associés à une région ou à un ensemble de barcodes d'intérêt
- créer rapidement des sous-FASTQ ciblés
- chaîner une extraction de barcodes dans un BAM avec une récupération des reads correspondants dans le FASTQ

### `utils`

Ce dossier contient des scripts utilitaires en Python.

Ils ne font pas partie du cœur C++ de LRez, mais ils sont utiles pour préparer certaines données en entrée. De ce que j'ai compris, ces scripts servent surtout au prétraitement de fichiers issus de certaines technologies de séquençage barcodé, comme :

- stLFR
- TELL-Seq.

Le but est d'adapter ou de reformater les données pour que les barcodes soient stockés de manière compatible avec LRez, notamment via le tag `BX:Z`.

En résumé, `utils` sert à préparer les données quand elles ne sont pas directement exploitables par l'outil principal.

### `docs`

Ce dossier contient la documentation HTML générée du projet.

<<<<<<< HEAD
Ce n'est pas du code source à modifier en priorité en Rust j'imagine. Il s'agit plutôt d'une documentation produite automatiquement à partir des commentaires du code C++.
=======
Ce n'est pas du code source à modifier en priorité pour le transfert en Rust je pense. Il s'agit plutôt d'une documentation produite automatiquement à partir des commentaires du code C++.

> > > > > > > 3ada55669b06bc70dcad75469ae1e2f183484dc8

Elle permet :

- de naviguer dans les fichiers sources
- de voir les fonctions, classes, structures et dépendances
- de mieux comprendre l'API C++ fournie par le projet.

Ce dossier est donc surtout utile pour consulter la documentation, pas pour développer directement de nouvelles fonctionnalités.

### `bamtools`

Ce dossier ne correspond pas à du code écrit directement pour LRez. C'est une dépendance externe ajoutée au projet sous forme de sous-module Git.

`bamtools` est une bibliothèque C++ spécialisée dans la lecture, l'écriture et la manipulation des fichiers BAM.

Elle est essentielle pour LRez, car une grande partie des fonctionnalités du projet repose sur l'analyse de fichiers BAM contenant des alignements de reads.

En pratique, `bamtools` permet notamment à LRez de :

- ouvrir des fichiers BAM
- lire les alignements un par un
- accéder aux régions génomiques
- récupérer les tags associés aux alignements, comme les barcodes
- utiliser les index BAM pour accéder rapidement à certaines zones

On peut donc considérer `bamtools` comme une bibliothèque externe sur laquelle LRez s'appuie pour toute la partie lecture et parcours des BAM.

### `CTPL`

Ce dossier contient également une dépendance externe, elle aussi ajoutée comme sous-module Git.

`CTPL` est une petite bibliothèque de parallélisation qui fournit un système de thread pool. Un thread pool permet de lancer plusieurs traitements en parallèle sans gérer manuellement chaque thread à bas niveau.

Dans LRez, cette bibliothèque est utilisée pour accélérer certains traitements coûteux, par exemple lors de l'indexation d'un fichier BAM en répartissant le travail sur plusieurs threads.

On peut donc dire que `CTPL` sert à gérer le multithreading dans LRez, afin d'améliorer les performances sur les grosses données.

## Fichiers importants à la racine

En plus des dossiers, quelques fichiers à la racine du projet sont importants pour comprendre son fonctionnement :

### `Makefile`

Il contient les règles de compilation du projet.

Il sert à construire :

- l'exécutable `LRez`
- la bibliothèque `liblrez`
- les dépendances nécessaires comme `bamtools`.

### `install.sh`

Ce script lance simplement la compilation du projet avec `make`.

Il automatise l'installation locale du programme et de la bibliothèque.

### `.gitmodules`

Ce fichier indique que le projet utilise des sous-modules Git, ici :

- `bamtools`
- `CTPL`

<<<<<<< HEAD

## Idées pour transformer le code C++ en Rust

Pour l'instant je n'y ai pas encore réfléchi

### Notions à travailler pour la transformation en Rust

#### Structure de données stockant les données de mapping (BAM)

TODO

#### Les différentes étapes de Leviathan et comment l'index est utilisé ?

TODO

#### Quelles sont les autres structures de données utilisées ?

TODO

#### Alternatives Rust aux libraries bamtools et CTPL

##### bamtools

**rust-htslib** - https://docs.rs/rust-htslib/latest/rust_htslib/ :
Ca suit l’écosystème HTS “classique”, car la crate fournit des bindings vers HTSlib avec une API Rust de plus haut niveau. Elle gère SAM, BAM et CRAM, propose la lecture/écriture, un IndexedReader pour les requêtes par région, et même un support de thread pool côté I/O/compression.

**noodles-bam** - https://docs.rs/noodles-bam/latest/noodles_bam/ :
C’est une alternative plus dans l'esprit de comment est codé Rust de ce que j'ai compris. La crate noodles-bam est dédiée au format BAM, et l’écosystème noodles couvre aussi d’autres formats bioinfo. Elle gère lecture/écriture BAM et fournit aussi des exemples de lecture complète et de requêtes sur enregistrements. En pratique, c’est souvent un bon choix si tu veux éviter des bindings C autant que possible et rester sur une architecture plus idiomatique Rust.

##### CPTL

**rayon** - https://docs.rs/rayon/latest/rayon/struct.ThreadPoolBuilder.html :
Son objectif est surtout de faire du data parallelism ou du fork-join. Elle permet de créer un pool personnalisé avec ThreadPoolBuilder, d’exécuter du travail dans ce pool, et fournit join, scope et les itérateurs parallèles.

**crossbeam** - https://docs.rs/crossbeam/latest/crossbeam/thread/struct.Scope.html :
C'est surtout bien pour executer plusieurs tâches qui empruntent des références locales sans s'occuper de tout ce que ça englobe. Ce n’est pas un thread pool complet comme rayon, mais les scoped threads sont très pratiques pour structurer du parallélisme bas niveau de façon sûre.
