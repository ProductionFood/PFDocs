# HU-27 · Alerta de stock bajo — Tarea Frontend

Pantalla de alertas más el indicador global de la barra superior. Debe llevar a la acción: de la alerta a la compra en un clic.

---

## 1. Pasos

1. `alerta.model.ts` y `alerta.service.ts`.
2. `alertas-stock.component`:
   - tabla ordenada por urgencia: agotadas primero (R-03);
   - columnas: materia prima, disponible, mínimo, **falta**, nivel, costo de reposición.
3. **Barra visual de nivel** por fila: `disponible / minimo`, en rojo si es cero.
4. Tarjetas de resumen: total en alerta, agotadas, costo de reposición estimado.
5. `sugerencia-compra.component`: propuesta agrupada por proveedor, con botón
   **"Crear compra"** que prellena el formulario de HU-13 con esos ítems.
6. **Indicador global** en la barra superior (`MatBadge`), que consulta `/alertas/resumen`
   al cargar y al volver a la pestaña.
7. Filtro para incluir materias primas inactivas (R-05), desactivado por defecto.
8. Estado vacío en **verde**: no tener alertas es una buena noticia.
9. Ruta: ADMIN/PRODUCCION/COMPRAS/CONSULTA. La sugerencia de compra, ADMIN y COMPRAS.

---

## 2. Archivos

```
frontend/src/app/features/inventario/
├── models/alerta.model.ts
├── services/alerta.service.ts
├── pages/{alertas-stock,sugerencia-compra}/
└── components/nivel-stock-bar/
frontend/src/app/layout/components/indicador-alertas/
```

---

## 3. Puntos de cuidado

- **El botón "Crear compra" es lo que cierra el ciclo.** Una alerta que obliga a apuntar en
  papel qué comprar y volver a teclearlo en otra pantalla se acaba ignorando.
- **El estado vacío va en verde con un mensaje positivo**: "Todas las materias primas están
  por encima del mínimo". Usar el mensaje genérico de "sin resultados" hace dudar de si la
  consulta falló.
- **El indicador global no debe refrescarse agresivamente**: al cargar y al volver a la
  pestaña basta. Un sondeo cada pocos segundos multiplica las consultas sin aportar nada.
- Las agotadas deben distinguirse visualmente de las bajas (R-03): son urgencias distintas.
- El costo de reposición es una referencia (HU-08 R-03); indicarlo para que no se tome como
  cotización.

---

## 4. Checklist de cierre

- [ ] Todos los CA cubiertos desde la interfaz.
- [ ] Los tres estados (cargando / vacío / error) implementados.
- [ ] Errores del servidor mostrados junto al campo correspondiente.
- [ ] Guard de rol aplicado en la ruta.
- [ ] Modelos TypeScript espejo de los DTO del backend.
- [ ] Probado en Chrome y Firefox a 1366×768.
- [ ] Sin `console.log` ni `any`.
