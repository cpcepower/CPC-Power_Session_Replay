# CPSR - CPC Power Session Replay - spécification v0.3 (brouillon)

Format de session (enregistrement + relecture) pour émulateurs Amstrad CPC, synchronisé sur la VBL de la machine émulée (464/664/6128/Plus/GX4000).

Ce document est la spécification de référence du format CPSR v0.3 (brouillon, 26/11/2026 ; v0.2 initial : 24/11/2026), conçu pour remplacer à terme le format SNR de WinAPE, dont les sessions décrochent pendant les chargements sur les autres émulateurs.

- Statut : brouillon v0.3.
- Retours de terrain (26/11/2026) : le diagnostic D-1 à D-4 est confronté à un cas réel exigeant, instrumenté trame par trame dans CPCEPower (section 1 bis).
- Décisions actées le 24/11/2026 : D1 (identité : CPSR, extension `.cpsr`, magic ASCII `CPSR`) ; D2 (l'image disque n'est pas liée au format : référence par nom dans `MDIA`, embarquement optionnel dans `EMDA`/`EMDB`) ; zéro dépendance (zlib, SHA-1 et MD5 retirés du v1, CRC32 par table intégrée).
- Décisions ouvertes : D3 (granularité horloge v1), D4 (politique de resynchronisation par défaut), D5 (périmètre v1 : disque et K7 en v2), D6 (heuristique des moments sûrs des points de contrôle).
- Auteur : Megachur. Licence : GPL-3.0 (cf. LICENSE). (c) 2026.
- Outils : convertisseur SNR -> CPSR : tools/cpsr-convert.html.

Projet : CPCEPower + compatibilité inter-émulateurs Amstrad CPC (464/664/6128/Plus/GX4000).
Objectif : remplacer à terme le format SNR de WinAPE par un format de session (enregistrement + relecture) robuste à la dérive de synchronisation, adoptable par tous les émulateurs CPC.

## 1. Diagnostic - pourquoi les SNR décrochent

Le format SNR (WinAPE) = un snapshot SNA v3 (chunks MEMx/ROMS/DSCA/DSCB/CPC+) + un chunk `SNR ` contenant 0x0A octets d'état initial du clavier, 72 octets (souris/RTC), puis un flux d'événements datés **en nombre de trames de l'émulateur** (codage observé dans `SNA.cpp` : delta de trames en 1 octet, ou 0x00 + u16 si > 254, suivi des frappes). Sources de désynchronisation :

| # | Source | Effet |
|---|--------|-------|
| D-1 | **Ancrage temporel = la trame de l'émulateur hôte**, pas un signal de la machine émulée | un événement appliqué en fin de trame hôte peut tomber 1 VBL avant/après la trame "vue" par le CRTC ; différent d'un émulateur à l'autre |
| D-2 | **Non-déterminisme inter-émulateurs pendant les chargements** | tempo FDC, latence d'interruption, type CRTC, loaders sur mesure : le moment où le programme interroge le clavier glisse de plusieurs trames entre WinAPE et les autres (CPCEPower inclus) ; la frappe datée trame N tombe hors de la fenêtre où le jeu écoute |
| D-3 | **Aucune reprise sur dérive** | pas de point de contrôle, pas d'empreinte d'état, pas de resynchronisation ; WinAPE affiche un indicateur vert/rouge mais ne corrige jamais |
| D-4 | **État FDC sous-défini dans SNA v3** | le snapshot ne fige pas tout l'état interne de l'upd765 (phase de commande, FIFO, PCN) : la reprise d'un chargement en cours est approximative par construction |

Ce diagnostic est cohérent avec l'expérience du corpus CPC-Power (444 sessions) : impossible pour les jeux multi-faces, obligation de démarrer l'enregistrement après les chargements, patchs manuels de SNR. Sources : aide WinAPE Recording Sessions (winape.net/help/sessions.html), article sessions CPC-Power (cpc-power.com/cpcarchives, page=articles&num=10), code CPCEPower `SNA.cpp`/`SNA.h`.

