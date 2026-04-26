# Snake (Vue 3 + Canvas)

Ein kleines Snake-Spiel in Vue 3, gerendert im HTML Canvas.

## Steuerung

- Pfeiltasten: Richtung der Schlange
- Enter: Neustart nach `Game Over`

## Schnellstart

```sh
npm install
npm run dev
```

## Build und Type-Check

```sh
npm run build
```

## Deploy auf GitHub Pages (GitHub Actions)

- Der Deploy-Workflow liegt in `.github/workflows/deploy.yml` und baut dieses Projekt aus dem Unterordner `snake`.
- In `vite.config.ts` wird der `base`-Pfad automatisch aus `GITHUB_REPOSITORY` berechnet.
- Lokal bleibt der `base`-Pfad auf `/`.

Einrichtung in GitHub:

1. Repository nach GitHub pushen (Branch `main`).
2. In GitHub unter `Settings -> Pages` als Source `GitHub Actions` auswaehlen.
3. Nach dem naechsten Push auf `main` wird automatisch nach GitHub Pages deployed.
