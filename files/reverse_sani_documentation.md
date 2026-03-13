# Documentation de reverse engineering — format `.sani` / `.sani_data` / `.resource` / `.nmb`

## 0. Portée et niveau de confiance

Ce document synthétise **tout ce qui a été établi** à partir des fichiers fournis et comparés entre eux :

- `primaris.nmb`
- `cadian_trooper.nmb`
- `hormagaunt.nmb`
- plusieurs animations `*.sani`
- plusieurs descripteurs `*.sani.resource`
- plusieurs sidecars `*.sani_data`
- fichiers de template/rig du personnage (`cc_sm_constructor_tank.*`)

Le document distingue :

- **Confirmé** : observé directement dans les fichiers, recoupé plusieurs fois.
- **Très probable** : forte corrélation structurelle, mais sémantique exacte non fermée.
- **Hypothèse** : piste crédible, mais non démontrée définitivement.

Objectif :
- documenter la **grammaire réelle** des fichiers,
- isoler les **relations entre formats**,
- et préciser ce qui reste encore ouvert.

---

## 1. Vue d’ensemble des fichiers

### 1.1. Rôle global des extensions

#### `.sani`
**Confirmé**

Fichier d’animation principal.

Il contient :
- un header stable,
- une table de noms (bones/channels),
- des groupes d’indices vers ces noms,
- des chunks d’animation / auxiliaires (`F2`, `F4`, `F5`, `F6`, `F7`).

#### `.sani.resource`
**Confirmé**

Descripteur logique JSON de ressource animation.

Il relie :
- le nom logique moteur,
- le fichier `.sani`,
- éventuellement un **sidecar** `.sani_data`.

#### `.sani_data`
**Confirmé**

Sidecar binaire structuré.

Il n’est ni un duplicata trivial du `.sani`, ni un simple cache opaque.
Il réorganise et enrichit une partie tardive du `.sani`, avec des tables par groupe/composante.

#### `.nmb`
**Confirmé**

Conteneur/catalogue du personnage.

Il contient au minimum :
- le catalogue des animations,
- leur type logique (`sani` / `sani+`),
- la table de rig principal,
- les channels extra,
- des tags moteur,
- des structures de contraintes/limits.

#### `cc_sm_constructor_tank.*`
**Confirmé / utile pour le mapping**

Ces fichiers décrivent le template du personnage et aident surtout à :
- reconstruire la hiérarchie réelle du rig,
- relier les noms de channels/bones à un template concret,
- distinguer bones, locators, composants visuels et collision.

Ils aident surtout au **mapping rig/template**, moins au **codec interne** des animations.

---

## 2. Le format `.sani`

## 2.1. Header global

**Confirmé** sur les fichiers testés.

Structure observée :

```c
struct SaniHeader {
    char  magic[4];      // "ETNA"
    u16   version;       // 2
    u16   unk_06;        // 241 observé sur les échantillons testés
    u32   names_end;     // offset de fin de la table de noms
    f32   duration;
    f32   fps;           // souvent 30.0
    u32   sample_count;  // ex. 4, 16, 37
    u32   bone_count;    // ex. 282 pour primaris
    u32   extra_count;   // ex. 4
};
```

Exemples confirmés :
- facial simple : `sample_count = 4`
- blink facial : `sample_count = 16`
- walk backward : `sample_count = 37`

### 2.2. Table des noms

**Confirmé**

À partir de `0x20`, le fichier contient une table de chaînes **length-prefixed**.

Pour le rig Primaris :
- `bone_count = 282`
- `extra_count = 4`
- total attendu = **286 noms**

Les 4 channels extra observés sont :
- `length`
- `footPos_L`
- `footPos_R`
- `lookAt`

Les noms de rig observés incluent notamment :
- `CharacterWorldSpaceTM`
- `CENTRE`
- `BACKdown`
- `BACKmiddle`
- `BACKup`
- `NECK`
- `HEAD`
- `helmet`
- `cam_root`
- `cam_anim`
- `surface`
- `foot_targ_L`
- `foot_targ_R`

