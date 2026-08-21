# CX Browser Update-Server

Dieser Ordner (`cx-browser-updates/`) ist die komplette, statische Ordnerstruktur,
die auf einem beliebigen HTTPS-Host liegen muss, damit CX Browser sich selbst
aktualisieren kann. Es ist **kein Anwendungsserver** — nur eine Sammlung
statischer Dateien. Jeder Static-File-Host funktioniert:

- GitHub Pages / GitHub Releases
- Amazon S3 + CloudFront
- Azure Blob Storage / Static Web Apps
- Ein eigener nginx-Server mit gültigem TLS-Zertifikat
- Cloudflare Pages/R2

## Struktur

```
cx-browser-updates/
├── releases/
│   └── windows/
│       ├── stable/
│       │   ├── CX Browser-Setup-1.1.0.exe
│       │   ├── CX Browser-Setup-1.1.0.exe.blockmap
│       │   ├── CX Browser-Portable-1.1.0.exe
│       │   └── latest.yml          ← wird von electron-updater gelesen
│       ├── beta/
│       └── dev/
└── metadata/
    ├── stable.json                  ← reichhaltige Metadaten für die UI
    ├── beta.json
    └── dev.json
```

`latest.yml` wird automatisch von `electron-builder` erzeugt (SHA-512 +
Dateigröße, für die eigentliche Signatur-/Integritätsprüfung durch
`electron-updater`). Die `metadata/<channel>.json`-Dateien liefern
zusätzlich Release Notes, `mandatory`/`securityUpdate`-Flags und einen
SHA-256-Hash für die zusätzliche eigene Prüfung von CX Browser (siehe
`src/updater/core/UpdateManager.ts`).

**Wichtig:** Alte Versionen NICHT löschen. Der Rollback-Mechanismus lädt bei
wiederholten Abstürzen nach einem Update die vorherige Version erneut über
genau diese Ordnerstruktur herunter.

## Release veröffentlichen

```bash
npm run release:build      # baut Installer + portable EXE
npm run release:metadata -- --channel=stable [--mandatory] [--security]
npm run release:publish -- --channel=stable
```

Alle drei Schritte laufen **komplett lokal** — nichts wird ins Internet
hochgeladen. Erst der letzte, manuelle Schritt macht eine Version live:

```bash
# Beispiel mit rsync über SSH:
rsync -avz cx-browser-updates/ user@server:/var/www/updates.veilgard.de/

# Beispiel mit AWS S3:
aws s3 sync cx-browser-updates/ s3://updates.veilgard.de/ --acl public-read
```

Trage die tatsächliche HTTPS-Basis-URL in CX Browser unter
**Einstellungen → Updates** ein (Standard-Platzhalter:
`https://updates.veilgard.de`).

## Lokal testen

Für Entwicklung/Tests kann `cx-browser-updates/` mit einem simplen lokalen
Server bereitgestellt werden, z. B.:

```bash
npx http-server cx-browser-updates -p 8443
```

`http://localhost:PORT` wird von CX Browser ausnahmsweise ohne HTTPS
akzeptiert (siehe `MetadataFetcher.assertHttps`) — ausschließlich für
`localhost`/`127.0.0.1`, niemals für echte Domains.
