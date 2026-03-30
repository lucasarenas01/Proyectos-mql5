# Proyectos MQL5 (lucasarenas01/Proyectos-mql5)

Repositorio con proyectos en **MQL5 / MetaTrader 5**, organizado siguiendo la estructura típica de **MQL5** (por ejemplo `Experts/` e `Include/`).

> Nota: este repo está pensado para usarse dentro de la carpeta de datos de MetaTrader 5, copiando sus carpetas `Experts/` e `Include/` dentro de `MQL5/`, respetando la misma estructura.

---

## Estructura del repositorio

- `Experts/`
  - Contiene los **Expert Advisors (EAs)** y componentes relacionados (por ejemplo, el Framework v4.0 y EAs que lo usan).
- `Include/`
  - Contiene el **framework reutilizable** (`.mqh`) y módulos comunes (agregadores, utilidades, trading, riesgo, tiempo, etc.).

Además, en la raíz hay archivos informativos:
- `Framework v4.0.txt` — notas/documentación rápida del Framework v4.0.
- `info.txt` — información breve adicional.
- `INSTRUCCIONES.md` — guía de instalación/compilación + diagrama de includes.

---

## Framework v4.0 (Arquitectura)

En el **Framework v4.0** el EA principal actúa como orquestador y delega lógica en un “experto multi-símbolo” y en estrategias por símbolo.

### Idea clave (lo más importante del repo)

Aunque en este repo existan **muchos bots/estrategias diferentes** (MA, Don, ORO, RM, RangeBreakout, etc.), dentro de `Experts/Framework v4.0/` casi todos siguen el mismo patrón de 3 capas:

1) **EA (entry point, `.mq5`)**
   - Es el archivo que se **compila** (F7 en MetaEditor) y se **ejecuta** en un gráfico de MT5.
   - Es el “main” del bot: crea el MS_Expert, le pasa inputs (`param...`) y reenvía los eventos de MT5 (`OnInit`, `OnTick`, `OnDeinit`) hacia el framework.
   - Normalmente incluye:
     - `../Framework.mqh` (framework puente)
     - `../MS_Expert/MS_Expert_*.mqh` (el orquestador multi‑símbolo)
     - un archivo local de inputs (por ejemplo `*_Inputs.mqh`) que define y organiza los `param...`

2) **MS_Expert (multi‑símbolo, `.mqh`)**
   - Vive en `Experts/Framework v4.0/MS_Expert/`.
   - Su responsabilidad **NO** es calcular señales (eso es de la estrategia). Su responsabilidad es **coordinar múltiples símbolos**.
   - Lo típico que hace un `MS_Expert_*`:
     - Crea `CMultisimbolo(...)` usando:
       - `paramGeneral.SimbolosOperables` (universo)
       - `paramGeneral.SimbolosEspecificados` (lista concreta, si aplica)
       - `paramGeneral.NumeroMagicoSemilla` (para generar magic numbers por símbolo)
     - Obtiene `mNumeroSimbolos`
     - Crea un array de “expertos por símbolo” (la estrategia):
       - `CExpertoX *ExpertoX[]`
     - Para cada símbolo `i`:
       - `ExpertoX[i] = new CExpertoX(MS.ObtenerSimbolo(i), MS.ObtenerNumeroMagico(i), ...params...)`
       - `ExpertoX[i].OnInitEvent()`
     - En cada tick:
       - recorre símbolos y llama `ExpertoX[i].OnTickEvent()`

3) **Strategy (por símbolo, `.mqh`)**
   - Vive en `Experts/Framework v4.0/EA_Strategies/`.
   - Implementa la lógica real de trading **para un único símbolo**:
     - señales de entrada y salida
     - filtros (MA/RSI/horarios/días/etc.)
     - cálculo de SL/TP (directo o delegado a módulos del framework)
     - gestión de posiciones (trailing, break-even, cierre por horario, etc.)
   - Normalmente expone:
     - `OnInitEvent()`
     - `OnTickEvent()`

### Cadena de ejecución (siempre la misma)

**EA (.mq5) → MS_Expert (.mqh) → Strategy (.mqh)**

En otras palabras:
- El `.mq5` es el “arranque y control general”
- el `MS_Expert_*.mqh` es el “manager multi‑símbolo”
- el `Expert_*.mqh` es “la estrategia por símbolo”

---

## Ejemplo completo (MA) — entry point documentado

### EA principal (entry point)
Ruta:
- `Experts/Framework v4.0/Portfolio Master EA MA/Portfolio Master EA MA_v4.0.mq5`

Incluye (a alto nivel):
- `../Framework.mqh`
- `../MS_Expert/MS_Expert_MA.mqh`
- `Portfolio Master EA MA_Inputs.mqh`

