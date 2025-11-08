# -- HINWEIS: Bitte beachten dass dieses Repo nur für Bildungszewecke gedacht ist, die credentials sind fake und daher nicht zu deployen! --   

# Cyber Tools Project – Stack: Pomerium + Keycloak + Grafana + Loki + Alloy

Dieses Repository startet eine lokale Demo-Umgebung, in der eine Grafana-Instanz über Pomerium (Zero‑Trust Proxy) abgesichert und über Keycloak (OIDC) authentifiziert wird. Logs der Container werden von Grafana Alloy gesammelt und an Loki gesendet. So kann in Grafana (nach Login) zentral in Logs gesucht werden.

## Dienste & Architektur
- Pomerium (Policy Enforcement Point)
  - Lauscht lokal auf Port 443 (HTTPS)
  - Terminates TLS und schützt Grafana per OIDC (Keycloak)
  - Reicht Claims/Identity-Header inkl. Rollen an Grafana weiter
- Keycloak + Postgres (Identity Provider)
  - Bereitstellung eines Realms und eines OIDC‑Clients für Grafana
  - Admin-Konsole unter http://localhost:8080
- Grafana (Demo-Applikation, via Pomerium geschützt)
  - Erhält das signierte JWT per Header `X-Pomerium-Jwt-Assertion`
  - Rollenzuordnung über Claim `roles` (Admin/Editor/Viewer)
- Loki (Logspeicher)
  - Nimmt Logs von Alloy entgegen, erreichbar unter http://localhost:3100
- Grafana Alloy (Log-Sammlung)
  - Liest Docker-Logs, reichert sie an und schreibt sie nach Loki

## Voraussetzungen
- Docker und Docker Compose installiert
- Freie lokale Ports: 443, 8080

## Schnellstart
1. Repository klonen und ins Projektverzeichnis wechseln
2. Dienste hochfahren:
   ```bash
   docker compose up
   ```
3. Warten, bis alle Container laufen. Wichtige URLs:
   - Grafana (über Pomerium): https://grafana.localhost.pomerium.io
   - Pomerium Authenticate:   https://authenticate.localhost.pomerium.io
   - Keycloak Admin Console:   http://localhost:8080

> Hinweis: Von den folgenden schritten, müsste eigentlich nur Keycloak konfigurieren ausgeführt werden und 
> das idp_client_secret in der pomerium-config.yaml setzen. Die resetlichen Schritte sind eigentlich schon erledigt. 


## Keycloak konfigurieren
Die Pomerium‑Konfiguration (`config/pomerium-config.yaml`) erwartet folgende Werte:
- Realm: `grafana_realm`
- Client ID: `grafana_client`
- Redirect URIs:
  - `https://grafana.localhost.pomerium.io/login/generic_oauth`
  - `https://authenticate.localhost.pomerium.io/oauth2/callback`

### Schritte in Keycloak
1. Keycloak aufrufen: http://localhost:8080 → Admin-Login
   - Standard-Admin: `admin` / `admin` (aus `docker-compose.yml`), bitte anschließend ändern
2. Realm anlegen
   - Name: `grafana_realm`
   - Enabled: `ON`
3. Client anlegen (im Realm `grafana_realm`)
   - Client ID: `grafana_client`
   - Client Authentication: `ON`
   - Standard Flow: `ON`
   - Direct Access Grants: `ON`
   - Valid Redirect URIs ergänzen (siehe oben)
4. Rollen im Client `grafana_client` erstellen
   - `admin`, `editor`, `viewer`
5. Client Scope `roles` erstellen und als „Default“ markieren
   - In `Client Scopes` → `roles` → Mappers:
     - Mapper-Typ: `User Client Role`
     - Token Claim Name: `roles`
     - Add to ID token: `ON`
     - Add to access token: `ON`
     - Add to userinfo: `ON`
6. Benutzer anlegen und zuordnen
   - `admin`  → Client-Rolle: `admin`,  E-Mail: `admin@interne-firma.xyz`
   - `editor` → Client-Rolle: `editor`, E-Mail: `editor@interne-firma.xyz`
   - `viewer` → Client-Rolle: `viewer`, E-Mail: `viewer@interne-firma.xyz`
   - `viewer-extern` → Client-Rolle: `viewer`, E-Mail: `viewer@externe-firma.xyz`

> Wichtig: Die Domain im E‑Mail‑Claim steuert Pomeriums Domain-basierte Policies (siehe `config/pomerium-config.yaml`). Für den Admin‑Bereich ist `interne-firma.xyz` erforderlich.

## Pomerium – relevante Konfiguration
Datei: `config/pomerium-config.yaml`
- `authenticate_service_url`: `https://authenticate.localhost.pomerium.io`
- `idp_provider`: `oidc`
- `idp_provider_url`: `http://keycloak.localhost.pomerium.io:8080/realms/grafana_realm`
- `idp_client_id`: `grafana_client`
- `idp_client_secret`: aus Keycloak kopieren (muss zum Client passen)
- `signing_key`: private Key (EC), zugehöriger Public Key liegt als JWKS in Grafana (`config/jwks.json`)
- Forwarded Identity Header inkl. Mapping:
  - `X-User-Email: email`, `X-User-Name: name`, `X-User-Roles: roles`, `X-User-Groups: groups`