**Conséquence de conception fondamentale** : la dérive inter-implémentations est inévitable (D-2, D-4). Un format fiable ne doit pas la nier (défaut du SNR), il doit l'assumer : ancrage sur la machine émulée + points de contrôle + resynchronisation + (option) attaches conditionnelles d'événements.

## 1 bis. Validation du diagnostic par mesures terrain (novembre 2026)

Un cas réel particulièrement exigeant (jeu à fenêtres d'interruptions critiques : une action de gameplay n'est possible que si la phase des événements dans la trame est exacte) a été instrumenté trame par trame dans CPCEPower : trace CSV de l'état GA/CRTC/mémoire à chaque front d'interruption, balayage exhaustif des phases d'ancrage (52 positions, 0 à 51 lignes) et session de reprise contrefactuelle depuis un snapshot intermédiaire. Résultats :

- **D-1 confirmé quantitativement** : deux moteurs temporellement identiques à l'échelle trame (même compte de cycles par trame, dérive bornée à ±1,5 microseconde sur tous les blocs de synchronisation d'une session complète) divergent en sous-trame : au point critique, la valeur de gameplay évolue à 3 unités du seuil d'échec. Aucun des 52 décalages d'ancrage testés ne restaure le comportement de référence : un réglage de phase par émulateur est une impasse ; seul l'ancrage matériel (livraison juste après acquittement de l'interruption) est reproductible.
- **D-3 confirmé** : la reprise depuis un snapshot intermédiaire ne reproduit pas la session d'origine (divergence mesurée avant la 240e interruption) : le SNA v3 ne figure pas la position raster au moment de la sauvegarde (compteurs CRTC en cours de trame, compteur GA). La réponse est le snapshot initial complet (STAT + ancre zéro), les points de contrôle (RSYN) et la resynchronisation : pas la reprise approximative.
- **Invariant matériel** : laisser le compteur d'interruptions GA en libre parcours (sans ré-ancrage à la VBL) détruit immédiatement la session : la position d'ancrage du GA est un invariant du matériel, pas un réglage d'implémentation.
- **Corollaire pour D-6** : les moments sûrs ne se décrètent pas, ils se mesurent (moteur disquette éteint, upd765 idle, aucune frappe en cours : stabilité vérifiable trame par trame).

## 2. Principes

- **P1 - L'horloge est la machine émulée** : l'unité de temps maîtresse est l'interruption Gate Array (front montant de la demande d'interruption, ~50,08 Hz), comptée depuis le chargement du snapshot. Jamais la trame hôte.
- **P2 - Les entrées sont physiques et adaptées à la machine** : événements exprimés en matrice clavier (ligne 0-9, masque 8 bits) et états joystick, pas en "touches logiques" mappées par l'émulateur.
- **P3 - La dérive est traquée et corrigée** : points de contrôle embarqués (snapshots) à des moments sûrs + événements d'attache conditionnels (PC/mémoire) pour les zones à chargement.
- **P4 - Conteneur extensible et auto-vérifié** : chunks TLV sautables (règle unknown = skip), CRC32 par chunk (code inline, aucune dépendance), réfs média par nom, média embarqué optionnel.
- **P5 - Compatibilité maximale à l'état initial** : le snapshot initial réutilise le corps SNA v3 (déjà compris partout, déjà implémenté dans CPCEPower) complété par des chunks dédiés pour l'état manquant (FDC).

## 3. Identité du format (décision D1 actée le 24/11/2026)

**CPSR (CPC Power Session Replay)**, extension `.cpsr`, magic ASCII `CPSR`.
Alternatives écartées : `.cpsn` (CPC SessioN), `.rsn` (collision probable avec les packs musique SNES), `.csn`, `.vbs` (VBL Synced).
Les noms internes de chunks commencent tous par une majuscule et font 4 caractères.

## 4. Conteneur

