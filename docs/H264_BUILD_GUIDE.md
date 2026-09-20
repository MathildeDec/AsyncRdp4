# Guide : compiler FreeRDP avec un vrai support H.264/GFX

Ce guide documente une procédure **entièrement vérifiée** — pas théorique. Chaque étape a été exécutée réellement, y compris un test de bout en bout avec de vraies frames H.264 reçues via un serveur `freerdp-shadow-cli` compilé selon cette même recette.

## Pourquoi c'est nécessaire

Le paquet `libfreerdp3`/`libfreerdp-client3` d'Ubuntu 24.04 (et très probablement d'autres distributions packagées de la même façon) est compilé avec :
```
WITH_GFX_H264=OFF
WITH_OPENH264=OFF
WITH_FFMPEG=OFF
```
Résultat vérifié : les symboles de l'API H.264 existent (`h264_context_new`, `h264_compress`...) mais ne sont raccordés à aucun décodeur réel. Activer le pipeline GFX/H.264 avec cette lib provoque un **crash (SIGSEGV, pointeur de fonction NULL)** dès qu'un vrai flux H.264 arrive — confirmé par trace `gdb` complète pointant vers un appel de fonction à l'adresse `0x0`.

Il faut donc recompiler FreeRDP soi-même avec `-DWITH_OPENH264=ON`.

## Étape 1 — Dépendances de build

La méthode la plus fiable : activer les dépôts sources Ubuntu et laisser `apt` calculer la liste exacte de dépendances de compilation pour la version installée, plutôt que deviner une liste à la main.

```bash
# Activer deb-src (fichier au format deb822 sur Ubuntu 24.04+)
sudo sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/ubuntu.sources
sudo apt update

sudo apt install -y devscripts libopenh264-dev git cmake
sudo apt-get build-dep -y freerdp3
```

`apt-get build-dep` installe automatiquement tout ce dont FreeRDP a besoin pour cette version précise (X11, Wayland, Kerberos, CUPS, PCSC, JPEG, etc.) — évite d'avoir à maintenir une liste manuelle qui se périmerait à chaque nouvelle version.

## Étape 2 — Récupérer les sources (même version que le paquet système)

```bash
git clone --depth 1 --branch v3.30.0 https://github.com/FreeRDP/FreeRDP.git freerdp-src
cd freerdp-src
```

Vérifier la version installée avec `pkg-config --modversion freerdp3` et adapter le tag `--branch` en conséquence — cloner une version différente de celle système peut fonctionner mais n'a pas été testé ici.

## Étape 3 — Configurer avec H.264 activé

```bash
mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/opt/freerdp-h264 \
  -DWITH_OPENH264=ON \
  -DBUILD_SHARED_LIBS=ON \
  -DWITH_CLIENT=ON \
  -DWITH_SERVER=ON \
  -DWITH_SHADOW=ON
```

Points vérifiés :
- `-DCMAKE_INSTALL_PREFIX=/opt/freerdp-h264` installe **en parallèle** du paquet système (`/usr`), sans rien écraser — recommandé, permet de revenir en arrière instantanément.
- `WITH_GFX_H264` n'est **pas** une option à positionner soi-même : elle est calculée automatiquement (`ON` dès que `WITH_OPENH264` — ou `WITH_VIDEO_FFMPEG`/`WITH_MEDIACODEC` — est activé). Voir `CMakeLists.txt` ligne ~445 du dépôt FreeRDP.
- Confirmer qu'OpenH264 a bien été détecté (pas juste accepté silencieusement) :
  ```
  -- Finding optional feature OpenH264 for codec (use OpenH264 library)
  -- Found OpenH264: /usr/lib/x86_64-linux-gnu/libopenh264.so
  ```
  Si cette ligne est absente ou dit autre chose, `libopenh264-dev` n'est pas trouvé par `pkg-config` — vérifier `pkg-config --exists openh264`.
- `-DWITH_SHADOW=ON` : nécessaire seulement si vous voulez aussi un **serveur** de test (`freerdp-shadow-cli`) capable de négocier H.264 pour valider votre client — pas nécessaire si vous avez déjà accès à un vrai serveur RDP qui supporte GFX/H.264 (Windows RDS, par exemple).

## Étape 4 — Compiler et installer

```bash
make -j$(nproc)
sudo make install
```

Vérifié : ~2m30 sur une seule machine mono-cœur, aucune erreur (seulement des warnings de dépréciation, normaux et sans conséquence).

