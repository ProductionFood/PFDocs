# HU-23 · Inventario automático por producción — Tarea Frontend

Sin pantallas propias: son las de HU-22 más la presentación de las alertas de stock. El foco está en que el usuario **vea el efecto sobre el inventario**.

---

## 1. Pasos

1. En el diálogo de registro de consumo (HU-22), mostrar el **impacto previsto** sobre
   cada materia prima antes de confirmar.
2. Tras registrar, mostrar los **saldos resultantes** de cada ingrediente.
3. `alertas-stock.component`: banner con las materias primas que quedaron bajo mínimo tras
   la operación (R-04), con enlace a HU-27 y a crear una compra (HU-13).
4. Distinguir visualmente `BAJO` (ámbar) de `AGOTADO` (rojo).
5. Manejar `409 STOCK_INSUFICIENTE` mostrando **disponible y requerido**, y sugiriendo el
   ajuste de inventario como camino (R-05).
6. Indicador global de alertas en la barra superior, visible desde cualquier pantalla para
   ADMIN, PRODUCCION y COMPRAS.
7. Si se produce `INVENTARIO_NO_INICIALIZADO`, mensaje explícito: es un defecto de datos, no
   un error del usuario.

---

## 2. Archivos

```
frontend/src/app/
├── features/produccion/components/impacto-inventario/
├── shared/components/alertas-stock/
└── layout/components/indicador-alertas/
```

---

## 3. Puntos de cuidado

- **Mostrar el impacto antes y los saldos después** cierra el ciclo: el usuario entiende
  que registrar consumo mueve inventario, en lugar de verlo como un formulario más.
- **El mensaje del `409` debe traer las dos cifras** (disponible y requerido) y sugerir qué
  hacer. Un "stock insuficiente" a secas deja al productor sin salida cuando la materia
  prima sí estaba físicamente.
- La alerta de stock bajo **no debe bloquear ni interrumpir**: es información posterior a
  una operación ya consumada (R-04).
- El indicador global conecta esta historia con HU-27 y con la decisión de comprar.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
