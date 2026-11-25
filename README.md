# Ponderix · Calculadora de notas ponderadas con CI/CD en Azure

[![Build Status](https://dev.azure.com/ponderix-equipo/Ponderix/_apis/build/status%2Fponderix-ci?branchName=main)](https://dev.azure.com/ponderix-equipo/Ponderix/_build/latest?definitionId=1&branchName=main)


Práctica final de **Despliegue de Software 2026-2**: Integración Continua (CI) y Entrega Continua (CD) con **GitHub**, **Azure DevOps** y **Azure App Service**.

> **Flujo demostrado:** commit en `main` (GitHub) → pipeline **CI** (`ponderix-ci`) → artefacto **`webapp`** → pipeline **CD** (`ponderix-cd`) → **Azure App Service** → aplicación actualizada y accesible.

---

## 1. Equipo **Ponderix**

| # | Integrante | Usuario GitHub | Responsabilidad | Contribuciones verificables |
|---|---|---|---|---|
| 1 | Juan Sebastián Echeverri Gallego | [@JuanSebastianEcheverriGallego](https://github.com/JuanSebastianEcheverriGallego) | Gestión de GitHub, aplicación base y documentación | Repositorio, protección de `main`, acceso del docente, código de la aplicación, README, plantilla de PR. [PR #1](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/1) · [PR #6](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/6) |
| 2 | David Stiven Franco López | [@Davdfranco](https://github.com/Davdfranco) | Pruebas automatizadas y calidad | Proyecto `tests/Ponderix.Tests` (pruebas unitarias y de integración), registro en la solución, guía de pruebas y capturas de evidencia de la CI. [PR #2](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/2) · [PR #4](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/4) · [PR #14](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/14) |
| 3 | Jhon Fernando Sánchez Álvarez | [@Jhonzn](https://github.com/Jhonzn) | Configuración de CI en Azure DevOps | `pipelines/azure-pipelines-ci.yml`, conexión GitHub ↔ Azure DevOps, disparador en `main`, artefacto `webapp`, validación de PR. [PR #3](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/3) · [PR #5](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/5) · [PR #12](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/12) |
| 4 | Marbel Juliana Mejía Bedoya | [@JULIANA-s22](https://github.com/JULIANA-s22) | Configuración de CD y Azure | App Service, Service Connection, entorno `ponderix-produccion`, `pipelines/azure-pipelines-cd.yml`, `scripts/crear-recursos-azure.*`. [PR #7](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/7) · [PR #8](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/8) · [PR #9](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/9) |
| 5 | Marlon García Sepúlveda | [@M3rl0n](https://github.com/M3rl0n) | Verificación extremo a extremo, evidencias y videos | Cambio visible v1.1.0, `scripts/verificar-despliegue.*`, `scripts/buscar-secretos.*`, carpeta `evidencias/`, coordinación de videos. [PR #10](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/10) · [PR #11](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/11) · [PR #13](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/13) |

El detalle de quién hizo qué se puede comprobar en la pestaña **Pull requests → Closed** y en `git log --merges`.

---

## 2. Descripción de la aplicación

**Ponderix** es una aplicación web + API REST en **ASP.NET Core 10 (Minimal API)** que calcula notas ponderadas en escala 0.0 – 5.0 y las clasifica con los niveles de la rúbrica de la práctica (Excelente ≥ 4.5, Bueno ≥ 3.5, Aceptable ≥ 2.5, Deficiente < 2.5). Incluye plantillas con los criterios y pesos de las rúbricas de CI, CD y nota final.

| Método | Ruta | Uso |
|---|---|---|
| GET | `/` | Interfaz web (HTML + JavaScript) |
| GET | `/health` | Comprobación de salud (`Healthy`). La usan Azure y la pipeline de CD |
| GET | `/api/version` | Versión, número de ejecución de CI y **commit** desplegado (trazabilidad) |
| GET | `/api/rubricas` | Plantillas de criterios (CI, CD, final) |
| POST | `/api/notas/calcular` | Calcula la nota ponderada; devuelve `400` con errores si los datos no son válidos |

Ejemplo:

```bash
curl -X POST http://localhost:5080/api/notas/calcular \
  -H "Content-Type: application/json" \
  -d '{"criterios":[{"nombre":"Nota CI","peso":50,"nota":4.5},{"nombre":"Nota CD","peso":50,"nota":5.0}]}'
# {"notaFinal":4.75,"nivel":"Excelente","detalle":[...]}
```

## 3. Tecnologías

| Ámbito | Tecnología |
|---|---|
| Lenguaje / framework | C# · .NET 10 (LTS) · ASP.NET Core Minimal API |
| Pruebas | xUnit · Microsoft.AspNetCore.Mvc.Testing (pruebas de integración) · coverlet (cobertura) |
| Repositorio | GitHub (rama principal `main`, ramas `feature/*`, Pull Requests) |
| CI / CD | Azure Pipelines (YAML) en Azure DevOps · agentes `ubuntu-latest` |
| Hospedaje | Azure App Service · Linux · pila `DOTNETCORE|10.0` |

## 4. Arquitectura y flujo CI/CD

```mermaid
flowchart LR
    A[Commit / PR fusionado en main<br/>GitHub] -->|trigger: main| B[CI · ponderix-ci<br/>Validar → Empaquetar]
    B -->|artefacto webapp<br/>ponderix-BUILD.zip + buildinfo.json| C[CD · ponderix-cd<br/>recurso de pipeline]
    C -->|AzureWebApp@1<br/>Service Connection| D[Azure App Service<br/>Linux .NET 10]
    D -->|/health y /api/version| E[Prueba de humo<br/>commit publicado = commit de CI]
```

- **CI** (`pipelines/azure-pipelines-ci.yml`): se dispara con cada push a `main` y valida cada PR hacia `main`.
  - Etapa **Validar**: instala .NET 10 → `dotnet restore` → `dotnet build` → `dotnet test` (publica resultados y cobertura).
  - Etapa **Empaquetar** (solo en `main`, no en PR): `dotnet publish` → genera `buildinfo.json` (número de ejecución, commit, rama, fecha) → comprime `ponderix-<build>.zip` → publica el artefacto **`webapp`** y etiqueta la ejecución con `commit-<sha corto>`.
- **CD** (`pipelines/azure-pipelines-cd.yml`): no tiene disparador por código (`trigger: none`); declara la CI como **recurso de pipeline** y se ejecuta automáticamente cuando `ponderix-ci` termina con éxito en `main`. Descarga el artefacto `webapp` **de esa ejecución** (no recompila), lo despliega con `AzureWebApp@1` y hace una prueba de humo que exige que `/api/version` devuelva el mismo commit que construyó la CI.

## 5. Estructura del repositorio

```text
.
├── .github/pull_request_template.md   Plantilla de Pull Request
├── evidencias/                        Capturas de CI, CD y extremo a extremo
├── pipelines/
│   ├── azure-pipelines-ci.yml         Pipeline de CI
│   └── azure-pipelines-cd.yml         Pipeline de CD
├── scripts/
│   ├── crear-recursos-azure.(sh|ps1)  Creación del App Service con Azure CLI
│   ├── verificar-despliegue.(sh|ps1)  Verificación funcional de la URL pública
│   └── buscar-secretos.(sh|ps1)       Búsqueda de credenciales en archivos e historial
├── src/Ponderix.Api/                  Aplicación (API + wwwroot)
├── tests/Ponderix.Tests/              Pruebas unitarias y de integración
├── CHANGELOG.md
├── CONTRIBUTING.md                    Flujo de ramas, commits y PR del equipo
├── global.json                        Fija el SDK .NET 10
└── Ponderix.slnx                      Solución
```

## 6. Rama principal y forma de trabajo

- Rama principal: **`main`** (protegida: cambios solo mediante Pull Request con 1 aprobación y la CI en verde).
- Cada integrante trabaja en `feature/<nombre>-<tema>` y abre un PR hacia `main`. Los PR se fusionan con **«Create a merge commit»** para conservar los commits individuales y su autoría.
- Detalle completo en [CONTRIBUTING.md](CONTRIBUTING.md).

## 7. Ejecución local

**Requisitos:** [.NET SDK 10](https://dotnet.microsoft.com/download/dotnet/10.0) y Git. Opcional: VS Code (extensión C# Dev Kit) o Visual Studio 2026.

```bash
git clone https://github.com/DespliegueOrganizacion/PonderixOficial.git
cd PonderixOficial
dotnet --version            # debe mostrar 10.0.x
dotnet restore Ponderix.slnx
dotnet build Ponderix.slnx
dotnet run --project src/Ponderix.Api --launch-profile http
```

Abrir <http://localhost:5080>. En local, `/api/version` muestra `build: "local"` y `commit: "local"` porque el archivo `buildinfo.json` solo lo genera la pipeline de CI.

## 8. Pruebas automatizadas

```bash
dotnet test Ponderix.slnx
# Con cobertura:
dotnet test Ponderix.slnx --collect "XPlat Code Coverage"
```

El proyecto `tests/Ponderix.Tests` contiene **25 pruebas** (xUnit):

- **Unitarias (19)** — `CalculadoraNotasTests`: promedio ponderado y redondeo, tolerancia de pesos (33.33 + 33.33 + 33.34),
  límites exactos de cada nivel (4.5, 3.5, 2.5) y validaciones (sin criterios, pesos que no suman 100, notas fuera de 0–5, nombre vacío).
- **Integración (6)** — `ApiTests`: levanta la API en memoria con `WebApplicationFactory` y comprueba `/health`,
  `/api/version`, `/api/rubricas`, `POST /api/notas/calcular` (200 y 400) y que `/` sirve la interfaz.

En la CI se ejecutan en la etapa **Validar**; si alguna falla, la etapa **Empaquetar** no se ejecuta y no hay despliegue.

**Alcance de la comprobación de salud:** la prueba de `/health` verifica que la API arranca y responde `Healthy`. Es una comprobación de disponibilidad del proceso; no valida servicios externos, porque la aplicación no depende de base de datos ni de almacenamiento persistente.

Ejecución local: `dotnet test Ponderix.slnx` → 25 superadas, 0 con error.

## 9. Pipeline de CI

- **Organización / proyecto:** `ponderix-equipo` / `Ponderix`
- **Pipeline:** [ponderix-ci](https://dev.azure.com/ponderix-equipo/Ponderix/_build?definitionId=1)
- **Ejecución de referencia:** [Build #4](https://dev.azure.com/ponderix-equipo/Ponderix/_build/results?buildId=4&view=results)
- **Primera ejecución en `main`:** [20261119.2](https://dev.azure.com/ponderix-equipo/Ponderix/_build/results?buildId=2) (merge del PR #3; *Individual CI*; 25/25 pruebas). Evidencia: `evidencias/02-ci/02-pruebas-en-pipeline.png`.
- **Ejecución de la demostración extremo a extremo:** [20261124.1](https://dev.azure.com/ponderix-equipo/Ponderix/_build/results?buildId=23) (ver sección 12).
- **Etapas:**
  - **Validar:** instalación de .NET SDK 10, restauración de dependencias (`restore`), compilación (`build`), ejecución de pruebas (`test`) y publicación de resultados y cobertura.
  - **Empaquetar:** publicación de la aplicación (`publish`), generación de `buildinfo.json`, creación del archivo `ponderix-<build>.zip` y publicación del artefacto `webapp`.

- Archivo: [`pipelines/azure-pipelines-ci.yml`](pipelines/azure-pipelines-ci.yml)
- Disparador: push a `main` (y validación de PR hacia `main`).
- **Artefacto:** nombre `webapp`; contiene `ponderix-<número de ejecución>.zip` y `buildinfo.json`.

## 10. Pipeline de CD y Azure App Service

- Organización / proyecto: `ponderix-equipo` / `Ponderix`
- Pipeline: [`ponderix-cd`](https://dev.azure.com/ponderix-equipo/Ponderix/_build) (definida en `pipelines/azure-pipelines-cd.yml`)
- Despliegue exitoso de referencia: ejecución «CD de CI 20261121.2», disparada por la ejecución `20261121.2` de `ponderix-ci` en `main` (commit `31d8282`). Pasos en verde: descarga del artefacto `webapp`, trazabilidad, despliegue con `AzureWebApp@1` y prueba de humo.
- Ejecución automática: «CD de CI 20261122.2», disparada sola por `ponderix-ci` tras fusionar el PR #8 en `main`, sin intervención manual. Evidencia: `evidencias/03-cd/08-cd-automatica.png`.
- Funcionamiento: `ponderix-cd` no reacciona a commits ni PR; se activa sola cuando `ponderix-ci` termina con éxito en `main`, descarga **ese** artefacto sin recompilar y lo publica en App Service. La prueba de humo exige `/health` = `Healthy` y que `/api/version` devuelva el commit construido por la CI.
- Creación reproducible del recurso: `scripts/crear-recursos-azure.ps1` / `.sh`.

| Configuración | Valor |
|---|---|
| Suscripción | Azure for Students |
| Grupo de recursos | `rg-ponderix` |
| Plan de App Service | `asp-ponderix` · Linux · F1 |
| Web App | `ponderix-equipo1` |
| Región | México Central (`mexicocentral`) |
| Pila | `DOTNETCORE|10.0` · comando de inicio `dotnet Ponderix.Api.dll` |
| Seguridad | HTTPS Only · TLS 1.2 mínimo · FTP deshabilitado · ruta de salud `/health` |
| Service Connection | `sc-azure-ponderix` (Azure Resource Manager, federación de identidad de carga de trabajo) |
| Entorno de Azure DevOps | `ponderix-produccion` · **sin aprobación manual**: cada ejecución exitosa de CI en `main` se despliega sin intervención humana (automatización extremo a extremo). Si se añadiera *Approvals and checks → Approvals*, la CD quedaría en pausa («Waiting for approval») hasta que la persona aprobadora acepte: más control, menos automatización. |

### Almacenamiento y alcance de las comprobaciones

**Almacenamiento.** La aplicación no guarda datos: no usa base de datos ni escribe archivos de usuario. Lo único que lee del disco es el paquete publicado (binarios, `wwwroot`) y `buildinfo.json`, que genera la CI y es de solo lectura. El plan **F1** incluye 1 GB de almacenamiento y no tiene *Always On*: tras un periodo de inactividad la primera petición puede tardar (arranque en frío). Cada despliegue sustituye la versión anterior; no hay datos de usuario que conservar ni migrar.

| Comprobación | Qué verifica | Qué **no** verifica |
|---|---|---|
| `/health` (ruta de salud de Azure y prueba de humo de la CD) | Que el proceso arrancó y responde `Healthy` | Dependencias externas (la aplicación no tiene) ni la lógica de negocio |
| `/api/version` (prueba de humo de la CD) | Que lo publicado es el **commit** construido por la CI | Que los cálculos sean correctos |
| `scripts/verificar-despliegue.*` | Funcionalidad real en la URL pública: `POST /api/notas/calcular` devuelve 4.75 «Excelente» y rechaza datos inválidos | Rendimiento, carga ni escalado |
| Pruebas automatizadas (CI, 25) | Lógica de cálculo y endpoints, en memoria, antes de empaquetar | El entorno real de Azure |

## 11. URL pública

**<https://ponderix-equipo1.azurewebsites.net>**

- Salud: <https://ponderix-equipo1.azurewebsites.net/health>
- Versión y commit publicados: <https://ponderix-equipo1.azurewebsites.net/api/version>

## 12. Trazabilidad del cambio demostrado (GitHub → CI → CD → Azure)

| Paso | Evidencia |
|---|---|
| Pull Request del cambio visible (v1.1.0) | [PR #11 · Versión 1.1.0](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/11): autor Marlon, aprobado por Marbel, fusionado por Marlon el 23-nov-2026 a las 22:52 (hora de Colombia) con *Create a merge commit*. Validación del PR: ejecución `20261123.4` (solo Validar, 25/25 pruebas) · [PR](evidencias/04-extremo-a-extremo/02-pr-cambio-visible.png) · [validación](evidencias/04-extremo-a-extremo/02b-validacion-del-pr.png) |
| Commit en `main` | [`b788a22`](https://github.com/DespliegueOrganizacion/PonderixOficial/commit/b788a221339abe3a1480b70acdd415bffe8da3d2), merge commit con padres `548c80a` + `7cd73ec` · [captura](evidencias/04-extremo-a-extremo/03-commit-en-main.png) |
| Ejecución de CI | [`ponderix-ci` · 20261124.1](https://dev.azure.com/ponderix-equipo/Ponderix/_build/results?buildId=23): motivo *Individual CI*, rama `main`, etiqueta `commit-b788a22`, 25/25 pruebas, 2 min 43 s · [captura](evidencias/04-extremo-a-extremo/04-ci-del-commit.png) |
| Artefacto `webapp` | `ponderix-20261124.1.zip` + `buildinfo.json` con `commit` = `b788a221339abe3a1480b70acdd415bffe8da3d2` y `fechaBuild` = `2026-11-24T03:55:09Z` (22:55 en Colombia) |
| Ejecución de CD | [«CD de CI 20261124.1»](https://dev.azure.com/ponderix-equipo/Ponderix/_build/results?buildId=24): *Automatically triggered* por `ci/20261124.1` a las 22:55, 1 min 29 s · [captura](evidencias/04-extremo-a-extremo/05-cd-de-esa-ci.png) |
| Versión publicada | `/api/version` → `"version":"1.1.0"`, `"build":"20261124.1"`, `"commit":"b788a221339abe3a1480b70acdd415bffe8da3d2"` · [app](evidencias/04-extremo-a-extremo/06-app-despues-v1-1-0.png) · [API](evidencias/04-extremo-a-extremo/07-api-version-mismo-commit.png) · [`verificar-despliegue`: superada](evidencias/04-extremo-a-extremo/08-script-verificacion.png) |

Nota: el número de ejecución (`20261124.1`) usa fecha UTC, por eso lleva el día 24 aunque el merge fue el 23 a las 22:52 en Colombia (UTC−5).

Del merge (22:52) a la versión 1.1.0 publicada (~22:57) pasaron unos 4,5 minutos, sin ejecuciones manuales: la CI arrancó por *Individual CI* y la CD por el recurso de pipeline (*Automatically triggered*).

## 13. Evidencias

Capturas organizadas en [`evidencias/`](evidencias/README.md) por actividad (repositorio, CI, CD, extremo a extremo y equipo).

## 14. Videos

| Video | Enlace |
|---|---|
| Video 1 · Integración Continua (CI) | [Ver en YouTube](https://www.youtube.com/watch?v=Lmi-mGvwmCo) |
| Video 2 · Entrega Continua (CD) | [Ver en YouTube](https://www.youtube.com/watch?v=uPKBFfj24mI) |

## 15. Seguridad

- No hay contraseñas, tokens, cadenas de conexión ni perfiles de publicación en el repositorio. `.gitignore` bloquea `.env`, `*.publishsettings`, `*.pfx`, `secrets.json`, etc.
- El despliegue se autentica con una **Service Connection** de Azure DevOps (federación de identidad, sin secreto almacenado). Los únicos valores en el YAML son nombres de recursos, que no son secretos.
- La CD solo puede usar la Service Connection y el entorno tras autorizarlos explícitamente en Azure DevOps.
- `scripts/buscar-secretos.sh` / `.ps1` revisa archivos e historial; además se activan *Secret scanning* y *Push protection* de GitHub.
- **Si se sube un secreto por error:** 1) revocarlo o rotarlo de inmediato (PAT, perfil de publicación, token); 2) eliminarlo con un PR `fix(seguridad): …`; 3) si debe salir del historial, limpiar con `git filter-repo` de forma coordinada y pedir a todos que vuelvan a clonar; 4) registrarlo en la sección 16.

## 16. Problemas encontrados y soluciones

| # | Problema | Causa | Solución | Responsable |
|---|---|---|---|---|
| 1 | La región elegida inicialmente para el App Service (`brazilsouth`) no estaba permitida en la suscripción | La política *Allowed resource deployment regions* de Azure for Students solo admite `southcentralus`, `mexicocentral`, `canadacentral`, `centralus` y `westus3` | Consultar la política con `az policy assignment list` antes de crear recursos y usar una región permitida | Marbel |
| 2 | «Operation cannot be completed without additional quota» al crear el plan F1 en `canadacentral` y `centralus` (y B1 en `centralus`) | La suscripción tenía cuota 0 de instancias F1/B1 en esas regiones | Probar las regiones permitidas hasta encontrar cuota: F1 funcionó en `mexicocentral`. Se eliminó el grupo de recursos de prueba y se recreó todo con `scripts/crear-recursos-azure.ps1` en esa región | Marbel |
| 3 | «Access denied. ... needs Create permissions» al crear el entorno `ponderix-produccion` | La cuenta no tenía permisos de creación en Environments de Azure DevOps | Jhon agregó a Marbel a *Project Administrators*; después se creó el entorno | Marbel |
| 4 | La CD quedó en «Waiting · Permission needed» en la primera ejecución | Azure DevOps exige autorizar explícitamente el uso de la Service Connection y del entorno por parte de una pipeline nueva | Pulsar *View → Permit* para ambos recursos en la primera ejecución manual | Marbel |
| 5 | `dotnet build` falló con «A compatible .NET SDK was not found. Requested SDK version: 10.0.100»: en el equipo solo quedaba el SDK 11.0.100 | Una actualización reemplazó el SDK 10.0.400 por el 11.0.100 y `global.json` (10.0.100 con `rollForward: latestFeature`) no pasa de la versión 10 a la 11 | Instalar el SDK 10.0.404 junto al 11 (`winget install Microsoft.DotNet.SDK.10`); `dotnet --version` → 10.0.404 y 25/25 pruebas. No se cambió `global.json` para mantener la misma versión que la CI (`10.0.x`) | Marlon |
| 6 | En Windows PowerShell 5.1 los `.ps1` mostraban tildes rotas y el patrón de `buscar-secretos.ps1` (con comillas) llegaba mal a `git grep` | PowerShell 5.1 lee UTF-8 sin BOM como ANSI y no escapa las comillas al llamar a `git.exe` | Guardar los `.ps1` en UTF-8 con BOM y pasar el patrón a `git grep` desde un archivo temporal (`-f`); las evidencias se tomaron con Git Bash | Marlon |
| 7 | En la revisión del PR #9, la captura `08-cd-automatica.png.jpeg` tenía doble extensión y el README la enlazaba como `.png` (enlace roto) | El archivo se exportó con la extensión equivocada | Se renombró a `08-cd-automatica.png` antes de aprobar y fusionar el PR #9 | Marlon (revisión) |
| 8 | Al hacer `git add` de `Ponderix.slnx` apareció «CRLF will be replaced by LF» | En Windows los archivos usan CRLF y `.gitattributes` normaliza a LF | Es un aviso informativo; el commit se hizo y `git status` quedó limpio | David |
| 9 | Las capturas `03` y `04` de David chocaban con la numeración de `evidencias/README.md` (eran de Jhon) | Se nombraron sin revisar los números ya asignados | Se renombraron a `09` y `10` y se registraron en la tabla de evidencias antes de aprobar el PR #2 | David |
| 10 | `git push` mostró «This repository moved» | El repositorio se trasladó a la organización `DespliegueOrganizacion` | `git remote set-url origin https://github.com/DespliegueOrganizacion/PonderixOficial.git` y `git fetch` | David |
| 11 | «0 collaborators, 5 invitations»: nadie podía subir ramas | Las invitaciones de GitHub no dan permiso de escritura hasta aceptarlas | Cada integrante aceptó su invitación; se verificó al aprobar el PR #1 y subir sus ramas | Juan |
| 12 | El PR #1 quedó abierto del 15 al 17 de noviembre sin poder fusionarse | La protección de `main` exige una aprobación y el autor no puede aprobar su propio PR | David revisó, aprobó y fusionó el PR #1 el 17 de noviembre (merge `38c1a25`) | Juan |
| 13 | El nombre «Ponderix» ya estaba ocupado por el repositorio de respaldo del material | GitHub no permite dos repositorios cuyos nombres solo difieran en mayúsculas | Se creó el repositorio oficial como `PonderixOficial` y se actualizaron las guías con la URL nueva | Juan |
| 14 | Conflictos al integrar cambios en `main` | La rama local estaba desactualizada frente a los cambios de otros integrantes | Sincronizar con `origin/main` y revisar los cambios antes de integrar | Jhon |
| 15 | La rama local y la remota tenían historiales de commits diferentes (divergencia) | Commits distintos en ambas ramas | `git fetch` y revisión del historial para identificar las diferencias antes de sincronizar | Jhon |
| 16 | Un merge quedó pendiente de finalizar | La integración quedó sin confirmar | `git merge --abort` y nueva sincronización con `origin/main`; revisar `git status` antes de iniciar un merge | Jhon |

## 17. Licencia y uso académico

Proyecto académico para la asignatura Despliegue de Software (2026-2).
