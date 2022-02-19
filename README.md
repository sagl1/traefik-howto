# HowTo: SmarthomeNG mit Traefik

## Beschreibung

Dieses HowTo zeigt, wie man den Reverse Proxy [Traefik](https://doc.traefik.io/traefik/) für [SmarthomeNG](https://www.smarthomeng.de/) und [smartVISU](https://www.smartvisu.de/) konfiguriert.

## **Warnung**

Server ins Internet zu stellen ist immer ein Sicherheitsrisiko. Die verwendete Software muss immer aktuell gehalten werden um bekannte Sicherheitslücken zu schließen. Online Scanner \(z.B. [SSL Labs](https://www.ssllabs.com/ssltest/)\) können helfen mögliche Konfigurationsfehler zu erkennen bzw. Sicherheitshinweise liefern, die über dieses HowTo hinausgehen.

## Voraussetzungen

1. Ein Rechner bzw. Server (`<DockerHOST>`) auf dem Docker und Docker-Compose laufen.
2. `<DockerHOST>` muss aus dem Internet auf Port 443 erreichbar sein. Vorgeschaltete Router oder Firewalls sind entsprechend konfiguriert.
3. Eine Domain (`<your.domain.tld>`) die auf `<DockerHOST>` zeigt. Grundsätzlich sind hier auch dynamische Domains \(z.B. von DynDNS\) möglich, aber das kann zu Problemen bei der Zertifikatsbeschaffung führen.
4. Eine gültige E-Mail Adresse (`<your.email.tld>`).
5. Grundverständnis für docker-compose und YAML Syntax setze ich vorraus.

## Vorbereitungen

Erst einmal erstellst du die Verzeichnisstruktur:
```
mkdir workdir
cd workdir
mkdir traefik
touch traefik/docker-compose.yaml
mkdir shng
touch shng/docker-compose.yaml
```

`workdir` ist ein beliebiges Arbeitsverzeichnis. Die Verzeichnisse `traefik` und `shng` dienen jeweils als separate docker-compose Konfigurationen für SmarthomeNG und Traefik. Ich empfehle die Konfigurationen zu trennen, falls man Traefik später auch noch für andere Dinge nutzen möchte.
Um Traefik mit SmarthomeNG zu verbinden nutzen wir das selbsterzeugte Netzwerk `traefik-net`. Das erzeugen wir mit dem Befehl:
```
docker network create -d bridge traefik-net
```

## Die Konfig

### Die Traefik Konfiguration

Zuerst öffnest du die Datei `traefik/docker-compose.yaml` mit einem Editor deiner Wahl und kopierst folgenden Inhalt hinein:
```
version: "3"

networks:
  traefik-net:
    external: true

services:
  traefik:
    image: "traefik:v2.6"
    command:
      - "--log.level=ERROR"
      #- "--log.level=DEBUG"
      
      # provider "docker"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-net"
      
      # entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      #- "--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      
      # lets encrypt
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=<your@email.tld>"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      
      # Traefik dashboard "http://<yourhost>:8080/"
      - "--api.insecure=true"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    networks:
      - traefik-net
    volumes:
      - ./volumes/letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

> `<your@email.tld>` muss durch eine gültige E-Mail Adresse ersetzt werden.

Ziemlich viel, oder? Die Traefik Konfig erkläre ich gleich noch mal Schritt für Schritt. 

```
networks:
  traefik-net:
    external: true
```
Das Netzwerk `traefik-net` ist als externes Netzwerk definiert. Es wird also beim starten und stoppen des Containers nicht erstellt bzw. zerstört, sondern muss schon vorhanden sein. Hatten wir ja als Vorbereitung schon angelegt.

```
services:
  traefik:
    image: "traefik:v2.6"
```
Hier wird der Service `traefik` definiert. Als Basis dient das Image `traefik` in der Version `v2.6` welches vom [DockerHub](https://hub.docker.com/) geladen wird. Häufig wird für die Version (bzw. tag) `latest` verwendet. Für Traefik empfehle ich es nicht, da neuere Traefik Versionen oft inkompatible Änderungen aufweisen.

```
    command:
```
Dieser Teil behandelt die statische Traefik Konfiguration.

```
      - "--log.level=ERROR"
      #- "--log.level=DEBUG"
```
Zur Fehlersuche kann man das Loglevel auf `DEBUG` setzen. Später sollte man es wieder zurücksetzen um Log Dateien nicht zu groß werden zu lassen.

```
      # provider "docker"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-net"
```
1. Die dynamsiche Traefik Konfiguration wird später aus *Labels* der Container bezogen.
2. Es muss explizit für jeden Container aktiviert werden.
3. Die relevanten Container sind am `traefik-net` angebunden.

```
      # entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"

```
1. Erster Entrypoint ist Port *80* und bekommt den Namen `web`.
2. Sämtliche Verbindungen werden vom Entrypoint `web` auf den Entrypoint `websecure` umgeleitet.
3. Dabei wird das Schema `https` verwendet. Somit werden aus *http* Verbindungen - *https* Verbindungen.
4. Der Port für den Entrypoint Namens `websecure` ist *443*.
Statt `web`und `websecure` könnte man auch `http` bzw. `https` als Namen wählen. Ich finde es aber aufgrund der Namensgleicheit zum *scheme* `https` gerade für Anfänger irreführend. Ausserdem behandeln wir ja später auch *ws* bzw. *wss* Verbindungen für den Websocket damit.

```
      # lets encrypt
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
      #- "--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      - "--certificatesresolvers.letsencrypt.acme.email=<your@email.tld>"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
```
Die Let's Encrypt Konfiguration.

Da Let's Encrypt [Rate Limits](https://letsencrypt.org/de/docs/rate-limits/) hat, empfielt es sich die Konfig erst mal auszuprobieren. Durch aktivieren der `caserver` Zeile leitet man die Zertifikat Anfragen an den Staging Server um und kann erst mal in Ruhe testen. Die gelieferten Zertifikate werden dann aber als ungültig ausgewiesen. An den Zertfifikatdetails kann man aber erkennen ob alles funktioniert. Für den regulären Betrieb muss die Zeile natürlich wieder auskommentiert werden.
Ich habe mich für die *TLS Challenge* entschieden, da hierdurch die Kommunikation mit Let's Encrypt ebenfalls verschlüsselt abläuft und ich nur den Port 443 ins Netz öffnen muss. Darüber hinaus gibt es noch die *http Challenge* und die *dns Challenge*.


```
      # Traefik dashboard "http://<Dockerhost>:8080/"
      - "--api.insecure=true"
```
Dies aktiviert das Traefik Dashboard. Dort kann man die aktive Konfiguration bzw. den Status der Services einsehen.

***

### Die SmarthomeNG Konfiguration

Als nächstes öffnest du die Datei `shng/docker-compose.yaml` mit einem Editor deiner Wahl und kopierst folgenden Inhalt hinein:
```
version: "3"

#https://www.smarthomeng.de/
#https://www.smartvisu.de/

networks:
  traefik-net:
    external: true

services:
  shng:
    image: sagl/shng:visu
    restart: "unless-stopped"
    environment:
      - TZ=Europe/Berlin
    labels:
      - "traefik.enable=true"
      
      # Websocket "wss://<your.domain.tld>/"
      - "traefik.http.routers.sh.entrypoints=websecure"
      - "traefik.http.routers.sh.rule=Host(`<your.domain.tld>`) && Path(`/`)"
      - "traefik.http.routers.sh.tls=true"
      - "traefik.http.routers.sh.tls.certresolver=letsencrypt"
      - "traefik.http.routers.sh.middlewares=shauth"
      - "traefik.http.routers.sh.service=sh"
      - "traefik.http.services.sh.loadbalancer.server.port=2424"
      
      # admin interface "https://<your.domain.tld>/admin/"
      - "traefik.http.routers.shadmin.entrypoints=websecure"
      - "traefik.http.routers.shadmin.rule=Host(`<your.domain.tld>`)"
      - "traefik.http.routers.shadmin.tls=true"
      - "traefik.http.routers.shadmin.tls.certresolver=letsencrypt"
      - "traefik.http.routers.shadmin.middlewares=shauth"
      - "traefik.http.routers.shadmin.service=shadmin"
      - "traefik.http.services.shadmin.loadbalancer.server.port=8383"
      
      # digestauth Example: user="test", pw="test" <- unbedingt ändern!
      # HINT: Generate Hash with echo -n "USER:REALM:PASSWORD" | md5sum
      - "traefik.http.middlewares.shauth.digestauth.users=test:SHNG:2eb7c3ae89f6812c8008aef4df7dc364"
      - "traefik.http.middlewares.shauth.digestauth.realm=SHNG"
      - "traefik.http.middlewares.shauth.digestauth.removeheader=true"
    volumes:
      - ./volumes/shng:/mnt
    networks:
      - traefik-net

  smartvisu:
    image: php:8.0-apache
    restart: unless-stopped
    #hostname: <your.domain.tld>
    depends_on:
      - shng
    labels:
      - "traefik.enable=true"
      
      # smartVISU web interface "https://<your.domain.tld>/smartvisu/"
      - "traefik.http.routers.sv.entrypoints=websecure"
      #- "traefik.http.routers.sv.rule=Host(`<your.domain.tld>`) && PathPrefix(`/smartvisu`)"
      - "traefik.http.routers.sv.rule=Host(`sh.thi0n.de`) && PathPrefix(`/smartvisu`)"
      - "traefik.http.routers.sv.tls=true"
      - "traefik.http.routers.sv.tls.certresolver=letsencrypt"
      - "traefik.http.routers.sv.middlewares=shauth"
      - "traefik.http.services.sv.loadbalancer.server.port=80"
    volumes:
      - ./volumes/html:/var/www/html
    networks:
      - traefik-net
```

> `<your.domain.tld>` muss durch die eigene Domain ersetzt werden.

Und noch einmal im Detail:
```
      # Websocket "wss://<your.domain.tld>/"
      - "traefik.http.routers.sh.entrypoints=websecure"
      - "traefik.http.routers.sh.rule=Host(`<your.domain.tld>`) && Path(`/`)"
      - "traefik.http.routers.sh.tls=true"
      - "traefik.http.routers.sh.tls.certresolver=letsencrypt"
      - "traefik.http.routers.sh.middlewares=shauth"
      - "traefik.http.routers.sh.service=sh"
      - "traefik.http.services.sh.loadbalancer.server.port=2424"
```
1. Zuerst wird der Entrypoint festgelegt. Der Websocket hört also nur auf Port *443*.
2. Danach wird der Host inklusive Pfad definiert. Alle Anfragen an das Stammverzeichnis (keine Sub-Pfade) der Domain `<your.domain.tld>` werden an diesen Container weitergeleitet.
3. Verschlüsselung wird aktiviert.
4. Als Zertifikatprovider wird letsencrypt ausgewählt.
5. Die *middlewares* Zeile aktiviert die Passwortabfrage die unter `shauth` definiert ist. Dazu später mehr.
`routers.sh.service`verweist auf den zuständigen service *sh*. Dies ist hier notwendig weil 2 services existerien.
6. Der *service* `sh` zeigt auf den zuständigen Port innerhalb des Docker Images.  *2424* ist der Websocket Port. Diese Angabe ist immer dann notwendig, wenn ein Container mehr als einen Port freigibt.
 > Hinweis: `sh` ist ein frei definierter Name des Routers und des Service. Über die Namensgleicheit wird keine Beziehung zwischen Router und Service hergestellt. Die Namen könnten auch völlig unterschiedlich gewählt werden. Die beziehung zwischen beiden stellt die Zeile `traefik.http.routers.sh.service=sh` her.

```
      # admin interface "https://<your.domain.tld>/admin/"
      - "traefik.http.routers.shadmin.entrypoints=websecure"
      - "traefik.http.routers.shadmin.rule=Host(`<your.domain.tld>`)"
      - "traefik.http.routers.shadmin.tls=true"
      - "traefik.http.routers.shadmin.tls.certresolver=letsencrypt"
      - "traefik.http.routers.shadmin.middlewares=shauth"
      - "traefik.http.routers.shadmin.service=shadmin"
      - "traefik.http.services.shadmin.loadbalancer.server.port=8383"
```
Dieser Abschnitt ist grundsätzlich vergleichbar zum vorherigen. Mit 2 wesentlichen Unterschieden. 
 * In der *rule* Zeile fehlt die *Path* Angabe. Hintergrund ist folgender: Traefik berechnet die Standardpriorität aus der Stringlänge Host & Path. Die Annahme dahinter: Je länger die Angabe desto spezifischer. Die Angabe von ``traefik.http.routers.sh.rule=Host(`<your.domain.tld>`) && Path(`/`)`` bedeutet alles was ins Stammverzeichnis der Domain geht gehört zum Websocket. Alles andere geht an ``traefik.http.routers.shadmin.rule=Host(`<your.domain.tld>`)``.
 * Der *service* `shadmin` zeigt auf Port *8383* - der Port des Admin Interfaces.

```
      # digestauth Example: user="test", pw="test" <- unbedingt ändern!
      # HINT: Generate Hash with echo -n "USER:REALM:PASSWORD" | md5sum
      - "traefik.http.middlewares.shauth.digestauth.users=test:SHNG:2eb7c3ae89f6812c8008aef4df7dc364"
      - "traefik.http.middlewares.shauth.digestauth.realm=SHNG"
      - "traefik.http.middlewares.shauth.digestauth.removeheader=true"
```
Die *middleware* `shauth` legt den Passwortschutz fest. Ich habe mich für *digestauth* entschieden, weil hier grundsätzlich eine Verschlüsselung des Passwortes erfolgt auch wenn die Verbindung nicht verschlüsselt ist. Da wir hier (wenn alles funktioniert) grundsätzlich verschlüsselt via *https* kommunizieren würde auch ein *basicauth* reichen. Für höherwertige Anmeldeverfahren bietet Traefik *forwardauth* welches entsprechende Dienste einbindet.
1. Hier wird der User `test` mit Passwort `test` festgelegt. Mehrere User werden durch Komma getrennt.
2. Der *realm*, hier `SHNG`, muss mit der Angabe beim *user* übereinstimmen.
3. *removeheader* entfehrnt die auth Header bevor die Anfrage an den Service geleitet wird. Ohne diese Angabe gab es Probleme mit dem Admin Interface.

```
      # smartVISU web interface "https://<your.domain.tld>/smartvisu/"
      - "traefik.http.routers.sv.entrypoints=websecure"
      #- "traefik.http.routers.sv.rule=Host(`<your.domain.tld>`) && PathPrefix(`/smartvisu`)"
      - "traefik.http.routers.sv.rule=Host(`sh.thi0n.de`) && PathPrefix(`/smartvisu`)"
      - "traefik.http.routers.sv.tls=true"
      - "traefik.http.routers.sv.tls.certresolver=letsencrypt"
      - "traefik.http.routers.sv.middlewares=shauth"
      - "traefik.http.services.sv.loadbalancer.server.port=80"
```
Zuletzt der smartVISU Part. Da der String *Host & Path* noch mal länger ist als die entsprechenden Definitionen für das Admin Interface und den Websocket hat dieser die höchste Priorität. *Pathprefix* gilt für den Subpath `smartvisu` und alle darunter. Im Gegensatz zu *Path* welches sich nur auf den definierten Pfad und nicht auf Unterpfade bezieht.
 > Beachte: Die Zeile `traefik.http.routers.sv.service=sv` wäre nicht verkehrt, ist aber nicht notwendig, da für diesen Container nur ein service definiert ist. Da der php Container nur den Port 80 freigibt ist die Zeile `traefik.http.services.sv.loadbalancer.server.port=80` eigentlich auch nicht notwendig.

## ...und los

Jetzt startest du die Container.
 * Ins Verzeichnis `traefik` wechseln und `docker-compose up -d` ausführen.
 * Ins Verzeichnis `shng` wechseln und `docker-compose up -d` ausführen.

Viel Erfolg
Sascha
