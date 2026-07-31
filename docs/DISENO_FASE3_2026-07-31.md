# CFO Personal — Fase 3: Diseño y roadmap

**Fecha:** 2026-07-31
**Punto de partida:** `docs/AUDITORIA_2026-07-31.md` (diagnóstico consolidado de la Fase 2.2). Este documento no vuelve a auditar ni genera evidencia nueva — parte del diagnóstico ya cerrado.
**Alcance de esta fase:** evaluar alternativas, definir reglas de negocio donde el comportamiento dependía de una decisión de producto, y formalizar bloques de trabajo con objetivo, problema, alcance, fuera de alcance, riesgos, dependencias, validaciones, rollback y criterio de éxito. **No se implementó nada en esta fase.**

---

## Reglas de negocio fijadas durante el diseño

Estas decisiones no están atadas a ningún bloque en particular — son la base normativa que los bloques implementan.

1. **Definición única de "Disponible del mes":** `Disponible = ingresos + uso de ahorro + movimientos internos − gastos fijos − gastos variables que impactan ese mes − ahorro aportado`. Toda funcionalidad que represente dinero disponible debe usar exactamente esta definición, salvo regla de negocio explícita en contrario.
2. **Invariante de cuotas:** la suma de las cuotas generadas por una compra debe ser exactamente igual al monto original del movimiento — con precisión a centavos en las cuotas intermedias y reconciliación del residuo en la última cuota.
3. **Criterio de caja para gastos variables:** un gasto variable descuenta de "Disponible" únicamente si el dinero efectivamente salió de la cuenta ese mes. Esto incluye todo medio de pago que represente esa salida real (hoy: Cuenta/Billetera y Débito) y excluye los que no (hoy: Crédito, hasta que se paga el resumen).
4. **Criterio de compromiso para gastos fijos:** un gasto fijo descuenta de "Disponible" en el momento en que se asume el compromiso, independientemente del medio de pago — sin excepción por el caso de que, excepcionalmente, se pague en cuotas de tarjeta.
5. **Invariante de no duplicación:** un mismo gasto nunca puede impactar dos veces el Disponible del mes, bajo ninguna circunstancia.

---

## Orden de implementación (priorizado)

Priorizado por impacto funcional (criterio principal), con riesgo de implementación, dependencias y relación beneficio/esfuerzo como desempate. Ningún bloque depende de otro — el orden es flexible por diseño. El Bloque F se adelantó al primer lugar por una razón práctica (documentar mientras el razonamiento está fresco), no por impacto funcional.

| # | Bloque | Impacto funcional | Riesgo | Dependencias | Beneficio/esfuerzo |
|---|---|---|---|---|---|
| 1 | F — Documentación | Nulo sobre comportamiento; alto sobre mantenibilidad | Mínimo | Ninguna | Alto |
| 2 | A — Motor de cálculo unificado | Alto | Medio | Ninguna | Alto |
| 3 | B.2 — Fijo en cuotas | Medio (confirmado, uso excepcional) | Bajo | Ninguna | Alto |
| 4 | C — Escrituras Objetivo/Config | Medio (preventivo) | Bajo-medio | Ninguna | Medio-alto |
| 5 | B.1 — Débito | Bajo-medio (sin manifestación en datos reales hoy) | Bajo | Ninguna | Alto |
| 6 | D — Concurrencia + resync | Bajo (preventivo) | Bajo | Ninguna | Medio |
| 7 | Hardening (Aplicación) | Nulo (preventivo) | Muy bajo | Ninguna | Alto |
| 8 | Hardening (Infraestructura) | Nulo (preventivo) | Bajo-medio | Ninguna | Medio-alto |
| 9 | Accesibilidad (labels) | Nulo sobre lo financiero | Mínimo | Ninguna | Alto |

**Fuera del roadmap de bloques:**
- `eliminarObjetivo()` sin confirmación y escaping faltante (`renderIngresos`, `renderTarjeta`) — bugs puntuales, se resuelven cuando se toque esa zona del código (reproducir → corregir → validar → commit), sin bloque propio.
- MFA — decisión operativa personal de Diego como único usuario, no trabajo de proyecto.
- Content-Security-Policy — bloque futuro específico, no diseñado todavía (requiere cuidado para no romper la carga de recursos externos).

---

## 1. Bloque F — Documentación del esquema y de decisiones

**Objetivo:** dejar documentado, dentro del repositorio, el estado actual del modelo de datos y las decisiones de negocio/arquitectura no reconstruibles desde el código o la base.

