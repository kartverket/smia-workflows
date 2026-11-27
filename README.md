# smia-workflows

Felles GitHub Actions-workflows som automatiserer release-, build- og deployment-prosesser for Team
Smias repoer.

Hele deployflyten, fra bygg til prod, er beskrevet i
[smia-docs](https://github.com/kartverket/smia-docs/blob/main/docs/felles/deploy.md).

## Sette opp deploy i et repo

Stegene gjelder både for nye repoer og for repoer som flyttes over fra en eldre deployflyt. Guiden
dekker bare deploy. Hvordan imaget bygges og testes, varierer fra app til app og står ikke her. Det
deployen trenger fra bygget, er et image i `ghcr.io/kartverket` med versjonen som tag, og digesten
til imaget.

1. **Lag versjonen i bygget.** Bruk commit-datoen i norsk tid, kort SHA og byggnummer, slik
   `komreg-backend` gjør:

   ```bash
   timestamp=$(git show -s --format=%ct HEAD)
   date=$(TZ=Europe/Oslo date -d "@$timestamp" +'%Y.%m.%d')
   echo "version=${date}-${GITHUB_SHA:0:7}-${GITHUB_RUN_NUMBER}" >> "$GITHUB_OUTPUT"
   ```

   Tag imaget med versjonen. Vi lager ikke Git-tags, så fjern `github-tag-action` hvis repoet bruker
   den.

2. **Legg appen inn i `smia-apps`.** Gjelder bare nye apper. Appen trenger en Jsonnet-fil og en
   versjonsfil (`<app>-version`) i namespace-mappa under `env/atkv3-dev/` og `env/atkv3-prod/`.
   Jsonnet-fila leser versjonen fra versjonsfila. Se `komreg-backend.jsonnet` i
   `env/atkv3-dev/komreg-main/` for et eksempel, og
   [README-en til `smia-apps`](https://github.com/kartverket/smia-apps#readme) for oppsettet.

3. **Gi repoet tilgang til `smia-apps`.** Deployen henter token med Octo STS, og feiler hvis
   identiteten mangler:
   - Legg til `.github/chainguard/<identitet>.sts.yaml` i `smia-apps`. Nye repoer må oppgis med
     org-ID og repo-ID i `subject`, slik `komreg-frontend.sts.yaml` gjør. Repo-ID-en finner du med
     `gh api repos/kartverket/<repo> --jq .id`. Se
     [SKIP-dokumentasjonen](https://skip.kartverket.no/docs/applikasjon-utrulling/github-actions/tilgang-til-repoer-med-tokens-fra-github-actions#etablere-tillit).
   - Merge identiteten før første deploy.

4. **Deploy til dev med [`deploy-app.yml`](#deploy-appyml).** Send med digesten fra bygget, og
   filnavnet uten `.sts.yaml` som `identity`. Sjekk etter første deploy at versjonsfila i
   `env/atkv3-dev/<namespace>/` i `smia-apps` er oppdatert, og at appen er synkronisert i Argo CD.

5. **Få deploy-PR til prod.** Legg versjonsfila inn i `VERSIONED_APPS` i `scripts/sync_versions.py`
   i `smia-apps`. Se [README-en til `smia-apps`](https://github.com/kartverket/smia-apps#readme).

6. **Flytter du et eksisterende repo, rydd bort det gamle.** Fjern jobber som deployer rett til
   prod, og kall til `release-app-octo-sts.yml` eller `promote-app-octo-sts.yml` i `smia-apps`.

## `deploy-app.yml`

Deployer en versjon av en applikasjon til ett namespace i ett miljø. Workflowen skriver
`ghcr.io/kartverket/<application>:<version>@<digest>` til
`env/<cluster>-<environment>/<namespace>/<application>-version` i
[`smia-apps`](https://github.com/kartverket/smia-apps) og committer direkte til `main`. Argo CD
ruller ut endringen derfra.

| Input         | Påkrevd | Beskrivelse                                                         |
| ------------- | ------- | ------------------------------------------------------------------- |
| `application` | ja      | Navnet på imaget i `ghcr.io/kartverket`, `[a-z0-9-]+`               |
| `version`     | ja      | Image-taggen, for eksempel `2026.11.25-e504fa6-403`                 |
| `digest`      | ja      | Digest til imaget, `sha256:<64 hex>`. Vi deployer alltid med digest |
| `environment` | ja      | `dev` eller `prod`                                                  |
| `namespace`   | ja      | Namespace-mappa under miljøet, `[a-z0-9-]+`                         |
| `identity`    | ja      | Octo STS-identiteten som har skrivetilgang til `smia-apps`          |
| `cluster`     | nei     | `atkv3` (standard) eller `atgcp1`                                   |

Versjonsfila må finnes i `smia-apps` fra før. Workflowen lager ikke nye namespaces. Skal appen
deployes til flere namespaces, kaller du workflowen én gang per namespace.

Kall workflowen med `environment: dev`. Til prod går applikasjonene via deploy-PR-en som `smia-apps`
lager automatisk.

Eksempel fra `komreg-backend`:

```yaml
deploy-dev:
  needs: [build]
  permissions:
    id-token: write
    contents: read
  uses: kartverket/smia-workflows/.github/workflows/deploy-app.yml@main
  with:
    application: komreg-backend
    version: ${{ needs.build.outputs.version }}
    digest: ${{ needs.build.outputs.digest }}
    environment: dev
    identity: komreg
    namespace: komreg-main
```
