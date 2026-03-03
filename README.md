# Guía de Deploy: SAPUI5 Adaptation Projects (On-Premise)

## Tabla de Contenidos

1. [Contexto](#contexto)
2. [Problemas Conocidos](#problemas-conocidos)
3. [Solución: ui5-deploy.yaml Manual](#solución-ui5-deployyaml-manual)
4. [Plantilla Base](#plantilla-base)
5. [Cómo Completar la Plantilla](#cómo-completar-la-plantilla)
6. [Configuración del package.json](#configuración-del-packagejson)
7. [Ejecución del Deploy](#ejecución-del-deploy)
8. [Notas SAP Relacionadas](#notas-sap-relacionadas)

---

## Contexto

Al trabajar con **SAPUI5 Adaptation Projects** en SAP Business Application Studio (BAS) o VS Code con extensiones SAP, el proceso estándar para configurar el deploy es:

1. Abrir **Application Info** (`Ctrl+Shift+P` → "Open Application Info")
2. Hacer clic en **Add Deploy Configuration**
3. Completar el wizard del **Deployment Configuration Generator**

Este wizard genera automáticamente un archivo `ui5-deploy.yaml` con la configuración necesaria para desplegar el App Variant al sistema ABAP On-Premise.

**Sin embargo**, existen errores conocidos que impiden generar este archivo de forma automática, obligando a crearlo manualmente.

---

## Problemas Conocidos

### Problema 1: Error del Generador ADP (CLOUD_READY_AND_ON_PREM)

**Error:**
```
@sap/fiori:adp generator failed - {"message":"Could not update method 'when' in
'projectType' question in generator '@sap/fiori:adp' - evaluateMethod() Cannot
read properties of undefined (reading 'CLOUD_READY_AND_ON_PREM')"}
```

**Causa:** Bug en el generador `@sap/generator-fiori` (versiones 1.20.x) que no puede resolver el tipo de proyecto al evaluar si es Cloud, On-Premise o ambos.

**Nota SAP:** [3722308](https://me.sap.com/notes/3722308) — *Error encountered during SAPUI5 adaptation project generation in SAP Business Application Studio*

**Workarounds:**
- Crear un **nuevo Dev Space** en BAS (el fix de SAP requiere un espacio limpio).
- Usar el template **"(Legacy) SAPUI5 Adaptation Project"** en vez del generador estándar al crear nuevos proyectos desde `File > New Project from Template`.

**Nota:** Incluso con el workaround del Legacy wizard, es posible que aparezca el segundo problema al intentar configurar el deploy.

---

### Problema 2: cloudReady vs onPremise

**Error:**
```
You cannot deploy a cloudReady project because the system only supports onPremise projects.
```

**Causa:** El Deployment Configuration Generator detecta incorrectamente el proyecto como `cloudReady` cuando el sistema destino es On-Premise. Esto fue introducido por un cambio reciente en las herramientas de BAS.

**Nota SAP:** [3651973](https://me.sap.com/notes/3651973) — *Error "the provided package is not intended for on-premise deployments" during adaptation project deployment in SAP BAS*

**Estado:** SAP indica que el fix está disponible en la última versión de BAS, pero en la práctica puede seguir ocurriendo dependiendo de la versión del Dev Space y del generador.

**Solución definitiva:** Crear el archivo `ui5-deploy.yaml` de forma **manual**. Esta es la solución más confiable y la que se documenta a continuación.

---

### Problema 3: Error 400 al seleccionar sistema y aplicación

**Error:**
```
Could not load applications: Request failed with status code 400
```

**Causa:** Efecto secundario del fix aplicado por SAP para el Problema 1. Incluso creando un nuevo Dev Space, el paso de selección de sistema puede fallar.

**Solución:** Crear el `ui5-deploy.yaml` manualmente (ver siguiente sección).

---

## Solución: ui5-deploy.yaml Manual

Cuando los wizards no funcionan, el archivo `ui5-deploy.yaml` se crea manualmente en la **raíz del proyecto** (al mismo nivel que `ui5.yaml` y `package.json`).

### Prerequisitos

1. **Paquete ABAP (Package):** Tener creado un paquete Z en el sistema destino (ej: `Z_ADAPTATION_PROJECT`).
2. **Orden de transporte (Transport Request):** Tener una orden de tipo Workbench abierta (ej: `NPLK900001`).
3. **Destination configurada:** El destination en BAS/subaccount debe apuntar al sistema On-Premise.
4. **Dependencia `@sap/ux-ui5-tooling`:** Ya incluida en tu `package.json` (proporciona la task `deploy-to-abap`).

---

## Plantilla Base

Crea el archivo `ui5-deploy.yaml` en la raíz del proyecto con el siguiente contenido:

```yaml
# yaml-language-server: $schema=https://sap.github.io/ui5-tooling/schema/ui5.yaml.json

specVersion: "4.0"
metadata:
  name: <APP_NAME_CON_GUIONES>
type: application
resources:
  configuration:
    propertiesFileSourceEncoding: UTF-8
builder:
  customTasks:
    - name: ui5-tooling-transpile-task
      afterTask: replaceVersion
      configuration:
        debug: true
        omitSourceMaps: true
        omitTSFromBuildResult: true
        transformModulesToUI5:
          overridesToOverride: true
    - name: deploy-to-abap
      afterTask: generateCachebusterInfo
      configuration:
        target:
          destination: <DESTINATION_NAME>
          url: <SYSTEM_URL>
          client: '<CLIENT>'
        app:
          package: <ABAP_PACKAGE>
          transport: <TRANSPORT_REQUEST>
        exclude:
          - /test/
          - .*\.ts
          - .*\.map
          - .*tsconfig\.json$
          - .*package\.json$
          - .*package-lock\.json$
          - .*ui5\.yaml$
          - .*\.gitignore$
          - .*\.Ui5RepositoryTextFiles$
        lrep: <LREP_NAMESPACE>
  resources:
    excludes:
      - /test/**
      - /localService/**
```

> **Nota sobre TypeScript:** El bloque `ui5-tooling-transpile-task` solo es necesario si usas TypeScript en tu proyecto. Si tu proyecto es solo JavaScript, puedes omitirlo.

---

## Cómo Completar la Plantilla

Todos los valores se extraen de dos fuentes: el archivo `manifest.appdescr_variant` y la configuración de tu sistema.

### Fuente 1: manifest.appdescr_variant

Abre el archivo `webapp/manifest.appdescr_variant` y localiza estos campos:

```json
{
  "reference": "ui.ssuite.s2p.mm.pur.po.manage.st.s1",
  "id": "customer.ui.ssuite.s2p.mm.pur.po.manage.st.s1",
  "namespace": "apps/ui.ssuite.s2p.mm.pur.po.manage.st.s1/appVariants/customer.ui.ssuite.s2p.mm.pur.po.manage.st.s1/"
}
```

### Mapeo de campos

| Placeholder en YAML | Fuente | Regla de Transformación | Ejemplo |
|---|---|---|---|
| `<APP_NAME_CON_GUIONES>` | Campo `id` del manifest | Reemplaza **todos los puntos** (`.`) por **guiones** (`-`) | `customer-ui-ssuite-s2p-mm-pur-po-manage-st-s1` |
| `<LREP_NAMESPACE>` | Campo `namespace` del manifest | Copia tal cual, **incluyendo la barra final** (`/`) | `apps/ui.ssuite.s2p.mm.pur.po.manage.st.s1/appVariants/customer.ui.ssuite.s2p.mm.pur.po.manage.st.s1/` |

### Regla del Name (Importante)

El campo `metadata.name` en el YAML **NO acepta puntos**. UI5 Tooling usa el name como identificador interno y los puntos causan conflictos de resolución de módulos.

```
❌ name: customer.ui.ssuite.s2p.mm.pur.po.manage.st.s1
✅ name: customer-ui-ssuite-s2p-mm-pur-po-manage-st-s1
```

Transformación: toma el `id` del manifest y reemplaza `.` por `-`.

### Regla del LREP (Importante)

El campo `lrep` le dice al deployer **dónde almacenar los Flex Changes** en el repositorio LREP del sistema ABAP. Debe coincidir **exactamente** con el `namespace` del manifest, incluyendo la barra `/` al final.

```
namespace del manifest: apps/ui.ssuite.s2p.mm.pur.po.manage.st.s1/appVariants/customer.ui.ssuite.s2p.mm.pur.po.manage.st.s1/
                                                                    ↓
lrep en YAML:           apps/ui.ssuite.s2p.mm.pur.po.manage.st.s1/appVariants/customer.ui.ssuite.s2p.mm.pur.po.manage.st.s1/
```

Si el `lrep` no coincide, los Flex Changes (fragments, controller extensions, addXML, codeExt, moveControls) no se registrarán correctamente y la app desplegada no mostrará tus extensiones.

### Fuente 2: Configuración del Sistema

| Placeholder | Dónde obtenerlo | Ejemplo |
|---|---|---|
| `<DESTINATION_NAME>` | BTP Cockpit → Destinations (o BAS Settings → Destinations) | `S4HANA22` |
| `<SYSTEM_URL>` | URL del sistema On-Premise (protocolo + host + puerto) | `http://s4h71.sap4practice.com:8071` |
| `<CLIENT>` | Mandante SAP (string entre comillas simples) | `'2020'` |
| `<ABAP_PACKAGE>` | Paquete Z creado en SE80 o ADT | `Z_ADAPTATION_PROJECT` |
| `<TRANSPORT_REQUEST>` | Orden de transporte Workbench (SE09/SE10) | `NPLK900001` |

### Ejemplo Completo (valores reales)

```yaml
specVersion: "4.0"
metadata:
  name: customer-ui-ssuite-s2p-mm-pur-po-manage-st-s1
type: application
resources:
  configuration:
    propertiesFileSourceEncoding: UTF-8
builder:
  customTasks:
    - name: ui5-tooling-transpile-task
      afterTask: replaceVersion
      configuration:
        debug: true
        omitSourceMaps: true
        omitTSFromBuildResult: true
        transformModulesToUI5:
          overridesToOverride: true
    - name: deploy-to-abap
      afterTask: generateCachebusterInfo
      configuration:
        target:
          destination: S4HANA25
          url: http://s4h71.sap4practice.com:8071
          client: '220'
        app:
          package: Z_ADAPTATION_PROJECT
          transport: WALK900795
        exclude:
          - /test/
          - .*\.ts
          - .*\.map
          - .*tsconfig\.json$
          - .*package\.json$
          - .*package-lock\.json$
          - .*ui5\.yaml$
          - .*\.gitignore$
          - .*\.Ui5RepositoryTextFiles$
        lrep: apps/ui.ssuite.s2p.mm.pur.po.manage.st.s1/appVariants/customer.ui.ssuite.s2p.mm.pur.po.manage.st.s1/
  resources:
    excludes:
      - /test/**
      - /localService/**
```

---

## Configuración del package.json

Agrega el script de deploy en tu `package.json`:

```json
{
  "scripts": {
    "build": "ui5 build --exclude-task generateFlexChangesBundle generateComponentPreload minify --clean-dest",
    "deploy": "npm run build && fiori deploy --config ui5-deploy.yaml",
    "start": "fiori run --open /test/flp.html#app-preview",
    "start-editor": "fiori run --open /test/adaptation-editor.html"
  }
}
```

El script `deploy` hace dos cosas en secuencia:
1. `npm run build` — Compila el proyecto (transpila TypeScript, genera dist/).
2. `fiori deploy --config ui5-deploy.yaml` — Despliega al sistema ABAP usando la configuración manual.

---

## Ejecución del Deploy

```bash
npm run deploy
```

Durante la ejecución:
1. El transpilador convierte `.ts` a `.js` y excluye los archivos TypeScript del resultado.
2. El deployer sube los artifacts al repositorio SAPUI5 ABAP en el paquete indicado.
3. Los Flex Changes se almacenan en el namespace LREP configurado.
4. Se te pedirá usuario y contraseña del sistema ABAP si no están cacheadas.

### Verificación post-deploy

1. Accede al **Fiori Launchpad** del sistema destino.
2. Busca tu App Variant en el catálogo o navega directamente.
3. Verifica que las extensiones (secciones custom, controller extensions) se cargan correctamente.
4. Si las extensiones no aparecen, revisa el `lrep` en el YAML — es la causa más común.

---

## Notas SAP Relacionadas

| Nota | Título | Problema | Estado |
|---|---|---|---|
| [3722308](https://me.sap.com/notes/3722308) | Error encountered during SAPUI5 adaptation project generation in SAP BAS | Error `CLOUD_READY_AND_ON_PREM` al generar proyecto ADP | Workaround: Legacy wizard + nuevo Dev Space |
| [3651973](https://me.sap.com/notes/3651973) | Error "the provided package is not intended for on-premise deployments" | Deploy wizard rechaza paquetes On-Premise | Fix en última versión de BAS (puede requerir nuevo Dev Space) |

### Recomendación General

Dado que los problemas del generador automático son recurrentes y dependen de versiones específicas de BAS y del generador `@sap/generator-fiori`, la creación manual del `ui5-deploy.yaml` es la opción **más estable y predecible**. Una vez configurado correctamente, el archivo no necesita modificaciones salvo para cambiar la orden de transporte entre despliegues.

---

## Checklist Rápida

- [ ] `manifest.appdescr_variant` tiene `id`, `reference` y `namespace` correctos.
- [ ] `ui5-deploy.yaml` existe en la raíz del proyecto.
- [ ] `metadata.name` usa guiones (no puntos).
- [ ] `lrep` coincide exactamente con el `namespace` del manifest (incluida la `/` final).
- [ ] Destination configurada y accesible desde BAS.
- [ ] Paquete Z existe en el sistema destino.
- [ ] Orden de transporte abierta y de tipo Workbench.
- [ ] Script `deploy` agregado en `package.json`.
- [ ] Si usas TypeScript: `ui5-tooling-transpile-task` presente en el YAML.
