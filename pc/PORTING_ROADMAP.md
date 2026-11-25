# Pokémon Cristal / Crystal — port PC natif par recompilation statique

Journal du port (en français). Le dépôt est le désassemblage pret
`pokecrystal` (assembleur SM83 + rgbds) ; **rien n'est modifié hors de `pc/`**
(sauf deux lignes de `.gitignore` pour `build/` et `dist/`) : la ROM se
reconstruit toujours à l'identique (`make compare` : `pokecrystal.gbc: OK`).

## Principe

Il n'y a pas de C à compiler : le code du jeu est de l'assembleur Game Boy.
Méthode : **recompilation statique** de la ROM en C + un **runtime Game Boy
Color** écrit à la main (pas d'émulateur tiers).

- `pc/tools/recomp/recomp.py` lit la ROM (octet pour octet celle du joueur),
  le `.sym` (adresse de chaque étiquette) et les sources asm (quelles
  étiquettes sont du code : l'instruction ou la macro qui suit ; les macros
  de code et les macros « transparentes » comme `vc_hook` sont détectées
  automatiquement). Racines supplémentaires : vecteurs `rst`/interruptions,
  motifs `farcall`/`homecall`, code caché dans les textes (`text_asm`),
  blocs `LOAD` (code copié en RAM, ex. la routine DMA en HRAM), adresses
  relevées par le runtime (`--extra`).
- Sortie : **une fonction C par banque de ROM** (`rc_<langue>_bank_NN(pc)`),
  registres dans des variables locales, chaque début de bloc est un `case`,
  les branches internes des `goto`. Les appels empilent la **vraie adresse de
  retour** sur la pile émulée : les astuces de pile du jeu (`push de / ret`,
  retours dépilés, `jp hl`, tables `JumpTable`) marchent comme sur la console.
- Le temps : chaque instruction ajoute ses cycles ; à chaque branche prise on
  compare à la prochaine échéance matérielle (changement de mode de l'écran,
  minuteur, liaison série) et on rend la main à la boucle d'aiguillage, qui
  met le matériel à jour et sert les interruptions.
- **Interpréteur de secours** généré à partir de la même sémantique
  (`interp_gen.c`) : adresses non vues par le traducteur (journalisées dans
  `rc_missing.txt`, réinjectables avec `--extra`) et ROM inconnues.
- Runtime (`pc/runtime/`) : mémoire en pages de 4 Kio, MBC3 + horloge (RTC,
  heure du PC), écran CGB/DMG ligne par ligne (effets de trame), HDMA/DMA,
  minuteur, son 4 canaux vers SDL, clavier + manette (SDL GameController),
  sauvegarde `.sav` au format des émulateurs (32 Kio + bloc RTC de 48 octets).
- **Un seul exécutable pour l'anglais et le français** : chaque ROM traduite
  a son préfixe (`--prefix en|fr`) ; au lancement, le CRC de la ROM choisit
  la traduction. Une ROM inconnue tourne entièrement dans l'interpréteur.

## Jalons

| Jalon | Contenu | État |
|---|---|---|
| C0 Outils | rgbds 1.0.4, ROM US reconstruite identique (sha1 OK) | ✅ |
| C1 Traducteur | ROM + sym + sources → C, tout le jeu | ✅ US 148 849 instructions, FR 148 281 |
| C2 Compilation | Clang + CMake + Ninja + SDL2 (statique) | ✅ Windows, ✅ Linux (WSL, compile et tourne) |
| C3 Démarrage | copyright, intro, écran titre | ✅ US et FR |
| C4 Jeu | nouvelle partie, chambre, maman, labo d'Orme, starter, Route 29, combat sauvage gagné, sauvegarde, CONTINUE au même endroit | ✅ (US ; FR jusqu'à la chambre) |
| C5 Son, manette | APU, manette SDL | ✅ validés par l'utilisateur (écoute + manette réelle) |
| C6 Français | traduction directe de la ROM française (voir décision) | ✅ démarre ; parcours complet à faire |
| C7 Finitions | paquet Windows (`pc/tools/package.py`), config pour le lanceur | ✅ paquet ; Linux en cours |
| C8 Or/Argent, Rouge/Bleu, Jaune | même outil, `pc/` copié dans chaque dépôt, un exécutable par version | ✅ démarrage + nouvelle partie pour les 5 (ROM US) ; `GB_DIFF` 0 écart ; jeu complet à tester |

## Décisions

- **ROM anglaise** : la ROM US du joueur (`Pokemon - Crystal Version (USA)`)
  a le même sha1 que `pokecrystal.gbc` (v1.0) : code traduit et données lues
  à l'exécution viennent de cette ROM. L'exécutable ne contient aucune donnée
  du jeu (vérifié par `package.py`).
