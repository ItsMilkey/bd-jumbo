# Especificación: Sistema de Gestión de Inventario — Pasillo de Carnes (Supermercado Jumbo)

Autores
- Hans Gómez
- Francisco Mardonez

Contexto y alcance
- Proyecto: Sistema de Gestión de Inventario para el pasillo de carnes del Supermercado Jumbo.
- Objetivo: automatizar y optimizar el control de existencias de productos cárnicos (por Kg), gestión de mermas y ciclo de Órdenes de Compra con frigoríficos. La implementación se realiza en Oracle PL/SQL (Oracle Database 19c, Oracle SQL Developer).
- Alcance funcional: registro/gestión de productos cárnicos (vacuno, pollo, cerdo, etc.), movimientos por peso (INGRESO, EGRESO, MERMA), gestión completa de OCs (crear cabecera, agregar detalle, recibir y conciliar), proveedores especializados (frigoríficos), alertas por stock mínimo y auditorías de conteo físico.

Resumen técnico
- Tecnología base: Oracle Database 19c, PL/SQL (paquetes, procedimientos, funciones, triggers), tipos compuestos (RECORD, VARRAY o Nested Tables).
- Patrón de diseño: centralizar la lógica de negocio en `pkg_inventario_carnes` — todas las modificaciones de `stock_actual_kg` deben pasar por `registrar_movimiento`.
- Tipos compuestos: usar RECORDs para representar un `producto_carne` y Nested Tables para pasar listas de detalle de OC en una sola llamada, reduciendo IO y mejorando rendimiento en operaciones por lote.

Requerimientos funcionales (alineados al informe)
1) Validaciones y reglas de negocio
   - Validar existencia y `estado` ('A'|'I') para IDs: producto, proveedor, empleado, supermercado, orden.
   - Validar cantidades en Kg: > 0 y por debajo de `max_delta_kg` (parámetro configurado en el package).
   - Tipos de movimiento permitidos: {'INGRESO','EGRESO','MERMA'}; normalizar entradas (mayúsculas) y rechazar aliases.

2) Gestión de Órdenes de Compra (OC)
   - Flujo: `crear_orden_compra_header` -> múltiples `agregar_detalle_orden` -> `recibir_orden_compra`.
   - `recibir_orden_compra` debe ser idempotente: detectar `estado = 'RECIBIDA'` y devolver código/exception `ex_orden_ya_recibida` (-20024).
   - Al recibir, cada línea del detalle actualizará stock vía `registrar_movimiento` (INGRESO) y marcará la OC como 'RECIBIDA'.

3) Control de mermas y ajustes
   - `MERMA` decrementa stock y debe lanzar `ex_stock_insuficiente` (-20010) si no hay suficiente stock.
   - Auditoría de conteo: `registrar_auditoria_conteo` registra el conteo real, calcula diferencia, guarda en `auditorias_inventario` y —si corresponde— ajusta stock a través de `p_actualizar_stock` (con control y logs).

4) Alertas y triggers
   - Trigger `trg_alerta_stock` se dispara AFTER UPDATE OF `stock_actual_kg` y crea registros en `alertas_inventario` cuando el stock cae por debajo del mínimo o llega a cero.
   - Alertas deben ser idempotentes por combinación (producto_id, tipo_alerta, fecha) o marcar la última alerta como no atendida.

5) Observabilidad y errores
   - Definir códigos de error estándar (-20002..-20030) y mensajes claros.
   - Registrar eventos importantes y errores en tablas `audit_logs` y/o `error_codes` para pruebas automáticas y trazabilidad.

Requerimientos no funcionales
- Rendimiento: índices necesarios en `productos_carne(producto_id,categoria_id,supermercado_id)`, `ordenes_compra(orden_id,proveedor_id)` y `movimientos_inventario(producto_id,fecha_movimiento)`.
- Seguridad: validar y usar binds en capa de aplicación; no construir SQL dinámico con concatenación insegura.
- Mantenibilidad: documentar contratos públicos del package; centralizar excepciones y constantes.

Criterios de aceptación (tests mínimos automatizables)
- Ingreso/egreso/merma actualizan stock y generan `movimientos_inventario`.
- Egreso/merma con stock insuficiente falla con -20010.
- Creación de OC y recepción: total calculado, stock incrementado, OC marcada como 'RECIBIDA'; reintento de recepción falla con -20024.
- Auditoría: `registrar_auditoria_conteo` inserta registro en `auditorias_inventario` y ajusta stock si hay diferencia.
- Obtener productos bajo mínimo por categoría devuelve colección (VARRAY/Nested Table) con IDs esperados.

Diseño interno recomendado (contrato y helpers)
- Constantes y parámetros públicos del package: `VARARRAY_MAX_SIZE`, `MAX_DELTA_KG`, `DEFAULT_STOCK_MINIMO`.
- Excepciones públicas declaradas en la especificación: `ex_stock_insuficiente`(-20010), `ex_cantidad_negativa`(-20011), `ex_tipo_movimiento_invalido`(-20012), `ex_producto_no_encontrado`(-20013), `ex_orden_ya_recibida`(-20024), `ex_orden_cancelada`(-20025).
- Helpers internos (package body):
  - `p_is_active_producto(p_producto_id) RETURN BOOLEAN` — valida existencia y estado.
  - `p_lock_producto(p_producto_id)` — SELECT FOR UPDATE para evitar race conditions.
  - `p_validar_cantidad(p_cantidad_kg)` — valida >0 y <= MAX_DELTA_KG.

Patrones de transacción
- Preferir que procedimientos complejos no hagan COMMIT implícito; documentar dónde se hace COMMIT (caller vs procedimiento). Para operaciones unitarias (p. ej. `registrar_movimiento`) se puede permitir commit controlado según contrato.

Uso de tipos compuestos y cursores
- Emplear RECORDS para representar filas de `productos_carne` en procedimientos que necesitan más atributos que solo `producto_id`.
- Preferir Nested Tables para el payload de detalle de OC si la aplicación puede enviar la colección; como alternativa usar VARRAYs con tamaño documentado.
- Cursores explícitos se usan para procesar líneas de OC en `recibir_orden_compra` y para construir colecciones en `obtener_productos_bajo_minimo_categoria`.

Gestión de excepciones
- Mapear excepciones de negocio a códigos y mensajes claros. Evitar `ROLLBACK` dentro del package body: re-lanzar `RAISE` y dejar control transaccional al caller salvo que el procedimiento sea transaccional por diseño.

Recomendaciones y trabajo futuro
- Integración con balanzas del mesón para registrar decrementos por venta en Kg.
- Desarrollo de una UI ligera (web/móvil) que consuma `pkg_inventario_carnes`.
- Analítica: explotar `movimientos_inventario` para predicción de demanda y optimización de OCs.

Referencias
- Informe del proyecto (Autores: Hans Gómez, Francisco Mardonez) — Taller de base de datos.
- Script base: `c:\Users\SB-Alumno\Downloads\Untitled-1.sql`

-- Fin de archivo: docs/inventario_spec.md