---

## 3. Chunks principaux du `.sani`

### 3.1. Structure générique d’un chunk principal

**Confirmé** pour les chunks `F2`, `F6`, `F7`.

Structure observée :

```c
struct SaniChunk {
    u16 tag;          // ex. F2, F6, F7
    u32 next_abs;     // offset absolu du chunk suivant
    u16 unk_06;       // souvent 2 observé sur F2/F6/F7
    u32 counts[6];    // 6 groupes
    u16 group_indices[sum(counts)];
    u8  payload[];
};
```

### 3.2. Groupes

**Confirmé**

Les groupes stockent des **indices u16** dans la table des noms.

Les counts observés sont toujours répartis en **6 groupes**.

Exemple d’interprétation sûre :
- un chunk ne référence pas des channels “anonymes” ;
- il référence explicitement des noms du rig/channels.

### 3.3. Régularités sur les `walk`

**Confirmé**

Pour les `walk`, on observe des régularités de partitions du type :
- `g0 + g3 = 282` ou voisinage très proche selon le chunk/variante
- `g1 + g4 = 282`
- `g5 = 4` tracks spéciales dans plusieurs cas

Cela montre que les groupes sont **structurels**, pas arbitraires.

---

## 4. Chunks auxiliaires du `.sani`

## 4.1. Chunk `F4`

**Confirmé**

Structure observée :

```c
u16 unk = 2;
float hdr[12];
int16 track1[frames][3];
int16 track2[frames][3];
```

Comportement observé :
- sur les `walk` : `track1` actif, `track2` nul
- sur les facials simples : quasi tout nul

Interprétation :
- **pas** un bloc de pose osseuse classique,
- très probablement une **courbe auxiliaire vec3**,
- possiblement liée à la locomotion / phase / foot lock / surface.

## 4.2. Chunk `F5`

**Confirmé**

Structure observée :

```c
u16 unk = 1;
u16 zero = 0;
u32 name_len;
char name[];         // ex. "Footsteps"
u8  guid[16];
u8  pad[5];         // zéros observés
u32 count;
struct {
    float a;
    float b;
    u32 x;
    u32 y;
} rec[count];
u64 tail_zero;
```

Nom confirmé :
- `Footsteps`

Exemples observés :
- facial simple : `count = 1`
- blink facial : `count = 2`
- walk : `count = 2`

Interprétation :
- bloc d’**événements / markers**,
- pas un flux de transformations osseuses.

---

## 5. Cas des facials simples

Fichiers typiques :
- `tank_face_aim_intercessor.sani`
- `tank_face_default_intercessor.sani`
- `tank_face_angry_intercessor.sani`
- `tank_face_devil_intercessor.sani`

### 5.1. Structure générale

**Confirmé**

Ils ont typiquement :
- un chunk principal `F2`
- puis `F4`
- puis `F5`

### 5.2. Codec scalaire 12-bit

**Confirmé**

Le payload principal des facials simples se décode proprement comme un flux de **valeurs 12-bit little-endian**.

Packing confirmé :

```python
a = b0 | ((b1 & 0x0F) << 8)
b = (b1 >> 4) | (b2 << 4)
```

Donc 3 octets contiennent 2 valeurs de 12 bits.

Les `samples_raw` déduits ainsi recollent avec les sorties intermédiaires observées côté script/JSON.

### 5.3. Conséquence

Pour ces facials simples :
- le `.sani` seul semble suffire,
- il n’y a pas de `animData` déclaré dans les `.resource` fournis,
- le cas est nettement plus simple que `blink`.

---

## 6. Cas spécial : `tank_face_blink_intercessor`

## 6.1. Pourquoi `blink` est spécial

**Confirmé**

Le fichier :
- est classé `sani+` dans `primaris.nmb`
- a un `.resource` qui référence un `animData`
- possède un `.sani_data`

Alors que les facials simples (`aim`, `default`, `angry`, `devil`) n’ont pas de sidecar déclaré.

