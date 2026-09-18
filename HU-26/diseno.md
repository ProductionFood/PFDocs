# HU-26 · Eficiencia de producción — Diseño

---

## 1. Pantallas

### Reporte de eficiencia

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│  Eficiencia de producción          Período ┌─────────────────────────┐             │
│                                            │ 01/09/2026 – 30/09/2026 │             │
│                                            └─────────────────────────┘             │
├────────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐       │
│  │ CUMPLIMIENTO DEL PLAN│ │ RENDIMIENTO AJUSTADO │ │ DESVIACIÓN MAT. PRIMA│       │
│  │                      │ │                      │ │                      │       │
│  │       87,4 %         │ │       96,8 %         │ │      +5,8 %  🟠      │       │
│  │                      │ │                      │ │                      │       │
│  │ Se produjo el 87,4 % │ │ Por cada unidad      │ │ Se gastó un 5,8 %    │       │
│  │ de lo planificado    │ │ producida se usó un  │ │ más de materia prima │       │
│  │                      │ │ 3,2 % menos de lo    │ │ de lo previsto       │       │
│  │ 12 planes · 2 cancel.│ │ previsto             │ │                      │       │
│  └──────────────────────┘ └──────────────────────┘ └──────────────────────┘       │
│                                                                                    │
│  ℹ 3 consumos sin rendimiento calculable (cantidad real en cero), excluidos del    │
│    promedio.                                                                       │
├────────────────────────────────────────────────────────────────────────────────────┤
│  DESVIACIÓN POR MATERIA PRIMA  (acumulado del período)                             │
│                                                                                    │
│  Harina de trigo   ████████████████░░  +7,0 %   180,0 → 192,6 kg   🟠 12 planes   │
│  Mantequilla       ███████░░░░░░░░░░░  +3,1 %    42,0 →  43,3 kg   🟡 12 planes   │
│  Levadura          ░░░░░░░░░░░░░░░░░░  −0,3 %     1,8 →   1,8 kg   🟢 12 planes   │
│  Azúcar            ◄◄◄◄░░░░░░░░░░░░░░  −4,2 %    60,0 →  57,5 kg   🟡 11 planes   │
├────────────────────────────────────────────────────────────────────────────────────┤
│  PLANES DEL PERÍODO                                                                │
│  PLAN  FECHA       RESPONSABLE       CUMPLIM.  RENDIM.  AJUSTADO   ESTADO         │
├────────────────────────────────────────────────────────────────────────────────────┤
│ ▾ #9   18/09/2026  Ismael Batalla     66,7 %   92,6 %    61,7 %   🟢 Completado   │
│    └─ Pan francés         planificado 90 und  ·  producido 60 und   66,7 %         │
│    └─ Harina de trigo     teórica 15,000  real 16,200   +8,0 %   ef. 92,6 %        │
│    └─ Sal                 teórica  0,300  real  0,000  −100,0 %   ef.    —  ⓘ     │
│ ▸ #8   17/09/2026  Natalia Góngora    100,0 %   98,1 %    98,1 %   🟢 Completado   │
├────────────────────────────────────────────────────────────────────────────────────┤
│  ⚠ PRODUCTOS SIN RECETA (fuera del análisis)                                       │
│     Pan integral — 2 planes. Sin receta no hay consumo teórico.  [ Crear receta → ]│
└────────────────────────────────────────────────────────────────────────────────────┘
```

**Cada tarjeta lleva una frase que explica el número.** "96,8%" no significa nada por sí
solo; "por cada unidad producida se usó un 3,2% menos de lo previsto" se entiende sin
consultar la fórmula.

**Las tres métricas están separadas** (R-01): cumplimiento y rendimiento responden a
preguntas distintas y nunca se promedian entre sí.

El `—` con ⓘ en la fila de Sal es `eficiencia: null` (R-03). El aviso sobre los 3 consumos
excluidos evita que el promedio parezca cubrir todo.

**La barra de desviación por materia prima es lo más accionable del reporte** (R-06): un
+7% sostenido en harina durante doce planes no es ruido, es una receta mal calibrada o
desperdicio sistemático.

### Por qué el ajustado importa — el caso del plan #9

```
┌──────────────────────────────────────────────────────────────────┐
│  ⓘ  Rendimiento vs. rendimiento ajustado                         │
│                                                                  │
│  Plan #9: se planificaron 90 panes, se produjeron 60.            │
│                                                                  │
│  El consumo teórico (15 kg) corresponde a 90 panes,              │
│  pero el consumo real (16,2 kg) corresponde a 60.                │
│                                                                  │
│  Rendimiento sin ajustar:   92,6 %  ← compara peras con manzanas │
│  Rendimiento ajustado:      61,7 %  ← comparable entre planes    │
│                                                                  │
│  El ajustado revela que, por pan producido, se consumió          │
│  bastante más harina de la que la receta indica.                 │
└──────────────────────────────────────────────────────────────────┘
```

Este tooltip explica R-02 en el punto donde importa. Sin el ajuste, el plan que peor cumplió
aparecería como el más eficiente del período.

---

## 2. Flujo

```
Administrador        Frontend                   Backend
     │                  │                          │
     │ Abre el reporte  │ GET /reportes/eficiencia?mes-actual
     ├─────────────────►├─────────────────────────►│
     │                  │                          │ planes con consumo (R-05)
     │                  │                          │ por cada plan:
     │                  │                          │   cumplimiento  = prod/plan
     │                  │                          │   rendimiento   = teo/real  ← NULL si real=0
     │                  │                          │   ajustado      = (teo×prod/plan)/real
     │                  │                          │   desviación    = (real−teo)/teo
     │                  │                          │ promedios EXCLUYENDO nulos  ← R-03
     │                  │ GET .../materias-primas  │
     │                  ├─────────────────────────►│ agregado del período  ← R-06
     │                  │◄─────────────────────────┤
     │  Ve el análisis  │                          │
     │◄─────────────────┤                          │
