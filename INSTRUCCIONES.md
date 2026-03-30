# INSTRUCCIONES (Proyectos-mql5)

Este documento describe cómo **instalar**, **compilar** y **trabajar** con el repositorio `lucasarenas01/Proyectos-mql5`, con foco en el **Framework v4.0**.

---

## 1) Requisitos

- MetaTrader 5 instalado
- MetaEditor (incluido con MT5)
- Acceso a la **carpeta de datos** de MT5 (File → Open Data Folder)

---

## 2) Instalación (copiar a la carpeta MQL5)

La estructura del repo está pensada para copiarse dentro de:

- `.../MQL5/Experts/`
- `.../MQL5/Include/`

### Pasos recomendados

1. Abre MT5 → **File → Open Data Folder**
2. Entra a la carpeta `MQL5/`
3. Copia desde este repo:
   - `Experts/` → a `MQL5/Experts/`
   - `Include/` → a `MQL5/Include/`

> Importante: mantén exactamente los nombres de carpetas (incluyendo espacios) para que los `#include` relativos funcionen.

---

## 3) Workflow recomendado (edición manual + push a GitHub)

Este repositorio se mantiene con un flujo de trabajo **manual**:

1. Se trabaja el código dentro de la carpeta de datos de MetaTrader 5 (`MQL5/`) usando MetaEditor.
2. Cuando hay cambios, se **copian/replican manualmente** los archivos modificados hacia el repositorio local `Proyectos-mql5/` (o viceversa).
3. Se versiona con git desde fuera de MetaTrader:
   - `git add`
   - `git commit`
   - `git push`

Motivo:
- MetaTrader/MetaEditor no tiene un flujo tipo VSCode con git integrado, así que no se asume que el repo viva “directamente” dentro de `MQL5/`.

---

## 4) Estructura espejo recomendada (PC)

En esta máquina se mantiene el código en dos ubicaciones, con estructura idéntica:

- Carpeta real de MetaTrader 5:
  - `.../MQL5/Experts/...`
  - `.../MQL5/Include/...`

- Repositorio (Git):
  - `.../Proyectos-mql5/Experts/...`
  - `.../Proyectos-mql5/Include/...`

Regla:
- Las carpetas `Experts/` e `Include/` deben mantenerse **sincronizadas** (mismos nombres y rutas) para evitar errores de `#include` y diferencias entre “lo que compila” y “lo que se versiona”.

---

## 5) Compilación (Framework v4.0)

### EA principal (Portfolio)
Compilar este archivo:
- `Experts/Framework v4.0/Portfolio Master EA MA/Portfolio Master EA MA_v4.0.mq5`

Pasos:
1. Abrir el `.mq5` en MetaEditor
2. Compilar (F7)
3. Si hay errores de includes: revisar que `Experts/` y `Include/` quedaron en el lugar correcto bajo `MQL5/`

---

## 6) Diagrama (texto) de dependencias / includes — Framework v4.0 (exacto)

### 6.1) Entry point (EA)
`Experts/Framework v4.0/Portfolio Master EA MA/Portfolio Master EA MA_v4.0.mq5`

Incluye:

- `#include "../Framework.mqh"`
  - `Experts/Framework v4.0/Framework.mqh` (**puente**)
    - `#include <Master Framework v4.0/Framework.mqh>`
      - `Include/Master Framework v4.0/Framework.mqh` (**agregador real**)
        - `Enums.mqh`
        - `Barra.mqh`
        - `Gestion Posiciones v2.mqh`
        - `Indicadores v3.mqh`
        - `Multisimbolo.mqh`  → define `CMultisimbolo`
        - `Gestion de Riesgo v2.mqh`
        - `Trade v2.mqh`
        - `Time.mqh`

- `#include "../MS_Expert/MS_Expert_MA.mqh"`
  - `Experts/Framework v4.0/MS_Expert/MS_Expert_MA.mqh`
    - `#include "../Framework.mqh"` → (mismo puente/agregador de arriba)
    - `#include "../EA_Strategies/Expert_MA.mqh"`
      - `Experts/Framework v4.0/EA_Strategies/Expert_MA.mqh`
        - define la estrategia por símbolo (clase `CExpertoMA`)

- `#include "Portfolio Master EA MA_Inputs.mqh"`
  - `Experts/Framework v4.0/Portfolio Master EA MA/Portfolio Master EA MA_Inputs.mqh`
    - inputs/configuración usados por el EA y el framework

> Nota: es normal que módulos como `MS_Expert_MA.mqh` incluyan también `../Framework.mqh` para ser autocontenibles.

### 6.2) Flujo de ejecución (eventos MT5)

**OnInit()**
- EA crea `CMS_Expert_MA`
- `CMS_Expert_MA.OnInitEvent()`
  - crea `CMultisimbolo`:
    - `MS = new CMultisimbolo(...)`
    - `MS.OnInitEvent()`
  - crea `mNumeroSimbolos` expertos por símbolo:
    - `ExpertoMA[i] = new CExpertoMA(...)`
    - `ExpertoMA[i].OnInitEvent()`

**OnTick()**
- `CMS_Expert_MA.OnTickEvent()`
  - loop de símbolos: `ExpertoMA[i].OnTickEvent()`

**OnDeinit()**
- `CMS_Expert_MA.OnDeinitEvent(reason)`
  - `delete ExpertoMA[i]` (si aplica)
  - `delete MS` (si aplica)

---

## 7) Nota crítica: existen 2 Framework.mqh (por diseño)

Hay dos archivos llamados `Framework.mqh`:

1) `Experts/Framework v4.0/Framework.mqh` (**puente**)  
- Se usa para permitir includes relativos simples desde EAs dentro de `Experts/Framework v4.0/...`
- Su responsabilidad es “redirigir” hacia el framework real en `Include/`

2) `Include/Master Framework v4.0/Framework.mqh` (**agregador real**)  
- Centraliza includes del framework (Enums, Multisimbolo, Trade, Riesgo, Tiempo, etc.)

Regla práctica:
- Los EAs dentro de `Experts/Framework v4.0/...` deben incluir el **puente** (`../Framework.mqh`)
- El código “core reutilizable” vive en `Include/Master Framework v4.0/...`

---

## 8) Versionado: cambios en progreso (WIP) vs versión estable

Este repo puede contener cambios en distintos estados:

- **WIP (Work In Progress)**:
  - Cambios parciales mientras se desarrolla/depura
  - Se suben para revisión, comprobaciones o para compartir contexto
  - Puede que todavía no compilen o no estén “operativos”

- **Estable / operativo**:
  - Versión ya probada y funcional
  - Se sube como “definitiva”

Recomendación:
- Usar mensajes de commit claros: `WIP: ...` mientras estás desarrollando y `Stable: ...` cuando quede operativo.

---

## 9) Troubleshooting (errores comunes)

### Error: cannot open include file ...
Causas típicas:
- `Experts/` y `Include/` no están dentro de `MQL5/`
- Se renombró una carpeta o archivo (incluyendo espacios)
- El EA se movió de carpeta y los includes relativos ya no apuntan bien

Solución:
- Repetir instalación (sección 2) y volver a compilar.

---