Donc `blink` est bien un **cas spécial**.

## 6.2. Structure du `.sani`

**Confirmé**

`blink.sani` utilise :
- `F6`
- `F7`
- `F4`
- `F5`

### 6.3. Structure du `.sani_data`

**Confirmé**

`blink.sani_data` contient 2 sections :
- `F2`
- `F3`

Header de section confirmé :

```c
u16 tag;      // F2 ou F3
u32 end_abs;
u16 count;    // 586 observé
```

### 6.4. Grammaire exacte des sections

**Confirmé**

#### Section `F2`
Body de `8670` octets :

```text
4
+ 25 * 18
+ 38 * 52
+ 65 * 18
+ 12
+ 281 * 18
```

#### Section `F3`
Body de `8094` octets :

```text
4
+ 25 * 18
+ 38 * 52
+ 59 * 18
+ 12
+ 255 * 18
```

### 6.5. Corrélation avec les groupes

**Très probable / fortement corrélé**

Pour `blink.sani` :
- `g0 = 38`
- `g1 = 65` sur `F6`
- `g1 = 59` sur `F7`

Corrélations exactes observées :
- `38 * 52` ↔ **g0**
- `65 * 18` ↔ **g1(F6)**
- `59 * 18` ↔ **g1(F7)**

### 6.6. Partie commune `F2` / `F3`

**Confirmé**

Le préfixe commun entre `F2` et `F3` fait exactement :

```text
4 + 25*18 + 38*52 = 2430
```

Donc les sections partagent une base commune structurée.

### 6.7. Lien avec le `.sani`

**Confirmé**

`blink.sani_data:F2` est structurellement lié à `blink.sani:F6`,
`blink.sani_data:F3` est structurellement lié à `blink.sani:F7`.

Le sidecar **réorganise et enrichit** une partie tardive du `.sani`, il ne la duplique pas naïvement.

### 6.8. Ce qui reste ouvert sur `blink`

- sens exact des records `18` octets
- sens exact des records `52` octets
- rôle précis du dernier gros bloc `281*18` / `255*18`

---

## 7. Cas des `walk backward` (`bwd = backward`)

Fichiers concernés :
- `tacticus_wpn0_walk_bwd_intercessor.sani`
- `tacticus_wpn1_walk_bwd_intercessor.sani`
- `tacticus_wpn2_walk_bwd_intercessor.sani`
- `tacticus_unarmed_walk_bwd_intercessor.sani`

## 7.1. Interprétation générale

**Très probable**

Ces fichiers représentent la **même locomotion de base** (marche arrière), déclinée selon l’état d’armement / la posture de tenue d’arme.

Indices forts :
- noms de fichiers
- différences de channels liés aux mains/arme
- grosse base commune entre variantes

Exemples de channels impliqués dans les différences :
- `Right_hand_hold`
- `Left_hand_hold`
- `Gun_pose`
- `GUN_ANIM`
- `melee_weapon_pivot`

## 7.2. Les sidecars des walk sont déclarés explicitement

**Confirmé**

Les `.resource` des 4 `walk` référencent tous un `.sani_data`.

Donc pour les `walk`, le sidecar n’est pas accessoire ; il fait partie du système.

## 7.3. F2/F3 côté sidecar

**Confirmé**

- `wpn0`, `wpn1`, `unarmed` : `.sani_data` avec **une seule section `F2`**
- `wpn2` : `.sani_data` avec **`F2 + F3`**

Corrélation confirmée :
- `F2` du sidecar ↔ `F6` du `.sani`
- `F3` du sidecar ↔ `F7` du `.sani`

## 7.4. Valeur `u16` globale de section

**Confirmé**

Le `count` du header de section `.sani_data` n’est pas un “nombre d’entrées”.
C’est la **valeur `u16` placée juste avant le suffixe externalisé** dans le `.sani`.