```

---

## 3. Componentes Angular Material

| Elemento | Componente |
|---|---|
| Tarjetas de métricas | `MatCard` con cifra, unidad y **frase explicativa** |
| Desviación por insumo | Barras divergentes (positivas y negativas desde el cero) |
| Tabla de planes | `MatTable` expandible |
| Métrica nula | `—` + `MatIcon` `info` con `MatTooltip` |
| Explicación del ajustado | `MatDialog` o `MatTooltip` extenso |
| Rango de fechas | `MatDateRangePicker` |
| Sin receta | `MatCard` ámbar con `RouterLink` a HU-12 |

**Cargar la guía de visualización de datos del proyecto antes de escribir los gráficos.**
Las barras divergentes requieren una paleta que distinga exceso de defecto sin depender solo
del color.

---

## 4. Validaciones en el formulario

Pantalla de consulta: **sin formularios de datos**. Solo el filtro de período:

| Filtro | Regla |
|---|---|
| Rango de fechas de producción | `fechaInicio <= fechaFin` |

Por defecto, **mes en curso**.

## 5. Estados de la pantalla

| Estado | Qué se muestra |
|---|---|
| Cargando | Tarjetas en esqueleto |
| Sin planes en el período | "No hay planes con consumo registrado en el período" |
| **Métrica no calculable** | `—` con tooltip "No se puede calcular: consumo real en cero" |
| Consumos excluidos del promedio | ℹ Aviso con el conteo |
| Productos sin receta | ⚠️ Sección aparte con enlace a HU-12 |
| Solo planes cancelados | Aviso de que el promedio puede no ser representativo |
| Error | Mensaje + "Reintentar" |

**Nunca mostrar `0%` donde el cálculo dio `null`** (R-03): un cero inventado es peor que un
hueco declarado, porque se promedia y se interpreta.

---

## 6. Accesibilidad y UX

- Todos los campos con `<mat-label>`; nunca solo *placeholder*.
- Navegación por teclado en orden lógico; `Enter` envía el formulario.
- Los mensajes de error del servidor se muestran **junto al campo** (`fieldErrors`).
- El botón de envío se deshabilita mientras la petición está en vuelo.
- Confirmación antes de descartar cambios sin guardar.
- Los tres estados obligatorios (cargando / vacío / error) según
  `00-base/05-ESTANDARES-QA.md` §7.
