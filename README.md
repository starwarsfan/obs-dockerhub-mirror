# OBS Docker Hub Mirror

Spiegelt Docker-Images von [ghcr.io/abeggled/openbridgeserver](https://github.com/abeggled/openbridgeserver/pkgs/container/openbridgeserver) 
automatisch nach [Docker Hub](https://hub.docker.com/r/starwarsfan/openbridgeserver).

## Hintergrund

Einige Nutzer haben Schwierigkeiten, Images von GHCR zu beziehen (z.B. wegen Authentifizierungs-Anforderungen oder Netzwerk-/Firewall-Restriktionen).
Dieses Repo stellt dieselben Images zusätzlich auf Docker Hub bereit, ohne dass am OBS-Build-Prozess selbst etwas geändert werden muss.

## Funktionsweise

Der Workflow [`mirror-to-dockerhub.yml`](.github/workflows/mirror-to-dockerhub.yml):

1. Läuft täglich per Cron (`0 3 * * *` UTC, 50 Min. nach Start des Nightly-Builds) sowie manuell per `workflow_dispatch`.
2. Ermittelt via `skopeo list-tags` alle aktuell auf GHCR vorhandenen Tags.
3. Filtert daraus:
   - `latest` und `nightly`
   - alle Release-Tags (Pattern `YYYY.M` bzw. `YYYY.M.P`)
   - alle Release-Candidate-Tags (Pattern `YYYY.M.P-RCx`)
   - die letzten `NIGHTLY_RETENTION` datierten Nightly-Tags (Pattern `nightly-YYYYMMDD`)
4. Kopiert die gefilterten Tags per `skopeo copy --all` (inkl. Multi-Arch-Manifest) 1:1 nach Docker Hub.
5. Löscht über die Docker Hub REST API alle datierten Nightly-Tags (`nightly-YYYYMMDD`), die nicht mehr in der Retention-Liste enthalten sind.

Reine Commit-Hash-Tags (z.B. `c08404b`) sowie Hash-Nightlies (z.B. `nightly-79df238`) werden bewusst **nicht** gemirrort, da sie inhaltlich identisch zu
einem bereits gemirrorten Versions- bzw. datierten Nightly-Tag sind.

Der Kopiervorgang ist idempotent: bereits vorhandene, unveränderte Tags werden beim nächsten Lauf einfach erneut überschrieben, ohne dass das ein Problem darstellt.

## Setup

### 1. Secrets

Im Repo unter **Settings → Secrets and variables → Actions** folgende Secrets anlegen:

| Secret | Beschreibung |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub Benutzername (`starwarsfan`) |
| `DOCKERHUB_TOKEN` | Docker Hub Access Token mit Read & Write Berechtigung ([Account Settings → Security → New Access Token](https://hub.docker.com/settings/security)) |

### 2. Konfiguration anpassen

Am Kopf der Workflow-Datei lassen sich folgende Werte anpassen:

```yaml
env:
  SRC_REPO: ghcr.io/abeggled/openbridgeserver
  DST_REPO: docker.io/starwarsfan/openbridgeserver
  NIGHTLY_RETENTION: 5
```

### 3. Manuell testen

Repo → **Actions** → *Mirror OBS images to Docker Hub* → **Run workflow**. Optional kann dabei ein einzelner zusätzlicher Tag (`extra_tag`) angegeben werden, der zusätzlich zur automatisch ermittelten Liste gemirrort wird.

## Bekannte Einschränkungen

- **Cleanup nur für datierte Nightlies**: Release- und RC-Tags, die auf GHCR gelöscht werden, bleiben auf Docker Hub bestehen. Schlägt die Anmeldung an der Docker Hub REST API fehl, wird der Cleanup übersprungen, ohne dass der Workflow fehlschlägt.
- **Anonymer GHCR-Zugriff**: Das Quell-Package wird ohne Login gelesen und muss daher öffentlich sein. Wird es privat, ist wieder ein `skopeo login ghcr.io` mit einem PAT (`read:packages`) nötig.
- **Docker Hub Rate Limits**: Bei sehr vielen Tags oder häufigen Läufen können abhängig vom Docker-Hub-Plan Rate Limits greifen.

## Manuelles Mirrorn (lokal)

Für einen initialen Abgleich oder Debugging kann derselbe Vorgang auch lokal mit [skopeo](https://github.com/containers/skopeo) ausgeführt werden:

```bash
skopeo login ghcr.io -u <github-user>
skopeo login docker.io -u <dockerhub-user>

skopeo copy --all \
  docker://ghcr.io/abeggled/openbridgeserver:<tag> \
  docker://docker.io/starwarsfan/openbridgeserver:<tag>
```