Exemples :
- `wpn0/wpn1/wpn2:F2` → `0x0C00 = 3072`
- `wpn2:F3` → `0x0FFF = 4095`
- `unarmed:F2` → `0x0E7D = 3709`
- `blink:F2/F3` → `0x024A = 586`

Donc le schéma réel est :

```text
.sani      = [...][u16 global][suffixe]
.sani_data = [tag,end,u16 global][suffixe réorganisé + tables]
```

## 7.5. Bloc tardif du `.sani` des walk

**Confirmé**

Dans les `walk`, le bloc tardif du `.sani` est bien :

```text
copyA + g1a
```

Et le sidecar fait :

```text
copyA + g0 + copyB + g1b
```

avec :
- `copyA + copyB = suffixe exact du .sani après le u16 global`
- `g1a = copyB`
- `g0` et `g1b` = tables ajoutées par le sidecar

Exemples de contiguïté confirmée :
- `wpn0 F6` : `copyA` finit à `7242`, `g1a` commence à `7242`
- `wpn1 F6` : `6110 + 1042 = 7152`
- `wpn2 F6` : `6164 + 1114 = 7278`
- `wpn2 F7` : `6182 + 1150 = 7332`
- `unarmed F6` : `6164 + 1318 = 7482`

Donc `g1a` est bien **déjà présent dans le `.sani`**, tandis que `g0` et `g1b` sont **purement sidecar**.

## 7.6. Grammaire exacte des sidecars `walk`

### `wpn0` (`F2`)

```text
16870
= (9*120 - 2)
+ 30*112
+ 56*54
+ 56*168
```

### `wpn1` (`F2`)

```text
16278
= (9*116 - 2)
+ 29*112
+ 54*54
+ 54*168
```

### `wpn2` (`F2`)

```text
16796
= (9*124 - 2)
+ 31*112
+ 55*54
+ 55*168
```

### `wpn2` (`F3`)

```text
16944
= (9*128 - 2)
+ 32*112
+ 55*54
+ 55*168
```

### `unarmed` (`F2`)

```text
16870
= (11*120 - 2)
+ 30*104
+ 56*66
+ 56*156
```

### 7.7. Interprétation de cette grammaire

**Confirmé / très probable**

Pour `wpn0/wpn1/wpn2` :
- `g0 = count(g0) * 112`
- `g1a = count(g1) * 54`
- `g1b = count(g1) * 168`

Pour `unarmed` :
- `g0 = count(g0) * 104`
- `g1a = count(g1) * 66`
- `g1b = count(g1) * 156`

Donc les tables sont **par channel**, alignées sur les groupes.

## 7.8. `g0`, `g1a`, `g1b` : arité des composantes

### `g0`

**Confirmé**

- armé : `112 = 2 * 56`
- unarmed : `104 = 2 * 52`

Donc `g0` = **2 composantes par channel**.

### `g1a`

**Confirmé**

- armé : `54 = 3 * 18`
- unarmed : `66 = 3 * 22`

Donc `g1a` = **3 composantes compactes par channel**.

### `g1b`

**Confirmé**

- armé : `168 = 3 * 56`
- unarmed : `156 = 3 * 52`

Donc `g1b` = **3 composantes étendues par channel**.

## 7.9. Nature probable des sous-blocs `56/52`

**Très probable**

Les sous-blocs de taille `56/52` se lisent proprement comme des **tableaux de paires `(u16,u16)`** :
- `56` = `28 u16` = **14 paires**
- `52` = `26 u16` = **13 paires**

Les longues fins du type :
- `(0,12064) × 8`
- `(0,13376) × 6`
- `(0,5312) × 10`

ressemblent à du **padding structuré de slots inutilisés**, pas à des keyframes.

Conséquence :
- les sous-blocs `56/52` stockent probablement **jusqu’à 14/13 paires utiles**,
- puis complètent avec du padding structuré.

## 7.10. Comportement de `g0`

**Confirmé / très probable**

`g0` est **plus sparse** que `g1b`.