```
Offset  Taille  Rôle
0x00    4       Magic "CPSR"
0x04    u16 LE  Version de format (0x0001)
0x06    u16 LE  Flags
                  bit0 : CRC32 présent sur chaque chunk
                  bit1 : réservé (compression - non défini en v1, aucune dépendance externe)
                  bit2 : checkpoints RSYN embarqués
                  bit3 : événements conditionnels WPC/WMEM présents
0x08    u32 LE  Nombre de chunks qui suivent
```

Puis une suite de chunks TLV :

```
ID       4 octets ASCII  ('HEAD','STAT','FDCS','KEYI','EVNT','RSYN','MDIA','EMDA','EMDB','META')
LEN      u32 LE          taille du payload
PAYLOAD  LEN octets
CRC32    u32 LE          sur ID+LEN+PAYLOAD (si flag bit0)
```

Règles : chunk inconnu = sauter via LEN (jamais rejeter le fichier) ; chunks requis = HEAD, STAT, EVNT ; ordre libre sauf EVNT après STAT et KEYI ; doublon d'un chunk requis = dernier gagne (usage ré-édition).

| Chunk | Contenu |
|-------|---------|
| `HEAD` | u8 modèle CPC (codes SNA v3 : 0=464,1=664,2=6128,3=inconnu,4=6128+,5=464+,6=GX4000), u8 type CRTC, u32 nb total d'interruptions du fichier (info), u32 nb total d'événements (info), u64 date d'enregistrement (epoch, info), chaîne ASCII émulateur+version (info), u32 durée totale en interruptions du flux (contrôle, info) |
| `STAT` | snapshot initial = corps SNA v3 complet (octets 0x10..fin, chunks MEMx/ROMS/... inclus) |
| `FDCS` | état upd765 : phase de commande (u8), FIFO (16x u8 + compteurs), u8 PCN par lecteur, et par lecteur : piste (u8), face (u8), moteur (u8), inséré (u8), pointeur position sous-piste (u32) |
| `KEYI` | 10 octets d'état initial de la matrice clavier (reprise sémantique du `SNR ` de WinAPE, sans les 72 octets souris/RTC) |
| `EVNT` | flux d'événements (section 6) |
| `RSYN` | table des points de contrôle (section 7) : N entrées {u32 interruption absolue, u8 type, u32 taille, snapshot inline} + éventuelles empreintes |
| `MDIA` | descripteurs média par lecteur : u8 lecteur, u8 format (0=DSK,1=HFE,2=RAW,3=IPF,4=CPR), u32 taille, u8 inséré, u8 longueur, chaîne nom de fichier (info ; identification par nom, aucun hash - cf. section 8) |
| `EMDA`/`EMDB` | média embarqué drive A/B : payload = image brute du média (décrit par MDIA) |
| `META` | UTF-8 : auteur, logiciel, commentaire, chrono, lien vers fiche CPC-Power (info) |

## 5. Horloge et livraison des événements

