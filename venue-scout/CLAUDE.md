# Venue Scout · Contexto para Claude Code

Herramienta táctica standalone del ecosistema **The Surf Sequence®** para que el coach analice un spot de competencia (o una sesión libre de surf) y le entregue al atleta un plan concreto: dónde empezar, cuándo moverse, qué esperar, dónde entrar y salir del agua, y qué hazards evitar.

**Versión actual: v1.7+** — dos modos de análisis (Competencia / Sesión Libre), 14 tipos de hazards con iconos SVG line-art Lucide, corrientes direccionales con 2 taps, perfil del surfista completo, reporte narrativo adaptado al perfil, mapa con fondo navy sólido para máximo contraste.

## Estructura del proyecto

```
venue-scout/
├── index.html          ← app entera (single file, ~2100 líneas, HTML + CSS + JS vanilla)
├── tss-logo-wave.png   ← logo cyan wave
├── tss-logo-white-h.png← logo horizontal blanco
└── CLAUDE.md           ← este archivo
```

**No hay build. No hay dependencias. No hay backend.** Todo corre en el browser, persiste en `localStorage` bajo la key `tss_venue_scout_v1`.

## Ubicación y deploy

- **Working copy**: `/Users/marcelocastellanos/Desktop/tss-elsalvador-fix/_repo_clone/venue-scout/`
- **Repo git**: `github.com/marcelocsurf/tss-elsalvador` (subcarpeta `/venue-scout/`)
- **URL pública** (GitHub Pages): `https://marcelocsurf.github.io/tss-elsalvador/venue-scout/`
- **Deploy**: commit + push a `main` → GH Pages rebuildea 1–3 min

```bash
# Deploy típico
cd /Users/marcelocastellanos/Desktop/tss-elsalvador-fix/_repo_clone
git add venue-scout/index.html
git commit -m "feat(venue-scout): ..."
git push origin main
```

## Stack

- Vanilla JS · sin framework · sin bundler
- SVG inline para el mapa (viewBox 600×600)
- CSS custom properties (variables `--bg`, `--cyan`, `--purple`, etc.)
- Fonts: Outfit (títulos), Work Sans (body), DM Mono (data), Instrument Serif (accents)
- PWA-ready (viewport meta apple-mobile)
- Sin `<canvas>` (todo SVG)
- Sin librerías externas (ni jQuery, ni Chart.js, ni lucide runtime)
- Iconos: SVG inline copiados de Lucide (no import)

## Brand (colores oficiales TSS)

```css
--bg:#0A1628         /* Ocean Navy */
--bg2:#0E1F3D
--bg3:#1a2842
--cyan:#00D2FF       /* Signature Cyan — botones primarios, labels */
--purple:#A855F7     /* uso especial (dibujo/entry-exit) */
--amber:#F59E0B      /* categoría media */
--foam:#34D399       /* categoría mejor / éxito */
--coral:#FCA5A5      /* peligro / destructivo */
--text:#E8F0F5       /* body en dark bg */
--text-soft:#B8CCDF
--grey:#6B7E8F
```

Fondo dark navy. Texto de lectura siempre blanco/soft. Cyan solo para labels y CTAs. **No usar emojis en UI** — todos los iconos van como SVG inline (Lucide).

## State model

Un solo objeto `state` global (línea ~246 de `index.html`). Persiste en LocalStorage por `save()` cada mutación:

