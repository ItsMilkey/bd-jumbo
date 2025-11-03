# Plan de trabajo detallado: Implementación y pruebas sobre el script PL/SQL

Propósito
- Traducir la especificación (c:\Users\SB-Alumno\Downloads\docs\documents\inventario_spec.md) en un plan de trabajo ejecutable para mejorar la robustez, validaciones y pruebas del sistema de inventario del pasillo de carnes (Supermercado Jumbo).

Entregables
1. `c:\Users\SB-Alumno\Downloads\docs\documents\inventario_plan.md` (este documento).
2. Cambios en `c:\Users\SB-Alumno\Downloads\docs\scripts\Untitled-1.sql` (package `pkg_inventario_carnes` y trigger `trg_alerta_stock`): helpers, validaciones y manejo de errores.
3. Scripts de pruebas automáticas en `c:\Users\SB-Alumno\Downloads\tests\` (SQL blocks) que cubran criterios de aceptación.
4. Scripts de despliegue/reversión (deploy / rollback SQL).

Supuestos razonables
- La base de datos objetivo es Oracle 19c.
- El entorno de trabajo para pruebas será Oracle SQL Developer o SQL*Plus conectado a una instancia de desarrollo.
- No hay consumidores externos críticos con dependencias directas en la implementación interna del package (solo se respetan las firmas públicas).

Fases, tareas, estimaciones y responsables
Duración estimada total: 4 días (sprint corto)

Fase 0 — Preparación y análisis (0.5d)
- T0.1: Revisar `Untitled-1.sql` para identificar puntos de extensión (firmas públicas, excepciones ya declaradas, cursores usados). (0.25d)
- T0.2: Ejecutar script en entorno dev (si está disponible) para confirmar esquema y datos de prueba. (0.25d)

Fase 1 — Diseño y constantes / códigos de error (0.5d)
- T1.1: Definir y documentar en spec la lista final de códigos de error (-20002..-20030). (0.1d)
- T1.2: Añadir constantes públicas en la spec y esqueleto en `pkg_inventario_carnes` (VARARRAY_MAX_SIZE, MAX_DELTA_KG, códigos de error). (0.2d)
- T1.3: Crear tabla auxiliar opcional `error_codes` y `audit_logs` (si se aprueba). (0.2d)

Fase 2 — Implementación de helpers y validaciones (1.0d)
- T2.1: Implementar helpers en package body:
  - `p_is_active_producto(p_producto_id) RETURN BOOLEAN` (comprueba existencia y estado).
  - `p_is_active_entity(p_table_name VARCHAR2, p_id NUMBER) RETURN BOOLEAN` (genérico, si procede).
  - `p_validar_cantidad(p_cantidad NUMBER)` — valida >0 y <= MAX_DELTA_KG.
  - `p_validar_tipo_movimiento(p_tipo VARCHAR2)` — normaliza y valida allowed set.
  - `p_lock_producto(p_producto_id)` — SELECT ... FOR UPDATE wrapper.
  Estimación: 0.5d
- T2.2: Refactorizar `registrar_movimiento` para usar helpers y centralizar validación y excepciones. (0.5d)

Fase 3 — Consistencia transaccional y concurrencia (0.5d)
- T3.1: Revisar puntos donde aplican `SELECT ... FOR UPDATE` y asegurar locks mínimos (p_lock_producto en registrar_movimiento y recibir_orden_compra). (0.25d)
- T3.2: Documentar política de commit (procedimientos no harán COMMIT salvo los scripts de prueba o wrappers que se definan explícitamente). (0.25d)

Fase 4 — Manejo de órdenes y idempotencia (0.5d)
- T4.1: Refactorizar `recibir_orden_compra`: asegurar detección de `estado = 'RECIBIDA'`, lanzar `ex_orden_ya_recibida` (-20024) y que cada línea de detalle invoque `registrar_movimiento` en modo INGRESO. (0.3d)
- T4.2: Añadir control de errores por línea (capturar errores por producto y acumular en `audit_logs` si falla parcialmente). (0.2d)

Fase 5 — Observabilidad y pruebas (0.75d)
- T5.1: Crear carpeta `c:\Users\SB-Alumno\Downloads\tests\` y archivos de test SQL:
  - `01_movimientos_happy.sql` — Ingresos/egresos/mermas exitosos.
  - `02_movimientos_errors.sql` — Cantidad negativa, stock insuficiente.
  - `03_ordenes_flow.sql` — Crear OC, agregar detalle, recibir y reintento (debe fallar con -20024).
  - `04_auditoria.sql` — Registrar auditoría y validar ajuste de stock.
  - `05_varray_tests.sql` — Probar `obtener_productos_bajo_minimo_categoria`.
  Estimación: 0.5d
- T5.2: Ejecutar tests en dev y corregir errores menores. (0.25d)

Fase 6 — Revisión, documentación y despliegue (0.25d)
- T6.1: Revisar cambios, añadir comentarios en package y actualizar `docs/inventario_spec.md` con los cambios realizados.
- T6.2: Crear scripts de despliegue y rollback (ficheros `deploy/` y `rollback/`).

Artefactos a crear / modificar
- `c:\Users\SB-Alumno\Downloads\docs\inventario_plan.md` (este archivo).
- Actualizaciones en `c:\Users\SB-Alumno\Downloads\Untitled-1.sql`:
  - Añadir constantes en la especificación del package.
  - Implementar helpers y refactorizar procedimientos (body).
- Tests en `c:\Users\SB-Alumno\Downloads\tests\` (5 scripts SQL).
- Opcional: `c:\Users\SB-Alumno\Downloads\deploy\deploy_pkg_inventario.sql` y `c:\Users\SB-Alumno\Downloads\deploy\rollback_pkg_inventario.sql`.

Pruebas automáticas (resumen de contenido de cada test)
- 01_movimientos_happy.sql
  - Inserta datos de prueba mínimos.
  - Ejecuta `pkg_inventario_carnes.registrar_movimiento` con INGRESO y verifica `verificar_stock_kg` y `movimientos_inventario`.
- 02_movimientos_errors.sql
  - Intenta EGRESO/MERMA mayor al stock; espera código -20010.
  - Intenta registrar cantidad <= 0; espera código -20011.
- 03_ordenes_flow.sql
  - Crea orden, agrega detalles, recibe la orden; valida stock incrementado y estado RECIBIDA.
  - Reintento de recibir la misma orden debe devolver -20024.
- 04_auditoria.sql
  - Ejecuta `registrar_auditoria_conteo` con diferencia y valida inserción en `auditorias_inventario` y ajuste de `productos_carne`.
- 05_varray_tests.sql
  - Llena datos que garanticen productos bajo mínimo y valida que `obtener_productos_bajo_minimo_categoria` retorne lista con COUNT>0.

Patrón de despliegue y rollback
- Despliegue en dev: ejecutar scripts en orden T1..T6 en una sesión controlada.
- Rollback: preservar copia del package actual (CREATE OR REPLACE con versión anterior) en `deploy/rollback`.

Riesgos conocidos y mitigaciones
- Deadlocks por locks prolongados: mitigar manteniendo transacciones cortas y aplicando locks sólo en `registrar_movimiento` y `recibir_orden_compra` por la fila necesaria.
- Cambios en commits automáticos pueden afectar integraciones: documentar la semántica exacta y comunicar a equipos consumidores.

Próximos pasos (inmediatos)
1. Aprobación: confirma que este plan cubre las expectativas y el alcance. Si falta algo (p. ej. integración con sistemas externos), indícalo.
2. Tras tu confirmación, empiezo la fase 0: revisión del script y creación de los helpers básicos (marcaré la tarea en la lista y crearé los cambios iniciales en el package).


---

## Análisis de la spec: puntos de mejora (resumen)
A continuación se listan observaciones prácticas sobre la especificación (`docs/inventario_spec.md`) y recomendaciones para ajustar la lógica de negocio antes de implementar cambios sobre el package y triggers.

1) Política transaccional y atomicidad en `recibir_orden_compra`
  - Observación: la spec indica que `recibir_orden_compra` iterará líneas y llamará a `registrar_movimiento`. No queda explícito si la operación debe ser atómica (todas las líneas se reciben o ninguna) o si acepta recepciones parciales.
  - Riesgo: si una línea falla (producto inactivo, cantidad inválida) y las demás se aplican, el inventario quedará inconsistente respecto a la OC y complicará auditorías.
  - Recomendación: definir y adoptar una política clara — sugerimos modalidad ATÓMICA por defecto (todas o ninguna). Si se necesita soporte para recepciones parciales, implementarlo como modo opcional (flag `p_allow_partial BOOLEAN`) con compensación y registro de errores por línea en `audit_logs`.

2) Idempotencia y manejo de re-ejecuciones
  - Observación: la spec exige idempotencia en `recibir_orden_compra` (detectar `RECIBIDA`). Falta definición en operaciones como `registrar_auditoria_conteo` o `registrar_movimiento` si un reintento es seguro.
  - Recomendación: definir claves de idempotencia (por ejemplo, `orden_id` + `empleado_id` + timestamp o `external_reference`) y/o restricción en tablas históricas para detectar duplicados. Añadir pruebas que re-ejecuten llamadas y verifiquen no duplicación de stock.

3) Manejo parcial de errores y estrategia de compensación
  - Observación: no queda claro si fallos por producto durante la recepción deben abortar todo el proceso. Tampoco está definido cómo reportar back-end/app esos fallos.
  - Recomendación: por defecto abortar y retornar error. Implementar un modo avanzado que registre errores por línea en `error_logs` y permita continuar (modo `p_allow_partial`), retornando un resumen con líneas aplicadas y líneas fallidas.

4) Validaciones genéricas vs específicas por entidad
  - Observación: la spec propone `p_is_active_entity(p_table_name, p_id)` genérica. Esto suele requerir SQL dinámico y aumenta superficies de fallo/inyección.
  - Recomendación: implementar validadores específicos por entidad (`p_is_active_producto`, `p_is_active_proveedor`, `p_is_active_empleado`, `p_is_active_supermercado`) evitando SQL dinámico. Si se mantiene genérico, restringirlo a una lista blanca de tablas y columnas.

5) Alertas desde trigger (`trg_alerta_stock`)
  - Observación: el trigger crea alertas AFTER UPDATE; si la lógica de deduplicación depende de consultas pesadas, puede penalizar throughput.
  - Recomendación: simplificar el trigger a una inserción mínima (producto_id, tipo_alerta, fecha) y mover la lógica de enriquecimiento / deduplicación a un proceso batch o a un procedimiento que se ejecute fuera del path crítico. Si la deduplicación debe ocurrir en línea, usar índices/unique constraints parciales (p. ej. única por producto+tipo+fecha_trunc) y evitar SELECTs complejos dentro del trigger.

6) Auditorías y `audit_logs`
  - Observación: la spec recomienda `audit_logs` y `auditorias_inventario`. Falta formato consistente para logs (quién hizo la acción, prev/new value, razón, referencia externa).
  - Recomendación: definir schema común para logs: `log_id, entidad, entidad_id, accion, usuario, session_id, antes_json, despues_json, motivo, fecha_hora`.

7) Uso de tipos compuestos (VARRAY vs Nested Table)
  - Observación: la spec sugiere VARRAY o Nested Tables. En entornos de prueba, VARRAY es más sencillo; en producción, Nested Table ofrece más flexibilidad.
  - Recomendación: si la aplicación cliente no puede pasar colecciones Oracle, mantener API de detalle por inserciones individuales, y ofrecer una overload que acepte Nested Table para cargas por lote. Documentar claramente la limitación.

8) Mapeo de códigos de error y mensajes localizables
  - Observación: la lista de códigos propuesta existe pero falta una tabla `error_codes` con texto y severidad.
  - Recomendación: crear `error_codes(code NUMBER PRIMARY KEY, short_msg VARCHAR2, long_msg VARCHAR2, severity VARCHAR2)` y usarla en tests para verificar mensajes. Esto ayuda a mantener consistencia y permite traducciones posteriores.

## Cambios propuestos en el plan (delta)
Basado en el análisis anterior, propongo las siguientes modificaciones al plan y nuevas tareas concretas.

1) Política transaccional (nueva tarea)
  - Añadir tarea: T2.3a "Definir y codificar política atómica para `recibir_orden_compra`".
  - Implementación: `recibir_orden_compra(p_orden_id, p_empleado_id, p_allow_partial BOOLEAN DEFAULT FALSE)` y pruebas para ambos modos.

2) Idempotencia y claves (nueva tarea)
  - Añadir tarea: T2.3b "Diseñar e implementar claves de idempotencia y restricciones en tablas históricas".
  - Implementación: añadir columnas opcionales `external_ref VARCHAR2` y unique indexes donde aplique; crear pruebas de re-ejecución.

3) Validadores por entidad (ajuste a T2.1)
  - En lugar de helper genérico, implementar helpers específicos por entidad (menos riesgo, más claro). Actualizar T2.1 para reflejar esto.

4) Alertas y trigger (ajuste a Fase 3)
  - Modificar T3.1 para simplificar el trigger: sólo INSERT minimal; crear job/procedimiento opcional para consolidación y enriquecimiento de alertas.

5) Error codes table y audit_logs (mover a Fase 1)
  - Mover creación de `error_codes` y `audit_logs` desde Fase 1 a tarea prioritaria (T1.3 -> T1.3a) para que tests y código puedan usar tablas desde el inicio.

6) Tests de fallo parcial y de idempotencia (ampliación de Fase 5)
  - Añadir tests que ejerciten recepciones parciales, re-ejecuciones y fallos por producto inactivo.

## Tareas nuevas resumidas (para plan)
- T1.3a: Crear `error_codes` y `audit_logs` (prioritario).
- T2.3a: Implementar política atómica y flag `p_allow_partial` en `recibir_orden_compra`.
- T2.3b: Implementar idempotencia (external_ref y unique constraints / índices).
- T3.1a: Simplificar `trg_alerta_stock` y crear procedimiento de consolidación de alertas.
- T5.3: Tests de idempotencia y recepción parcial.

## Impacto en estimaciones
Las tareas nuevas añaden ~0.5d a la estimación total (principalmente por pruebas y diseño de idempotencia). Nueva estimación total: ~4.5 días.

## Próximos pasos (después de tu aprobación)
1. Aprobar las decisiones arriba (atomicidad por defecto, validadores por entidad, trigger simple, crear `error_codes` y `audit_logs`).
2. Marcar T3 (análisis) como completada y actualizar la lista de tareas para iniciar la implementación (helpers y changeset en `pkg_inventario_carnes`).

-- Fin de la sección de análisis y cambios propuestos
