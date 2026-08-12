# Checklist de Migración a maiwei-app

> Generado tras Fase 0 (discovery) y Fase 1 (plan). Ejecución NO autorizada aún —
> cada bloque requiere confirmación explícita antes de ejecutarse (HITL).
> Estado: ⬜ pendiente · ✅ hecho · ⚠️ falló (ver nota) · 🚫 excluido de alcance

## Decisiones fijadas (no reabrir sin motivo)

- Cuenta origen: `colomr-cc` (renombrada desde `colomr-dev`)
- SonarCloud: estándar único y permanente = **CI-based Analysis** (Automatic Analysis descartado)
- `SONAR_TOKEN`: org secret, visibility **All repositories**
- `DEVTO_API_KEY`: org secret, visibility **All repositories** (temporal hasta migrar `colomr-cc`/`dev-to-backup`, luego restringir a Selected — ver punto pendiente en Pre-migración)
- `APP_CLIENT_ID` / `AUTOMATION_APP_PRIVATE_KEY`: org-level (App `maiwey-automation`, transferida a `maiwei-app`, instalada con `repository_selection: all`) — ver sección "PR approval flow"
- Reusable workflows viven en `maiwei-app/workflows`; community health files en `maiwei-app/.github`
- Aprobación de PR: `required_approving_review_count: 1` **manteniéndose** — la autoaprobación NO es posible en GitHub (verificado, corrijo suposición anterior). Resuelto vía App `maiwey-automation`: PRs abiertos con token de instalación de la App (autor: bot), commits autorados como `F Colomer <colomr@pm.me>`, aprobación real de `colomr-cc`. Ver sección dedicada más abajo
- No-atribución de IA: required workflow adaptada del step ya existente en `claude-sync/.github/workflows/ci.yml`. Nota técnica: como "required workflow" de ruleset, tuvo que llevar trigger `pull_request` propio, no solo `workflow_call`
- Firma GPG (`required_signatures`): decidido dejar como fine-tuning **post-migración**, para no mezclar dos fuentes de fricción durante la migración
- Licencia: `GPL-3.0` en repos públicos (placeholder en `repo-template`), sin LICENSE en repos privados/SaaS (ej. `chameleon`), `BSL 1.1` como referencia si algún día se abre el código de un SaaS
- `claude-sync`: 🚫 **excluido de esta migración** (tooling personal, no código de negocio) — pendiente de confirmación final tuya, revertible
- `colomr-cc` (repo del README de perfil): 🚫 **excluido de esta migración** — el Easter egg de perfil de GitHub (repo con mismo nombre que el usuario) solo funciona si el owner es exactamente ese username; transferirlo lo rompe sin trasladar el beneficio a la org. Se queda personal, mantiene sus propios `SONAR_TOKEN`/`DEVTO_API_KEY` (mismo valor que los de org, rotados a la vez en los dos sitios)
- `fles`, `mainframes`, `notebooks` (archived): 🚫 **excluidos de esta migración** — sin CI ni secrets, cero beneficio real de centralizar, solo ruido cosmético
- Cola de migración real: **4 repos** — `dev-to-backup`, `colomr-v1-theme`, `chameleon`, `colomr.cc` (orden sin cambios respecto al original, salvo las exclusiones)
- Perfil org (`github.com/maiwei-app`): `profile/README.md` creado en `maiwei-app/.github` (minimalista, placeholder hasta tener imagen corporativa). Membership de `colomr-cc` en la org puesta en público. Pendiente (opcional, cuando quieras): pinear repos de la org en tu perfil personal (manual, gusto tuyo) + línea mencionando `@maiwei-app` en el README de `colomr-cc/colomr-cc`

---

## Pre-migración (capa organizacional, va primero)