- Il a souvent de longues queues `(0, param)` répétées.
- Une des 2 composantes peut être fortement occupée, l’autre très peu.
- La composante “active” dépend du channel.

Exemples `wpn0` :
- `CharacterWorldSpaceTM` → sous-unité 2 avec `8` paires de padding
- `CENTRE` → sous-unité 2 avec `6`
- `BACKdown` → sous-unité 2 avec `4`
- `shoulder_L` → sous-unité 2 avec `2`

Puis plus loin, inversion de la composante occupée :
- `hose1_l_bind` → sous-unité 1 avec `10`
- `hose2_l_bind` → sous-unité 1 avec `8`
- `hose3_l_bind` → sous-unité 1 avec `6`

Interprétation :
- `g0` code **2 familles de paramètres distinctes**,
- pas deux copies d’un même champ.

## 7.11. Comportement de `g1a`

**Confirmé / très probable**

`g1a` est une **forme compacte** du groupe `g1`.

- armé : `18 octets = 9 u16`
- unarmed : `22 octets = 11 u16`

Les valeurs observées contiennent :
- beaucoup de `0`
- beaucoup de `65535`
- des grands `u16`
- un dernier `u16` qui ne ressemble pas à un simple count

Conclusion :
- `g1a` **n’est probablement pas** une simple table de compteurs,
- mais plutôt un **mini-descripteur compact avec sentinelles**.

## 7.12. Comportement de `g1b`

**Confirmé / très probable**

`g1b` est **plus dense** que `g0`.

Il montre une asymétrie structurelle forte entre ses 3 composantes :

### Variante armée (`wpn0` typique)
- composante 1 finit souvent par **`(0, x)`**
- composante 2 finit souvent par **`(x, 0)`**
- composante 3 est généralement **pleinement renseignée**

Exemples `wpn0` :
- `foot_targ_L`
  - comp1 → `(0, 54867)`
  - comp2 → `(20988, 0)`
  - comp3 → `(44750, 53472)`
- `CENTRE`
  - comp1 → `(0, 55499)`
  - comp2 → `(16337, 0)`
  - comp3 → `(51490, 45370)`

Positions souvent nulles par composante (exemple `wpn0`) :
- comp1 : positions `25/26` souvent nulles
- comp2 : positions `3/4` toujours nulles, `21/22/27` souvent nulles
- comp3 : position `0` souvent nulle, `23/24` souvent nulles

Interprétation :
- `g1b` a un **layout de champs fixe**,
- certains slots sont structurellement vides selon la composante,
- les 3 composantes n’ont clairement pas le même rôle.

## 7.13. Ce que l’on peut affirmer pour les `walk`

**Confirmé / très probable**

Pour les `walk` :
- `.sani` = payload runtime minimal
- `.sani_data` = enrichissement structuré
- `g1a` est déjà dans le `.sani`
- `g0` et `g1b` sont ajoutés par le sidecar
- `g0` = 2 composantes, sparse/paramétriques
- `g1` = 3 composantes, avec version compacte (`g1a`) et étendue (`g1b`)

---

## 8. Le fichier `.resource`

**Confirmé**

Format JSON lisible, avec au minimum :
- type logique de ressource
- chemin logique/naming moteur
- lien vers le `.sani`
- parfois lien vers `animData`

Observation structurante :
- les facials simples n’ont pas de `animData`
- `blink` en a un
- les 4 `walk` en ont tous un

Conséquence :
- le sidecar n’est pas universel,
- mais il est explicitement déclaré quand requis.

---

## 9. Le fichier `.nmb`

## 9.1. Rôle global

**Confirmé**

Pour `primaris.nmb`, le fichier contient au moins :
- une table d’animations,
- une table de types d’animation,
- le rig principal,
- les channels extra,
- des tags moteur,
- des structures de type “limit/constraint”.

## 9.2. Catalogue d’animations

**Confirmé**

`primaris.nmb` contient :
- **1747 noms d’animations**
- **1747 types** associés

Types observés :
- majoritairement `sani`
- une partie `sani+`

