# Ponderix · Calculadora de notas ponderadas con CI/CD en Azure

<!-- Jhon reemplaza esta línea por la insignia de estado de la pipeline CI (Azure DevOps → pipeline ponderix-ci → ⋯ → Status badge → Markdown). -->
- **Organización / proyecto:** `ponderix-equipo` / `Ponderix`
- **Pipeline:** [ponderix-ci](https://dev.azure.com/ponderix-equipo/Ponderix/_build?definitionId=1)
- **Ejecución de referencia:** [Build #4](https://dev.azure.com/ponderix-equipo/Ponderix/_build/results?buildId=4&view=results)
- **Etapas:**
  - **Validar:** instalación de .NET SDK 10, restauración de dependencias (`restore`), compilación (`build`), ejecución de pruebas (`test`) y publicación de resultados y cobertura.
  - **Empaquetar:** publicación de la aplicación (`publish`), generación de `buildinfo.json`, creación del archivo `ponderix-<build>.zip` y publicación del artefacto `webapp`.

En las ejecuciones correspondientes a Pull Requests únicamente se ejecuta la etapa **Validar**. La etapa **Empaquetar** se ejecuta en las ejecuciones de la rama principal (`main`).

Práctica final de **Despliegue de Software 2026-2**: Integración Continua (CI) y Entrega Continua (CD) con **GitHub**, **Azure DevOps** y **Azure App Service**.

> **Flujo demostrado:** commit en `main` (GitHub) → pipeline **CI** (`ponderix-ci`) → artefacto **`webapp`** → pipeline **CD** (`ponderix-cd`) → **Azure App Service** → aplicación actualizada y accesible.

---

## 1. Equipo **Ponderix**

| # | Integrante | Usuario GitHub | Responsabilidad | Contribuciones verificables |
|---|---|---|---|---|
| 1 | Juan Sebastián Echeverri Gallego | [@JuanSebastianEcheverriGallego](https://github.com/JuanSebastianEcheverriGallego) | Gestión de GitHub, aplicación base y documentación | Repositorio, protección de `main`, acceso del docente, código de la aplicación, README, plantilla de PR. [PR #1](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/1) |
| 2 | David Stiven Franco López | [@Davdfranco](https://github.com/Davdfranco) | Pruebas automatizadas y calidad | Proyecto `tests/Ponderix.Tests` (pruebas unitarias y de integración), registro en la solución, guía de pruebas. [PR #2](https://github.com/JuanSebastianEcheverriGallego/PonderixOficial/pull/2) |
| 3 | Jhon Fernando Sánchez Álvarez | [@Jhonzn](https://github.com/Jhonzn) | Configuración de CI en Azure DevOps | `pipelines/azure-pipelines-ci.yml`, conexión GitHub ↔ Azure DevOps, disparador en `main`, artefacto `webapp`, validación de PR. [PENDIENTE: enlace al PR] |
| 4 | Marbel Juliana Mejía Bedoya | [@JULIANA-s22](https://github.com/JULIANA-s22) | Configuración de CD y Azure | App Service, Service Connection, entorno `ponderix-produccion`, `pipelines/azure-pipelines-cd.yml`, `scripts/crear-recursos-azure.*`. [PR #7](https://github.com/DespliegueOrganizacion/PonderixOficial/pull/7) |
| 5 | Marlon García Sepúlveda | [@M3rl0n](https://github.com/M3rl0n) | Verificación extremo a extremo, evidencias y videos | Cambio visible v1.1.0, `scripts/verificar-despliegue.*`, `scripts/buscar-secretos.*`, carpeta `evidencias/`, coordinación de videos. [PENDIENTE: enlace al PR] |

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

[PENDIENTE (Jhon): nombre de la organización/proyecto de Azure DevOps, enlace a la pipeline `ponderix-ci`, enlace a una ejecución exitosa y explicación de las etapas.]

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

## 11. URL pública

**<https://ponderix-equipo1.azurewebsites.net>**

- Salud: <https://ponderix-equipo1.azurewebsites.net/health>
- Versión y commit publicados: <https://ponderix-equipo1.azurewebsites.net/api/version>

## 12. Trazabilidad del cambio demostrado (GitHub → CI → CD → Azure)

| Paso | Evidencia |
|---|---|
| Pull Request del cambio visible (v1.1.0) | [PENDIENTE: enlace al PR] |
| Commit en `main` | [PENDIENTE: SHA y enlace] |
| Ejecución de CI | [PENDIENTE: número de ejecución y enlace] |
| Artefacto `webapp` | [PENDIENTE: ponderix-<build>.zip] |
| Ejecución de CD | [PENDIENTE: nombre «CD de CI <build>» y enlace] |
| Versión publicada | [PENDIENTE: salida de `/api/version` con el mismo commit] |

## 13. Evidencias

Capturas organizadas en [`evidencias/`](evidencias/README.md) por actividad (repositorio, CI, CD, extremo a extremo y equipo).

## 14. Videos

| Video | Enlace |
|---|---|
| Video 1 · Integración Continua (CI) | [PENDIENTE: ENLACE DE YOUTUBE] |
| Video 2 · Entrega Continua (CD) | [PENDIENTE: ENLACE DE YOUTUBE] |

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

## 17. Licencia y uso académico

Proyecto académico para la asignatura Despliegue de Software (2026-2).