**Problema que resuelve:** todo el modelo de datos se reconstruyó en la auditoría exclusivamente por consulta directa a Supabase, sin ningún artefacto en el repositorio que lo describa. Tampoco existe registro de por qué se tomaron ciertas decisiones.

**Alcance:**
- `docs/SCHEMA.md`: estado actual de las 4 tablas (`movimientos`, `subcategorias_config`, `config_usuario`, `fijos_config`) — columnas, tipos, constraints (PK/FK), políticas RLS, índices.
- `docs/DECISIONS.md`: incluye, como mínimo, los criterios ya fijados en esta Fase 3 (fórmula de Disponible, tratamiento de medios de pago e invariante de no duplicación para fijos en cuotas), y los umbrales existentes del motor de cálculo (score, anomalías y límite de tarjeta), documentando su valor y el comportamiento que implementan actualmente. Cuando el repositorio no permita reconstruir el motivo histórico de una decisión, el documento debe indicarlo explícitamente, sin inferir justificaciones.

**Fuera de alcance:** migraciones SQL retroactivas o hacia adelante; automatización de sincronización de `SCHEMA.md`; detalle de implementación de los demás bloques.

**Riesgos:** `SCHEMA.md` puede desactualizarse si el esquema cambia sin actualizar el documento — mitigado solo por la aclaración explícita de que la base prevalece, no por ningún mecanismo automático.

**Dependencias:** ninguna.

**Validaciones:** `SCHEMA.md` coincide con el estado real de Supabase; `DECISIONS.md` incluye como mínimo las decisiones de Bloque A, B.1, B.2, C y D; ambos documentos indican explícitamente qué es autoritativo (código/base) y qué es documentación.

**Rollback:** no aplica — archivos nuevos, sin efecto sobre comportamiento existente.

**Criterio de éxito:** alguien sin acceso a Supabase puede entender la estructura de datos y el razonamiento de las reglas de negocio principales leyendo solo estos dos archivos.

---

## 2. Bloque A — Motor de cálculo financiero unificado

**Objetivo:** corregir la discrepancia funcional de `capacidadAhorro()` y unificar el cálculo de "Disponible del mes" en una única implementación reutilizada por Inicio, Historial y Objetivo de ahorro, junto con una política de redondeo/reconciliación de cuotas que garantice que la suma de las cuotas coincide exactamente con el monto original. Como decisión de implementación —no como objetivo en sí— se evita además el recálculo redundante de estos resultados dentro de un mismo ciclo de render mediante un caché efímero.

**Problema que resuelve:** tres implementaciones independientes de la fórmula de "libre", una de ellas (`capacidadAhorro()`) estructuralmente incompleta — omite el término de ahorro aportado. Incluye uno de los hallazgos funcionales de mayor impacto identificado en la auditoría: la diferencia quedó cuantificada con datos reales ($1.100.000, movimiento del 2026-07-07) y su efecto se producirá cuando ese período pase a formar parte de los meses cerrados utilizados por el algoritmo, según el comportamiento verificado del código. Además, redondeo de cuotas sin reconciliación: la suma de cuotas generadas puede ser hasta $0,99 menor que el monto original (caso real GYM). `buildCuotas()` y `getAnomalias()` se ejecutan 2 veces por ciclo de render sin compartir resultado; `getFijosParaMes()` se ejecuta 21 veces por `renderAll()`.

**Alcance:**
- Función pura que centraliza, para un mes dado: ingresos, uso de ahorro, movimiento interno, gasto fijo, gasto variable, ahorro aportado y "libre" — sin leer variables globales directamente.
- Migración de `renderInicio()`, `renderHistorial()` y `capacidadAhorro()` a esa única función.
- Nueva política de redondeo en `buildCuotas()`: precisión a centavos en cuotas intermedias, última cuota reconciliada para que la suma sea exactamente igual al monto original.
- Caché efímero, vigente solo durante un ciclo de `renderAll()`, para el resumen financiero por mes y para `buildCuotas()` (resuelve la duplicación entre `renderTarjeta()` y `renderTCHorizonte()`).
- Alcance acotado a: libre, cuotas. `getAnomalias()` queda deliberadamente fuera.

**Fuera de alcance:** `getAnomalias()` y otros derivados (score, insights); la asimetría de medio de pago (Causa B, bloques separados); `eliminarObjetivo()`; cambios de arquitectura adicionales (store, multi-archivo, tests); cambios visuales — Inicio y Historial deben verse idénticos a hoy.