Les fichiers fournis sont bien retrouvés dans ce catalogue.

## 9.3. Type `sani+`

**Très probable**

`sani+` semble désigner une **catégorie additive / overlay / correctif**,
mais **pas** “présence de `F7` = automatiquement `sani+`”.

Indices :
- `blink` = `sani+`
- beaucoup d’animations `sani+` ont des noms évocateurs (`blink`, `*_add_*`, `aim_idle`, `talk_add`, etc.)
- `wpn2_walk_bwd` a `F6 + F7` mais reste classé `sani`

## 9.4. Rig principal et channels extra

**Confirmé**

`primaris.nmb` contient :
- une table du rig principal de **282 noms**
- une table des 4 extra :
  - `length`
  - `footPos_L`
  - `footPos_R`
  - `lookAt`

## 9.5. Tags moteur

**Confirmé**

Le fichier contient une table de tags/événements avec notamment :
- `Footsteps`
- `Sound`
- `Camera`
- `terrain`
- `terrain_l`
- `terrain_r`
- `lock_foot_l`
- `lock_foot_r`
- `Phase_l`
- `Phase_R`
- `footLock`
- `event_track`

Conséquence :
- `F5` s’insère bien dans un système d’événements moteur réel,
- `F4` est vraisemblablement lié à locomotion/phase/footing,
- ce n’est pas une reconstruction arbitraire.

## 9.6. Limits / contraintes

**Confirmé**

`primaris.nmb` contient des records réguliers liés à des `*Limit` :
- `BACKdownLimit`
- `BACKmiddleLimit`
- `NECKLimit`
- `HEADLimit`
- `LEG1LLimit`
- `LEG2RLimit`
- `FOOT1RLimit`
- `hose1_l_bindLimit`
- etc.

Le `nmb` ne catalogue donc pas seulement les anims ; il embarque aussi des métadonnées de contrainte / limitation.

---

## 10. Fichiers de template / rig (`constructor_tank`)

## 10.1. `geom_dbg`

**Confirmé**

Expose une hiérarchie réelle du personnage du type :
- `ROOT|CENTRE|BACKdown|BACKmiddle|BACKup|NECK|HEAD|...`

Il confirme :
- la hiérarchie du rig,
- la présence de nombreux bones faciaux sous `HEAD`,
- la présence de bones auxiliaires / jiggle / décoratifs.

Très utile pour :
- rattacher les noms vus dans `.sani` / `.nmb` à un vrai rig.

## 10.2. `tpl`

**Confirmé**

Fichier template de ressource (`1SERtpl` observé), contenant beaucoup de chaînes liées aux composants du personnage.

Utile pour :
- comprendre le template du personnage,
- les composants visuels,
- les éléments attachés.

## 10.3. `tpl_markup`

**Confirmé**

Expose des liaisons template/objets/locators, ex. :
- `surface`
- `helmet`
- `HEAD`
- `vfx_iron_halo`

Aide au mapping de certains noms vus dans les animations.

## 10.4. `lods_base`

**Confirmé**

Expose des proxies/collision/LOD du type :
- `HEAD_cdt`
- `ARM2L_cdt`
- `CENTRE_cdt`
- etc.

Utile pour distinguer :
- bones,
- objets,
- collision/visibilité.

## 10.5. `cdt`

Peu utile pour le codec d’animation à ce stade, mais cohérent avec les structures de collision/proxy.

---

## 11. Ce qu’on peut affirmer avec un bon niveau de confiance

### 11.1. Sur le système global

**Confirmé / très probable**

Le système complet ressemble à :

```text
nmb        = catalogue + rig + channels extra + tags/limits
resource   = liaison logique
sani       = animation runtime principale
sani_data  = sidecar structuré de reconstruction / enrichissement
```

### 11.2. Sur les facials simples

**Confirmé**

Ils utilisent un flux de samples 12-bit dans le `.sani` principal,
sans `animData` requis dans les cas observés.

### 11.3. Sur `blink`

**Confirmé / très probable**

