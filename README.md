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

Pour le streaming Icecast et les métadonnées (sections 10 et 11), ajoute
`glasscoder` et `python3-mysqldb`.

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

## 10. Diffuser vers Icecast avec GlassCoder

[GlassCoder](https://github.com/ElvishArtisan/GlassCoder) encode un flux JACK
et l'envoie à un serveur Icecast. Le paquet Debian suffit :

```sh
sudo apt install glasscoder
```

Le principe : `caed` lance GlassCoder comme un client JACK de plus, dans le même
serveur jackd que Rivendell, et on relie `playout_0` à ses entrées. Le flux
démarre et s'arrête donc avec le service `rivendell`.

### a) Identifiants Icecast

Crée `/etc/glasscoder/webradio.cred`, lisible par root seulement :

```ini
[Credentials]
Username=source
Password=MOT_DE_PASSE_SOURCE
```

```sh
sudo install -d -m 700 /etc/glasscoder
sudo chmod 600 /etc/glasscoder/webradio.cred
```

### b) Script de lancement

Comme `caed` n'utilise pas de shell, un petit script lance GlassCoder, connecte
ses ports dès qu'ils apparaissent, et le relance s'il s'arrête. Enregistre-le
dans `/usr/local/bin/rd-glasscoder` (`sudo chmod 755`), puis adapte l'URL du
serveur, le format et le nom du flux :

```bash
#!/bin/bash
# Diffuse playout_0 de Rivendell vers Icecast avec glasscoder(1).
CLIENT=glasscoder
SOURCE_L=rivendell_0:playout_0L
SOURCE_R=rivendell_0:playout_0R

export JACK_PROMISCUOUS_SERVER=${JACK_PROMISCUOUS_SERVER:-audio}

GC_PID=
trap '[ -n "$GC_PID" ] && kill "$GC_PID" 2>/dev/null; exit 0' TERM INT

connect_ports() {
  for i in $(seq 1 30); do
    if jack_lsp 2>/dev/null | grep -qx "$CLIENT:input_2"; then
      jack_connect "$SOURCE_L" "$CLIENT:input_1" 2>/dev/null
      jack_connect "$SOURCE_R" "$CLIENT:input_2" 2>/dev/null
      logger -t rd-glasscoder "ports JACK connectés ($SOURCE_L/R -> $CLIENT)"
      return 0
    fi
    sleep 1
  done
  logger -t rd-glasscoder "ports $CLIENT:input_* introuvables après 30 s"
  return 1
}

while :; do
  /usr/bin/glasscoder \
    --audio-device=jack \
    --jack-client-name="$CLIENT" \
    --audio-format=mp3 \
    --audio-bitrate=256 \
    --audio-samplerate=44100 \
    --audio-channels=2 \
    --server-type=icecast2 \
    --server-url=http://icecast.example.org:8000/webradio \
    --credentials-file=/etc/glasscoder/webradio.cred \
    --stream-name="Ma radio" \
    --errors-to=syslog \
    --errors-string=webradio &
  GC_PID=$!
  connect_ports
  wait "$GC_PID"
  logger -t rd-glasscoder "glasscoder s'est arrêté (code $?), relance dans 5 s"
  GC_PID=
  sleep 5
done
```

GlassCoder rééchantillonne lui-même de 48 kHz (JACK) vers la fréquence
demandée avec `--audio-samplerate`.

### c) Le faire démarrer par Rivendell

Dans rdadmin, va dans Manage Hosts → *ton hôte* → **JACK Settings**, puis
ajoute `/usr/local/bin/rd-glasscoder` à **JACK Clients to Start**, et
redémarre le service :

```sh
sudo systemctl restart rivendell
```

### d) Vérifier

```sh
pgrep -a glasscoder
journalctl -t rd-glasscoder -t glasscoder --since "-5min"
JACK_PROMISCUOUS_SERVER=audio jack_lsp -c | grep -A2 playout_0
```

Chaque port `playout_0L/R` doit aller vers `system:playback_*` et vers
`glasscoder:input_*`. Le point de montage doit aussi apparaître sur la page
d'état d'Icecast (`/status.xsl`).

## 11. Envoyer le titre en cours à Icecast avec PyPAD

PyPAD est le système de métadonnées (PAD) de Rivendell. Le script
`pypad_icecast2.py`, fourni avec Rivendell, met à jour le titre d'un point de
montage Icecast à chaque nouveau morceau. Il a besoin de :

```sh
sudo apt install python3-mysqldb
```

Dans rdadmin, va dans Manage Hosts → *ton hôte* → **PyPAD Instances** →
**Add**, puis choisis
`/usr/lib/rivendell/pypad/pypad_icecast2.py`. La configuration proposée vient
de `pypad_icecast2.exemplar`. Adapte au minimum ces lignes de la section
`[Icecast1]` :

```ini
[Icecast1]
Username=source
Password=MOT_DE_PASSE_SOURCE
Hostname=icecast.example.org
Tcpport=8000
Mountpoint=/webradio
FormatString=%a - %t
MasterLog=Yes
Aux1Log=No
Aux2Log=No
```

`%a` est l'artiste et `%t` le titre. Les autres codes sont décrits dans
l'exemplar. `MasterLog=Yes` envoie les titres du log principal de rdairplay.
Le compte doit pouvoir accéder à `/admin/metadata` d'Icecast : c'est le cas du
compte source du point de montage.

L'instance démarre tout de suite. Pour vérifier, lance un titre dans rdairplay,
puis :

```sh
journalctl -u rivendell --since "-5min" | grep pypad_icecast2
# ... Updating http://icecast.example.org/webradio: song=Artiste - Titre
```

En cas d'erreur, rdadmin affiche le code de sortie et l'erreur de l'instance
(bouton **Error Log**). `No module named 'MySQLdb'` signifie qu'il manque
`python3-mysqldb`.

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