**Riesgos:** al centralizar, un error en la función compartida afecta simultáneamente a Inicio, Historial y Objetivo de ahorro. El número de "Objetivo de ahorro" va a cambiar respecto al actual (esperado). La última cuota de una compra podrá diferir en centavos de las demás. Sin tests automatizados, la validación depende de comparación manual. El caché efímero es un mecanismo nuevo — mitigado manteniéndolo como variable local del ciclo de render.

**Dependencias:** ninguna sobre Supabase ni el modelo de datos. Depende de que `esFijo()`/`getFijosParaMes()` no cambien simultáneamente por la Causa B.

**Validaciones:**
- Para cada mes de `mesList`, "libre" de la función nueva coincide con lo que hoy muestran Inicio/Historial.
- Para un mismo mes, Inicio, Historial y Objetivo de ahorro obtienen exactamente el mismo valor de "Disponible" desde la función compartida.
- `capacidadAhorro()` verificado contra cálculo manual de referencia.
- Caso GYM: la suma de las 3 cuotas es exactamente $47.979,99.
- Tarjeta muestra las cuotas activas correctamente.
- Inicio/Historial no cambian respecto al estado previo.

**Rollback:** commit único versionado en git, sin migración de datos — revertir y redesplegar.

**Criterio de éxito:**
- Una sola función calcula "Disponible del mes", sin reimplementación paralela.
- `capacidadAhorro()` incluye el ahorro aportado y coincide con la fórmula canónica.
- La suma de cuotas coincide exactamente con el monto original.
- Inicio y Historial muestran los mismos valores que antes.
- *(Derivado, no objetivo en sí)* `buildCuotas()` no se recalcula más de una vez por ciclo de `renderAll()`.

---

## 3. Bloque B.2 — Fijo pagado en cuotas: compromiso único, sin duplicación

**Objetivo:** garantizar que un gasto fijo pagado excepcionalmente en cuotas de tarjeta se reconozca como compromiso una única vez —en el mes de la compra—, sin volver a impactar Disponible en los meses siguientes.

**Problema que resuelve:** caso real GYM — el compromiso se cuenta correctamente en el mes de la compra, y se vuelve a contar en el mes siguiente sin pago real vía el mecanismo de "referencia".

**Alcance:** una compra en cuotas utilizada para reconocer el compromiso inicial de un fijo no puede volver a ser utilizada por el mecanismo de referencia en meses posteriores. (El resultado de que hoy eso derive en el estado "sin datos" cuando no hay otra referencia disponible es una consecuencia de la implementación actual, no un requisito del bloque.) La pestaña Tarjeta no cambia de comportamiento.

**Fuera de alcance:** el mecanismo de referencia para fijos recurrentes normales; interfaz manual para marcar exclusiones; rediseño de Tarjeta/`buildCuotas()`; fijo pagado parcialmente en cuotas y parcialmente en efectivo el mismo mes (sin evidencia de que ocurra).

**Riesgos:** si un mismo fijo tuviera varias compras en cuotas superpuestas, podría quedar sin referencia por varios meses — aceptable dado que es un caso ya reconocido como excepcional. Cambio acotado a `getFijosParaMes()`, riesgo de implementación bajo.

**Dependencias:** ninguna con B.1. Depende de que `es_cuota` siga siendo confiable (ya verificado: 0 inconsistencias en datos reales).

**Validaciones:** reproducir el caso GYM y confirmar que diciembre 2025 ya no muestra referencia basada en esa compra; un fijo recurrente normal sigue usando la referencia igual que hoy; el monto de la compra en cuotas se contó una única vez.

**Rollback:** commit único, sin migración de datos.

**Criterio de éxito:** un fijo pagado en cuotas impacta Disponible una única vez; el mecanismo de referencia normal no cambia; se cumple el invariante de no duplicación también en este caso excepcional.

---

## 4. Bloque C — Supabase como única fuente de verdad (Objetivo/Configuración)

**Objetivo:** que una escritura de Objetivo o Configuración solo se considere exitosa cuando el servidor confirma, con error visible si falla, y que `localStorage` funcione exclusivamente como caché posterior a esa confirmación.

**Problema que resuelve:** `guardarConfigRemota()` se dispara sin esperar confirmación y cualquier error se atrapa con `console.warn`, invisible para el usuario — riesgo de que un cambio se revierta silenciosamente en la siguiente sincronización.

