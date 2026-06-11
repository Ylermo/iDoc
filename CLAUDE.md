# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**iDoc** is a Java EE 7 web application (JSF + PrimeFaces) for the Municipalidad
Distrital de Nuevo Imperial, integrating document management (STD — Sistema de
Trámite Documentario) and citizen-relations modules (SRC). The codebase and
domain language are **Spanish**: class names, variable names, UI text, and
database columns are all in Spanish, and new code should follow that convention.

Packaged as a `.war` and deployed to a Java EE container (GlassFish v3 is the
configured target; the app context path is `/iDoc`). The data layer talks to a
remote **SQL Server** database over JDBC.

## Build & run

This is a Maven project. There is **no test suite** — there are no `src/test`
sources, so `mvn test` does nothing useful.

```bash
mvn clean package          # compile + build target/iDoc-1.0-SNAPSHOT.war
mvn clean install          # same, plus install to local repo
```

Notes:
- Java source/target level is **1.7** (see `pom.xml` compiler plugin). Stay
  within Java 7 language features.
- `failOnMissingWebXml` is false, but a `web.xml` is present.
- The `maven-dependency-plugin` copies the `javaee-endorsed-api` into
  `target/endorsed` during the `validate` phase — required for the Java EE 7 API
  to compile against an older JDK.
- PrimeFaces 7.0 is pulled from the custom `repository.primefaces.org` repo
  declared in `pom.xml`.
- NetBeans is the intended IDE (`nb-configuration.xml`, `nbactions.xml`). The
  `run`/`debug` actions deploy the WAR to the configured GlassFish server.

There is no linter, formatter config, or CI configured in the repo.

## Architecture

Classic 3-tier JSF MVC, with one vertical slice per domain entity (Documento,
Persona, Empresa, Trabajador, Acta, Tupa, Incidencia, Transferencia, etc.).
For each entity there is typically a model + DAO + controller + Facelet page.

```
src/main/java/
  modelo/        Plain POJOs (domain models). Fields use UPPERCASE DB-column
                 names (e.g. IDDOC, CODDOC). Related entities are nested object
                 fields (Documento has Tupa, Login, Empresa, Persona...). Many
                 models expose a no-op/reset clear() used by controllers.
  dao/           Data access. *Impl classes extend Conexion and implement
                 ICrud<T> (and IReporte<T> / IDashboard where relevant).
  controlador/   JSF managed beans (the C suffix = "Controlador"). @Named +
                 @SessionScoped, one per entity. Hold the model + selected
                 model + list, call the matching DAO, push FacesMessages.
  servicios/     Cross-cutting helpers (EncriptarS — hashing/encoding).
  websockets/    ServerW — JSR-356 @ServerEndpoint for live UI refresh.

src/main/webapp/
  Pages/         One .xhtml Facelet per entity (the views).
  Templates/     Default.xhtml master layout (Ultima/Materialize theme) +
                 reusable Municipalidad fragments.
  Reportes/      Compiled JasperReports (.jasper) grouped by entity.
  resources/     css/js/img/fonts/icons static assets (JSF resource library).
  WEB-INF/web.xml, META-INF/context.xml
```

### Data access layer (`dao/`)

- `ICrud<T>` defines the standard CRUD contract: `registrar`, `editar`,
  `eliminar`, `listar()`, `listar(T)`, `obtenerModelo(T)`. Most `*Impl` classes
  implement the methods they need and `throw new UnsupportedOperationException`
  for the rest — that is the established pattern, not a bug.
- `Conexion` is the JDBC base class. `*Impl` classes `extends Conexion` and call
  `this.conectar()` to get a shared static `Connection`, then `this.desconectar()`
  in a `finally`. **The SQL Server host, database, and credentials are hardcoded
  in `Conexion.conectar()`** — that is where to change the DB target.
- DAOs use raw `PreparedStatement` + hand-written SQL strings. Reads map
  `ResultSet` rows into nested model objects by hand. There is no ORM.
- **Deletes are soft deletes**: e.g. `UPDATE ... SET ESTDOC='I'` rather than
  `DELETE`. Listings filter with `WHERE EST... != 'I'`. Preserve this pattern.