### Inputs
- `Portfolio Master EA MA_Inputs.mqh`
  - Define y organiza los inputs que luego se usan a lo largo del EA y del framework.

### Qué pasa en runtime (flujo de eventos MT5)
- **OnInit()**
  - el EA crea `CMS_Expert_MA`
  - `CMS_Expert_MA.OnInitEvent()`:
    - crea `CMultisimbolo`
    - crea `CExpertoMA` por símbolo
    - llama `CExpertoMA.OnInitEvent()` por símbolo
- **OnTick()**
  - `CMS_Expert_MA.OnTickEvent()`:
    - loop de símbolos → `CExpertoMA.OnTickEvent()` por símbolo
- **OnDeinit()**
  - destruye `CExpertoMA[]` y `CMultisimbolo`

---

## “Familias” de bots: cómo ubicar archivos (MS_Expert + Strategy + EA)

Dentro de `Experts/Framework v4.0/` vas a ver que hay muchos pares:

- `MS_Expert/MS_Expert_XXX.mqh`
- `EA_Strategies/Expert_XXX.mqh`

Eso significa:
- `Expert_XXX.mqh` = la estrategia por símbolo (lógica)
- `MS_Expert_XXX.mqh` = el orquestador multi‑símbolo que crea N instancias de esa estrategia

Y (normalmente) existe además un `.mq5` que es el entry point para esa familia.

Ejemplos (no exhaustivo):
- **RM‑Keltner**
  - MS_Expert: `Experts/Framework v4.0/MS_Expert/MS_Expert_RM-Keltner.mqh`
  - Strategy: `Experts/Framework v4.0/EA_Strategies/Expert_RM-Keltner.mqh`
  - EA (entry point): *(si existe, documentar aquí la ruta del `.mq5` que instancia `CMS_Expert_KELTNER`)*

- **Don**
  - MS_Expert: `Experts/Framework v4.0/MS_Expert/MS_Expert_Don.mqh`
  - Strategy: `Experts/Framework v4.0/EA_Strategies/Expert_Don.mqh`
  - EA (entry point): *(documentar ruta del `.mq5` correspondiente)*

- **ORO / OROv2 / RangeBreakout / etc.**
  - Mismo patrón:
    - `MS_Expert/MS_Expert_*.mqh`
    - `EA_Strategies/Expert_*.mqh`
    - un `.mq5` entry point que instancia el MS_Expert y pasa los `param...`

> Recomendación: si querés que cualquiera (humano o IA) entienda rápido “qué compilar/correr” para cada familia, agregá aquí la lista de entry points `.mq5` existentes y cuál MS_Expert usan.

---

## Nota importante: existen 2 `Framework.mqh`

Esto es intencional y ayuda a separar responsabilidades:

1) **Framework puente (en `Experts/`)**  
Ubicación:
- `Experts/Framework v4.0/Framework.mqh`

Rol:
- Sirve como “puente” para que los EAs dentro de `Experts/Framework v4.0/...` puedan hacer includes relativos simples (por ejemplo `../Framework.mqh`) sin depender de rutas largas.

2) **Framework agregador real (en `Include/`)**  
Ubicación:
- `Include/Master Framework v4.0/Framework.mqh`

Rol:
- Archivo agregador que centraliza la inclusión de módulos del framework, por ejemplo:
  - `Enums.mqh`
  - `Barra.mqh`
  - `Gestion Posiciones v2.mqh`
  - `Indicadores v3.mqh`
  - `Multisimbolo.mqh`  → define `CMultisimbolo`
  - `Gestion de Riesgo v2.mqh`
  - `Trade v2.mqh`
  - `Time.mqh`

Regla práctica:
- Los EAs dentro de `Experts/Framework v4.0/...` deben incluir el **puente** (`../Framework.mqh`)
- El código “core reutilizable” vive en `Include/Master Framework v4.0/...`

---

## Cómo compilar / usar (guía rápida)

1) Copia el contenido de este repo respetando carpetas dentro de tu carpeta `MQL5/`:
   - `MQL5/Experts/...`
   - `MQL5/Include/...`

2) Abre MetaEditor y compila el EA (ejemplo MA):
   - `Experts/Framework v4.0/Portfolio Master EA MA/Portfolio Master EA MA_v4.0.mq5`

3) Ejecuta el EA en un gráfico desde MetaTrader 5.

> Recomendación: si tienes errores de includes, revisa que la estructura de carpetas dentro de `MQL5/` coincida exactamente con la del repo.

---

## Convenciones / notas

- `.mq5` = entry points (EAs / indicadores / scripts).
- `.mqh` = módulos/include (framework, estrategias, utilidades).
- El framework está pensado para separar:
  - orquestación portfolio (`.mq5`)
  - ejecución multi-símbolo (`MS_Expert_*.mqh`)
  - estrategia por símbolo (`Expert_*.mqh`)

---