**Alcance:** `guardarObjetivo()`, `eliminarObjetivo()` y `guardarCierre()`/`setCierreDia()` esperan confirmación remota antes de darse por exitosas; si falla, error visible y `localStorage` no se actualiza; `localStorage` pasa a escribirse solo tras confirmación remota (caché derivada, no fuente inmediata). `cargarConfig()` no cambia.

**Fuera de alcance:** control de versión entre dispositivos; eliminar `localStorage`; `resumen_open`; `movimientos`/`subcategorias_config` (ya con manejo de error visible).

**Riesgos:** sin conexión, Objetivo/Configuración pasan a requerir confirmación remota (igual que Movimiento ya requiere hoy). El cambio se refleja con la latencia real de red.

**Dependencias:** ninguna.

**Validaciones:** guardar sin conexión → error visible, sin cambios en `localStorage`; tras escritura exitosa, `localStorage` coincide con `config_usuario`; mismo comportamiento en `eliminarObjetivo()` y `guardarCierre()`; el valor no se revierte tras un `syncData()` completo. Se considera confirmada una escritura únicamente cuando la operación remota devuelve éxito (`await` resuelto sin error) — solo a partir de ese momento la UI y `localStorage` reflejan el nuevo valor.

**Rollback:** commit único, sin migración de datos.

**Criterio de éxito:** ninguna escritura se da por exitosa sin confirmación del servidor; el usuario nunca queda creyendo que un cambio se guardó cuando en realidad no ocurrió.

---

## 5. Bloque B.1 — Débito como salida real de dinero

**Objetivo:** que todo cálculo cuyo significado de negocio sea "dinero que efectivamente salió de la cuenta" trate de forma idéntica los medios de pago que representan esa salida real, sin asimetría entre Cuenta/Billetera y Débito.

**Problema que resuelve:** un gasto variable pagado con Débito no participa hoy en ningún cálculo de ese tipo, pese a ser equivalente a un pago por Cuenta/Billetera según la definición de negocio confirmada.

**Alcance:** el criterio queda expresado por semántica de negocio ("todo medio de pago que represente una salida efectiva de dinero ese mes"), no por nombres de medios — su traducción técnica (enumerar medios vs. excluir Crédito) es una decisión de implementación. Ubicaciones: cálculo de gasto variable/libre, saldo actual, desglose de la pestaña Gastos, determinación de meses con actividad.

**Fuera de alcance:** B.2; tratamiento de Crédito (ya confirmado correcto); arquitectura del Bloque A (este bloque no depende de que exista una función compartida); decisión de implementación standalone vs. absorbido en Bloque A (se resuelve al implementar).

**Riesgos:** los valores de Disponible, saldo y Gastos pueden cambiar para meses con gastos a Débito (esperado). Riesgo de omitir alguna de las ~8-9 ubicaciones si se implementa de forma dispersa. La cuenta activa de Diego no tiene hoy movimientos a Débito reales — la validación end-to-end requiere un movimiento de prueba.

**Dependencias:** ninguna funcional con Bloque A ni B.2. Depende de que el conjunto de medios de pago siga limitado a los 3 valores actuales.

**Validaciones:** para cada ubicación, un gasto a Débito produce el mismo efecto que uno a Cuenta/Billetera; Crédito sigue sin afectar; validación transversal — todos los cálculos que representan salida real de dinero coinciden entre sí sobre el mismo conjunto de movimientos, no solo contra su propio valor esperado de forma aislada; un mes cuya única actividad sea un gasto a Débito aparece correctamente en el selector y el Historial.

**Rollback:** commit único, sin migración de datos.

**Criterio de éxito:** Débito y Cuenta/Billetera se tratan de forma idéntica en todas las ubicaciones; Crédito no se ve afectado; ninguna ubicación queda sin corregir.

---

## 6. Bloque D — Guard de concurrencia + resincronización

**Objetivo:** evitar que sincronización completa y borrado de movimiento se ejecuten en paralelo consigo mismos, y mantener los datos actualizados al volver de segundo plano — sin que esa resincronización interrumpa una interacción activa del usuario (por ejemplo, un modal de edición abierto) ni se dispare mientras ya hay una operación en curso. El mecanismo concreto para garantizarlo queda para la implementación.

**Problema que resuelve:** sin flag de "operación en curso", dos `syncData()` en rápida sucesión corren en paralelo; `eliminarMovimiento()` no deshabilita ningún control durante la espera; sin `visibilitychange`, la app no se refresca sola al volver de segundo plano (relevante en uso por iPhone).

