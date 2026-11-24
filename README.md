# Rivendell 4.5.0 pour Debian 13 « Trixie »

Ce dépôt est un fork de [Rivendell](https://github.com/ElvishArtisan/rivendell),
le système d'automatisation radio de Paravel Systems. Il permet de compiler et
d'installer **Rivendell 4.5.0 sur Debian 13 « Trixie »**, ce que la version
officielle ne permet pas encore.

## Pourquoi ce fork ?

Deux dépendances de Rivendell 4.5.0 ont changé dans Debian 13 :

| Problème sur Trixie | Correctif intégré |
|---|---|
| **QtWebKit a été retiré** de Debian 13. Sans lui, `configure` s'arrête, car le widget de messages de rdairplay en a besoin. | [PR #1013](https://github.com/ElvishArtisan/rivendell/pull/1013) : utilise **QtWebEngine** s'il est présent, et QtWebKit sinon |
| Debian 13 ne fournit plus **ImageMagick 6** (seulement le 7), et `configure` ne cherche que Magick++ 6. | [PR #1019](https://github.com/ElvishArtisan/rivendell/pull/1019) : accepte **Magick++ 7** ou Magick++ 6 |

Ces deux PR, écrites par `k2fc`, sont encore ouvertes sur le dépôt officiel.

### Branches

| Branche | Contenu |
|---|---|
| `trixie-prs` (par défaut) | `v4` officiel (v4.5.0), avec les PR #1019 et #1013 fusionnées |
| `v4` | Copie conforme de la branche `v4` officielle, sans modification |

Le code de Rivendell n'est pas modifié en dehors de ces deux PR. Seul le
fichier `ChangeLog` a demandé une résolution manuelle, pour garder les deux
entrées.

---

## 1. Dépendances de compilation

```sh
sudo apt install \
  autoconf automake libtool g++ make debhelper \
  qtbase5-dev libqt5sql5-mysql qtwebengine5-dev qttools5-dev-tools \
  libexpat1 libexpat1-dev libssl-dev libsamplerate-dev libsndfile-dev \
  libcdparanoia-dev libcoverart-dev libdiscid-dev libmusicbrainz5-dev \
  libid3-dev libtag1-dev libcurl4-gnutls-dev libpam0g-dev libsoundtouch-dev \
  docbook5-xml libxml2-utils docbook-xsl-ns xsltproc fop \
  libsystemd-dev libjack-jackd2-dev libasound2-dev \
  libflac-dev libflac++-dev libmp3lame-dev libmad0-dev libtwolame-dev \
  libmagick++-dev \
  python3 python3-pycurl python3-pymysql python3-serial python3-requests \
  python3-virtualenv
```

Il faut aussi les services nécessaires au fonctionnement :

```sh
sudo apt install mariadb-server apache2 jackd2
```

`qtwebengine5-dev` installe Chromium embarqué, ce qui représente plusieurs
centaines de Mo.

## 2. Récupérer les sources

```sh
git clone git@github.com:samuelvermeulenpro/rivendell_for_trixie.git rivendell
cd rivendell
git switch trixie-prs
```

## 3. Configurer

```sh
export DOCBOOK_STYLESHEETS=/usr/share/xml/docbook/stylesheet/docbook-xsl-ns
./autogen.sh
./configure_build.sh
```

`DOCBOOK_STYLESHEETS` est **obligatoire**. Sans cette variable, la génération
de la documentation échoue pendant `make`, avec l'erreur
`failed to load external entity "../../helpers/docbook/template/titlepage.xsl"`.

`configure_build.sh` détecte Debian et lance `./configure` avec
`--prefix=/usr --libdir=/usr/lib --libexecdir=/var/www/rd-bin --sysconfdir=/etc/apache2/conf-enabled`.

Dans la sortie, vérifie ces lignes :

```
checking for Qt5WebEngineWidgets... yes
checking for Magick++-7.Q16... yes
...
|                 JACK Audio Connection Kit ... Yes       |
```

Le pilote JACK est activé automatiquement quand `libjack-jackd2-dev` est
installé. Pour le désactiver, utilise `--disable-jack`. Les options
`--disable-alsa` et `--disable-hpi` fonctionnent de la même façon.

## 4. Compiler

```sh
make -j"$(nproc)"
```

Vérifications facultatives :

```sh
ldd rdairplay/.libs/rdairplay | grep -E 'WebEngine|Magick'   # QtWebEngine + Magick++ 7
ldd cae/.libs/caed | grep jack                                # libjack.so.0
```

## 5. Installer

```sh
sudo make install
sudo ldconfig
sudo systemctl restart apache2
```

Les fichiers sont installés aux emplacements suivants :

| Élément | Emplacement |
|---|---|
| Applications (`rdairplay`, `rdadmin`, `rdlibrary`…) | `/usr/bin` |
| Démons (`caed`, `ripcd`, `rdcatchd`, `rdservice`, `rddbmgr`…) | `/usr/sbin` |
| Bibliothèques (`librd-4.5.0.so`, `librdalsa-4.5.0.so`) | `/usr/lib` |
| Scripts web (rdxport) | `/var/www/rd-bin` |
| Configuration Apache | `/etc/apache2/conf-enabled/rd-bin.conf` |
| Service systemd | `/usr/lib/systemd/system/rivendell.service` |

`make install` **n'écrase pas** un `/etc/rd.conf` existant.

## 6. Configuration initiale (première installation)

### Compte système et dossier audio

```sh
sudo useradd --system --create-home --groups audio rivendell   # si le compte n'existe pas
sudo mkdir -p /var/snd
sudo chown rivendell:rivendell /var/snd
sudo chmod 0775 /var/snd
```

Ajoute aussi ton utilisateur aux groupes `audio` et `rivendell` :
`sudo usermod -aG audio,rivendell $USER`, puis rouvre ta session.

### `/etc/rd.conf`

```sh
sudo cp conf/rd.conf-sample /etc/rd.conf
```

Modifie ensuite ces valeurs :

- `[Identity]` : `Password=` (la valeur par défaut est `letmein`, à changer),
  `AudioOwner`/`AudioGroup` (par défaut `rivendell`) et
  `RnRmlOwner`/`RnRmlGroup` (l'utilisateur qui exécute les commandes RML
  `RN`, par exemple ton propre compte).
- `[mySQL]` : `Hostname=`, `Loginname=`, `Password=` et `Database=`.

### Base de données (MariaDB local)

Remplace `MOT_DE_PASSE` par celui que tu as mis dans `[mySQL]` :

```sh
sudo mysql <<'SQL'
CREATE DATABASE Rivendell CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'rduser'@'localhost' IDENTIFIED BY 'MOT_DE_PASSE';
GRANT ALL ON Rivendell.* TO 'rduser'@'localhost';
FLUSH PRIVILEGES;
SQL

sudo rddbmgr --create --generate-audio
```

`rddbmgr --create` refuse de travailler sur une base qui n'est pas vide.
Si tu mets à jour une installation existante, utilise plutôt
`sudo rddbmgr --modify`, après une sauvegarde avec `mysqldump`. Rivendell 4.5.0
utilise le schéma **377**.

## 7. Configurer JACK

### a) Libérer la carte son pour JACK

Si `rdalsaconfig` a déjà été utilisé, `/etc/asound.conf` peut contenir un
bloc `pcm.rd0` qui attribue la carte au **pilote ALSA** de Rivendell. Dans ce
cas, `caed` ouvre la carte en ALSA avant que jackd puisse démarrer, et JACK ne
fonctionne pas. Supprime ce bloc :

```sh
sudo cp -a /etc/asound.conf /etc/asound.conf.bak
sudo sed -i '/Start of Rivendell configuration/,/End of Rivendell configuration/d' /etc/asound.conf
```

### b) Faire démarrer jackd par Rivendell

Dans **rdadmin**, va dans Manage Hosts → *ton hôte* → **JACK Settings** :

- coche **Start JACK Server** ;
- dans **JACK Command Line**, mets par exemple
  `/usr/bin/jackd -R -d alsa -d hw:PCH,0 -r 48000 -p 1024 -n 2`.
  Adapte `hw:…` à ta carte (liste-les avec `aplay -l`), et garde `-r` égal à la
  fréquence système de Rivendell, 48000 par défaut ;
- laisse **JACK Server Name** vide pour utiliser le serveur par défaut.

`caed` démarre alors jackd en root à chaque démarrage du service. Comme le
service définit `JACK_PROMISCUOUS_SERVER=audio`, les membres du groupe `audio`
peuvent s'y connecter.

### c) Relier les sorties de Rivendell aux haut-parleurs

Au départ, aucun port n'est connecté. Dans la même boîte de dialogue, ajoute
ces lignes à **JACK Clients to Start**, une par canal (`caed` n'utilise pas de
shell, donc on ne peut pas enchaîner deux commandes sur une même ligne) :

```
/usr/bin/jack_connect rivendell_0:playout_0L system:playback_1
/usr/bin/jack_connect rivendell_0:playout_0R system:playback_2
```

`caed` les lance environ 6 secondes après s'être connecté à JACK, à chaque
démarrage du service.

⚠️ Tant que Rivendell tourne, jackd occupe la carte son. Si c'est la carte de
ton bureau, PipeWire ou PulseAudio ne pourra plus l'utiliser.

## 8. Démarrer Rivendell

```sh
sudo systemctl enable --now apache2
sudo systemctl enable --now rivendell
```

## 9. Vérifier

```sh
systemctl is-active rivendell apache2
pgrep -a 'jackd|caed|ripcd|rdcatchd'

# Ports et connexions JACK (même variable que le service)
JACK_PROMISCUOUS_SERVER=audio jack_lsp -c | grep -A1 playout_0
```

Pour tester le widget de messages de rdairplay avec QtWebEngine, lance
`rdairplay`, puis :

```sh
rmlsend 'LM https://example.org/!'
```

Pour revenir à un message texte : `rmlsend 'LB Bonjour!'` (ou `LC <couleur> <texte>!` pour un texte en couleur). Attention, l'URL
chargée avec `LM` est mémorisée et rechargée au prochain démarrage de
rdairplay.

## Mettre à jour depuis le dépôt officiel

```sh
git remote add upstream https://github.com/ElvishArtisan/rivendell.git   # une seule fois
git fetch upstream
git switch trixie-prs
git merge upstream/v4
```

Si les PR #1019 et #1013 sont intégrées au dépôt officiel, la branche `v4`
officielle suffira, et ce fork n'aura plus de raison d'être.

## Licence

Rivendell est distribué sous licence **GNU GPL v2** (voir
`LICENSES/GPLv2.txt`), © Fred Gleason / Paravel Systems et les contributeurs.
La documentation d'origine est dans `README` et `INSTALL`.
