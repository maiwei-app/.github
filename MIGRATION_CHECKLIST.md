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
  - Backlog post-migración: ~~`release-please.yml` (versionado)~~ ✅ hecho 2026-08-15, ver sección dedicada más abajo; `inline-comments.yml` sigue pendiente
- [x] ✅ Definir schema de custom properties en la org (verificado vía API):
  - [x] ✅ `ci-profile`: `python-quality | flutter-quality | hugo-static | none`
  - [x] ✅ `sonar-enabled`: `true | false` — **doble propósito desde 2026-08-14**: además de indicar si el `ci.yml` del repo llama a `sonar-scan.yml`, es ahora también el criterio de targeting del ruleset "Require SonarCloud Quality Gate" (ver más abajo). Todo repo con Sonar wired en su CI debe tener esta property en `true`, o el check dejará de ser bloqueante para mergear aunque corra igualmente. Repos verificados con `sonar-enabled=true`: `dev-to-backup`, `repo-template`, `workflows`. `.github` se queda deliberadamente en `false`/sin valor (no tiene CI ni código que analizar)
- [x] ✅ Crear repo `maiwei-app/repo-template` (`is_template: true`): `ci.yml` cableado a `no-ai-attribution`+`ci-python`+`sonar-scan`, `.github/dependabot.yml`, `sonar-project.properties` (placeholder), `README.md`, `LICENSE` (GPL-3.0, política: público=GPL-3.0, privado/SaaS=sin LICENSE, BSL 1.1 si algún día se abre un SaaS). `delete_branch_on_merge: true` y custom properties (`ci-profile=python-quality`, `sonar-enabled=true`) preseteadas. **No** incluye CODEOWNERS (se hereda del `.github` org). Verificado vía API (`git/trees`, `properties/values`)
- Nota: CODEOWNERS se omite intencionadamente del template — lo hereda de `maiwei-app/.github`
- [x] ✅ Crear ruleset org-wide "Protect default branch" (id `20768236`, `enforcement: active`, target `~ALL` repos / `~DEFAULT_BRANCH`):
  - [x] ✅ Require PR before merging (`required_approving_review_count: 1`, dismiss stale reviews, resolver hilos)
  - [ ] ⬜ **Post-migración (fine-tuning)**: `required_signatures` (firma GPG) — se deja para cuando la migración esté estable, para no mezclar dos fuentes de fricción
  - [x] ✅ Block force-push (`non_fast_forward`)
  - [x] ✅ Restrict branch deletion (`deletion`)
  - [x] ✅ Required workflow: `no-ai-attribution.yml` — nota técnica: tuvo que llevar trigger `pull_request` propio además de `workflow_call` (GitHub lo exige para "required workflows" en rulesets, es un mecanismo distinto a `uses:`). Se quitó la llamada duplicada del `ci.yml` del `repo-template`
  - [ ] ⬜ Required workflow: `sonar-scan.yml` — no aplicable como "required workflow" (necesita `secrets`/`inputs` por repo), se mantiene como llamada `uses:` en cada `ci.yml`, no vía ruleset
  - Sin bypass actors
  - ⚠️ **Corregido 2026-08-14**: este ruleset llegó a tener el status check `SonarCloud Code Analysis` como required, aplicando a `~ALL` repos sin condición. Rompió la primera PR en un repo sin código (`maiwei-app/.github`, sin CI propio): el check nunca se reportaba y la PR quedaba bloqueada para siempre ("Expected — Waiting for status to be reported"). El check de Sonar se sacó de aquí y se movió al ruleset dedicado de abajo.

