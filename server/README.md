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

Alle Schritte bis einschließlich `release:commit` laufen **komplett
lokal** (auch das Git-Commit) — nichts wird ins Internet hochgeladen. Erst
`git push` macht eine Version tatsächlich live.

### Aktuelles Setup: GitHub Pages + Git LFS für updates.veilgard.de

Dieser Ordner ist bereits ein eigenes lokales Git-Repository (`git init` +
`git lfs install` wurden hier ausgeführt, `.gitattributes` trackt `*.exe`/
`*.blockmap` per LFS, da sie über GitHub Pages, 100-MB-Hardlimit von
normalem Git). Einmalig einzurichten:

1. Auf github.com ein neues **öffentliches** Repo anlegen, z. B. `cx-browser-updates` (leer lassen, keine README/Lizenz automatisch erzeugen).
2. In diesem Ordner:
   ```bash
   git remote add origin https://github.com/<dein-username>/cx-browser-updates.git
   git push -u origin master
   ```
3. Im GitHub-Repo unter **Settings → Pages** bei "Custom domain" `updates.veilgard.de` eintragen.
4. Bei deinem Domain-Anbieter (wo `veilgard.de` registriert ist) einen DNS-Eintrag anlegen:
   ```
   Typ:  CNAME
   Name: updates
   Wert: <dein-username>.github.io
   ```
5. Warten, bis der DNS-Eintrag propagiert ist (paar Minuten bis Stunden) — GitHub stellt danach automatisch ein HTTPS-Zertifikat aus.

Ab dann reicht bei jedem neuen Release:

```bash
npm run release              # baut, verifiziert, kopiert, committet lokal
cd cx-browser-updates && git push   # macht die neue Version live
```

**Alternativen** (falls du später umsteigen willst): Amazon S3 + CloudFront, Azure Static Web Apps, Cloudflare Pages/R2, oder ein eigener nginx-Server — jeder Static-File-Host mit gültigem HTTPS-Zertifikat funktioniert, da CX Browser nur normale HTTPS-GET-Requests macht.

Die Server-Base-URL ist in CX Browser **bewusst keine Nutzer-Einstellung**
(ein manipulierbarer Update-Server wäre ein Sicherheitsrisiko) — sie steht
fest in `src/shared/types.ts` (`DEFAULT_UPDATE_SETTINGS.serverBaseUrl`) und
in `electron-builder.yml` (`publish.url`), aktuell auf
`https://updates.veilgard.de` gesetzt.

## Lokal testen

Für Entwicklung/Tests kann `cx-browser-updates/` mit einem simplen lokalen
Server bereitgestellt werden, z. B.:

```bash
npx http-server cx-browser-updates -p 8443
```

`http://localhost:PORT` wird von CX Browser ausnahmsweise ohne HTTPS
akzeptiert (siehe `MetadataFetcher.assertHttps`) — ausschließlich für
`localhost`/`127.0.0.1`, niemals für echte Domains.
