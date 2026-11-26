# CHANGELOG

## 202611

lot 202611.1.1 (24/11/2026) :

- SPEC.md : spécification CPSR v0.2 (brouillon) - diagnostic des décrochages SNR, horloge interruption Gate Array, conteneur TLV avec CRC32 sans dépendance, blocs HEAD/STAT/FDCS/KEYI/EVNT/RSYN/MDIA/EMDA/EMDB/META, points de contrôle et resynchronisation, médias référencés par nom ; décisions actées D1 (identité CPSR, extension .cpsr, magic CPSR) et D2 (le média n'est pas lié au format, embarquement optionnel).
- tools/cpsr-convert.html : convertisseur autonome HTML5/JavaScript (miroir du lecteur SNR de CPCEPower, SNR.cpp/SNA.cpp) - conversion .snr vers .cpsr avec CRC32 et/ou embarquement média optionnels, mode vérification .cpsr (chunks, CRC32, décodage HEAD/KEYI/EVNT/MDIA/EMD), self-test 32 contrôles intégré, rejet propre des fichiers hostiles.
- README.md : présentation du dépôt, usage du convertisseur, statut, licence.
- CHANGELOG.md : création du fichier.
lot 202611.1.2 (26/11/2026) :

- SPEC.md : v0.3 - nouvelle section 1 bis "validation du diagnostic par mesures terrain" (identité à l'échelle trame mais divergence sous-trame à 3 unités d'un seuil critique ; 52 décalages d'ancrage testés sans restauration ; reprise depuis snapshot intermédiaire non fidèle ; compteur GA libre = session détruite) ; ligne Sugarbox actualisée (V2 = cœur CPCCore en C++17/Qt 6 : plus de portage C#, adoption directe de la lib C) ; ligne RetroArch complétée (core SugarLibRetro) ; statut du plan d'implémentation ajouté (phase 1 démarrée côté lecture).
- README.md : statut actualisé (v0.3, validation terrain, phase 1 entamée, convertisseur validé sur une session réelle complète).

