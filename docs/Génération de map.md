# Génération de map

## Algorithmes utilisés

- Perlin Noise : Création du terrain avec altitude (dénivelé / eau), positionnement des "grosses" forêts sur terrain plat et début dénivelé positif
- Poisson Disk : Positionnement des arbres solo, positionnement des lieux uniques plus gros
- Wave Function Collapse : Génération du contenu des lieux uniques

## Principe général

### Déterminisme

La génération doit être entièrement déterministe. C'est à dire que l'intégralité du terrain ; forme, contenu, arbres, minerais, coffres, doit pouvoir être construit de manière exactement identique à chaque fois qu'il est généré, en fonction d'une seed.

Et ce, même si le terrain n'est pas généré dans le même ordre chacune des fois (génération au fur et à mesure, en fonction du déplacement du personnage sur le terrain par exemple)

Pour ce faire, l'aléatoire est bien sûr permis tout au long de la génération, largement, mais son utilisation doit être réglé et ordonnancé quoi qu'il arrive.

Pour le Perlin Noise, pas de problème puisque l'aléatoire n'est utilisé qu'à l'initialisation de la génération du terrain uniquement (pour la création de ce qui va servir à Perlin Noise tout au long de la génération du terrain. Celle-ci est dite "globale")

Pour le Poisson Disk (partie lieux unique), c'est pareil parce que les lieux uniques sont choisi à l'initialisation

Pour le Poisson disk (partie arbre indépendant, minerais etc) et le WFC (contenu du sol et lieux au fur et à mesure), c'est plus compliqué.
En effet, ces algorithmes ne peuvent pas utiliser l'aléatoire de façon déterministe puisqu'on ne sait pas quelle partie du terrain sera généré avant une autre. 

C'est pourquoi il est nécessaire d'avoir un aléatoire dit "local", sur l'espace de "chunk" . Cet aléatoire provient d'une seed, qui est un dérivé de la seed de la map globale et de la position du chunk.
Quand la moindre case d'un chunk doit être connue, son chunk entier est généré (d'une seule et même manière, avec la seed du chunk)
À l'échelle de la map c'est donc un entre deux ; entre tout générer d'un coup (impossible), et générer case par case (perte de cohérence) 

Ainsi, avec ce système de chunk (micro parties locales) même si les cases ne sont pas dévoilées toutes en même temps, l'intégralité de la map a la possibilité d'être déterministe.

### Ordre

Pour générer le terrain de façon cohérente, l'ordre suivant est adopté pour chacun des chunks et chacune des cases :<br/>
(Chunk de 8 x 8 unités de Perlin x / y (8 cases par unité x / y) donc 64 x 64 cases finales par chunk)

- (par case, seed random global) Perlin Noise pour l'altitude de la case<br/>
 Fonction de lissage pour les altitudes :
  - max moins d'une dizaine de cases pour les pentes (les plus hautes)
  - max 3 pour l'eau (les plus basses)
  - le reste étant du plat
- (chunk, seed random global) Perlin noise pour les grandes forêts
- (chunk, seed random local) Poisson disk pour placer les éléments suivants :
  - Arbres solo
  - Minerais
  - Cueillette
- (par case, seed random local) Wave function collapse pour le contenu (choix des sprites, topping)

## Processus de génération

La génération de découpe en deux grandes parties :
- l'initialisation, pour créer toutes les constantes utiles à la génération des chunks et débuter la génération du terrain pour placement déterministe des lieux uniques
- "en continu" tout le long de la vie du terrain, la génération des chunks, à la demande

### Initialisation
L'initialisation sert :
- d'une part à générer toutes les constantes utiles à tout le terrain (seed globale, tables de permutation pour les deux utilisations de Perlin Noise)
- d'autre part à générer les premières parties des 25 chunks autour du point de départ pour établir de façon déterministe la localisation des lieux uniques du terrain avec la seed globale.<br/>
  Pour ce faire, seule l'utilisation de Perlin Noise pour l'altitude (première étape) est nécessaire. Le reste de la génération se fera au moment voulu (partie deux)

```
// Les méthodes préfixées par "ukn" sont des méthodes inconnues dont on assumera leur bon fonctionnement. Leur utilité étant suggérée par leur nom.

// Start

CHUNK = []
SEED = ukn_timestamp_millis()
RANDOM = ukn_create_random(SEED)

perlin_terrain = Perlin(create_permutation_table(RANDOM))
perlin_forest = Perlin(create_permutation_table(RANDOM))


fn create_permutation_table(random) {
  table = (0..255).ukn_to_array()           // [0, 1, 2, 3 ... 254, 255]
  for (i = table.length - 1; i > 0; i--) {
    ukn_swap(table[i], table[random.rand() * (i - 1)])
  }
}

struct Map {
// chunk [] btreemap
// get_x_y() 
}

struct Chunk {
// tiles [] btreemap 
// get_x_y()
// generate(preload_only:bool))
} 


```

### Chunk
À chaque fois que l'utilisateur se déplace et lorsqu'un chunk non généré doit apparaître, il doit être généré de façon déterministe.<br/>
Il aura donc attribué une seed en fonction de sa position.
Ensuite, l'ordre de génération du chunk est déroulé.

####CODE