```js
state = {
  screen: 'setup' | 'observe' | 'breathing' | 'closing' | 'report',
  config: {
    spotName: string,
    date: 'YYYY-MM-DD',
    conditions: { sizeFt:1-12, tideLevel:0-100, tideDir:'subiendo'|'bajando'|'parada', wind:string, period:string },
    durationMin: number,           // duración de la observación (5-60 min)
    mode: 'comp' | 'free',         // COMPETENCIA o SESIÓN LIBRE (v1.6+)
    surferProfile: {               // solo se usa en modo free (v1.7+)
      name: string,
      level: 'principiante'|'intermedio'|'avanzado',
      focus: 'free-surf'|'training',
      intention: string,           // solo si focus='free-surf' (diversión, fluir, etc.)
      goal: string,                // solo si focus='training' (practicar maniobra, ganar volumen, etc.)
      conditionsMatch: 'ideales para mi nivel'|'desafiantes / al límite'|'por debajo de mi nivel',
      hazards: []                  // preocupaciones declaradas (piedras, crowd, corrientes, etc.)
    }
  },
  map: {
    lines: { outside:{x1,y1,x2,y2}|null, inside:..., orilla:... },
    refs: [{id, x, y, label}],
    photoDataUrl: string|null,     // foto del lineup como fondo
    photoOpacity: 0-1,
    freehand: [[[x,y],...],...],   // trazos libres con dedo
    entryPoint: {x,y}|null,        // dónde entra el atleta al agua
    exitPoint: {x,y}|null,         // dónde sale
    hazards: [{                    // v1.5+ · sistema de hazards
      id, type,                    // type es id del catálogo HAZARDS (rip, rocks, coral, etc.)
      x, y,                        // posición en el mapa
      angle?                       // opcional · solo para hazards con directional:true (corriente)
    }]
  },
  waves: [{
    id: string,
    gridCol: 1..15,
    gridRow: 1..17,                // filas etiquetadas A..R
    category: 'mejor'|'media'|'otra'|'ola'|'pending',  // 'ola' en modo free · 'pending' en modo rápido sin score
    score: 1-10,
    direction: 'L'|'R'|null,       // sticky modifier
    ts: ms desde inicio timer,
    absTime: timestamp real
  }],
  timer: { startedAt:ms, durationMs:ms, paused:bool, finished:bool },
  activeMode: 'mejor'|'media'|'otra'|'ola'|'line-out'|'line-in'|'line-ori'|'ref'|'erase'|'draw'|'entry'|'exit'|'predict'|`hazard-${string}`,
  activeDirection: 'L'|'R'|null,
  activePrediction: {col,row}|null,
  predictionStats: { tried, hit, near },
  pendingLineStart: {x,y}|null,   // primer tap para línea outside/inside/orilla
  pendingRipStart: {x,y,hazardId}|null,  // v1.8 · primer tap para corriente direccional
  closing: {
    description: string,
    scores: { mejor:8.5, media:6.5, otra:5.0 },
    heatObjective: 14.5,
    heatDurationMin: 25
  },
  rapidMode: true   // default ON en modo comp: tap primero, asignás score después
}
```

**Catálogo de hazards** (const `HAZARDS`, ~línea 249):
14 tipos agrupados en 4 packs, cada uno con `id`, `label`, `cat`, `color` (brand TSS) y `svg` (paths Lucide-style dentro de un viewBox 24×24).
- **Peligros físicos**: rip (Corriente · **directional:true**), rocks, coral, danger
- **Otros humanos**: crowd, judges (gavel), buoy, camera
- **Naturaleza y agua**: marine, freshwater, wind
- **Seguridad**: lifeguard (life-buoy), safeentry (shield-check), meeting (users-round)

Hazards direccionales (`directional:true`) usan flujo de **2 taps** en `handleMapTap`: primer tap = origen, segundo tap = destino, se calcula `angle = atan2(dy,dx)*180/PI + 90` y el SVG rota `<g transform="rotate(angle 12 12)">` alrededor de su centro.

**Migraciones de state en `load()`** (línea ~280) — SIEMPRE agregar la próxima versión abajo, nunca cambiar las viejas:
- v1 → v1.1: `conditions.size` string → `sizeFt` número + `tideLevel`
- v1.4 → v1.5: agregar `map.hazards = []`
- v1.5 → v1.6: agregar `config.mode = 'comp'` + `config.surferProfile`
- v1.6 → v1.7: `surferProfile.concerns` → `surferProfile.hazards` + campos `focus`/`intention`/`conditionsMatch`
- v1.7 → v1.8: (implícito) hazards viejos sin `angle` se renderizan sin rotate

