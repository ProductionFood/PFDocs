# HU-04 · Bitácora de auditoría — Diseño

---

## 1. Pantallas

### Consulta de bitácora (solo ADMIN, solo lectura)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Bitácora de auditoría                                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│  Desde ┌──────────┐  Hasta ┌──────────┐  Acción ┌─────────┐ Tabla ┌────────┐ │
│        │18/09/2026│        │18/09/2026│         │ Todas ▾ │       │Todas ▾ │ │
│        └──────────┘        └──────────┘         └─────────┘       └────────┘ │
├──────────────────────────────────────────────────────────────────────────────┤
│  FECHA ▼          USUARIO         ACCIÓN          TABLA       DETALLE        │
├──────────────────────────────────────────────────────────────────────────────┤
│  18/09 14:32:11   Admin Inicial   ●CREAR         usuarios    Usuario crea…   │
│  18/09 14:30:02   ⚙ Sistema       ●LOGIN_FALLIDO usuarios    Intento falli…  │
│  18/09 14:28:45   Carlos Cochero  ●EDITAR        proveedores Proveedor act…  │
│  18/09 14:15:30   María Gómez     ●CAMBIAR_ESTADO pedidos    Pedido 45: PEN… │
├──────────────────────────────────────────────────────────────────────────────┤
│                          Elementos por página: 20 ▾   1–4 de 128   ◀  ▶     │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Sin botones de acción**: la pantalla no tiene "Nuevo", ni menú `⋮` por fila, ni
posibilidad de editar. Es un registro, no un CRUD (CA-04).

El detalle se trunca en la celda; el texto completo aparece en un tooltip al pasar el cursor.

**"⚙ Sistema"** identifica los registros con `id_usuario = NULL`: acciones sin usuario
autenticado, como los intentos de login fallidos o el script de semilla.

---

## 2. Flujo

```
Cualquier servicio anotado                 AuditoriaAspect            bitacora
         │                                        │                       │
         │ clienteService.crear(...)              │                       │
         │ ───────── ejecuta ─────────►           │                       │
         │                                        │                       │
         │ ◄──── retorna con éxito ────           │                       │
         │                                        │                       │
         │              @AfterReturning se dispara│                       │
         │                                        │ idUsuarioActual()     │
         │                                        │ (null si es sistema)  │
         │                                        │ resumen del resultado │
         │                                        │ recorte a 255         │
         │                                        ├──────────────────────►│
         │                                        │      INSERT           │
         │                                        │                       │
         │                          si el INSERT falla: log, sin propagar │
```

**Si el método lanza excepción, `@AfterReturning` no se ejecuta** y no se audita nada
(R-02). **Si la auditoría falla, la operación de negocio ya terminó bien** y no se revierte
(R-03).

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tabla | `MatTable` + `MatPaginator` (sin acciones de fila) |
| Rango de fechas | `MatDateRangePicker` |
| Filtros | `MatSelect` (acción, tabla) |
| Acción | `MatChip` con color por tipo |
| Detalle largo | `MatTooltip` |
| Usuario "Sistema" | `MatIcon` `settings` + texto en cursiva |

---

## 4. Validaciones en el formulario

Esta pantalla **no tiene formularios de datos**. Solo los filtros:

| Filtro | Regla |
|---|---|
| Rango de fechas | `fechaInicio <= fechaFin`, validado antes de llamar |
| Acción | Opcional, del catálogo de la API |
| Tabla afectada | Opcional |

Por defecto se cargan los **últimos 7 días**. La bitácora crece rápido y cargarla completa
es lento sin aportar nada: quien la consulta suele buscar algo reciente.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | `MatProgressBar` |
| Vacío | "No hay registros en el período seleccionado" + sugerencia de ampliar el rango |
| Error | Mensaje + "Reintentar" |
| Con datos | Tabla ordenada por fecha descendente |

El estado vacío sugiere ampliar el rango porque la causa habitual es un filtro de fechas
demasiado estrecho, no la ausencia de actividad.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
