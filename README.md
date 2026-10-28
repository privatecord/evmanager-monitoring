# EVManager Monitoring

Docker- und Host-Monitoring für einen Dokploy-Host mit Grafana, Prometheus, cAdvisor und node-exporter.

## Einrichtung in Dokploy

1. Diesen Repository-Branch als Compose-Anwendung auswählen.
2. Das externe Docker-Netzwerk `dokploy-network` muss bereits existieren.
3. Die Variablen aus `.env.example` in Dokploy setzen. Ein starkes `GRAFANA_ADMIN_PASSWORD` und `GRAFANA_DOMAIN` ohne Protokoll oder Pfad sind verpflichtend.
4. DNS der Domain auf den Host richten. Die Traefik-Konfiguration erwartet den EntryPoint `websecure` und den Zertifikatsresolver `letsencrypt`.
5. Manuell deployen und Grafana unter `https://<GRAFANA_DOMAIN>` öffnen.

Grafana erfordert eine Anmeldung. Bei bestehenden Grafana-Daten ändern die Admin-Umgebungsvariablen das bereits gespeicherte Passwort nicht.

## Dashboards und Daten

- **Docker & Host Monitoring**: Container-Auslastung, Host-CPU, RAM, Dateisystem und I/O.
- **EVManager Monitoring**: zusätzliche Containeransichten und vorbereitete App-Panels.

Prometheus scrapt aktuell ausschließlich Prometheus, cAdvisor und node-exporter. App-Panels bleiben ohne zusätzliche Integration leer; das ist kein Nachweis eines App-Ausfalls. Für die Integration müssen der tatsächliche, erreichbare App-Service und Port in `monitoring/prometheus/prometheus.yml` ergänzt werden. Der Jobname muss zum Dashboard passen, etwa `evmanager`.

Die App muss dafür passende Metriken bereitstellen:
- `http_requests_total{route,method,status}`
- `http_request_duration_seconds_bucket{route,method,le}`

Die Containerfilter im EVManager-Dashboard erwarten Namen mit evmanager, frontend, backend bzw. dev/prod. Vor Nutzung gegen die tatsächlichen cAdvisor-Labels prüfen. Datenbankpanels zeigen Containerressourcen, keine Query-Metriken.

Alarmierung und zentrale Logs sind derzeit **nicht implementiert**.

## Betrieb

Metriken werden standardmäßig 15 Tage aufbewahrt; `PROMETHEUS_RETENTION` passt das Zeitfenster an. Den Speicherverbrauch zusätzlich überwachen.

Persistente Volumes heißen weiterhin `docker_prometheus_data` und `docker_grafana_data`. Vor Updates sichern; `docker compose down -v` löscht Daten. Die festen Container- und Netzwerknamen erlauben nur eine solche Installation pro Host.

Dashboard-Änderungen im Grafana-UI sind nicht automatisch im Repository gespeichert und können durch Provisionierung überschrieben werden. Dauerhafte Änderungen als JSON ins Repository übernehmen.

Grafana ist auf `grafana/grafana:13.3.0` festgelegt (stabile Veröffentlichung vom 27.10.2026). Die übrigen Images verwenden noch `latest`. Vor deren Update getestete Versionen festlegen und den Wechsel mit den bestehenden Volumes prüfen. cAdvisor benötigt in dieser Konfiguration privilegierten Hostzugriff.

## Prüfung vor Deployment

Mit gesetzten Variablen bzw. einer lokalen `.env`:

```sh
docker compose config --quiet
docker compose run --rm --no-deps --entrypoint promtool prometheus check config /etc/prometheus/prometheus.yml
```

Nach Deployment in Grafana über Explore `up` abfragen: alle drei konfigurierten Targets sollten 1 liefern. Anschließend Hostwerte und Containernamen mit dem tatsächlichen Host vergleichen.

Es werden keine Hostports veröffentlicht; der Zugriff erfolgt über Traefik. Ein unverändertes lokales `docker compose up` ohne Dokploy-Netzwerk und Traefik ist daher kein vollständiges lokales Setup.

## Grafana aktualisieren

1. Aktuelle Grafana-Version und Image-ID notieren.
2. Grafana stoppen und das vollständige Volume `docker_grafana_data` sichern (einschließlich SQLite-Datenbank). Die Sicherung außerhalb des Volumes aufbewahren. Danach Grafana bei Bedarf wieder starten, bis das Deployment bereit ist.
3. In Dokploy den Branch `feature/evmanager-monitoring-dashboard` mit der aktualisierten Compose-Datei manuell deployen. Das Image `grafana/grafana:13.3.0` muss dabei heruntergeladen werden.
4. Startlogs auf Migrations- und Provisionierungsfehler prüfen. Login, beide Dashboards und die Prometheus-Datenquelle testen. Die laufende Version über `/api/health` prüfen; erwartet wird `13.3.0`.

Benutzer, Passwort und Einstellungen bleiben im bestehenden Volume erhalten. Kein neues Volume anlegen und keine Volumes löschen.

Grafana 13 migriert beim ersten Start einer älteren Installation Dashboard- und Ordnerdaten in Unified Storage. Für einen Rollback die Sicherung von vor dem Update zusammen mit dem vorherigen Image wiederherstellen; ein bloßer Wechsel auf einen älteren Image-Tag genügt nicht.

Offizielle Hinweise:
- https://grafana.com/grafana/download/13.3.0?edition=oss
- https://grafana.com/docs/grafana/latest/upgrade-guide/upgrade-v13/upgrade-v13.0/

Die Änderung ist im Repository vorbereitet. Ein Laufzeittest und die Migration müssen auf dem Dokploy-Host geprüft werden.