## State machine (screens)

```
setup     → renderSetup()          — modo (comp/free) · perfil surfista (si free) · spot · condiciones · duración
  ↓ startBreathingThenObserve()
breathing → showBreathingScreen()  — box breathing 4-4-4-4 pre-scout (~1 min · opcional)
  ↓ endBreathing()
observe   → renderObserve()        — mapa + botonera + timer + captura de olas/hazards
  ↓ timer termina o coach cierra
report    → renderReport()         — despacha a renderReportComp() o renderReportFree() según config.mode
```

`render()` (línea ~344) es el dispatcher que llama a la función de la screen actual. No hay `closing` como screen — `state.closing` es solo el bag de datos de scoring/heatObjective usado por el reporte comp.

## Funciones principales (líneas aproximadas)

| Función | Línea | Qué hace |
|---|---:|---|
| `save() / load() / reset()` | 279–308 | Persistencia LocalStorage |
| `openAppMenu()` | 310 | Menú hamburguesa (rapid mode toggle, print, reset) |
| `render()` | 344 | Dispatcher de screen |
| `renderSetup()` | 402 | UI setup |
| `renderChipGroup()` | 466 | Chips para dropdowns |
| `startBreathingThenObserve()` | 491 | Arranca box breathing |
| `showBreathingScreen()` | 495 | UI respiración 4-4-4-4 |
| `runBreathingCycle()` | 514 | Ciclo animación respiración |
| `startTimerLoop()` | 563 | Loop del timer de observación |
| `renderObserve()` | 585 | UI mapa + botonera captura |
| `renderPredictionWidget()` | 710 | Widget modo juego predicción |
| `renderMapSVG()` | 798 | SVG del mapa (600×600) con grid, líneas, fotos, pins |
| `groupWavesByCell()` | 965 | Agrupa olas por celda de grilla |
| `attachMapHandlers()` | 982 | Handlers touch/mouse del mapa |
| `handlePhotoUpload()` | 1039 | Cargar foto del lineup como fondo |
| `handleMapTap()` | 1063 | Tap en el mapa → agrega pin/ola |
| `scoreToCategory(score, scores)` | ~ | Convierte score 1-10 → mejor/media/otra (solo para COLOR de pins; el motor táctico no usa categorías) |
| `assignScore()` | ~ | Asigna score a un pin pendiente (tap-first) |
| `computeZonesV2()` | ~ | **CORE v2**: clustering de zonas por vecindad (Chebyshev ≤1) + stats por zona (ritmo, scores, dirección) |
| `simulateStrategy(stints, target, trials)` | ~ | **CORE v2**: Monte Carlo de un heat — llegadas Poisson, scores bootstrap, top-2 |
| `buildTacticalPlan(zones)` | ~ | **CORE v2**: genera estrategias (stay / split A→B al 33/50/66%), simula todas, elige la mejor |
| `renderTacticalPlanCard(plan)` | ~ | Card ejecutiva del plan con pasos, pts esperados y probabilidad de objetivo |
| `renderZoneRanking(zones, cl)` | ~ | Ranking de zonas ordenado por valor simulado en heat |

Para hallazgos exactos:
```bash
grep -n "^function nombreFuncion" venue-scout/index.html
```

## Motor táctico v2 (imprescindible entender)

**Premisa**: en un heat contás tus 2 mejores olas, no el promedio. Por eso el motor NO compara zonas por score medio — **simula el heat completo (Monte Carlo, 1.200 trials por estrategia)** y elige la estrategia con más puntos esperados.

### Captura (modo comp)
Score-only, tap-first SIEMPRE: tap donde salió la ola → pin gris pendiente → asignás score 1-10 + dirección. No hay botones MEJOR/MEDIA/OTRA (removidos). `scoreToCategory` solo persiste para colorear pins.

