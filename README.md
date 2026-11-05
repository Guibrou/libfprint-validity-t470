# libfprint-validity-t470

**[Français](#français) · [English](#english)**

libfprint with [merge request !626](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/626)
(Validity/Synaptics VCSFW driver) plus three fixes that make the
**138a:0097** sensor of the **Lenovo ThinkPad T470** work: firmware upload,
enroll and verify.

The original libfprint README is in [README.libfprint.md](README.libfprint.md).

---

## Français

### Ce que ce dépôt corrige

La MR !626 détecte le capteur 138a:0097, l'appaire et ouvre le canal TLS, mais
sur le T470 l'enrôlement échouait. Trois commits par-dessus la MR :

| Commit | Correction |
|---|---|
| `validity: pace fwext chunks and add an erase-only mode` | Pause de 50 ms entre les blocs de 4 Ko lors de l'envoi du firmware, et mode `VALIDITY_FWEXT_ERASE_ONLY=1` pour effacer une partition firmware à moitié écrite. |
| `validity: add the missing 55E (major 0x0071) device table entries` | Le capteur répond `major=0x0071 version=0x000d` (VSI 55E FM209-001), absent de la table : identification impossible. |
| `validity: support capture on 55E (0xdb) sensors via line update type 2` | La MR ne savait capturer que sur les capteurs « type 1 ». Portage depuis python-validity du chemin « type 2 » du type 0x00db : programme de capture, vrai blob de calibration (celui de la MR était provisoire), construction de la commande de capture. |

### Testé sur

- ThinkPad T470, Fedora 44, capteur `138a:0097`, ROM 6.7, type 0x00db.
- Envoi du firmware, enrôlement de l'index droit, `verify-match` avec ce doigt
  et `verify-no-match` avec un autre, avec le fprintd du système.

Un seul appareil testé. Les autres capteurs de la MR ne sont pas concernés par
ces changements, sauf le blob de calibration du type 0x00db.

### Prérequis

1. **Données du pilote** (blobs `init`, `db_write_enable`, etc.) dans
   `/usr/share/libfprint/validity/`, comme pour la MR !626.
2. **Firmware Lenovo** `6_07f_lenovo_mis_qm.xpfwext`, à extraire de
   l'installeur Lenovo `nz3gf09w.exe`, dans `/usr/share/libfprint/validity/`
   ou `/usr/share/python-validity/`. Il n'est envoyé que si le capteur n'a pas
   déjà son firmware.
3. Pas de python-validity ni d'open-fprintd installés en même temps
   (conflit avec fprintd).

### Compiler et tester sans installer

```sh
git clone https://github.com/Guibrou/libfprint-validity-t470.git
cd libfprint-validity-t470
meson setup ../build -Ddrivers=validity -Ddoc=false -Dintrospection=false \
  -Dgtk-examples=false -Dudev_rules=disabled -Dudev_hwdb=disabled
ninja -C ../build
meson test -C ../build validity
```

Pour essayer avec fprintd sans rien installer, lancer fprintd à la main
depuis un terminal (pas avec `systemd-run` : SELinux fait ignorer
`LD_LIBRARY_PATH` dans le domaine `fprintd_t`) :

```sh
sudo systemctl stop fprintd
sudo env LD_LIBRARY_PATH="$PWD/../build/libfprint" G_MESSAGES_DEBUG=all \
  /usr/libexec/fprintd -t 2>&1 | tee ~/fprintd-test.log
# dans un autre terminal :
fprintd-enroll
fprintd-verify
```

Le journal doit contenir le chemin `../libfprint/libfprint/drivers/...` :
sinon, c'est la bibliothèque du système qui est chargée.

### Construire un RPM (Fedora)

Partir du paquet source de Fedora et remplacer son archive par ce dépôt :

```sh
dnf download --source libfprint
rpm -i libfprint-*.src.rpm
git archive --format=tar --prefix=libfprint-v1.94.100/ main \
  | gzip > ~/rpmbuild/SOURCES/libfprint-v1.94.100.tar.gz
# Dans ~/rpmbuild/SPECS/libfprint.spec : retirer les lignes Patch*/%patch,
# et mettre une Release plus haute que celle de Fedora, par ex. 99.t470
sudo dnf builddep ~/rpmbuild/SPECS/libfprint.spec
rpmbuild -bb --nocheck ~/rpmbuild/SPECS/libfprint.spec
sudo dnf install ~/rpmbuild/RPMS/x86_64/libfprint-*99.t470*.rpm
sudo dnf versionlock add libfprint   # évite le retour au paquet Fedora
```

Adapter `1.94.100` à la version du paquet source.

### Si l'envoi du firmware échoue

Symptôme : « transfer failed » pendant l'envoi, puis le capteur se déconnecte
(`USB disconnect`, erreurs `-71`). Ce qui a marché sur le T470 :

1. **Couper complètement l'alimentation** : BIOS > Config > Power >
   *Disable built-in battery*, débrancher le secteur et la batterie amovible,
   attendre quelques minutes.
2. Si le capteur répond `0x044f` (partition firmware à moitié écrite) au lieu
   de `0xb004`, l'effacer avec le mode dédié, puis relancer l'envoi normal :

   ```sh
   sudo env LD_LIBRARY_PATH=... VALIDITY_FWEXT_ERASE_ONLY=1 /usr/libexec/fprintd -t
   fprintd-verify   # déclenche l'ouverture, donc l'effacement
   ```

   ⚠️ Ce mode **efface la partition 2** du capteur. Il n'efface rien d'autre,
   ne réappaire pas, et arrête l'ouverture juste après.

`VALIDITY_FWEXT_CHUNK_DELAY_MS` change la pause entre blocs (50 ms par défaut).
Son utilité n'est pas prouvée : elle était active lors de l'envoi réussi, en
même temps que la coupure d'alimentation et l'effacement.

### Limites connues

- Pas de calibration pour le type 0x00db : python-validity n'y fait aucune
  itération et n'en garde qu'une image de référence, qu'il écrit en flash ; ce
  pilote n'écrit jamais cette image. La reconnaissance fonctionne sans.
- La ligne clé de calibration (0x48) est notée « devinée » dans python-validity.

### Crédits et licence

- libfprint : LGPL-2.1, voir [COPYING](COPYING).
- Pilote Validity : MR !626 et ses auteurs.
- Le programme de capture, le blob de calibration et la logique « line update
  type 2 » viennent de [python-validity](https://github.com/uunicorn/python-validity),
  Copyright (c) 2020 uunicorn, licence MIT.

---

## English

### What this repository fixes

MR !626 detects the 138a:0097 sensor, pairs it and sets up the TLS channel,
but on the T470 enrollment failed. Three commits on top of the MR:

| Commit | Fix |
|---|---|
| `validity: pace fwext chunks and add an erase-only mode` | 50 ms pause between the 4 KB chunks of the firmware upload, and a `VALIDITY_FWEXT_ERASE_ONLY=1` mode to erase a half-written firmware partition. |
| `validity: add the missing 55E (major 0x0071) device table entries` | The sensor reports `major=0x0071 version=0x000d` (VSI 55E FM209-001), missing from the table, so identification failed. |
| `validity: support capture on 55E (0xdb) sensors via line update type 2` | The MR could only capture on "type 1" sensors. Port of python-validity's "type 2" path for type 0x00db: capture program, the real calibration blob (the MR's was a placeholder), capture command assembly. |

### Tested on

- ThinkPad T470, Fedora 44, sensor `138a:0097`, ROM 6.7, type 0x00db.
- Firmware upload, right index enrollment, `verify-match` with that finger
  and `verify-no-match` with another one, with the system fprintd.

Only one device was tested. The MR's other sensors are not affected by these
changes, except for the type 0x00db calibration blob.

### Requirements

1. **Driver data** (`init`, `db_write_enable`... blobs) in
   `/usr/share/libfprint/validity/`, as for MR !626.
2. **Lenovo firmware** `6_07f_lenovo_mis_qm.xpfwext`, extracted from Lenovo's
   `nz3gf09w.exe` installer, in `/usr/share/libfprint/validity/` or
   `/usr/share/python-validity/`. It is only uploaded when the sensor does not
   already have its firmware.
3. No python-validity or open-fprintd installed at the same time (they
   conflict with fprintd).

### Build and test without installing

```sh
git clone https://github.com/Guibrou/libfprint-validity-t470.git
cd libfprint-validity-t470
meson setup ../build -Ddrivers=validity -Ddoc=false -Dintrospection=false \
  -Dgtk-examples=false -Dudev_rules=disabled -Dudev_hwdb=disabled
ninja -C ../build
meson test -C ../build validity
```

To try it with fprintd without installing anything, run fprintd by hand from
a terminal (not with `systemd-run`: SELinux makes `LD_LIBRARY_PATH` ignored in
the `fprintd_t` domain):

```sh
sudo systemctl stop fprintd
sudo env LD_LIBRARY_PATH="$PWD/../build/libfprint" G_MESSAGES_DEBUG=all \
  /usr/libexec/fprintd -t 2>&1 | tee ~/fprintd-test.log
# in another terminal:
fprintd-enroll
fprintd-verify
```

The log must show the path `../libfprint/libfprint/drivers/...`; otherwise
the system library was loaded.

### Build an RPM (Fedora)

Start from Fedora's source package and replace its tarball with this
repository:

```sh
dnf download --source libfprint
rpm -i libfprint-*.src.rpm
git archive --format=tar --prefix=libfprint-v1.94.100/ main \
  | gzip > ~/rpmbuild/SOURCES/libfprint-v1.94.100.tar.gz
# In ~/rpmbuild/SPECS/libfprint.spec: remove the Patch*/%patch lines and set
# a Release higher than Fedora's, e.g. 99.t470
sudo dnf builddep ~/rpmbuild/SPECS/libfprint.spec
rpmbuild -bb --nocheck ~/rpmbuild/SPECS/libfprint.spec
sudo dnf install ~/rpmbuild/RPMS/x86_64/libfprint-*99.t470*.rpm
sudo dnf versionlock add libfprint   # keeps Fedora's package from coming back
```

Adjust `1.94.100` to the source package version.

### If the firmware upload fails

Symptom: "transfer failed" during the upload, then the sensor disconnects
(`USB disconnect`, `-71` errors). What worked on the T470:

1. **Cut power completely**: BIOS > Config > Power > *Disable built-in
   battery*, unplug AC and the removable battery, wait a few minutes.
2. If the sensor answers `0x044f` (half-written firmware partition) instead of
   `0xb004`, erase it with the dedicated mode, then run the normal upload
   again:

   ```sh
   sudo env LD_LIBRARY_PATH=... VALIDITY_FWEXT_ERASE_ONLY=1 /usr/libexec/fprintd -t
   fprintd-verify   # triggers the open, hence the erase
   ```

   ⚠️ This mode **erases partition 2** of the sensor. It erases nothing else,
   does not re-pair, and stops the open right after.

`VALIDITY_FWEXT_CHUNK_DELAY_MS` changes the pause between chunks (50 ms by
default). Its usefulness is not proven: it was active during the successful
upload, together with the power cut and the erase.

### Known limitations

- No calibration for type 0x00db: python-validity runs no iterations there and
  only keeps a reference image, which it writes to flash; this driver never
  writes that image. Recognition works without it.
- The calibration key line (0x48) is marked as a guess in python-validity.

### Credits and license

- libfprint: LGPL-2.1, see [COPYING](COPYING).
- Validity driver: MR !626 and its authors.
- The capture program, calibration blob and "line update type 2" logic come
  from [python-validity](https://github.com/uunicorn/python-validity),
  Copyright (c) 2020 uunicorn, MIT License.
