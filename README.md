# LinOpt — Linear Programming & Optimization Studio

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PWA Ready](https://img.shields.io/badge/PWA-100%25%20Offline-green.svg)](https://linopt.app)
[![Vite](https://img.shields.io/badge/Built%20with-Vite%20%2B%20React-blue.svg)](https://vitejs.dev)

**LinOpt** es una plataforma profesional de Programación Lineal e Investigación Operativa moderna, didáctica y 100% offline. Inspirada en la fluidez de las herramientas científicas modernas (estilo Jupyter Notebooks para optimización lineal), LinOpt resuelve problemas complejos utilizando aritmética exacta de fracciones y matrices Simplex paso a paso, método gráfico bidimensional interactivo con curvas de nivel, formulación primal/dual, análisis de sensibilidad post-óptimo y exportación a formatos estándar (LP, MPS, JSON, PDF).

---

## Características Principales

### 1. Motor Matemático y Solucionadores
- **Método Simplex Primal y Dos Fases**: Resolución canónica con variables de holgura ($s_i$), exceso ($e_i$) y artificiales ($a_i$).
- **Aritmética Exacta de Fracciones**: Cero acumulación de errores de punto flotante ($1/3$ se mantiene exactamente como $1/3$, evitando ciclados numéricos).
- **Método Gráfico 2D Interactivo**:
  - Trazado de rectas frontera con colores contrastantes.
  - Sombreado translúcido de la región factible con polígonos convexos.
  - Identificación automática de vértices de esquina y evaluación de $z$.
  - Punto óptimo resaltado con tooltip dinámico y anillo pulsante.
  - Línea de nivel ($Z$) deslizable para ver el punto de contacto óptimo.
- **Análisis de Sensibilidad Completo**:
  - Precios sombra (valores duales) para cada recurso.
  - Rangos de optimalidad para coeficientes de la función objetivo ($c_j$).
  - Rangos de factibilidad para lados derechos ($b_i$).
  - Costos reducidos para variables no básicas.

### 2. Cuaderno LP & Editor Algebraico (Estilo Jupyter)
- **Editor de Celda Algebraica**: Escribe o pega tus modelos en lenguaje matemático natural (compatible con CPLEX, Gurobi, LINDO).
- **Ejecución y Parseo Instantáneo**: Interpreta la sintaxis en tiempo real, detecta variables y restricciones y resuelve el problema al presionar *Ejecutar Celda*.
- **Exportación e Importación**: Soporte completo para formatos `.lp`, `.mps`, y `.json`.

### 3. Interfaz de Usuario y Experiencia Visual
- **Diseño Frosted Glass con Material Design 3**: Formas orgánicas en tonos pastel, tarjetas de vidrio esmerilado con desenfoque de fondo (`backdrop-blur-2xl`), bordes sutiles y tipografía matemática nítida.
- **Asistente de Formulación Paso a Paso**: Navegación fluida por etapas: Función Objetivo &rarr; Variables de Decisión &rarr; Restricciones &rarr; Modelo Formal.
- **Buscador Semántico**: Localiza modelos guardados, variables o ejemplos clásicos directamente desde la barra superior.
- **Sin Distracciones**: Interfaz limpia sin notificaciones invasivas, selectores innecesarios ni perfiles de usuario artificiales.

### 4. Persistencia Local y PWA Offline
- **Base de Datos IndexedDB**: Todos los modelos se guardan automáticamente en tu navegador sin requerir registro en la nube.
- **Copia de Seguridad**: Exporta o importa toda tu biblioteca de modelos en un solo clic.
- **Instalable como App Nativa**: Cumplimiento con especificaciones PWA para funcionar en Windows, Android, macOS y Linux.

---

## Getting Started

### Prerequisites
- Node.js 18+ or modern Bun / npm / pnpm

### Installation
```bash
# Clone the repository
git clone <repo-url>
cd linopt

# Install dependencies
npm install

# Start the local development server (binds to http://localhost:3000)
npm run dev
```

### Production Build
```bash
# Build the optimized production bundle
npm run build

# Preview the production build locally
npm run preview
```

---

## Architecture & Code Structure

```
├── public/
│   └── manifest.webmanifest   # Complete PWA manifest (PWABuilder compliant)
│                              # (the service worker itself is generated at build time
│                              #  by vite-plugin-pwa — see Engineering & Logic below)
├── src/
│   ├── components/            # UI (Nuevo Modelo wizard, Simplex viewer, Graphical viewer, Notebook, Reports)
│   │   ├── ErrorBoundary.tsx        # React Error Boundary for resilient failure recovery
│   │   ├── LinOptApp.tsx            # Main screen shell: sidebar nav + all 6 top-level views
│   │   ├── SimplexResultViewer.tsx  # Simplex/MIP result screen
│   │   ├── SimplexTableauViewer.tsx # Tableau iteration viewer
│   │   ├── MathematicalModelViewer.tsx # Formal mathematical model rendering
│   │   ├── LinOptGraph.tsx          # 2D Canvas plot of feasible region
│   │   └── ...
│   ├── engine/                # Core mathematical algorithms
│   │   ├── fraction.ts        # Arbitrary exact fraction arithmetic
│   │   ├── simplex.ts         # Primal Simplex & Two-Phase Simplex (exact tableau)
│   │   ├── graphical.ts       # 2D geometry: line intersections, feasible region, convex hull
│   │   └── lpParser.ts        # LP/MPS text notebook parsing & export
│   ├── lib/
│   │   └── solver.ts          # javascript-lp-solver wrapper (see Engineering & Logic below)
│   ├── storage/               # IndexedDB repository for local problem saving
│   ├── types.ts               # Complete TypeScript interfaces
│   ├── App.tsx                # Main controller and navigation
│   └── main.tsx               # React entry point
├── package.json
└── vite.config.ts
```

---

## Engineering & Logic

**Why `javascript-lp-solver`:** MIT-licensed, pure JavaScript, runs entirely client-side with no server or WASM dependency, so it works fully offline once the PWA is cached. It's used specifically for **Integer (Entera) and Binary (Binaria) variables**, where it runs true branch-and-bound rather than rounding a continuous LP relaxation. The wrapper lives entirely in `src/lib/solver.ts` — that file is the only place that touches the library, so it could be swapped later without touching UI code.

**Two solving paths, by design:**
- `src/engine/simplex.ts` is a from-scratch, exact-fraction Simplex/Two-Phase implementation. It exists so the app can show the actual tableau iterations, shadow prices, and sensitivity ranges step by step — output a black-box solver can't give you.
- `src/lib/solver.ts` (`javascript-lp-solver`) is used for the final optimal point wherever integer/binary correctness matters, e.g. inside `src/engine/graphical.ts`, so a 2-variable mixed-integer model's plotted optimum respects integer/binary feasibility rather than just the continuous relaxation's vertex.

**Internal model schema** (`InternalModel` in `src/lib/solver.ts`):
```ts
{
  objective: { type: 'max' | 'min', coefficients: Record<string, number> },
  variables: [{ name, lowerBound?, upperBound?, type: 'Continua' | 'Entera' | 'Binaria' }],
  constraints: [{ expression: Record<string, number>, relation: '<=' | '>=' | '=', value: number }]
}
```
`solveLP()` normalizes this into `javascript-lp-solver`'s expected shape and returns `{ status: 'optimal' | 'infeasible' | 'unbounded', objectiveValue, variableValues }`.

**Graphical method scope:** enabled only when a model has exactly 2 decision variables, regardless of variable type — a 2-variable mixed-integer model still renders the feasible region graphically. Sensitivity analysis and iteration tracing are considered core teaching features of this version and are intentionally kept (unlike an earlier draft of this spec, which proposed dropping them).

**Offline/PWA architecture:** the service worker is generated at build time by `vite-plugin-pwa` (`registerType: 'autoUpdate'`), which also injects its own registration script — nothing in application code registers a service worker manually in production. All static assets and the bundled solver are precached via Workbox so the app is fully usable with no network connection after first load.

---

## License

This project is licensed under the **MIT License** — see the [LICENSE](./LICENSE) file for details.