### `computeZonesV2()`
Zonas = clusters de celdas ADYACENTES con olas (flood-fill, vecindad Chebyshev ≤1) — D7 y D8 son la misma zona si ambas tienen olas. Por zona: `ratePerMin` (n/minutos observados), `meanIntervalMs`, `scores[]` (reales; olas legacy sin score usan estimado por categoría), `topHalfAvg` ("potencial de sus mejores"), dirección dominante, refs cercanas.

### `simulateStrategy(stints, target, trials)`
Un trial = un heat simulado: en cada estancia las olas llegan como **proceso de Poisson** al ritmo medido (`-ln(1-U)/λ`); cada ola agarrada consume `WAVE_COST_MIN` (2.5 min) del reloj; su score se sortea de los anotados en esa zona (**bootstrap**). Puntaje del trial = suma de las 2 mejores. Devuelve `{expected, probTarget, p25, p75}`.

### `buildTacticalPlan(zones)`
Compite estrategias: quedarse en cada zona (top 4 con ≥2 olas) + splits A→B con cambio al 33/50/66% del heat, descontando remada (`moveCostMin` ≈ 0.5 min/celda de distancia). Gana la de mayor `expected`. `alt` = mejor estrategia con zona inicial o tipo distinto (Plan B). La regla de switch mostrada al user usa el percentil 70 de los scores de la zona A como umbral.

### Constantes ajustables
```js
WAVE_COST_MIN = 2.5      // min por ola surfeada (ride + remada de vuelta)
MOVE_MIN_PER_CELL = 0.5  // min de remada por celda entre zonas
MC_TRIALS = 1200         // simulaciones por estrategia
```
Supuesto declarado en el reporte: no se modela prioridad de rivales.

## Convenciones críticas

### Surf: dirección visual VS dirección real
- Una ola **izquierda** para el surfista **visualmente va hacia la derecha** desde la playa (rompe hacia la derecha del observador).
- En el código, `direction: 'L'` = ola izquierda de surf; la **flecha visual** apunta a la derecha.
- Fix histórico: commit `e64f199` invierte las flechas visuales para respetar esta convención sin cambiar los nombres internos.
- **Si tocás algo de flechas, no cambies el código semántico (`L`/`R`) — cambiá solo la orientación visual del SVG.**

### Mobile-first
- El mapa ocupa la mitad superior en portrait, y hay una botonera fija abajo.
- `touch-action: pan-y` en el mapa permite scroll vertical incluso cuando se dibuja (fix `e8599e7`).
- El header y la botonera nunca se pintan sobre el mapa (dos zonas distintas).

### Captura tap-first (siempre, no es toggle)
- Modo comp: tap donde salió la ola → pin gris pendiente → score 1-10 + dirección. `scoreToCategory` deriva la categoría SOLO para el color del pin.
- Modo free: un solo pin "OLA BUENA" sin score.
- El toggle rapidMode del menú quedó sin efecto en comp (siempre tap-first).

### Storage
- Un solo key: `tss_venue_scout_v1`.
- No incrementar la versión salvo cambio breaking del schema — usar migración en `load()` como se hizo con `sizeFt`/`tideLevel`.

## Cómo probar local

```bash
# Opción 1: abrir directo en el browser
open /Users/marcelocastellanos/Desktop/tss-elsalvador-fix/_repo_clone/venue-scout/index.html

# Opción 2: servidor local (mejor para PWA/manifest)
cd /Users/marcelocastellanos/Desktop/tss-elsalvador-fix/_repo_clone/venue-scout
python3 -m http.server 8000
# → http://localhost:8000
```

Para resetear estado de prueba: DevTools → Application → LocalStorage → borrar key `tss_venue_scout_v1`, o click "Nuevo análisis" en el menú.

## Cómo deployar

```bash
cd /Users/marcelocastellanos/Desktop/tss-elsalvador-fix/_repo_clone
git status
git add venue-scout/
git commit -m "feat(venue-scout): descripción"
git push origin main
# esperar 1-3 min · verificar en marcelocsurf.github.io/tss-elsalvador/venue-scout/
```

