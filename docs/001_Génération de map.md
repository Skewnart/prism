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
(Chunk de 2 x 2 unités de Perlin x / y (16 cases par unité x / y) donc 32 x 32 cases finales par chunk)

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
  Pour ce faire, seule l'utilisation de Perlin Noise pour l'altitude (première étape) est nécessaire. Le reste de la génération se fera au moment voulu (partie deux)<br/>
  C'est pour cela que l'initialisation d'un chunk se fait en deux parties (`init()` et `end_init()`)

```
// Les méthodes préfixées par "ukn" sont des méthodes inconnues dont on assumera leur bon fonctionnement. Leur utilité étant suggérée par leur nom.

// Start

SEED = ukn_timestamp_millis()
RANDOM = ukn_create_random(SEED)

perlin_terrain = Perlin::init(create_permutation_table_1024(RANDOM))
perlin_forest = Perlin::init(create_permutation_table_1024(RANDOM))
permutation_table_seed = create_permutation_table_1024(RANDOM)

map = Map::create(&perlin_terrain, &perlin_forest, &permutation_table_seed);

// Crée une table de permutation de taille 1024
fn create_permutation_table_1024(random) {  // Fisher-yates (je savais pas que cet algo avait un nom omg)
  table = (0..1024).ukn_to_array()   [0, 1, 2 ... 1022, 1023]        
  for (i = table.length - 1; i > 0; i--) {
    ukn_swap(table, i, random.rand() * i)
  }
  return table
}
```

```
struct Map {
  chunks []  // btreemap

  fn create(perlin_terrain, perlin_forest, permutation_table_seed) {
    // génération des 25 cases autour du joueur
    foreach (x_chunk, y_chunk) :
      chunks.add(Chunk::init(x_chunk, y_chunk, perlin_terrain, perlin_forest, permutation_table_seed))
    
    //listage des lieux uniques possibles
    //Choix des lieux
    chunk.ukn_set_unique_area()
  }
  
  // ...
}
```

```
struct Chunk {
  tiles []   // btreemap
  load_state // enum 
  
  // ...

  fn init(x_chunk, y_chunk, perlin_terrain, perlin_forest, permutation_table_seed) {
    
    // ...

    load_state = State.Init

    foreach(x, y) :
      // Pour un chunk en (3, 5), le point (x, y) qui à l'intérieur du chunk en (14, 27)
      // Un chunk de 32 x 32 et 2 unités de perlin par chunk pour rappel
      // Pour Perlin, le point à calculer sera en (3 x 2 + 14 / (32 / 2), 5 x 2 + 27 / (32 / 2))
      //     = ((3 + 14 / 32) x 2, (5 + 27 / 32) x 2)
      //     = (6,875, 11,6875)

      perlin_terrain.get(x, y)
      perlin_forest(x, y)
  }
  
  // ...
} 
```

### Chunk
À chaque fois que l'utilisateur se déplace et lorsqu'un chunk non généré doit apparaître, il doit être généré de façon déterministe.<br/>
Il aura donc attribué une seed en fonction de sa position.
Ensuite, l'ordre de génération du chunk est déroulé.<br/>
A ce moment-là, les deux fonctions d'init pour un chunk sont lancées.

```
struct Map {

  // ...
  
  fn get(x, y) {
    (x_chunk, y_chunk) = Chunk::convert_position(x, y)
    chunk = chunks.try_get(x_chunk, y_chunk)
    if !(chunk exists) {
      chunk = Chunk::init(x_chunk, y_chunk, perlin_terrain, perlin_forest, permutation_table_seed)
      chunk.end_init()
      chunks.add(chunk)
    }
    else {
      if (chunk.state = State.Init) {
        chunk.end_init()
      }
    }

    return chunk.get(x, y)
  }
}
```

