[← index](README.md) — session 13 — 2026-09-05

## Test d'absence de dépendances circulaires ajouté (2026-09-05, nouvelle session)

Tâche choisie dans le backlog 🟢 (candidat explicitement identifié
comme le plus simple à porter tel quel lors de la comparaison
`PATTERNS.md` ci-dessus) : le test anti-cycle, seul des trois écarts
qui ne nécessite ni GTK4/Xvfb/serveur réel ni modification du code de
production — faisable entièrement hors ligne.

**Méthode**, reprenant celle décrite dans `PATTERNS.md` : graphe des
imports internes construit par analyse `ast` (pas une relecture
manuelle) sur les fichiers de `src/asyncrdp/` et `integrations/gtk4/`
(`tests/` volontairement exclu — un test qui importe le code testé
n'est pas un cycle applicatif), puis validation acyclique par un
parcours en profondeur classique (couleurs blanc/gris/noir). Le
détecteur (`find_cycle`) est testé isolément sur des graphes fabriqués
à la main (cycle direct à 2 nœuds, cycle indirect à 3 nœuds, diamant
acyclique, dépendance partagée visitée par deux branches sans cycle)
avant d'être appliqué au vrai graphe du projet — même discipline que
les autres tests « preuve que ça détecterait une vraie régression » du
projet.

**Résultat sur le vrai graphe** : acyclique, comme attendu vu la
simplicité de la structure du projet — `asyncrdp` (le paquet) dépend
de `asyncrdp._core` ; `_core` ne dépend d'aucun autre module interne
(seulement de son extension cffi compilée, externe à ce graphe) ; les
deux ponts GTK4 (`gcm_gtk4_display_bridge`/`gcm_gtk4_clipboard_bridge`)
dépendent du paquet `asyncrdp`, sans dépendance en sens inverse. Un
test dédié (`test_import_graph_matches_known_dependencies`) fige ces
arêtes attendues, pour qu'un futur ajout d'import interne oublié soit
visible même sans introduire de cycle.

**Validation réelle du détecteur contre une vraie régression** (pas
seulement contre des graphes fabriqués à la main) : un import
temporaire de `gcm_gtk4_display_bridge` ajouté dans `src/asyncrdp/
_core.py` (créant un vrai cycle `asyncrdp -> asyncrdp._core ->
gcm_gtk4_display_bridge -> asyncrdp`), restauré aussitôt après.
Résultat : `test_project_internal_imports_are_acyclic` échoue
proprement avec le cycle exact affiché dans le message d'assertion,
`test_import_graph_matches_known_dependencies` échoue aussi (arêtes
attendues plus respectées), les 5 autres tests du fichier continuent
de passer — confirme que ces deux tests détectent bien une régression
réelle, pas seulement les cas synthétiques. `_core.py` revérifié
byte-identique à l'original après restauration (`diff` contre une
sauvegarde prise avant la mutation).

**Contrainte d'environnement** : `pytest` n'est pas installé dans ce
sandbox (pas d'accès réseau pour le récupérer) — fichier écrit avec
des `assert` nus et un petit exécuteur maison en bas de fichier
(`python3 tests/test_no_circular_imports.py`), même convention que
`test_gtk4_clipboard_file_download.py`. Reste compatible `pytest` tel
quel pour un environnement où il est disponible.

**Fichiers modifiés cette session** :
- Ajouté : `tests/test_no_circular_imports.py` (nouveau).
- `features.md` et ce fichier.
- Aucun fichier de code de production modifié au final (`_core.py`
  temporairement muté puis restauré identique, voir ci-dessus).
