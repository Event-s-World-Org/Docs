# Sistema de diseño de Event's World · Salar Cristal

* **Versión:** 1.0 · **Fecha:** 28 oct 2026 · **Estado:** decisión final
* **Decisión:** [ADR-0008](../architecture/adr/0008-salar-cristal-design-system.md). Mantiene la base técnica de [ADR-0007](../architecture/adr/0007-styling-and-components.md): Tailwind CSS v4 + Angular CDK + componentes propios.
* **Requisitos:** NFR-01.1 y NFR-01.2 de [requirements.md](../requirements.md).
* **Referencia visual:** opción "Salar Cristal" del [laboratorio de estilos](style-lab.html), con tipografía Outfit y esquinas suaves.

Este documento es la **fuente única** de colores, tipografía, superficies y reglas visuales de **todas las vistas** de la app: públicas, participante, autenticación, organizador y Super Admin, en modo claro y oscuro.

---

## 1. Concepto

**Salar Cristal** se inspira en el Salar de Uyuni: un cielo azul que se refleja en el suelo y la luz coral del atardecer.

* **Superficies:** toda la interfaz flota en **paneles de vidrio translúcido** sobre un fondo de cielo con reflejos azules y coral.
* **Marca:** el azul transmite confianza en los pagos, y el coral aparece solo como acento puntual.
* **Equilibrio:** el vidrio es intenso y visible, pero cada texto se verifica en el **peor caso** de contraste (§6).

---

## 2. Colores

### 2.1 Tokens semánticos

| Token | Claro | Oscuro | Uso |
|---|---|---|---|
| `bg` | `#E6F0FB` | `#0A121C` | Color base del fondo de cielo (§4) |
| `surface` | `#FFFFFF` | `#111C29` | Superficie sólida; base del vidrio y del modo "reducir transparencia" |
| `surface-2` | `#EBF1F7` | `#182638` | Superficie secundaria sólida (estados neutros) |
| `border` | `#D3DEE9` | `#28394F` | Borde sutil sobre superficies sólidas |
| `control-border` | `#5A6B80` | `#7F95B0` | Borde de campos y controles (≥ 3:1) |
| `text` | `#0E1B2A` | `#EAF2FA` | Texto principal |
| `muted` | `#3E5367` | `#9AB0C6` | Texto secundario |
| `primary` | `#0B63B6` | `#5AB0FF` | Botón principal, enlaces, foco, selección activa |
| `primary-hover` | `#09528F` | `#84C4FF` | Hover y presionado |
| `on-primary` | `#FFFFFF` | `#04192E` | Texto sobre `primary` |
| `accent` | `#E5533B` | `#FF8A73` | Acento coral puntual: dato destacado del gráfico, avatar. **Nunca** botones, fondos ni texto |
| `chart` | `#0B63B6` | `#5AB0FF` | Barras y series del gráfico |

### 2.2 Estados (pagos y eventos)

Son fijos: el color nunca cambia de significado entre pantallas. Se muestran siempre con **texto y un punto de color**, nunca solo con color (WCAG 1.4.1). Los badges son **sólidos**, no de vidrio.

| Estado | Claro (texto / fondo) | Oscuro (texto / fondo) |
|---|---|---|
| Pendiente · Inscripciones cerradas | `#8A3B0A` / `#FEF3C7` | `#FCD34D` / `#3A2A08` |
| Aprobado · Publicado | `#166534` / `#DCFCE7` | `#86EFAC` / `#0B3B22` |
| En observación | `#5B21B6` / `#EDE9FE` | `#C4B5FD` / `#2E1A5E` |
| Rechazado · Suspendido · Cancelado | `#991B1B` / `#FEE2E2` | `#FCA5A5` / `#4A1515` |
| Finalizado (info) | `#1E40AF` / `#DBEAFE` | `#93C5FD` / `#172554` |
| Sin pago · Borrador (neutro) | `muted` / `surface-2` | `muted` / `surface-2` |

---

## 3. Vidrio

### 3.1 Niveles

| Nivel | Dónde | Claro | Oscuro |
|---|---|---|---|
| **Panel** | Tarjetas, tablas, mosaicos, menú lateral, hoja del detalle, campos, chips, botones secundarios | `rgba(255, 255, 255, .50)` | `rgba(17, 28, 41, .45)` |
| **Barra** | Barra superior y barras fijas (inscripción, acciones), barra inferior móvil | Igual que el panel | Igual que el panel |
| **Overlay** | Modales, drawers (menú ☰), menús emergentes | `rgba(255, 255, 255, .72)` + fondo atenuado `rgba(10, 18, 28, .35)` | `rgba(17, 28, 41, .72)` + fondo atenuado `rgba(0, 0, 0, .55)` |