```
struct Chunk {
  // ...
  random    // Random

  fn init(x_chunk, y_chunk, perlin_terrain, perlin_forest, permutation_table_seed) {
    SEED = ukn_sizeof(u32) / 1024
          * permutation_table_seed[(permutation_table_seed[x_chunk & 1023] + y_chunk) & 1023]
    random = ukn_create_random(SEED)

    // ...
  }

  fn end_init() {
    load_state = State.EndInit

    Poisson::solo_tree(random)
    Poisson::solo_minerai(random)
    foreach(x, y) :
      WFC
  }

  fn get(x, y) {
    return tiles[x, y]
  }
} 
```

## Utilisation des algorithmes

### Perlin Noise

Perlin noise va nous permettre de générer un terrain avec altitude ou un forêt avec un effet pseudo aléatoire mais cohérent (comme une image bruitée fluidement, utilisé dans le cinéma notamment pour des FX de nuages, de feu, et même les jeux vidéos pour de la génération de map, comme on va faire finalement)

Perlin Noise en tant que tel possède des améliorations de l'algorithme, certaines seront utilisées (interpolation en fading, octaves, grande table de permutation)

TODO : Faire des schémas

```
struct Perlin {
  static VECTOR_TABLE : [...]  //table de concordance entre les entrées de la table de permutation avec un vecteur unique de norme 1 (tous les vecteurs formant un cercle exact), abrégé "VT"

  permutation_table  // [], abrégé "PT"

  fn init (permutation_table) {
    self.permutation_table = permutation_table
  }

  fn get(x, y) {
    // gestion des octaves de perlin avec la fréquence et l'amplitude
    // Pour rappel : Octave, somme de bruits à différentes échelles pour donner du détail
    //  fréquence : Offset, donne du détail. Zoom du bruit. Variable
    //  amplitude : Poids d'une couche. Variable
    //  persistance : Evolution de l'amplitude entre les couches. Constant (classique : 0.5)
    //  lacunarité : Evolution de la fréquence entre les couches. Constant (classique : 2)

    frequence = 0.005
    amplitude = 1

    noise_result = 0
    foreach (i in 0..5) {
      noise_result += noise(x * frequence, y * frequence) * amplitude

      frequence *= lacunarité
      amplitude *= persistance
    }
    return noise_result
  } 

  fn noise(x, y) {
    // Récupération des sommets (nombres entiers) autour du point -> (xy1, xy2, xy3, xy4)
      x1 = (u32)(x), x2 = x1+1
      y1 = (u32)(y), y2 = y1+1
      xy1 = point(x1, y1), xy2 = point(x2, y1), xy3 = point(x1, y2), xy4 = point(x2, y2)
    // Récupération des vecteurs gradients associés -> (vg1, vg2, vg3, vg4)
      vg1 = VT[PT[(PT[xy1.x&1023]+xy1.y)&1023] / 4]    //"/ 4" car on passe de 0..1024 à 0..256 pour la table de vecteurs
      ...
    // Création des vecteurs entre les sommets et le point (x,y) -> (vs1, vs2, vs3, vs4)
      vs1 = vecteur(x - xy1.x, y - xy1.y)
      ...
    // Création des produits scalaires entre les vecteurs gradients et les vecteurs sommet-point (ps1, ps2, ps3, ps4)
      ps1 = vg1.x * vs1.x + vg1.y * vs1.y     // À y penser il faut vérifier les résultats avec des exemples vu que le vecteur gradient n'a pas la même échelle que les vecteurs sommets (vecteur d'un cercle de rayon 1, l'autre un rayon de racine de 2...), sinon faudra utiliser le cosinus mais bof
      ...
    // Interpolation faded du point (x,y) (sur une base 0..1)
      isx = ((6 * x - 15) * x + 10) * x * x * x
      isy = ((6 * y - 15) * y + 10) * y * y * y
    // Interpolation linéaire horizontale puis verticale des produits scalaires et du point faded
      ilh1 = ps1 + isx * (ps2 - ps1)
      ilh2 = ps3 + isx * (ps4 - ps3)
      noise = ilh1 + isy * (ilh2 - ilh1)

    //TODO : Faire des schémas parce que là c'est imbouffable, (tu seras content quand tu reviendras dessus dans deux ans hein... 👀)

    return noise;
  }
}
```