- Routen:
  - `https://grafana.localhost.pomerium.io` → `http://grafana:3000`
    - Pfad `/admin`: Zulässig nur für Domain `interne-firma.xyz`
    - Allgemein: Zulässig für `interne-firma.xyz` ODER `externe-firma.xyz`

## Grafana – relevante Konfiguration
Datei: `config/grafana.ini`
- Server
  - `root_url = https://grafana.localhost.pomerium.io`
  - `serve_from_sub_path = true`
- Auth
  - `oauth_auto_login = true`
  - `disable_login_form = true`
  - `signout_redirect_url = https://grafana.localhost.pomerium.io/.pomerium/sign_out`
- JWT
  - `enabled = true`
  - `header_name = X-Pomerium-Jwt-Assertion`
  - `jwk_set_file = /etc/grafana/jwks/jwks.json` (aus `config/jwks.json` gemountet)
  - `email_claim = sub`, `username_claim = sub`
  - `role_attribute_path = contains(roles, 'admin') && 'Admin' || contains(roles, 'editor') && 'Editor' || 'Viewer'`
  - `allow_assign_grafana_admin = true`

## Loki & Alloy
- Loki-Konfiguration: `config/loki-config.yml`
  - Lauscht auf `3100` und speichert lokal unter `./.loki-data`
- Alloy-Konfiguration: `config/alloy-config.alloy`
  - Entdeckt Docker-Container, parst JSON‑Felder, labelt Streams und pusht nach Loki (`http://loki:3100/loki/api/v1/push`)

### Loki als Datenquelle in Grafana hinzufügen
1. In Grafana (nach Login via Pomerium) → „Connections“ → „Add data source“ → „Loki“
2. URL: `http://loki:3100`
3. Speichern → „Explore“ öffnen und Logs abfragen, z. B.:
   ```
   {service="pomerium"}
   ```
   oder Container-Label `container` verwenden, z. B. `container="/pomerium-1"` (je nach Compose‑Namen).

## Standard-URLs & Ports
- Keycloak: http://localhost:8080 (Wird das genutzt um Keycloak initial zu konfigurieren)
- Pomerium (Proxy): https://grafana.localhost.pomerium.io (443)
- Pomerium Authenticate: https://authenticate.localhost.pomerium.io (443)

## Volumes & Persistenz
- Postgres: benannter Volume `keycloak_postgres_data`
- Grafana: benannter Volume `grafana_data`
- Loki: lokales Verzeichnis `./.loki-data` (Wird zur laufzeit erstellt)

## Häufige Probleme & Tipps
- Zertifikatswarnung: Bei erstem Zugriff auf Pomerium die Browser‑Warnung akzeptieren (selbstsigniertes Zertifikat).
- 403/401 hinter Pomerium: Prüfen, ob Benutzer die richtige E‑Mail‑Domain hat und die Rolle im Client `grafana_client` zugewiesen ist.
- Falsche Redirect URI: In Keycloak exakt die beiden URIs hinterlegen (Groß/Kleinschreibung, HTTPS beachten).
- `invalid_client` oder `unauthorized_client`: `idp_client_id`/`idp_client_secret` zwischen Pomerium und Keycloak abgleichen.
- Kein Login in Grafana möglich: `jwks.json` muss zum `signing_key` in Pomerium passen; Zeit/Zeitzonen‑Drift (<5 min) vermeiden.
- Loki leer: Prüfen, ob Alloy läuft und Docker‑Socket gemountet ist. In Compose ist `/var/run/docker.sock` vorhanden.
- Ports belegt: Sicherstellen, dass 443/8080 frei sind oder Port‑Mapping anpassen.
- idp_client_secret in der `config/pomerium-config.yaml` inkorrekt

## Stoppen & Aufräumen
- Container stoppen: `docker compose down`
- Mit Volumes: `docker compose down -v` (löscht Persistenz – Vorsicht)

## Dateien im Überblick
- `docker-compose.yml`: Startet alle Dienste und Volumes
- `config/pomerium-config.yaml`: Pomerium‑Routen, OIDC‑Integration, Signing Key
- `config/grafana.ini`: Grafana‑Server‑, Auth‑ und JWT‑Einstellungen
- `config/jwks.json`: Öffentlicher Schlüssel (JWKS) für Grafana zur Verifikation des Pomerium‑JWT
- `config/loki-config.yml`: Loki‑Speicher/Schema/Server
- `config/alloy-config.alloy`: Alloy‑Pipeline, Discovery und Write‑Ziel

Viel Erfolg mit der Demo-Umgebung!