# CPC-Power Session Replay (CPSR)

CPSR (CPC Power Session Replay) est un format de session (enregistrement + relecture) pour émulateurs Amstrad CPC (464/664/6128/Plus/GX4000), synchronisé sur la VBL de la machine émulée et conçu pour être adopté par tous les émulateurs. Il vise à remplacer à terme le format SNR de WinAPE, dont les sessions décrochent pendant les chargements dès qu'on les rejoue sur un autre émulateur.

Principe : l'horloge maîtresse est l'interruption Gate Array de la machine émulée (jamais la trame de l'émulateur hôte), les entrées sont exprimées en matrice clavier physique (ligne 0-9, masque 8 bits), la dérive est traquée par points de contrôle embarqués et resynchronisation. Conteneur à chunks TLV sautables, CRC32 par table intégrée, aucune dépendance.

## Contenu du dépôt

- SPEC.md : spécification de référence v0.3 (brouillon).
- tools/cpsr-convert.html : convertisseur de sessions WinAPE .snr vers .cpsr et outil de vérification .cpsr.
- CHANGELOG.md : historique des livraisons.
- LICENSE : licence GPL-3.0.

## Utilisation du convertisseur

1. Télécharger tools/cpsr-convert.html et l'ouvrir dans un navigateur : c'est un fichier local autonome, il fonctionne hors ligne (file://), sans dépendance ni accès réseau.
2. Déposer le fichier .snr (et, facultativement, l'image disque .dsk, .hfe, .raw, .ipf ou .cpr référencée par la session, reconnue par son nom).
3. Options : CRC32 par chunk (recommandé, défaut coché) ; embarquement des médias dans EMDA/EMDB (facultatif : par défaut l'image reste à côté du conteneur, référencée par son nom dans MDIA, décision D2) ; champ auteur (repris dans META).
4. Convertir : le fichier .cpsr est téléchargé.
5. Déposer un .cpsr existant pour le vérifier : liste des chunks, CRC32 (OK/KO), HEAD/KEYI/EVNT/MDIA/EMD décodés, contrôle croisé durée/EVNT.
6. Le bouton Self-test rejoue une session synthétique complète (32 contrôles : analyse, conversion, relecture, entrées hostiles).

L'outil accepte les deux magics de session WinAPE ("RW - SNR" et "MV - SNA"), les extensions sans tenir compte de la casse, et rejette proprement les fichiers tronqués ou corrompus (avertissement, jamais de crash). Le décodage du flux SNR est un miroir du lecteur de l'émulateur CPCEPower (SNR.cpp / SNA.cpp), lui-même compatible WinAPE.

## Statut

Spécification v0.3 (brouillon, 26/11/2026 ; v0.2 : 24/11/2026) : la phase 0 (spécification + convertisseur statique) est livrée et le diagnostic est validé par mesures sur un cas réel exigeant (SPEC.md, section 1 bis). L'implémentation native dans CPCEPower est démarrée côté lecture (conteneur + CRC32 + reconstruction du flux EVNT) : enregistrement, points de contrôle RSYN et attaches conditionnelles WPC/WMEM suivent, de même qu'une lib C d'adoption pour d'autres émulateurs (cf. SPEC.md, section 12 ; Sugarbox V2 disposant désormais d'un cœur C++17, l'adoption y est directe). Le convertisseur, validé sur une session réelle complète (round-trip exact), marque les fichiers produits "second-hand" dans META (traduction 1-1, sans points de contrôle).

## Licence

GPL-3.0 (cf. LICENSE), (c) 2026 Megachur.
