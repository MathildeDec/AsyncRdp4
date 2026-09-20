---
name: sessions-index
---

# Sessions asyncrdp — index

Journal détaillé, une session par fichier, classé par ordre chronologique (le fichier `CLAUDE.md` original mélangeait l'ordre par endroits — remis d'aplomb ici). Deux en-têtes du fichier d'origine qui n'étaient que la suite immédiate l'une de l'autre ont été fusionnées dans un seul fichier (sessions 01, 19, 20).

À rouvrir seulement si vous avez besoin du raisonnement détaillé ou d'une décision de conception précise — pas à chaque démarrage. Pour l'état courant, voir `../../CLAUDE.md`.

| # | Date | Session |
|---|------|---------|
| 01 | 2026-09-01 | [Stub GTK4 pour tester la logique des ponts sans environnement graphique](session-01.md) |
| 02 | 2026-09-02 | [Revérification des noms de champs `_shim.c` contre FreeRDP `master`](session-02.md) |
| 03 | 2026-09-02 | [Exécution réelle des ponts GTK4 dans un vrai GTK4 + Xvfb](session-03.md) |
| 04 | 2026-09-02 | [Pont fichiers GTK4 en conditions réelles, sens local → RDP](session-04.md) |
| 05 | 2026-09-02 | [Diagnostic affiné du bug mime-type presse-papier image GTK4](session-05.md) |
| 06 | 2026-09-02 | [Tentative de correctif du bug mime-type presse-papier image](session-06.md) |
| 07 | 2026-09-04 | [Robustification du test disque : quatre bugs réels trouvés et corrigés](session-07.md) |
| 08 | 2026-09-05 | [Audio réel : PipeWire monté, charge utile RDPSND confirmée](session-08.md) |
| 09 | 2026-09-05 | [Imprimante/série/parallèle : confirmation définitive via le code source de xrdp](session-09.md) |
| 10 | 2026-09-05 | [Audio réel, deuxième sens : capture (client → serveur) validée bout en bout](session-10.md) |
| 11 | 2026-09-05 | [Pont fichiers GTK4, sens RDP → local : couverture par tests de logique pure](session-11.md) |
| 12 | 2026-09-05 | [Comparaison avec `PATTERNS.md` d'un autre projet](session-12.md) |
| 13 | 2026-09-05 | [Test d'absence de dépendances circulaires ajouté](session-13.md) |
| 14 | 2026-09-06 | [Décorateur de traçabilité entrée/sortie ajouté](session-14.md) |
| 15 | 2026-09-06 | [Segfault GTK4 live diagnostiqué et contourné](session-15.md) |
| 16 | 2026-09-06 | [Pont fichiers RDP → local en conditions réelles GTK4 : gbulb intégré](session-16.md) |
| 17 | 2026-09-07 | [Bug d'écriture du ContentProvider custom : la cause n'est pas là où on la cherchait](session-17.md) |
| 18 | 2026-09-07 | [Module d'énumération USB : rattrapage de packaging](session-18.md) |
| 19 | 2026-09-07 | [Première session sur un vrai poste physique (+ addendum même jour : second serveur RDP externe)](session-19.md) |
| 20 | 2026-09-08 | [Revérification complète : compilation, connexion xrdp réelle, et suite GTK4 live](session-20.md) |
| 21 | 2026-09-09 | [Nettoyage ruff : 75 → 0 erreurs, deux vrais bugs trouvés au passage](session-21.md) |
| 22 | 2026-09-10 | [Règle udev USB + diagnostic de permissions ; rattrapage de documentation (README obsolète)](session-22.md) |
| 23 | 2026-09-10 | [Régression réelle des tests GTK4 « logique pure » : le stub `asyncrdp` ne suivait pas l'ajout du décorateur `traced`](session-23.md) |
| 24 | 2026-09-11 | [Revérification complète (GTK4/Xvfb enfin disponibles) et diagnostic affiné du bug mime-type presse-papier image via le code source GDK](session-24.md) |
| 25 | 2026-09-11 | [Recherche d'un rapport de bug PyGObject existant et rédaction d'un rapport prêt à déposer pour le bug presse-papier image](session-25.md) |
| 26 | 2026-09-11 | [Reprise depuis une archive zip dans une nouvelle conversation ; bug d'abandon silencieux dans `install.sh` trouvé et corrigé](session-26.md) |
| 27 | 2026-09-14 → 2026-09-15 | [Presse-papier image GTK4 résolu par une sous-classe `GdkContentProvider` écrite en C ; bug de timing `GI_TYPELIB_PATH` trouvé en cours de route](session-27.md) |
| 28 | 2026-09-15 | [Imprimante/série/parallèle : piste FreeRDP-serveur explorée (parquée) ; migration pip→uv livrée](session-28.md) |
| 29 | 2026-09-16 | [Imprimante acceptée par un vrai serveur (une première) ; série/parallèle bloqués par un désaccord interne à FreeRDP ; le « blocage sandbox » de la session 28 était un mauvais diagnostic](session-29.md) |
| 30 | 2026-09-16 | [Premier run combiné complet depuis la session 20 (112 tests, 0 échec) ; dépendance Pillow manquante depuis la session 20 enfin déclarée ; le conteneur a redémarré en plein milieu de la session](session-30.md) |
| 31 | 2026-09-18 | [Assemblage GTK4 dans une vraie application hôte (`RdpSession` + visualiseur de référence) ; Xvfb sans module GLX découvert, bloquant la vérification live](session-31.md) |
| 32 | 2026-09-18 | [Assemblage GTK4 revalidé en conditions réelles (Xvfb avec GLX cette fois) ; deux bugs réels trouvés et corrigés (`install.sh --gtk4` cassé, collision de GType entre deux tests)](session-32.md) |
| 33 | 2026-09-18 | [Revérification complète dans un nouveau bac à sable : aucun bug trouvé, zéro régression (127 passed contre un vrai serveur xrdp local)](session-33.md) |