### 3.2 Propiedades comunes

| Propiedad | Claro | Oscuro |
|---|---|---|
| Desenfoque | `blur(22px) saturate(180%)` | `blur(22px) saturate(160%)` |
| Borde (1 px) | `rgba(255, 255, 255, .72)` | `rgba(255, 255, 255, .16)` |
| Brillo superior | `inset 0 1px 0 rgba(255, 255, 255, .85)` | `inset 0 1px 0 rgba(255, 255, 255, .14)` |
| Sombra | `0 10px 30px rgba(14, 27, 42, .14)` | `0 10px 30px rgba(0, 0, 0, .45)` |

### 3.3 Qué NO es vidrio

* **Botón principal:** `primary` sólido, con sombra de su propio color (`0 6px 18px` de `primary` al 35 %).
* **Badges de estado:** sólidos (§2.2).
* **Portadas de eventos y comprobantes:** son imágenes, nunca se desenfocan.
* **Textos largos** (términos y privacidad, PUB-07): van en un panel de nivel overlay, porque es más opaco.

---

## 4. Fondo de cielo

Va detrás de toda la app, fijo (no se desplaza con el contenido), para que el vidrio muestre movimiento al hacer scroll.

```css
/* Claro */
--backdrop:
  radial-gradient(42% 30% at 10% 6%,  rgba(11, 99, 182, .45), transparent 70%),
  radial-gradient(40% 30% at 94% 22%, rgba(229, 83, 59, .42), transparent 72%),
  radial-gradient(50% 34% at 26% 64%, rgba(90, 176, 255, .55), transparent 72%),
  radial-gradient(40% 28% at 86% 92%, rgba(229, 83, 59, .30), transparent 70%),
  linear-gradient(180deg, #E6F0FB, #F6EEEC);

/* Oscuro */
--backdrop:
  radial-gradient(42% 30% at 10% 6%,  rgba(90, 176, 255, .45), transparent 70%),
  radial-gradient(40% 30% at 94% 22%, rgba(255, 138, 115, .30), transparent 72%),
  radial-gradient(50% 34% at 26% 64%, rgba(11, 99, 182, .55), transparent 72%),
  radial-gradient(40% 28% at 86% 92%, rgba(255, 138, 115, .22), transparent 70%),
  linear-gradient(180deg, #08172A, #0A121C);
```

* Se pinta en un `body::before` con `position: fixed`, porque `background-attachment: fixed` no funciona en Safari de iOS.
* Las manchas de color no deben superponerse más que en el diseño: el contraste del §6 se calculó con la mancha más intensa sola.

---

## 5. Tipografía, espaciado y formas

### 5.1 Tipografía

* **Familia única:** **Outfit** (Google Fonts), con la pila de respaldo `'Outfit', system-ui, -apple-system, 'Segoe UI', sans-serif`. Se cargan solo los pesos 400, 600 y 700, con `display=swap`.
* **Pesos:**

  | Peso | Uso |
  |---|---|
  | 400 | Cuerpo |
  | 600 | Etiquetas, botones, nombres, encabezados de tabla |
  | 700 | Títulos y cifras |

  Ningún peso supera 700. El laboratorio mostraba algunos títulos más gruesos; en la implementación se limitan a 700, según la skill de tipografía.
* **Escala Minor Third (1,2):**

| Rol | Tamaño | Peso | Interlineado | Ejemplos |
|---|---|---|---|---|
| Meta | 12 px (0,75 rem) | 400 | 1,4 | Horas, códigos bajo un nombre, ayudas |
| Secundario | 14 px (0,875 rem) | 400 / 600 | 1,5 | Celdas de tabla, etiquetas de campo, badges |
| Cuerpo | 16 px (1 rem) | 400 | 1,55 | Descripción, valores de campos, botones |
| Subtítulo | 19,2 px (1,2 rem) | 600 | 1,25 | Precio en la barra, títulos de tarjeta |
| Título de página | 23 px (1,44 rem) | 700 | 1,2 | "Revisión de pagos", KPIs |
| Título de evento | 27,6 px (1,728 rem) | 700 | 1,15 | PUB-02 |
| Display | 33,2 px (2,074 rem) | 700 | 1,1 | Inicio (PUB-01), escritorio |

  Ningún texto baja de 12 px. Las cifras usan `font-variant-numeric: tabular-nums`.

### 5.2 Espaciado