- **Compteur maître** : `frame` = nombre d'interruptions Gate Array délivrées depuis le chargement de `STAT`. Incrémenté par l'émulateur sur chaque front d'interruption matériel émulé (PAS sur la trame hôte, PAS sur un timer réel).
- **Convention de livraison** : un événement daté `frame N` est appliqué **juste après l'acquittement de l'interruption N, avant le premier opcode suivant** (l'état de la matrice est stable pour toute la trame). C'est le remède direct à D-1 : tout émulateur conforme obtient le même couple (état clavier, position dans la trame) au même moment machine.
- **Ancre zéro** : le snapshot fige les compteurs internes du CRTC (SNA v3 0xa9-0xb7) : la trajectoire du CRTC depuis t=0 est donc identique pour tout émulateur précis, l'ancrage VBL est reproductible par construction.
- **Sous-trame optionnelle** : les protections qui échantillonnent le clavier en cours de trame sont couvertes par l'événement `SCAN` (section 6), précision ligne+CRTC, sans l'imposer partout.
- **Frein volontaire** : la fréquence nominale (50,08 Hz) n'est jamais gravée : elle découle du matériel émulé. Aucune synchronisation sur l'horloge murale n'est requise pour une relecture conforme.

## 6. Flux d'événements `EVNT`

Encodage générique : `[u8 type][u16 LE delta]` + charge utile propre au type.
`delta` = nombre d'interruptions écoulées depuis l'événement précédent (0 = même trame, réservé aux types qui précisent une sous-position). Deltas > 65534 = insérer des `NOP` explicites (type 0x0F).

| Type | Nom | Payload | Sémantique |
|------|-----|---------|------------|
| 0x00 | END | (aucune) | fin du flux |
| 0x01 | KEYDOWN | u8 ligne (0-9), u8 masque | bit à 1 dans la matrice |
| 0x02 | KEYUP | u8 ligne, u8 masque | bit à 0 |
| 0x03 | KEYSYNC | 10 octets | ré-alignement explicite de toute la matrice (toujours émis par l'enregistreur **après chaque point de contrôle**, pour restaurer la cohérence auto-repeat/état) |
| 0x04 | JOY | u8 index, u8 état (bit par direction+feu) | joystick émulé (port ou extension) |
| 0x05 | SCAN | u8 ligne vidéo (0-311), u8 position CRTC char. (0-63) | s'applique **à l'événement suivant** : précision sous-trame au lieu de l'ancre trame |
| 0x0F | NOP | (aucune) | occupant de delta long |
| 0x10 | SNAP | u8 index RSYN | signal de relecture : point de contrôle atteint, appliquer la politique de resynchronisation (section 7) |
| 0x20 | WMEM | u16 adresse, u8 valeur, u8 masque, u16 timeout_int | attente jusqu'à (mem[adresse] AND masque) == valeur, abandon à timeout (l'événement suivant est alors livré immédiatement) |
| 0x21 | WPC | u16 PC min, u16 PC max, u16 timeout_int | attente jusqu'à ce que le PC entre dans la fenêtre (fenêtres dans la ROM = identiques sur tous les émulateurs, attache la plus fiable contre D-2) |
| 0x22 | WJOY | néant, marqueur | l'événement suivant se déclenche à la première lecture du port joystick par le programme |
| 0xF0 | MARK | u16 index | marqueur nommé (noms dans META), navigation dans la session |
| 0xF1 | NOTE | u8 longueur + chaîne UTF-8 | annotation hors temporel (info) |

Règles : consommation stricte séquentielle ; tout octet non couvert par le type selon sa table = fin propre de la relecture + erreur signalée (règle projet : fichiers hostiles = rejet propre sans crash).

Coût moyen : 1 frappe = 5 octets (KEYDOWN) + 5 octets (KEYUP), soit ~10 octets/frappe, comparable au SNR.

**Attaches conditionnelles (réponse directe au problème "chargements")** : à l'enregistrement, chaque frappe peut être dualisée en `KEYDOWN/WPC` : l'émulateur connaît le PC à l'instant de la frappe et l'inscrit comme fenêtre (élargie de quelques octets pour la tolérance). À la relecture, la frappe part à la date nominale ou légèrement en avance, mais est **retenue jusqu'à satisfaction de la condition** : un loader qui finit 10 trames plus tard/tôt sur un autre émulateur ne fait plus décrocher la session. C'est le complément indispensable de l'ancrage VBL : P1 corrige la dérive de repère, WPC corrige la dérive de chargement.

## 7. Points de contrôle et resynchronisation

- L'enregistreur insère automatiquement un point de contrôle (snapshot complet SNA v3 inline dans `RSYN`, RLE comme pour les MEMx) dès que des conditions de **moment sûr** sont réunies ensemble pendant M trames consécutives : moteur(s) disquette(s) off + upd765 idle + pas d'événement clavier en cours + interruption constatée. Max 1 point de contrôle par K interruptions (défaut 1250 = 25 s). Au moins un point de contrôle à la fin d'un chargement si possible.
- Impact taille : un snapshot ~130 Ko, le RLE des chunks MEMx (schéma 0xE5, celui du SNA v3) réduit à ~30-60 Ko typiques ; la densité par défaut reste raisonnable. Sans point de contrôle possible sur une plage > N trames, l'enregistreur insère un point de contrôle forcé quand même (avertissement dans META).
- L'événement `SNAP` dans `EVNT` porte l'index ; à la relecture, **trois politiques** (config joueur) :
  - **douce** : indicateur seul (comparaison d'empreinte, cf ci-dessous), aucune action ;
  - **stricte (défaut recommandé)** : comparaison, et si empreinte diverge, rechargement du point de contrôle + reprise du flux immédiatement après le `KEYSYNC` suivant (resynchronisation effective, fin du vert/rouge qui ne fait rien) ;
  - **ferme** : rechargement systématique à chaque SNAP (determinisme par tranches ; mode compatibilité maximale inter-émulateurs).
- **Empreinte** (pour l'indicateur et la détection) : CRC32 de zones RAM choisies par l'enregistreur (fenêtres de 256 octets lues en 3 zones stables heuristiques lors de l'émission du point de contrôle) + u32 interruption attendue. Tolérance par fenêtre de ±4 interruptions sur la relecture : la comparaison est indicative, jamais fatale.

## 8. Médias

- **Le média n'est pas lié au format** (décision D2 actée le 24/11/2026) : par défaut l'image disque reste À CÔTÉ du conteneur, référencée par son nom dans `MDIA`. L'embarquement `EMDx` reste possible pour figer la session : si `EMDx` est présent, le média embarqué fait foi, le fichier externe est ignoré.
- Embarquement `EMDx` possible mais non requis (D2) : pratiques selon le besoin ; le corpus CPC-Power qui distribue déjà SNR+DSK en zip reste lisible tel quel (session CPSR + image à côté), et l'embarquement produit un fichier unique figé.
- Un changement de média en cours de session (changement de face) **ne fait pas partie de v1** : ré-enregistrer une session par-face est la règle v1 ; v2 étudiera un événement `MDIA-CHG` (éjection + insertion datée, suspend la session jusqu'à fourniture du média).
- K7 : hors périmètre v1 (décision D5), le positionnement bande (compression/contre-pression) n'est pas couvert ; v2 : chunk `TAPE` (position bloc, image CDT/WAV + hash) sur le même modèle.

## 9. Exigences enregistreur / lecteur

- **Enregistreur** : les événements sont capturés lors du front d'interruption Gate Array (tampon applicatif), pas au fil de l'eau de la boucle hôte. L'enregistrement se fait depuis l'état initial (snapshot OBLIGATOIRE en début d'enregistrement, pas de record depuis le "READY" du BASIC - cause de D-1 documentée par CPC-Power elle-même pour le SNR).
- **Lecteur** : état de la matrice clavier piloté par le noyau en mémoire directement (comme pour l'OSD), ancrage du compteur sur le front Gate Array (site déjà identifiable via `ga->gaHsyncCounter`/`crtc->VS` dans CPCEPower).
- **Zéro dépendance** (acté le 24/11/2026) : enregistrement et relecture sont du code pur de l'émulateur, aucune bibliothèque tierce (zlib retiré du v1 ; CRC32 = table inline ~15 lignes). Une lib C99 optionnelle (`libcpsr-lect`, `libcpsr-ecrit`) pourra accompagner l'adoption par d'autres émulateurs (C/C++), avec une couche d'adaptation minimale par émulateur (fournir : état clavier matrice, front interruption, PC, mémoire, FDC), sans jamais être un prérequis de CPCEPower.
- **Portabilité** : il faut que 464, 664, 6128, 6128+, 464+ et GX4000 passent tous (chunks CPC+ de SNA v3 inclus), aucun modèle dans un sous-ensemble strict.

## 10. Stratégie de compatibilité inter-émulateurs

| Cible | Canal | Effort estimé |
|-------|-------|---------------|
| CPCEPower | intégration directe du noyau v1 | moyen |
| WinAPE (Richard Wilson) | courtoisie + spéc ; pas d'obligation | nul (juste spéc + courrier) |
| Sugarbox V2 (T. Guillemin) | approche directe : la V2 a migré de C# (V1) vers un cœur multi-plateforme CPCCore en C++17 (Qt 6, actif), qui alimente aussi le core libretro SugarLibRetro ; la lib C99 CPSR s'y intègre sans portage de langage (compilation C native à côté du cœur ou adaptation triviale en C++) | faible |
| Caprice32 / nouveaux forks | issues avec la lib C | faible |
| JS et web émulateurs (cpcecpc.eu ; CPC-Power / player Ym-Ay interne) | couche JS/TS de la même arborescence de spéc | moyen |
| RetroArch (cores caprice32, SugarLibRetro) | via Caprice32, ou via le cœur CPCCore de Sugarbox (core libretro SugarLibRetro) dès que Sugarbox adopte | faible dès qu'un des deux cœurs passe |
| CPCWiki / forums | page de spéc + débat forum émulateurs (thread "Playback format for scenes" déjà existant, 2010, sans suite) | communication |

Le spéc final est publié comme page unique (HTML/PDF versionné, campagne CPCWiki + forum émulateurs), le lib C en GPL-3.0 ; l'adoption réelle se jouera sur les trois atouts face au SNR : ancre VBL (P1), points de contrôle avec resynchronisation effective (P3), attaches conditionnelles (WPC/WMEM).

## 11. Migration SNR -> CPSR

- **Convertisseur SNR -> CPSR** (outillage CPCEPower) : traduction 1-1 des deltas de trames en interruptions + reconstruction de HEAD/STAT/KEYI/MDIA, sans points de contrôle (traducteur pur, sans exécution).
  Livré : `cpsr-convert.html` (HTML5/JS autonome, W3C, zéro dépendance, fonctionne hors ligne), dépôt GitHub cpcepower/CPC-Power_Session_Replay (tools/cpsr-convert.html).
- Le résultat couvre D-1 (ancre) mais pas D-2/D-3 : pour un corpus qui décroche, la récupération passe par **ré-enregistrement en lisant le SNR** (CPCEPower joue le SNR et ré-enregistre en CPSR avec points de contrôle WPC) - médiation pragmatique, héritant de la tempo propre du CPCEPower ; les fichiers convertis restent marqués "second hand" dans META.
- Le support lecture SNR actuel **est conservé** (règle de non-régression : un nouveau format ne "vole" rien).

## 12. Plan d'implémentation dans CPCEPower

| Phase | Contenu | Oracles |
|-------|---------|----------------------|
| 0 | spéc v0.2 + décisions D1/D2 actées + convertisseur statique SNR->CPSR (cpsr-convert.html) | revue + validation user |
| 1 | lib pure (parse/write) + intégration CPCEPower (record, playback, horloge interruption) : objectif sessions simples sans chargement | H1 : session synthétique entreprise (octets attendus exacts) ; H2 : round-trip lecture = écriture octet-exact ; H3 : non-régression SNR (le nouveau format ne détruit pas le chemin SNR actuel, corpus CPC-Power 444 fichiers en lecture) |
| 2 | convertisseur SNR->CPSR | H4 : conversion trame par trame d'un petit corpus + vérification diff hexa |
| 3 | RSYN (points de contrôle) + politique stricte en défaut | H5 : session avec chargement disque relue avec et sans injections de dérive artificielle (retard FDC simulé) : relecture stable en mode stricte |
| 4 | WPC/WMEM + SCAN sous-trame | H6 : une session SNR historique qui décrochait est convertie puis ré-enregistrée en CPSR : la relecture reste stable |
| 5 | lib publique + campagne adoption | publication + discussions |

Statut au 26/11/2026 : phase 0 livrée, diagnostic validé par mesures terrain (section 1 bis) ; phase 1 démarrée côté lecture dans CPCEPower (conteneur + CRC32 + reconstruction du flux depuis EVNT) : il reste l'enregistrement natif, l'horloge interruption liée à la livraison et les oracles H1/H2/H3 ; phases 2 à 5 à faire. Le convertisseur statique est en outre validé sur une session réelle complète de 20 minutes (parsing exact, round-trip événement par événement, contrôle croisé durée/EVNT).