- **Version française : traduite directement depuis la ROM française** (pas
  de désassemblage FR). Le traducteur part des octets ; il lui faut les points
  d'entrée. `match_roots.py` les retrouve par correspondance d'octets avec la
  ROM US (opérandes d'adresse masqués), puis par prédiction du décalage local
  (le code se décale par blocs) : **16 660 fonctions sur 16 933** retrouvées ;
  le reste est couvert par la descente récursive et les adresses relevées à
  l'exécution (`pc/roots_crystal_fr_runtime.txt`). Résultat : démarrage FR
  sans quasi aucune instruction interprétée.

## Bugs trouvés (classe → contrôle)

- `pop af` restaurait des drapeaux aléatoires : la macro `SET_F` déclarait une
  variable `_f` qui masquait celle de `POP_AF` (`uint8_t _f = (_f)`). Symptôme :
  escalier sans effet, crash de la décompression en mode interprété.
  Contrôle : `GB_DIFF=1` (ci-dessous) et mode `GB_INTERP=1` qui sert les
  interruptions à chaque instruction (comme la console) et révèle ces bugs.
- Étiquettes de code prises pour des données (`depixel`, `vc_hook`, code
  après `text_asm`) → détection automatique des macros.

## Outils de test et de débogage

- `pc/tools/play.py` : parcours scriptés (`wait`, `press`, `hold`, `mash`,
  `shot`, `save`), captures assemblées en planche PNG.
- États : `GB_STATE_SAVE=frame:fichier`, `GB_STATE_LOAD=fichier`,
  `GB_STATE_WHEN=ADDR:VAL:fichier` (ex. `D22D:1:combat.bin` = début de combat).
- `pc/tools/peek.py état.bin pokecrystal.sym wMapGroup wXCoord ...`.
- **`GB_DIFF=1`** : chaque bloc traduit est rejoué par l'interpréteur depuis le
  même état et les deux états comparés (243 909 blocs, 0 écart sur l'intro).
- `GB_INTERP=1`, `GB_ITRACE=de:à` (trace instruction par instruction),
  `GB_WATCHPC=hex`, `GB_PCLOG=1`, `GB_WAV=fichier.wav` (son enregistré).

## Commandes

```
# outils : rgbds 1.0.4 (libre) dans D:/LUC/pokemonall/tools/rgbds-1.0.4
PATH=/d/LUC/pokemonall/tools/rgbds-1.0.4:$PATH mingw32-make -j16 crystal
python pc/tools/recomp/recomp.py --rom pokecrystal.gbc --sym pokecrystal.sym --src . \
       --out build/rc_en --name crystal --prefix en --lang en
python pc/tools/recomp/match_roots.py --rom pokecrystal.gbc --sym pokecrystal.sym --src . \
       --target "<ROM FR>.gbc" --out pc/roots_crystal_fr.txt
python pc/tools/recomp/recomp.py --rom "<ROM FR>.gbc" --extra pc/roots_crystal_fr.txt \
       --extra pc/roots_crystal_fr_runtime.txt --out build/rc_fr --name cristal --prefix fr --lang fr
cmake -S pc -B build/pc -G Ninja -DCMAKE_C_COMPILER=clang "-DRC_DIRS=build/rc_en;build/rc_fr" \
      -DSDL2_PREBUILT_DIR=<SDL2 statique déjà compilé>   # ou FETCHCONTENT_SOURCE_DIR_SDL2
cmake --build build/pc
build/pc/pokecrystal.exe [--lang fr|en] [--fullscreen] [--scale N] ["<ROM>.gbc"]
python pc/tools/package.py --exe build/pc/pokecrystal.exe --out dist --rom <ROM>...
```

Configuration pour le lanceur : `pokecrystal.ini` à côté de l'exécutable
(`lang=`, `rom_en=`, `rom_fr=`, `fullscreen=`, `scale=`, `save=`) ; la ligne de
commande l'emporte. Sauvegardes : `pokemon_crystal.sav` / `pokemon_cristal.sav`.

Clavier : flèches, X/Espace = A, Z/Ctrl = B, Entrée = Start, Retour arrière =
Select, Tab = avance rapide, F11 = plein écran. Manette : automatique.

## Journal

- 2026-11-25 : rgbds 1.0.4 ; AVG bloquait les outils compilés de la décomp
  (exception ajoutée par l'utilisateur sur `decomp/`). ROM US reconstruite
  identique. Traducteur + runtime écrits ; intro → titre → nouvelle partie →
  monde → combat → sauvegarde/rechargement en anglais ; version française
  traduite depuis la ROM FR et démarrée ; exécutable unique EN/FR ; paquet.
- 2026-11-25 (suite) : Linux (WSL) compile et donne des images identiques pixel
  pour pixel à Windows. `pokecrystal.ini` : commentaires en fin de ligne
  (`lang=en #en ou fr`), espaces et guillemets acceptés (bug signalé par
  l'utilisateur). Son et manette validés par l'utilisateur.