* Ritmo de **4/8 px**: `4, 8, 12, 16, 24, 32, 48`.
* Margen lateral de 16 px en el celular.
* Separación entre paneles de 12–16 px.

### 5.3 Esquinas suaves

| Token | Valor | Uso |
|---|---|---|
| `radius-sm` | 6 px | Campos, badges pequeños |
| `radius-md` | 12 px | Botones, mosaicos, chips cuadrados |
| `radius-lg` | 18 px | Paneles, tarjetas, hoja del detalle, modales |
| `radius-pill` | 999 px | Chips de filtro y badges de estado |

### 5.4 Foco, íconos y movimiento

* **Foco:** anillo de 2 px en `primary`, separado 2 px del elemento (`outline-offset`), visible en vidrio y en sólido.
* **Íconos:** Lucide, de 20 px (o 24 px en la barra móvil), con trazo de 1,75. Sin íconos decorativos dentro de los campos; solo la lupa en la búsqueda.
* **Movimiento:** solo transiciones de color y opacidad de 150 ms. **Nunca** animar un elemento con desenfoque. Con `prefers-reduced-motion`, todo se vuelve instantáneo.

---

## 6. Accesibilidad y contraste

El contraste sobre vidrio depende de lo que hay detrás, así que cada par se verifica en el **peor caso**: el vidrio sobre la mancha más oscura del fondo (en claro) o la más brillante (en oscuro).

| Par | Claro | Oscuro | Requerido |
|---|---|---|---|
| Texto sobre vidrio (peor caso) | 11,85 | 9,77 | 4,5 |
| Texto secundario sobre vidrio (peor caso) | 5,43 | 4,94 | 4,5 |
| Borde de campo sobre vidrio (peor caso) | 3,72 | 3,59 | 3 |
| Título grande directamente sobre el fondo (peor caso) | 7,68 | 6,48 | 3 |
| Botón principal | 6,04 | 7,68 | 4,5 |
| Estados (§2.2) | 6,49–7,57 | 7,85–9,62 | 4,5 |

**Reglas:**

1. Todo texto de tamaño normal va sobre vidrio o sobre una superficie sólida. Solo los títulos grandes pueden ir directamente sobre el fondo.
2. Con `prefers-reduced-transparency: reduce`, y en navegadores sin `backdrop-filter` (`@supports not`), todo el vidrio pasa a `surface` sólido con borde `border`.
3. Las áreas táctiles miden al menos **44 × 44 px** en pantallas táctiles (`pointer: coarse`).
4. Cualquier cambio de color, de opacidad o del fondo exige recalcular el peor caso con el laboratorio.

---

## 7. Rendimiento (importante para celulares de gama media y baja)

* **Desenfoque en contenedores, no en cada elemento:** la tabla es un panel de vidrio; sus filas, no.
* **Límite:** como máximo **6 capas con desenfoque** visibles a la vez en el celular.
* **Sin cambios de tamaño ni posición animados** en paneles de vidrio, y nunca `will-change` permanente.
* **Prueba obligatoria:** antes de cada versión, en un Android de gama baja con Chrome. Si el desplazamiento baja de unos 50 cuadros por segundo, se reduce el desenfoque a `blur(14px)` en esa pantalla.

---

## 8. Implementación en Angular (Tailwind v4)

`src/styles.css` (frontend.md §7):