- [x] ✅ Crear ruleset org-wide **"Require SonarCloud Quality Gate"** (nuevo, 2026-08-14) — separado del anterior a propósito, porque el targeting por repositorio en GitHub Rulesets es todo-o-nada por ruleset: no se puede condicionar una regla suelta dentro de un ruleset ya targeteado a `~ALL`.
  - Target repositories: **Matching a filter** → `props.sonar-enabled:true` (dinámico: cualquier repo futuro con esa property en `true` queda incluido automáticamente, sin tocar el ruleset)
  - Única regla activa: Require status checks to pass → `SonarCloud Code Analysis`
  - Sin bypass actors
  - A fecha de creación matchea 3 repos: `dev-to-backup`, `repo-template`, `workflows`
  - El resto de protecciones (PR obligatoria, no force-push, no borrado de rama, no-ai-attribution) las sigue cubriendo el ruleset universal de arriba, que aplica a `~ALL` sin condición — este ruleset nuevo solo añade la exigencia de Sonar encima, donde aplica

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

## Aislamiento de maibot (completado 2026-08-14)

**Contexto:** La App ahora se llama `maibot-app` (vocación de agente). Para evitar que Claude Code pueda ejecutar comandos con credenciales del usuario, se implementó aislamiento a nivel de OS en WSL2.

- [x] ✅ Usuario `maibot` creado en WSL2 (home: `/home/maibot`)
- [x] ✅ Clave privada copiada a `~maibot/.ssh/maibot.pem` (permisos 600, propiedad maibot:maibot)
- [x] ✅ SSH config configurado (`~maibot/.ssh/config` → usa maibot.pem para GitHub)
- [x] ✅ JWT generation implementado (genera tokens de instalación automáticamente)
- [x] ✅ GitHub CLI autenticación: Token de instalación configurado en `~maibot/.config/gh/hosts.yml`
- [x] ✅ Permisos verificados en maibot-app (NO write access to main): `contents:write`, `metadata:read`, `pull_requests:write` — ampliados a lo largo de la migración con `workflows:write` (necesario para tocar `.github/workflows/*`, GitHub lo bloquea aparte de `contents`) y `secrets:read` a nivel de **repo** (org secrets requieren un permiso de organización aparte, todavía no concedido)
- [x] ✅ Validado: PR creation desde usuario maibot como `maibot-app[bot]` funciona end-to-end (varios PRs mergeados)
- Nota operativa: al ampliar permisos de la App en `https://github.com/organizations/maiwei-app/settings/apps/maibot-app/permissions`, hay que pulsar **Save changes** al fondo de esa misma página — no basta con cambiar el desplegable, y no hay un paso de re-aprobación aparte en otra pantalla

**Arquitectura:**
- **fcolomer** (user): trabajo interactivo, sin acceso a credenciales de la App
- **maibot** (user): ejecuta acciones de CI/CD (git push, PR creation) con credenciales del bot
- Seguridad por aislamiento (OS), no por confianza (IA)

---

## Sem-Ver / Release-Please (implementado y validado en 3 repos)

**Estado (actualizado 2026-08-15):** funcionando de extremo a extremo en los
3 repos que hoy viven en `maiwei-app`. Los 4 repos originales de la cola de
migración (`dev-to-backup`, `colomr-v1-theme`, `chameleon`, `colomr.cc`) NO
son lo mismo que "repos con release-please" — solo `dev-to-backup` está
migrado hoy; los otros 3 siguen en `colomr-cc` personal y release-please
ahí es tarea futura, parte de su propia migración (ver secciones de abajo).

| Repo | Release-please | Config validado contra schema real | Última release |
|---|---|---|---|
| `workflows` | ✅ | ✅ (corregido 2026-08-15) | v1.0.0 shipeada 2026-08-15; PR `chore(main): release 1.0.1` pendiente de merge |
| `dev-to-backup` | ✅ | ✅ (corregido 2026-08-15) | v1.0.0 shipeada 2026-08-15; PR `chore(main): release 1.0.1` pendiente de merge |
| `repo-template` | ✅ (de fábrica, para cualquier repo nuevo creado desde el template) | ✅ | primera release `0.1.0` pendiente de merge |