**Alcance:** guard de "operación en curso" para `syncData()` y `eliminarMovimiento()` (patrón ya usado en `guardarMovimiento()`); listener de `visibilitychange` que dispare `syncData()` al volver a primer plano, respetando la restricción del objetivo.

**Fuera de alcance:** comunicación entre pestañas/dispositivos; Service Worker/manifest/caché offline; listeners de `online`/`offline`/`pagehide`/`beforeunload`; cola de reintentos.

**Riesgos:** tráfico de red adicional al volver a primer plano (impacto bajo hoy); necesidad de decidir en implementación cómo detectar "interacción activa".

**Dependencias:** ninguna.

**Validaciones:** dos sincronizaciones en rápida sucesión → la segunda no inicia una nueva paginación en paralelo; el botón de eliminar se deshabilita durante la espera; simular segundo plano → primer plano dispara resincronización; ningún otro comportamiento cambia.

**Rollback:** commit único, sin migración de datos.

**Criterio de éxito:** ninguna sincronización ni borrado se dispara dos veces en paralelo sobre sí mismo; la app se resincroniza automáticamente al volver de segundo plano; la resincronización automática nunca interrumpe una interacción activa del usuario (por ejemplo, un modal de edición abierto); ningún otro comportamiento existente cambia.

---

## 7. Bloque Hardening (Aplicación)

**Objetivo:** eliminar el comportamiento impredecible de la versión flotante de `supabase-js`.

**Problema que resuelve:** verificado que `@2` resuelve a una versión distinta según el momento (hoy: `2.111.0`), sin control de versión propio del proyecto.

**Alcance:** fijar `supabase-js` a una versión exacta. Sin SRI (descartado por costo de mantenimiento sin evidencia que lo justifique).

**Fuera de alcance:** SRI, CSP, grants y cabeceras HTTP (bloque de Infraestructura).

**Riesgos:** deja de recibir actualizaciones automáticas del SDK.

**Dependencias:** ninguna.

**Validaciones:** la app funciona correctamente (login, sincronización, CRUD) con la versión fija.

**Rollback:** revertir el commit.

**Criterio de éxito:** versión fija, sin flotar; ninguna funcionalidad afectada.

---

## 8. Bloque Hardening (Infraestructura)

**Objetivo:** reducir privilegios innecesarios del rol `anon` en Supabase y agregar cabeceras HTTP de bajo riesgo.

**Problema que resuelve:** el rol `anon` tiene grants amplios (incluido `TRUNCATE`, no cubierto por RLS) más allá de lo necesario; ausencia de `X-Frame-Options`/`X-Content-Type-Options` en el deploy.

**Alcance:** revocar en Supabase los privilegios de `anon` no necesarios para el funcionamiento normal vía PostgREST (sin tocar RLS); agregar las dos cabeceras vía configuración de Netlify.

**Fuera de alcance:** CSP, MFA, políticas RLS existentes.

**Riesgos:** revocar grants podría romper alguna operación legítima no identificada — validar con cuidado. Cabeceras de bajo riesgo.

**Dependencias:** ninguna.

**Validaciones:** operaciones normales de la app siguen funcionando tras revocar grants; `curl` confirma las cabeceras en la respuesta del deploy.

**Rollback:** volver a otorgar los grants revocados (documentados antes de aplicar el cambio); revertir el archivo de configuración de Netlify.

**Criterio de éxito:** `anon` con privilegios mínimos necesarios; cabeceras presentes; ninguna funcionalidad afectada.

---

## 9. Bloque Accesibilidad — asociación de labels

**Objetivo:** que cada campo de formulario quede correctamente asociado a su etiqueta visible (`for=`/`id=`).

**Problema que resuelve:** confirmado por ejecución real — ningún `<label>` usa `for=`; un lector de pantalla identifica el campo de contraseña por su placeholder, no por su etiqueta.

**Alcance:** vincular cada `<label>` con su `<input>`/`<select>` correspondiente.

**Fuera de alcance:** ARIA adicional, roles semánticos de las pestañas de auth, `user-scalable=no` — deuda técnica documentada, sin evidencia de problema actual.

**Riesgos:** mínimos — cambio aditivo; cuidado con colisiones de `id`.

**Dependencias:** ninguna.

**Validaciones:** repetir la lectura del árbol de accesibilidad real del navegador y confirmar identificación por etiqueta.

**Rollback:** revertir el commit.

**Criterio de éxito:** todos los campos correctamente asociados a su label, verificado por el árbol de accesibilidad real.