- [x] ✅ Activar `two_factor_requirement_enabled` — hecho manualmente en la web (la API lo bloqueaba silenciosamente). Verificado: `true`
- [x] ✅ Security Configuration org-wide (`maiwei-app-org-config-1`, id 266636, `enforcement: enforced`, default para repos nuevos): Secret Protection **disabled** ($0/mes, decisión consciente por coste — $19+$30 por committer activo no justificado con 1 seat), Code Security **disabled**, Dependabot alerts + security updates **enabled** (gratis). Nota: los campos clásicos `advanced_security_enabled_for_new_repositories` etc. quedan en `false` — es esperado, esta configuración los sustituye. Verificado vía `GET /orgs/maiwei-app/code-security/configurations`
- [ ] ⬜ (pendiente, se resuelve en el punto del `repo-template`) `dependabot.yml` básico — no se localizó el que tenían antes "a nivel personal", se define desde cero
- [x] ✅ Crear org secret `SONAR_TOKEN` (visibility: All repositories) — token único, rotación cada 90 días. Creado por el usuario vía `gh secret set` local (nunca pasó por el chat). Verificado vía API
- [x] ✅ Crear org secret `DEVTO_API_KEY` — key única en dev.to (consolidada de "Dev.to Latest Posts" + "Dev.to Backups Repo"), rotación 90 días. Creado con `visibility: all` temporalmente (repos aún no migrados). Verificado vía API
- [ ] ⬜ **Post-migración**: restringir `DEVTO_API_KEY` a `visibility: selected` (colomr-cc, dev-to-backup) una vez esos repos estén transferidos; borrar las 2 API keys antiguas en dev.to
- [x] ✅ Crear org variable `SONAR_ORGANIZATION=maiwei` — verificado vía API
- [x] ✅ Crear repo `maiwei-app/.github` (PR template, issue templates, CODEOWNERS default `@colomr-cc`, SECURITY.md, README) — verificado vía API (`git/trees`)
- [x] ✅ Crear repo `maiwei-app/workflows` con:
  - [x] ✅ `ci-python.yml` (job: `quality`) — lint (ruff) + validación JSON schemas + tests (pytest)
  - [x] ✅ `ci-flutter.yml` (job: `quality`) — flutter analyze + flutter test
  - [x] ✅ `ci-hugo.yml` (job: `quality`) — build estricto (`--panicOnWarning`)
  - [x] ✅ `sonar-scan.yml` (job: `sonar`) — CI-based, usa `SONAR_TOKEN` + `SONAR_ORGANIZATION`, input `project-key`
  - [x] ✅ `no-ai-attribution.yml` (job: `check`) — portado 1:1 del step de `claude-sync/ci.yml`
  - Verificado vía API (`git/trees`)
  - Backlog post-migración: `release-please.yml` (versionado), `inline-comments.yml`
- [x] ✅ Definir schema de custom properties en la org (verificado vía API):
  - [x] ✅ `ci-profile`: `python-quality | flutter-quality | hugo-static | none`
  - [x] ✅ `sonar-enabled`: `true | false`
- [x] ✅ Crear repo `maiwei-app/repo-template` (`is_template: true`): `ci.yml` cableado a `no-ai-attribution`+`ci-python`+`sonar-scan`, `.github/dependabot.yml`, `sonar-project.properties` (placeholder), `README.md`, `LICENSE` (GPL-3.0, política: público=GPL-3.0, privado/SaaS=sin LICENSE, BSL 1.1 si algún día se abre un SaaS). `delete_branch_on_merge: true` y custom properties (`ci-profile=python-quality`, `sonar-enabled=true`) preseteadas. **No** incluye CODEOWNERS (se hereda del `.github` org). Verificado vía API (`git/trees`, `properties/values`)
- Nota: CODEOWNERS se omite intencionadamente del template — lo hereda de `maiwei-app/.github`
- [x] ✅ Crear ruleset org-wide "Protect default branch" (id `20768236`, `enforcement: active`, target `~ALL` repos / `~DEFAULT_BRANCH`):
  - [x] ✅ Require PR before merging (`required_approving_review_count: 1`, dismiss stale reviews, resolver hilos)
  - [ ] ⬜ **Post-migración (fine-tuning)**: `required_signatures` (firma GPG) — se deja para cuando la migración esté estable, para no mezclar dos fuentes de fricción
  - [x] ✅ Block force-push (`non_fast_forward`)
  - [x] ✅ Restrict branch deletion (`deletion`)
  - [ ] ⬜ Required status checks (`quality`/`sonar` por `ci-profile`/`sonar-enabled`) — pendiente hasta tener un run real por repo migrado
  - [x] ✅ Required workflow: `no-ai-attribution.yml` — nota técnica: tuvo que llevar trigger `pull_request` propio además de `workflow_call` (GitHub lo exige para "required workflows" en rulesets, es un mecanismo distinto a `uses:`). Se quitó la llamada duplicada del `ci.yml` del `repo-template`
  - [ ] ⬜ Required workflow: `sonar-scan.yml` — no aplicable como "required workflow" (necesita `secrets`/`inputs` por repo), se mantiene como llamada `uses:` en cada `ci.yml`, no vía ruleset
  - Sin bypass actors

---

## PR approval flow (bloqueante, resuelto)

Descubrimos en vivo que GitHub bloquea la autoaprobación de PRs (ni por API ni por UI) — con `required_approving_review_count: 1` y un único colaborador, ningún PR de `colomr-cc` podía mergearse nunca. Solución implementada (definitiva, no parche):