**Receta de 3 ficheros por repo** (usar exactamente esta forma, ya
validada — ver `.release-please-config.json` real en cualquiera de los 3
repos de arriba como referencia):
- `.release-please-config.json` — claves reales del schema de
  release-please en **kebab-case** (`release-type`,
  `bump-minor-pre-major`, `changelog-sections` como array plano de
  `{type, section, hidden}`). **No existe** la propiedad `repositoryUrl`
  — la action resuelve la URL del repo sola vía `GITHUB_REPOSITORY`.
- `.release-please-manifest.json` — `{".": "X.Y.Z"}`, versión de arranque
  (`1.0.0` si el repo ya tenía código real en producción antes de adoptar
  release-please; `0.1.0` si arranca de cero, coherente con
  `bump-minor-pre-major: true`)
- `.github/workflows/release.yml` — dispara en `push` a `main`, usa
  `actions/create-github-app-token@v1` (App `maibot-app`, org var
  `MAIBOT_APP_ID` + org secret `MAIBOT_APP_PRIVATE_KEY`, ambos scope "All
  repositories") en vez de `secrets.GITHUB_TOKEN` — el token por defecto
  no puede disparar checks `pull_request` en PRs que él mismo crea, así
  que el PR de release se queda bloqueado sin este paso

**Bugs encontrados y corregidos en esta ronda (2026-08-14/15), todos ya
arreglados en los 3 repos:**
1. Acción mal referenciada: `google/release-please-action` no existe, es
   `googleapis/release-please-action`
2. Token por defecto no dispara checks en PRs auto-creados → App token
3. `no-ai-attribution.yml` marcaba `maibot-app[bot]` como falso positivo
   de autoría IA → excepción añadida
4. `no-ai-attribution.yml` marcaba el footer estándar de release-please
   ("generated with Release Please") como falso positivo → patrón
   acotado a exigir nombre real de herramienta IA
5. `repo-template` nunca había tenido un PR real que disparase su CI
   completo — salieron 3 bugs latentes de golpe: `ci-python.yml` exigía
   `requirements-ci.lock`/`tests/` que no existían; `secrets: inherit`
   no mapea `sonar-token` (nombre distinto a `SONAR_TOKEN`); validación
   JSON era solo sintaxis, no contra schema real
6. **El más serio**: `.release-please-config.json` llevaba claves en
   camelCase (`releaseType`, `bumpMinorPreMajor`, `repositoryUrl`,
   `changelog.sections` anidado) que **no existen** en el schema real de
   release-please (verificado contra `v17.6.1`, la versión que empaqueta
   `release-please-action@v4`, y contra `v17.11.1`). Como
   `additionalProperties: false` en la raíz, es muy probable que esas
   claves se ignorasen en silencio desde el principio — la v1.0.0 de
   `workflows`/`dev-to-backup` puede haberse generado con el
   comportamiento por defecto de release-please, no con el ajuste fino
   que creíamos haber configurado. Corregido en los 3 repos.

**Mejoras de calidad que salieron de este trabajo** (ahora en
`ci-python.yml`, compartido por todos los repos que lo llamen):
- Validación JSON real contra schema (`check-jsonschema`), con mapa
  fichero→schema documentado en el README de `workflows` — mantenerlo al
  añadir nuevos tipos de fichero JSON con schema real
- `pytest` tolera "no hay tests" solo si el repo no tiene ningún `.py` de
  lógica real fuera de `tests/` — nunca tests placeholder para forzar un
  check en verde

**Pendiente, sin urgencia:**
- [ ] ⬜ Extender release-please a `colomr-v1-theme`, `chameleon`,
      `colomr.cc` cuando se migren a `maiwei-app` (no antes — no tiene
      sentido instalarlo en un repo que sigue en `colomr-cc` personal)
- [ ] ⬜ Configurar tags múltiples (v1.2.3 / v1 / latest) — release-please
      ya crea el tag `vX.Y.Z`; los tags `v1`/`latest` rodantes no están
      implementados todavía, evaluar si hacen falta de verdad
- [ ] ⬜ Linter anti-SHA en CI (`lint-no-shas.yml` ya existe en
      `workflows` — confirmar que corre en los repos migrados)

---

## Migración por repo (ordenado por prioridad)

🚫 `fles`, `mainframes`, `notebooks` — excluidos (ver Decisiones fijadas), se quedan en `colomr-cc` personal, sin acción.

🚫 `colomr-cc` (repo del README de perfil) — excluido (ver Decisiones fijadas), se queda en `colomr-cc` personal. Mantiene `SONAR_TOKEN`/`DEVTO_API_KEY` locales, mismo valor que los de org, se rotan a la vez en ambos sitios.

#### Repo: `dev-to-backup` (Complejidad: 3, CI simple + 1 secret duplicado) — ✅ COMPLETO (2026-08-14)
- [x] ✅ Transfer repo a maiwei-app — confirmado, `repos/colomr-cc/dev-to-backup` redirige a `maiwei-app/dev-to-backup`
- [x] ✅ Verificar redirect URL antigua funciona — `https://github.com/colomr-cc/dev-to-backup` → 301 a `maiwei-app/dev-to-backup`
- [x] ✅ Asignar custom property `ci-profile=python-quality`, `sonar-enabled=true` — corregido: el plan original dejaba Sonar fuera, pero el org-wide ruleset ya exige el check "SonarCloud Code Analysis" en todo repo, así que se activó (ver PR #3)
- [x] ✅ Activar `delete_branch_on_merge` — verificado vía API
- [x] ✅ CI usa `ci-python.yml` + `sonar-scan.yml` reusables (PR #3, `maiwei-app/dev-to-backup#3`)
- [x] ✅ Verificar CI usa `DEVTO_API_KEY` org — confirmado: `GET /repos/maiwei-app/dev-to-backup/actions/secrets` devuelve `total_count: 0`, no hay override repo-level, cae al secret de org
- [x] ✅ Borrar secret `DEVTO_API_KEY` del repo — no aplica, nunca existió a nivel repo tras el transfer
- [x] ✅ Verificar ruleset org-wide aplica correctamente — confirmado, incluye ya el required status check "SonarCloud Code Analysis" (añadido 2026-08-14 08:32)
- [x] ✅ Fix bloqueante no contemplado en el plan original: `backup.yml` hacía `git push` directo a `main`; con el ruleset ya aplicando (sin bypass actors) el próximo run automático habría fallado. Reescrito para abrir PR en vez de pushear (PR #3) — revisión y merge manual mensual, sin auto-merge
- [x] ✅ Fix bloqueante en `maiwei-app/workflows`: `ci-python.yml` traía `actions/setup-python@<SHA inválido>`, resuelto en `c2f1224` antes de que esta PR corriera en verde
- [x] ✅ Quitado `--require-hashes` de `ci-python.yml` (`maiwei-app/workflows#8`) — bloqueaba el quality check y no había tooling (`pip-compile`) disponible; la integridad de dependencias se resolverá vía sem-ver + release-please, no hashes por repo
- [x] ✅ Smoke test: push → PR → checks verdes → review → merge → rama borrada (`maiwei-app/dev-to-backup#4`, docs fix de README)
- Nota: `README.md` actualizado para reflejar el flujo PR-based de `backup.yml` (ya no dice que commitea directo a `main`)

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
- ⚠️ **Bloqueado**: `repos/colomr-cc/chameleon` devuelve 404 tanto vía API con el token de instalación de `maibot-app` como en `repos/maiwei-app/chameleon`, y no aparece en el listado público de repos de `colomr-cc`. Al ser privado, puede ser que la instalación de la App en `colomr-cc` no tenga acceso concedido a este repo (`repository_selection: selected` sin incluirlo), o que el repo ya no exista/se haya renombrado. Requiere confirmación del desarrollador (permisos de admin) antes de continuar con este bloque.
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
