# HU-24 · Dashboard resumen — Diseño

---

## 1. Pantallas

### Pantalla de inicio

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  Resumen del día · miércoles 18 de septiembre de 2026      Actualizado 10:42 [⟳] │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌────────────────────────────┐  ┌────────────────────────────┐                 │
│  │ 🔴 REQUIERE ATENCIÓN       │  │ 🟠 STOCK BAJO              │                 │
│  │                            │  │                            │                 │
│  │        4                   │  │        3                   │                 │
│  │  productos por vencer      │  │  materias primas           │                 │
│  │                            │  │                            │                 │
│  │  • Croissant  VENCIDO  8u  │  │  • Levadura      AGOTADA   │                 │
│  │  • Pan francés  3 días 40u │  │  • Harina        faltan 3,7│                 │
│  │  • Torta        5 días  3u │  │  • Sal           faltan 1,2│                 │
│  │                            │  │                            │                 │
│  │            [ Ver todos → ] │  │      [ Registrar compra → ]│                 │
│  └────────────────────────────┘  └────────────────────────────┘                 │
│                                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐              │
│  │ PEDIDOS DE HOY   │  │ COMPRAS          │  │ PRODUCCIÓN HOY   │              │
│  │                  │  │ PENDIENTES       │  │                  │              │
│  │       7          │  │       2          │  │       2          │              │
│  │  $ 284.500       │  │  $ 620.000       │  │  planes          │              │
│  │                  │  │                  │  │                  │              │
│  │ ⚪3 pendientes    │  │ ⚠ la más antigua │  │ ⚪1 planificado   │              │
│  │ 🔵2 en prep.      │  │   lleva 16 días  │  │ 🔵1 en proceso    │              │
│  │ 🟢2 entregados    │  │                  │  │                  │              │
│  │                  │  │                  │  │ Avance: 45,5%    │              │
│  │   [ Ver → ]      │  │    [ Ver → ]     │  │   [ Ver → ]      │              │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘              │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Las tarjetas están ordenadas por urgencia, no por número de criterio de aceptación.**
Lo vencido y lo agotado arriba, en formato grande: son las dos cosas que exigen una decisión
hoy. El resto es información de contexto.

Cada cifra es un enlace a la pantalla donde se actúa. El dashboard detecta; no resuelve.

El aviso "la más antigua lleva 16 días" convierte un número neutro en algo accionable: dos
compras pendientes no dicen nada, una compra pendiente de hace más de dos semanas sí.

### Vista de un rol operativo (VENTAS)

```
┌──────────────────────────────────────────────────────────────────┐
│  Resumen del día · miércoles 18 de septiembre de 2026            │
├──────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────┐  ┌──────────────────┐           │
│  │ 🔴 PRODUCTOS POR VENCER    │  │ PEDIDOS DE HOY   │           │
│  │        4                   │  │       7          │           │
│  │  ...                       │  │  $ 284.500       │           │
│  └────────────────────────────┘  └──────────────────┘           │
└──────────────────────────────────────────────────────────────────┘
```

Cada rol ve lo que le concierne (R-05). Mostrar a VENTAS el stock de materia prima solo
añade ruido.

---

## 2. Flujo

```
Usuario            Frontend                     Backend
   │                  │                            │
   │ Inicia sesión    │                            │
   ├─────────────────►│ (redirige a /dashboard)    │
   │                  │                            │
   │                  │ GET /api/v1/dashboard      │
   │                  ├───────────────────────────►│
   │                  │                            │ 5 consultas agregadas:
   │                  │                            │  · pedidos hoy   (curdate)
   │                  │                            │  · compras PENDIENTE
   │                  │                            │  · v_inventario (BAJO/AGOTADO)
   │                  │                            │  · planes hoy    (curdate)
   │                  │                            │  · lotes ≤ 7 días con stock
   │                  │◄───────────────────────────┤ 200 (< 2 s)
   │  Ve el resumen   │ (filtra tarjetas por rol)  │
   │◄─────────────────┤                            │
   │                  │                            │
   │ Clic en          │                            │
   │ "Registrar compra"│ → navega a HU-13          │
   ├─────────────────►│                            │
```

**Sin caché** (R-04): cada apertura consulta datos frescos.

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjetas | `MatCard` con jerarquía visual por urgencia |
| Cifra principal | Tipografía grande, con color según estado |
| Desglose por estado | `MatChip` pequeños |
| Semáforo | `MatIcon` con color |
| Enlaces | `MatButton` con `RouterLink` |
| Recarga | `MatIconButton` `refresh` + marca de hora |
| Carga por tarjeta | `MatProgressSpinner` individual |

Si se añaden gráficos, cargar antes la guía de visualización de datos del proyecto para
mantener la coherencia cromática con HU-26.

---

## 4. Validaciones en el formulario

El dashboard **no tiene formularios** (R-04): es solo lectura.

El único parámetro configurable es el horizonte de vencimiento, fijado en **7 días** por
CA-05. Se expone como parámetro opcional (`?diasVencimiento=`) para que HU-27 y la pantalla
de lotes puedan reutilizar el endpoint con otro horizonte.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | Spinner **por tarjeta**, no uno global |
| Tarjeta vacía sin incidencias | ✅ "Sin productos próximos a vencer" en verde |
| Stock bajo vacío | ✅ "Todas las materias primas por encima del mínimo" |
| Sin pedidos hoy | "Aún no hay pedidos registrados hoy" |
| Base vacía | Mensaje de bienvenida con enlaces a los maestros |
| Error en una tarjeta | Esa tarjeta muestra "No se pudo cargar" + reintentar; **las demás siguen funcionando** |

**Que una tarjeta falle no debe tumbar el dashboard** (R-04). Y los estados vacíos de las
tarjetas de alerta son buenas noticias: se muestran en verde, no con el mensaje genérico de
"sin datos".

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