Vérification post-installation :
```bash
ldd /opt/freerdp-h264/lib/libfreerdp3.so.3 | grep -i h264
# Doit afficher : libopenh264.so.7 => ...
```
Si cette ligne est absente après compilation, `WITH_OPENH264` n'a probablement pas été réellement activé — revérifier la sortie de `cmake` à l'étape 3.

## Étape 5 — Recompiler `asyncrdp` contre cette nouvelle lib

`asyncrdp` (le shim cffi) doit être recompilé pour lier contre `/opt/freerdp-h264` plutôt que les paquets système :

```bash
export PKG_CONFIG_PATH=/opt/freerdp-h264/lib/pkgconfig
export LD_LIBRARY_PATH=/opt/freerdp-h264/lib
python3 build_ffi.py
```

**Important** : `LD_LIBRARY_PATH` doit aussi être positionné à l'exécution (pas seulement à la compilation), sinon le programme retombera sur la lib système sans H.264 au chargement dynamique. En production, préférer `-Wl,-rpath` au build ou un fichier `/etc/ld.so.conf.d/` plutôt que de dépendre d'une variable d'environnement à chaque lancement.

## Étape 6 — Correctif nécessaire côté `asyncrdp` : le callback `DesktopResize`

En testant contre un vrai flux GFX, un deuxième problème est apparu (après la résolution du premier par la recompilation) : un `WINPR_ASSERT` sur `update->DesktopResize` dans `gdi_ResetGraphics` (`libfreerdp/gdi/gfx.c`). Le pipeline GFX exige ce callback en plus d'`EndPaint` — **déjà corrigé dans `asyncrdp_shim.c`** livré avec ce projet (voir la fonction `asyncrdp_set_end_paint_callback`, qui enregistre maintenant aussi `ctx->update->DesktopResize`). Si vous partez d'une version antérieure du shim, ajoutez :

```c
static BOOL asyncrdp_on_desktop_resize(rdpContext* ctx) { return TRUE; }
// dans asyncrdp_set_end_paint_callback() :
ctx->update->DesktopResize = asyncrdp_on_desktop_resize;
```

## Étape 7 — Tester pour de vrai

Si vous avez aussi compilé le serveur shadow (étape 3, `-DWITH_SHADOW=ON`) :

```bash
# Un vrai bureau X11 à partager (ex: une session existante, ou Xvfb + un WM)
DISPLAY=:N XAUTHORITY=~/.Xauthority \
  LD_LIBRARY_PATH=/opt/freerdp-h264/lib \
  /opt/freerdp-h264/bin/freerdp-shadow-cli /port:3391 -auth -sec-nla

# IMPORTANT : sur un bureau parfaitement statique, RDP n'envoie qu'une frame
# initiale puis plus rien (comportement normal, delta-based) — lancez
# quelque chose qui bouge réellement pour valider un vrai flux continu :
DISPLAY=:N xclock -update 1 &
```

Côté client :
```python
options = asyncrdp.RdpOptions(
    enable_graphics_pipeline=True,
    enable_h264=True,
    enable_h264_444=True,
    enable_progressive_codec=True,
)
async with asyncrdp.connect(host, port, username, password, options=options) as client:
    frame = await client.get_frame()
    print(client._gfx is not None)  # True si le canal GFX a bien pris le relais
```

Résultat obtenu lors de la validation de ce guide : 5 frames GFX reçues en continu, `client._gfx` non-`None` du début à la fin, taille et résolution correctes, déconnexion propre sans crash.

## Limitations restantes (honnêtes)

- Testé contre `freerdp-shadow-cli` (même codebase FreeRDP côté serveur) — **pas testé contre Windows RDS**, qui reste la cible de production la plus probable pour du vrai GFX/H.264 en environnement d'entreprise.
- Testé en NLA désactivée (`-auth -sec-nla` côté serveur, `enable_nla=False` côté client) pour simplifier — le pipeline GFX/H.264 lui-même est indépendant du mécanisme d'authentification, mais la combinaison n'a pas été revérifiée avec NLA actif.
- Le crash initial (décodeur NULL) était spécifique aux **frames H.264 elles-mêmes** — RemoteFX/Progressive (codecs GFX non-H.264) fonctionnaient probablement déjà avec la lib système, seul H.264/AVC444 nécessitait cette recompilation.