- `IReporte<T>` / Jasper: report DAOs (e.g. `DocumentoImpl.generarReporteIndividual`)
  load a `.jasper` from `webapp/Reportes/...` via `FacesContext` real path, fill
  it against the live JDBC connection, and stream the PDF straight to the
  `HttpServletResponse` (`Content-disposition: attachment`).
- `ConexionAlfresco` (Alfresco/OpenCMIS document store) is fully commented out —
  dead/aspirational code; the related `pom.xml` dependencies are also commented.

### Controllers (`controlador/`)

- `@Named(value="xxxC") @SessionScoped implements Serializable`. Bound from
  Facelets as `#{xxxC...}`.
- Constructor initializes the model, selected model, DAO, and empty list;
  `@PostConstruct iniciar()` loads the initial listing.
- CRUD action methods wrap the DAO call in try/catch, add a `FacesMessage`
  (success or error), and — critically — trigger a WebSocket broadcast:
  `PrimeFaces.current().executeScript("enviar('Documento');")`.
- `LoginC` owns authentication/session: `iniciarSesion()` validates credentials
  and redirects to `Home.xhtml`; `seguridadSesion()` is a page guard that
  redirects unauthenticated users (IDLOG==0) back to `Login.xhtml`;
  `cerrarSesion()` clears the session map. Active records use status `'A'`.

### Real-time refresh (WebSockets)

This is the one non-obvious cross-cutting mechanism. When a controller mutates
data, it calls the JS `enviar('<EntityName>')` which sends the entity name over a
WebSocket to `ServerW` (`@ServerEndpoint("/ws/server")`). `ServerW.broadcast`
relays that string to **all** open sessions. The client `websocket.js`
`notificarActualizar()` switches on the entity name and calls the matching
`actualizar<Entity>()` function, which fires a PrimeFaces remote AJAX update
(`PrimeFaces.ab({...})`) to re-render that entity's data table for every logged-in
user. So: **adding a new CRUD entity that needs live updates requires three
coordinated edits** — the controller's `executeScript("enviar('X')")`, a `case "X"`
in `websocket.js`, and an `actualizarX()` function targeting the right form/table
component IDs.

### Views (`webapp/Pages` + `Templates/Default.xhtml`)

- Facelets/XHTML with PrimeFaces (`p:`) components, composed against the
  `Default.xhtml` master template (Ultima admin theme + Materialize CSS).
- `web.xml`: `Faces Servlet` mapped to `/faces/*`; welcome page is
  `faces/Pages/Login.xhtml`; `PROJECT_STAGE` is `Development`; PrimeFaces theme
  is `none` (styling comes from the bundled CSS, not a PrimeFaces theme).
- The `PrimeFaces.ab(...)` calls in `websocket.js` reference generated component
  client IDs (e.g. `j_idt110:btnActualizar...`). These are brittle — if you
  restructure a page's component tree, the auto-generated `j_idt*` IDs can shift
  and silently break live refresh.

## Conventions

- **Language**: Spanish throughout. New classes follow the suffix scheme —
  `*C` = controller, `*Impl` = DAO implementation, `I*` = interface,
  `*S` = service. Models are the bare entity name.
- **DB column naming**: short UPPERCASE codes (`IDDOC`, `RAZSOCEMP`, `ESTDOC`),
  mirrored as model field names.
- **Status columns** (`EST...`): `'A'` = activo, `'I'` = inactivo (soft-deleted).
- Passwords are hashed with **MD5** (`EncriptarS.encriptarPssw`); documents get a
  single check-character via `EncriptarS.encriptarDocumento`.
- Error handling is overwhelmingly `try/catch { e.printStackTrace(); }` —
  consistent with the codebase, though not robust.

## Security note (existing state)

The repository currently commits **plaintext production-looking DB credentials**
in `dao/Conexion.java` and uses MD5 for password hashing. Be aware of this when
touching those files; do not propagate new secrets into source, and flag rather
than silently replicate these patterns if asked to harden the app.