Cache-bust del logo: agregar `?v=N` al `<img src>` cuando reemplaces el PNG (ver commit `b18924c`).

## Gotchas conocidos

1. **GitHub Pages cachea agresivo**. Si un push no aparece en producción, hacer commit vacío para forzar rebuild:
   ```bash
   git commit --allow-empty -m "chore: trigger rebuild"
   git push
   ```
2. **jsPDF NO renderiza SVG** — cualquier icono para PDF debe ser texto plano o dibujado con `doc.circle/rect/path`. En este proyecto no hay PDF nativo; el "PDF" es `window.print()`.
3. **LocalStorage 5MB** — la foto del lineup es dataURL en base64. En sesiones muy pesadas puede llenarse. Si empieza a fallar, comprimir la foto antes de guardar.
4. **Direcciones de ola** — leer sección "Surf: dirección visual VS dirección real" antes de tocar flechas.
5. **iOS Safari** — `viewport-fit=cover` + `env(safe-area-inset-*)` ya está aplicado en el header/botonera para evitar tapar por el notch.

## Features implementadas (v1.7+)

### Base (v1 – v1.4)
- ✅ Setup del spot (condiciones, duración, sliders)
- ✅ Box breathing 4-4-4-4 pre-scout (opcional)
- ✅ Mapa SVG 600×600 con grilla 17×15 (filas A-R)
- ✅ Foto del lineup como fondo (con slider de opacidad)
- ✅ Dibujo libre con dedo (freehand)
- ✅ 3 líneas guía (outside, inside, orilla)
- ✅ Referencias con label (palmera, casa, etc.)
- ✅ Captura de olas con categoría + dirección L/R
- ✅ Modo rápido (tap primero → score auto-clasifica según thresholds adaptativos)
- ✅ Múltiples hot zones por categoría
- ✅ Timer con pausa
- ✅ Modo predicción (juego de acertar dónde cae la próxima MEJOR)
- ✅ Replay temporal animado
- ✅ Reporte narrativo storytelling + stats
- ✅ Plan táctico del heat (duración + objetivo → recomienda combos factibles)
- ✅ Rebrand oficial TSS v5.0 (colores, fonts, logo)
- ✅ Impresión / guardar como PDF via `window.print()`

### v1.5 · Sistema de Hazards
- ✅ 14 tipos de hazards con iconos SVG line-art Lucide
- ✅ 4 packs: Peligros físicos · Otros humanos · Naturaleza y agua · Seguridad
- ✅ Modal de selección tipo grid 4-col agrupado por pack
- ✅ Cada hazard con color, icono coherente con lo que representa (life-buoy real para salvavidas, shield-check para entrada segura, gavel para jueces, users-round para meeting point, etc.)
- ✅ Persistencia + render en observe + reporte + replay
- ✅ Borrado con modo erase

### v1.6 · Modo dual Competencia / Sesión Libre
- ✅ Selector "Tipo de análisis" en el setup con 2 tarjetas grandes
- ✅ Modo COMPETENCIA: flujo actual completo con scoring 1-10, categorías, plan táctico
- ✅ Modo SESIÓN LIBRE: un solo pin "OLA BUENA", sin scoring, sin plan táctico
- ✅ Reporte adaptado: renderReportComp vs renderReportFree con lógicas separadas

### v1.7 · Perfil del surfista (Free mode)
- ✅ Nombre, nivel (principiante/intermedio/avanzado)
- ✅ Condiciones para tu nivel (ideales / desafiantes / por debajo)
- ✅ Enfoque de la sesión: FREE SURF o TRAINING (dos botones grandes)
- ✅ Si FREE SURF → intención (diversión, desestresarse, ejercicio, jugar con el mar, fluir, lo que salga) + input libre
- ✅ Si TRAINING → objetivo (practicar maniobra, ganar volumen, mejorar timing, prep competencia, evaluación técnica, foto/video) + input libre
- ✅ Peligros/hazards a considerar (multi-select)
- ✅ Reporte free con:
  - Zona ideal (cluster con más olas alejado de hazards)
  - Zonas a evitar (lista de hazards con celda de grilla)
  - Entrada/salida sugeridas
  - Checklist adaptado a nivel + condiciones + hazards + focus

