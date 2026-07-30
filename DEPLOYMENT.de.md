# Deployment mit Docker Compose

Die Anwendung speichert Datenbank, Konfiguration und automatische SQLite-Sicherungen im Verzeichnis `./data`. Dieses Verzeichnis darf bei Updates nicht gelöscht werden.

## Voraussetzungen

- Linux-Server mit Docker Engine und Docker Compose Plugin
- DNS/Reverse-Proxy auf den Server-Port `7973`, falls die Anwendung öffentlich erreichbar sein soll
- Repository auf dem Server, zum Beispiel unter `/opt/stremio-rss-catalog`

Einmalig vorbereiten:

```bash
git clone https://github.com/Aerya/stremio-rss-catalog.git /opt/stremio-rss-catalog
cd /opt/stremio-rss-catalog
mkdir -p data
cp .env.example .env
nano .env
```

In `.env` mindestens `WEBUI_PASSWORD` ändern und ein starkes Geheimnis erzeugen:

```bash
openssl rand -hex 32
```

## Variante A: Direkt auf dem Server bauen

```bash
cd /opt/stremio-rss-catalog
git pull --ff-only
docker compose -f docker-compose.build.yml up -d --build
docker compose -f docker-compose.build.yml ps
```

Die WebUI läuft lokal unter `http://SERVER-IP:7973`. Der Manifest-Endpunkt bleibt:

```text
http://SERVER-IP:7973/manifest.json
```

Hinter einem Reverse-Proxy wird die externe HTTPS-URL in Stremio verwendet.

## Variante B: Eigenes Registry-Image

Lokal oder in einer CI-Umgebung anmelden und Image bauen:

```bash
docker login ghcr.io
docker build -t ghcr.io/DEIN-ACCOUNT/stremio-rss-catalog:1.0.0 .
docker push ghcr.io/DEIN-ACCOUNT/stremio-rss-catalog:1.0.0
```

Auf dem Server `.env` um die Image-Werte ergänzen:

```dotenv
REGISTRY_IMAGE=ghcr.io/nexis-cmyk/stremio-rss-catalog
IMAGE_TAG=1.0.0
```

Danach starten oder aktualisieren:

```bash
cd /opt/stremio-rss-catalog
docker login ghcr.io
docker compose -f docker-compose.registry.yml pull
docker compose -f docker-compose.registry.yml up -d
docker compose -f docker-compose.registry.yml ps
```

## Betrieb und Updates

Logs ansehen:

```bash
docker compose -f docker-compose.build.yml logs -f --tail=200
```

Container stoppen oder starten:

```bash
docker compose -f docker-compose.build.yml down
docker compose -f docker-compose.build.yml up -d
```

Vor einem Update den persistenten Datenordner sichern:

```bash
tar -czf "stremio-rss-data-$(date +%F-%H%M).tar.gz" data
```

Die Migration auf das neue Sortierschema erzeugt zusätzlich automatisch eine SQLite-Sicherung unter `data/backups/`. Bei Problemen kann der vorherige Image-Tag gestartet und die Datenbanksicherung zurückgespielt werden.

Nach dem Update im WebUI-Bereich **Kataloge** den gewünschten Sortiermodus auswählen. Standardmäßig wird das RSS-Datum absteigend verwendet; bei mehreren Releases desselben Films zählt deren jeweils neuestes Veröffentlichungsdatum.