- [x] ✅ Localizada la GitHub App propia `maiwey-automation` (antes solo instalada en `colomr-cc/colomr.cc`, usada por `sync-badges.yml`)
- [x] ✅ Transferido el ownership de la App a `maiwei-app` (antes: intento de "Make public" en `colomr-cc`, descartado — transferir ownership es la solución más limpia dado que el único repo que la usaba, `colomr.cc`, también migra a la org)
- [x] ✅ App instalada automáticamente en `maiwei-app` al transferir (`installation id 153295344`, `repository_selection: all`, permisos `contents:write` + `pull_requests:write`)
- [x] ✅ Private key regenerada (la antigua no estaba guardada, era write-only; el usuario la revocó) y promovida a `AUTOMATION_APP_PRIVATE_KEY` (org secret, visibility all)
- [x] ✅ `APP_CLIENT_ID` promovido a org variable (visibility all)
- [x] ✅ Patrón de PR validado end-to-end: commit autorado como `F Colomer <colomr@pm.me>` (git local, respeta el contrato de no-atribución-IA) + PR abierto con token de instalación de la App (`maiwey-automation[bot]` como autor) + aprobado por `colomr-cc` (posible al no ser el autor) + merge. Probado con un PR real en `maiwei-app/.github` (#1 cerrado sin mergear por ser autoría directa, #2 mergeado con éxito vía el flujo del bot)
- **Pendiente de revisar**: el contrato global (`~/.claude/CLAUDE.md`) sobre atribución de IA — el usuario quiere repasarlo a la luz de este flujo con bot
- **Nota para el resto de la migración**: cualquier PR que abra en repos de `maiwei-app` de aquí en adelante debe usar este patrón (token de instalación de la App), no el token personal de `colomr-cc`, o volveremos a bloquearnos
- Pendiente (cuando migremos `colomr.cc`): decidir si su `AUTOMATION_APP_PRIVATE_KEY`/`APP_CLIENT_ID` locales se borran (ya cubiertos por los de la org) — la key vieja ahí ya no vale, quedó huérfana

## Migración por repo (ordenado por prioridad)

🚫 `fles`, `mainframes`, `notebooks` — excluidos (ver Decisiones fijadas), se quedan en `colomr-cc` personal, sin acción.

🚫 `colomr-cc` (repo del README de perfil) — excluido (ver Decisiones fijadas), se queda en `colomr-cc` personal. Mantiene `SONAR_TOKEN`/`DEVTO_API_KEY` locales, mismo valor que los de org, se rotan a la vez en ambos sitios.

#### Repo: `dev-to-backup` (Complejidad: 3, CI simple + 1 secret duplicado)
- [ ] ⬜ Transfer repo a maiwei-app
- [ ] ⬜ Verificar redirect URL antigua funciona
- [ ] ⬜ Actualizar remote local (si aplica)
- [ ] ⬜ Asignar custom property `ci-profile=python-quality`, `sonar-enabled=false`
- [ ] ⬜ Activar `delete_branch_on_merge`
- [ ] ⬜ Migrar `backup.yml` a usar `ci-python.yml` reusable
- [ ] ⬜ Verificar CI usa `DEVTO_API_KEY` org (ya no repo-level)
- [ ] ⬜ Borrar secret `DEVTO_API_KEY` del repo
- [ ] ⬜ Verificar ruleset org-wide aplica correctamente
- [ ] ⬜ Smoke test: push → PR → checks verdes → merge → rama borrada

#### Repo: `colomr-v1-theme` (Complejidad: 3, CI + Sonar Automatic → migrar a CI-based)
- [ ] ⬜ Transfer repo a maiwei-app
- [ ] ⬜ Verificar redirect URL antigua funciona
- [ ] ⬜ Actualizar remote local (si aplica)
- [ ] ⬜ Asignar custom property `ci-profile=hugo-static`, `sonar-enabled=true`
- [ ] ⬜ Activar `delete_branch_on_merge`
- [ ] ⬜ Migrar `ci.yml` a usar `ci-hugo.yml` + `sonar-scan.yml` reusable (CI-based)
- [ ] ⬜ Desactivar Sonar Automatic Analysis para este proyecto
- [ ] ⬜ Re-mapear proyecto en SonarCloud a `maiwei-app/colomr-v1-theme`
- [ ] ⬜ Verificar Quality Gate sigue aplicando
- [ ] ⬜ Verificar ruleset org-wide aplica correctamente
- [ ] ⬜ Smoke test: push → PR → checks verdes → merge → rama borrada

#### Repo: `chameleon` (Complejidad: 5, privado, primera vez con protección real)
- [ ] ⬜ Transfer repo a maiwei-app
- [ ] ⬜ Verificar redirect URL antigua funciona
- [ ] ⬜ Actualizar remote local (si aplica)
- [ ] ⬜ Confirmar que branch protection/ruleset ahora SÍ aplica (ya no 403 "Upgrade to Pro")
- [ ] ⬜ Asignar custom property `ci-profile=flutter-quality`, `sonar-enabled=true`
- [ ] ⬜ Activar `delete_branch_on_merge`
- [ ] ⬜ Migrar `flutter-analyze.yml` a usar `ci-flutter.yml` reusable
- [ ] ⬜ Migrar `sonarcloud.yml` a usar `sonar-scan.yml` reusable
- [ ] ⬜ Re-mapear proyecto en SonarCloud a `maiwei-app/chameleon`
- [ ] ⬜ Verificar CI usa `SONAR_TOKEN` org (ya no repo-level)
- [ ] ⬜ Borrar secret `SONAR_TOKEN` del repo (mantener `SUPABASE_KEEPALIVE_DB_URL` a nivel repo, no se centraliza)
- [ ] ⬜ Verificar `supabase-keepalive.yml` sigue funcionando
- [ ] ⬜ Verificar ruleset org-wide aplica correctamente
- [ ] ⬜ Smoke test: push → PR → checks verdes → merge → rama borrada

#### Repo: `colomr.cc` (Complejidad: 5, mayor superficie externa: Firebase + GitHub App)
- [ ] ⬜ Transfer repo a maiwei-app
- [ ] ⬜ Verificar redirect URL antigua funciona
- [ ] ⬜ Actualizar remote local (si aplica)
- [ ] ⬜ Asignar custom property `ci-profile=hugo-static`, `sonar-enabled=true`
- [ ] ⬜ Activar `delete_branch_on_merge`
- [ ] ⬜ Migrar `ci.yml` a usar `ci-hugo.yml` + `sonar-scan.yml` reusable (CI-based)
- [ ] ⬜ Desactivar Sonar Automatic Analysis para este proyecto
- [ ] ⬜ Re-mapear proyecto en SonarCloud a `maiwei-app/colomr.cc`
- [ ] ⬜ Verificar `deploy.yml` sigue funcionando con `FIREBASE_SERVICE_ACCOUNT` (repo-level, no se centraliza)
- [ ] ⬜ Verificar `sync-badges.yml` sigue funcionando — puede pasar a usar `AUTOMATION_APP_PRIVATE_KEY`/`APP_CLIENT_ID` org-level (ya promovidos) en vez de sus copias locales
- [ ] ⬜ Borrar `AUTOMATION_APP_PRIVATE_KEY`/`APP_CLIENT_ID` locales del repo (la key vieja quedó huérfana e inválida tras la regeneración)
- [ ] ⬜ Verificar `GEMINI_API_KEY` sigue disponible (repo-level, no se centraliza)
- [ ] ⬜ Verificar Quality Gate sigue aplicando
- [ ] ⬜ Verificar ruleset org-wide aplica correctamente
- [ ] ⬜ Smoke test: push → PR → checks verdes → merge → rama borrada

---

## Post-migración

- [ ] ⬜ Audit: los 4 repos de la cola transferidos (`dev-to-backup`, `colomr-v1-theme`, `chameleon`, `colomr.cc`)
- [ ] ⬜ Verificar secret scanning + push protection org-wide activo en todos
- [ ] ⬜ Verificar 2FA requirement activo
- [ ] ⬜ Verificar que no quedan secrets huérfanos en ningún repo individual migrado
- [ ] ⬜ Restringir `DEVTO_API_KEY` a `visibility: selected` (colomr-cc personal ya no aplica al no migrar ese repo — revisar si sigue teniendo sentido "all" o restringir solo a `dev-to-backup`)
- [ ] ⬜ Borrar branch protections repo-level redundantes con rulesets org (donde existían: `colomr-v1-theme`, `colomr.cc`)
- [ ] ⬜ Verificar SonarCloud: 3 proyectos (`chameleon`, `colomr-v1-theme`, `colomr.cc`) re-mapeados a `maiwei-app` y en modo CI-based. `colomr-cc` (perfil) se queda en `colomr-cc` personal, sin re-mapear
- [ ] ⬜ Habilitar `required_signatures` (firma GPG) en el ruleset org-wide, una vez todo estable
- [x] ✅ Revisar el contrato global sobre atribución de IA a la luz del bot — PR mergeado: [colomr-cc/claude-sync#22](https://github.com/colomr-cc/claude-sync/pull/22). Regla final: "todo PR se abre con un bot/App de GitHub de su propiedad — GitHub no permite auto-aprobación de PRs", sin excepciones (el flujo estructural no depende de quién escribió el código)
- [ ] ⬜ Pinear repos de la org en el perfil personal de `colomr-cc` (manual, opcional)
- [ ] ⬜ Añadir mención a `@maiwei-app` en el README de `colomr-cc/colomr-cc` (opcional)
- [ ] ⬜ Actualizar links en Notion
- [ ] ⬜ Confirmar destino final de `claude-sync` (excluido salvo indicación contraria)