### v1.8 · Corrientes direccionales
- ✅ Icono corriente cambia a chevrons apilados con línea central
- ✅ Flujo de 2 taps: 1er tap = origen, 2do tap = destino
- ✅ Preview animado del origen esperando el 2do tap
- ✅ Ángulo se calcula con atan2 y se guarda en el hazard como `angle`
- ✅ El icono rota alrededor de su centro con `<g transform="rotate(angle 12 12)">`
- ✅ Extensible: cualquier hazard con `directional:true` entra al mismo flujo

### Estética y contraste
- ✅ Fondo del mapa navy sólido (`#0A1628`) con banda inferior verde-oliva tenue como hint de arena
- ✅ Todos los pins de olas, hazards y refs contrastan uniformemente
- ✅ Flechas de entrada/salida claramente opuestas: arrow-up-from-line (entra al mar) vs arrow-down-to-line (llega a la orilla)
- ✅ Convención surf de dirección: 'L' visualmente va a la derecha desde la playa

## Historia de commits relevante

```
e78ce54 feat(venue-scout): corrientes direccionales (2 taps origen → destino)
92c10b5 style(venue-scout): iconos de hazards más literales a lo que representan
5084b20 style(venue-scout): flechas entrada/salida claramente opuestas
9188047 style(venue-scout): iconos Lucide log-in/log-out para entrada/salida
a1555b5 style(venue-scout): fondo del mapa navy sólido para mejor contraste
23a5eec feat(venue-scout): perfil del surfista rediseñado con enfoque + intención
8f2cea0 feat(venue-scout): modo dual Competencia / Sesión Libre
a4e3448 feat(venue-scout): sistema de hazards (corrientes, piedras, crowd, etc)
1b2329d feat(venue-scout): categoría se deriva del score
7df73ae feat(venue-scout): plan táctico del heat + modo rápido tap-first
62d5229 feat(venue-scout): entry/exit points para plan táctico del atleta
e8599e7 fix(venue-scout): scroll vertical en mobile
e64f199 fix(venue-scout): convención surf de dirección
8463546 feat(venue-scout): rediseño visual + dirección L/R
33f2789 feat(venue-scout): rediseño del reporte hero
e2e0d01 feat(venue-scout): rebrand con identidad oficial TSS
5f33212 feat(venue-scout): v1.4 foto del lineup + dibujo libre
9ab93b4 feat(venue-scout): v1.3 múltiples hot zones
373eb0a feat(venue-scout): v1.2 reporte con promedio real de scores
9784f07 fix(venue-scout): v1.1 sliders + tap-en-pin + modal refs
ae6a186 feat(venue-scout): v1 standalone tool
```

`git log --oneline -- venue-scout/` te lo saca actualizado.

## Backlog / ideas open

- Exportar el análisis a PDF nativo (hoy es `window.print()`)
- Compartir el reporte por URL (state en query params, sin backend)
- Registrar múltiples scouts del mismo spot y comparar (histórico)
- Integrar con app principal TSS El Salvador para asignar el plan al atleta
- Overlay de dirección del viento animada
- Sonido/vibración cuando entra ola MEJOR mientras estás observando

## Cómo pedirle cambios a Claude Code

Cuando abras esta carpeta en Claude Code Desktop, ejemplos de tareas útiles:

- *"Agregá un contador de olas en tiempo real arriba a la derecha del mapa"*
- *"Cambiá el modo rápido para que el score también acepte enter directo por teclado"*
- *"Cuando el timer llega a los últimos 60s, poné una animación de urgencia"*
- *"Exportá el reporte como PDF real (no `window.print()`) usando jsPDF inline sin import"*

Claude Code va a leer este `CLAUDE.md`, entender la estructura, y editar `index.html` en el lugar correcto. Si querés deploy: pedile *"commit + push a main"* al terminar.