```css
@import "tailwindcss";
@custom-variant dark (&:where(.dark, .dark *));

:root {
  --ew-bg: #E6F0FB; --ew-surface: #FFFFFF; --ew-surface-2: #EBF1F7; --ew-border: #D3DEE9; --ew-control-border: #5A6B80;
  --ew-text: #0E1B2A; --ew-muted: #3E5367; --ew-primary: #0B63B6; --ew-primary-hover: #09528F; --ew-on-primary: #FFFFFF; --ew-accent: #E5533B;
  --ew-glass: rgba(255, 255, 255, .50); --ew-glass-overlay: rgba(255, 255, 255, .72);
  --ew-glass-border: rgba(255, 255, 255, .72); --ew-glass-highlight: rgba(255, 255, 255, .85);
  --ew-glass-shadow: 0 10px 30px rgba(14, 27, 42, .14); --ew-blur: blur(22px) saturate(180%);
  /* + --ew-backdrop (§4) y los estados (§2.2) */
}
.dark {
  --ew-bg: #0A121C; --ew-surface: #111C29; --ew-surface-2: #182638; --ew-border: #28394F; --ew-control-border: #7F95B0;
  --ew-text: #EAF2FA; --ew-muted: #9AB0C6; --ew-primary: #5AB0FF; --ew-primary-hover: #84C4FF; --ew-on-primary: #04192E; --ew-accent: #FF8A73;
  --ew-glass: rgba(17, 28, 41, .45); --ew-glass-overlay: rgba(17, 28, 41, .72);
  --ew-glass-border: rgba(255, 255, 255, .16); --ew-glass-highlight: rgba(255, 255, 255, .14);
  --ew-glass-shadow: 0 10px 30px rgba(0, 0, 0, .45); --ew-blur: blur(22px) saturate(160%);
}

@theme inline {
  --font-sans: 'Outfit', system-ui, -apple-system, 'Segoe UI', sans-serif;
  --radius-sm: 6px; --radius-md: 12px; --radius-lg: 18px;
  --color-bg: var(--ew-bg); --color-surface: var(--ew-surface); --color-surface-2: var(--ew-surface-2);
  --color-border: var(--ew-border); --color-control: var(--ew-control-border);
  --color-text: var(--ew-text); --color-muted: var(--ew-muted);
  --color-primary: var(--ew-primary); --color-primary-hover: var(--ew-primary-hover); --color-on-primary: var(--ew-on-primary);
  --color-accent: var(--ew-accent);
}

@utility glass {
  background: var(--ew-glass); border: 1px solid var(--ew-glass-border);
  backdrop-filter: var(--ew-blur); -webkit-backdrop-filter: var(--ew-blur);
  box-shadow: inset 0 1px 0 var(--ew-glass-highlight), var(--ew-glass-shadow);
}
@utility glass-overlay {
  background: var(--ew-glass-overlay); border: 1px solid var(--ew-glass-border);
  backdrop-filter: var(--ew-blur); -webkit-backdrop-filter: var(--ew-blur);
}
@media (prefers-reduced-transparency: reduce) {
  .glass, .glass-overlay { background: var(--ew-surface); backdrop-filter: none; -webkit-backdrop-filter: none; border-color: var(--ew-border); }
}
@supports not (backdrop-filter: blur(1px)) {
  .glass, .glass-overlay { background: var(--ew-surface); border-color: var(--ew-border); }
}

body { font-family: var(--font-sans); color: var(--ew-text); background: var(--ew-bg); }
body::before { content: ""; position: fixed; inset: 0; z-index: -1; background: var(--ew-backdrop); }
```

El bloque "Tokens de la combinación elegida" del laboratorio (opción Salar Cristal) genera los valores exactos de `:root` y `.dark`.

---

## 9. Aplicación por componente (`shared/ui`)

| Componente | Superficie | Detalle |
|---|---|---|
| Barra superior y barra inferior móvil | `glass` (barra) | Fija; borde inferior o superior de vidrio |
| Panel, tarjeta, KPI, mosaico | `glass` | `radius-lg` (mosaicos: `radius-md`) |
| Tabla y lista | Un solo `glass` para todo el bloque | Divisores con `glass-border`. Fila seleccionada: `primary` al 14 % y barra lateral de 3 px en `primary` |
| Campo de texto y select nativo | `glass` | Borde `control-border`; foco con anillo `primary` |
| Botón principal | `primary` sólido | `radius-md`, peso 600, sombra de color |
| Botón secundario | `glass` | Texto `text` |
| Aprobar, Observar, Rechazar | Color de estado sólido | Nunca vidrio |
| Chip de filtro | `glass`; el activo es `primary` sólido | `radius-pill` |
| Badge de estado | Sólido (§2.2) | `radius-pill`, punto de color y texto |
| Modal y drawer (☰) | `glass-overlay` + fondo atenuado | Foco atrapado; Escape cierra (CDK `Dialog`) |
| Toast | `text` sólido sobre `bg` | Sin vidrio, para que se lea siempre |
| Portada de evento | Imagen | La hoja del detalle se superpone 28 px |
| Comprobante (ORG-08) | Imagen sobre `surface` sólido | Sin desenfoque; texto alternativo obligatorio |

---

## 10. Excepciones a las skills de UI/UX

Este sistema contradice a propósito una regla de `vectorlab-ux-skills:anti-slop`: el vidrio en la UI del producto (S3, S4 y S15). Es una **decisión de marca** del equipo, registrada en ADR-0008. Cumple las condiciones que la misma skill pide si se usa vidrio:

* un límite visible;
* un respaldo opaco;
* contraste suficiente, verificado en el peor caso.

Todas las demás reglas de las skills siguen vigentes: un solo acento interactivo, 3 pesos, tamaños de 12/14/16 px y verbos claros en los botones.