`blink` est un cas enrichi :
- `sani+`
- `.sani.resource` avec `animData`
- `.sani_data` en deux sections structurées `F2/F3`

### 11.4. Sur les `walk`

**Confirmé / très probable**

Les `walk` sont des variantes d’une même locomotion (`bwd = backward`) selon l’état d’armement.
Leur `animData` est structurellement important.

---

## 12. Ce qui reste ouvert

## 12.1. Sémantique exacte des tables de descripteurs

Toujours ouverte :
- sens précis des paires utiles dans `56/52`
- rôle exact des composantes `g0`
- rôle exact des 3 composantes de `g1`
- nature du dernier `u16` dans les blocs compacts `18/22`

## 12.2. Sur `g0`

On sait qu’il est :
- plus sparse,
- structuré en 2 composantes,
- rempli partiellement avec padding structuré.

Mais on ne peut **pas encore** dire honnêtement :
- composante 1 = translation,
- composante 2 = rotation,
- etc.

## 12.3. Sur `g1`

On sait qu’il est :
- structuré en 3 composantes,
- avec forme compacte et forme étendue,
- asymétrique entre les 3 composantes.

Mais la sémantique exacte n’est pas fermée.

## 12.4. Sur `sani+`

Très probable :
- catégorie additive/overlay/correctif.

Mais il manque encore assez d’exemples complets pour figer une définition exacte.

---

## 13. Ce que le reverse permet déjà de faire

### 13.1. Ce qui est réaliste dès maintenant

- parser proprement les headers `.sani`
- extraire la table des noms
- parser les groupes et leurs indices
- identifier les chunks présents
- lire `F4` et `F5`
- décoder les facials simples en 12-bit
- parser `.resource`
- parser la structure générale des `.sani_data`
- relier un sidecar à son `.sani` correspondant
- documenter la logique `walk` par groupe/composante

### 13.2. Ce qui reste partiellement bloqué

- reconstruction complète fidèle de certains canaux complexes
- interprétation fine des tables `g0/g1`
- reproduction exacte “comme en jeu” sans ambiguïté

---

## 14. État d’avancement estimé

Estimation honnête :

- **70–80 %** sur la **structure globale du format**
- **40–50 %** sur la **sémantique interne réelle**
- donc environ **60 % d’avancement global utile**

Avec nuance :
- pour documenter le format et faire un parseur sérieux : plutôt **65–70 %**
- pour refaire un importeur/playback exact “comme en jeu” : plutôt **50–55 %**

---

## 15. Fichiers qui apporteraient le meilleur gain supplémentaire

Sans être strictement nécessaires pour continuer, les plus rentables seraient :

1. un autre couple complet `sani + resource + sani_data` d’une **autre famille d’animation** sur le même rig Primaris :
   - idle
   - run
   - turn
   - strafe
   - aim idle
   - reload
   - hit

2. d’autres exemples complets de **`sani+`**

3. éventuellement le modèle/skinning final si l’objectif devient le playback Blender fidèle

---

## 16. Résumé final condensé

Ce qui est aujourd’hui **solidement établi** :

- `.sani` = animation runtime principale
- `.resource` = descripteur logique qui déclare éventuellement un `animData`
- `.sani_data` = sidecar structuré, pas accessoire
- `.nmb` = catalogue d’animations + rig + channels extra + tags moteur + limits
- facials simples = flux 12-bit dans le `.sani`
- `blink` = cas enrichi avec `sani+` + `animData`
- `walk` = même locomotion backward, variantes selon l’armement
- pour les `walk` :
  - le `.sani` embarque un suffixe tardif `copyA + g1a`
  - le sidecar ajoute `g0 + g1b`
  - `g0` = 2 composantes
  - `g1` = 3 composantes
  - les gros sous-blocs `56/52` sont très probablement des tables de paires avec padding structuré

Le **vrai verrou restant** n’est plus la structure générale.
C’est la **signification exacte des champs utiles** dans les descripteurs `g0/g1`.
