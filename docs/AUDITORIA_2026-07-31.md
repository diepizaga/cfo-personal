# CFO Personal — Diagnóstico consolidado de auditoría

**Fecha:** 2026-07-31
**Alcance:** repositorio `diepizaga/cfo-personal` (`main`, HEAD `7fe0253`) + proyecto Supabase `fzumqhrpiexgkutybcrc` + deploy Netlify (`tourmaline-torrone-175f2e.netlify.app`)
**Método:** auditoría por invariantes, evidencia obligatoria (código, datos reales, historial de git, ejecución real contra el deploy publicado), clasificación hecho comprobado / inferencia / hipótesis / pendiente de validar en cada hallazgo original.
**Este documento:** cierra formalmente la Fase 2.2 (Auditoría). No propone soluciones, no prioriza, no diseña. Es el diagnóstico único del que debe partir la Fase 3 (Diseño). Los hallazgos se agrupan por causa raíz, no por las 9 áreas originales — para el detalle de evidencia de cada uno (líneas de código, valores reales, consultas SQL ejecutadas), remitirse al historial de la auditoría por áreas.

---

## 1. Fortalezas comprobadas del sistema

Antes de listar problemas, esto es lo que la auditoría confirmó que funciona correctamente, con evidencia real:

- **Seguridad de acceso a datos:** RLS habilitado en las 4 tablas (`movimientos`, `subcategorias_config`, `config_usuario`, `fijos_config`), con políticas que restringen cada operación a `auth.uid()=user_id`; foreign keys con `ON DELETE CASCADE`; **0 filas huérfanas** verificadas en las 4 tablas.
- **Integridad de los datos reales:** sobre los 630 movimientos de la cuenta activa — 0 montos negativos, 0 fechas nulas, 0 inconsistencias lógicas entre columnas (cuota/medio/tarjeta), 0 desvíos entre `movimientos` y `subcategorias_config`.
- **Ausencia de materialización de derivados:** ningún cálculo (libre, score, saldo, anomalías) se persiste nunca — la UI nunca es fuente de verdad, sin dependencias circulares en el motor de cálculo.
- **Manejo explícito de errores en la ruta principal de datos:** `guardarMovimiento()`/`eliminarMovimiento()` muestran error visible, no cierran el modal, no pierden lo tipeado.
- **Fechas sin ambigüedad de zona horaria:** único punto de conversión string→Date, con patrón que evita el desplazamiento de día típico de UTC.
- **Deploy sincronizado con el repositorio:** verificado por hash real — sin deriva entre lo publicado y lo auditado.
- **Infraestructura de red:** HSTS activo, compresión Brotli real (~77% de reducción de transferencia).
- **Escaping consistente:** ~13 de ~15 puntos de interpolación de datos de usuario en el DOM usan `esc()` correctamente.
- **Sistema de diseño:** responsive y modo oscuro verificados por captura real contra el deploy publicado (superficie de login), sin desbordamientos ni contraste roto.
- **Consistencia de mensajería:** canales de error y estados vacíos uniformes en toda la aplicación.
- **Índices de base de datos** alineados al patrón de consulta real.

---

## 2. Hallazgos agrupados por causa raíz

### Causa raíz A — Ausencia de una capa de cálculo compartida, pura y aislable del DOM

**Qué es:** ningún cálculo financiero (salvo `calcScore()`) se implementa una sola vez y se reutiliza — cada función de render recalcula desde `movimientos` lo que necesita, mezclado con la escritura directa en el DOM y la lectura de variables globales.

**Manifestaciones concretas:**
- Tres implementaciones independientes de la fórmula de "libre" (Inicio, Historial, capacidad de ahorro). Incluye uno de los hallazgos funcionales de mayor impacto identificado en la auditoría: la discrepancia entre `capacidadAhorro()` y las demás implementaciones del cálculo de disponible — la primera omite el término de ahorro aportado, presente en las otras dos. La diferencia quedó cuantificada con datos reales ($1.100.000, movimiento del 2026-07-07) y su efecto se producirá cuando ese período pase a formar parte de los meses cerrados utilizados por el algoritmo, según el comportamiento verificado del código.
- `buildCuotas()` y `getAnomalias()` se ejecutan 2 veces cada una dentro de un mismo ciclo de render, sin compartir resultado.
- `getFijosParaMes()` se ejecuta 21 veces por cada `renderAll()` (1 en Inicio + 20 en Historial, una por mes), sin memoización.
- Imposibilidad de testear cualquier cálculo de forma aislada — es la causa directa, ya razonada en la auditoría, de que no exista ningún test en el repositorio.

**Clasificación:** el defecto funcional confirmado es la fórmula incompleta de `capacidadAhorro()` (con impacto financiero real y cuantificado). El resto (duplicación, recálculo redundante, acoplamiento cálculo-DOM) es **deuda técnica**: no produce hoy un resultado incorrecto, pero es la causa estructural que permitió que la fórmula incompleta pasara desapercibida, y es lo que hace crecer el costo de cálculo de forma superlineal con el uso.

**Impacto actual:** la discrepancia está confirmada y cuantificada (capacidadAhorro); su efecto sobre el valor mostrado en la tarjeta de Objetivo de ahorro se produce cuando julio 2026 pase a integrar los meses cerrados que usa el algoritmo, según el comportamiento verificado del código — no observado todavía en pantalla dentro de esta auditoría.
**Riesgo de escalabilidad:** la cantidad de trabajo de cálculo crece aproximadamente con el producto entre meses históricos y movimientos registrados — sin evidencia de que esto sea perceptible hoy (630 movimientos, 20 meses), pero es una tendencia de crecimiento verificada, no acotada por ningún mecanismo de caché.

---

### Causa raíz B — Asimetría del criterio "medio de pago" entre gasto fijo y gasto variable

**Qué es:** la función que determina el gasto fijo del mes (`getFijosParaMes`) no filtra por `medio_pago`; la función que determina el gasto variable (`gastosLibres`, repetida idénticamente en ~8 lugares) sí lo restringe a `"Cuenta / Billetera"`. Ningún cálculo concilia lo que pasa con Débito o Crédito del lado variable.

**Manifestaciones concretas:**
- Un gasto a Débito no-fijo no impacta en ningún cálculo agregado (verificado con un movimiento real de $6.800, cuenta de prueba).
- Un gasto a Crédito no-fijo vive únicamente en la pestaña Tarjeta, invisible en "Disponible del mes", en el desglose de Gastos y en el Historial.
- Un gasto fijo pagado a Crédito en cuotas (caso real: GYM, $47.979,99, 3 cuotas) se contabiliza **completo** en el mes de la compra según `getFijosParaMes()`, y **prorrateado en meses distintos** según `buildCuotas()` — reconstruí la línea de tiempo completa mes a mes con datos reales: en diciembre 2025, "Disponible del mes" restaba $47.979,99 mientras la pestaña Tarjeta mostraba $15.993 por el mismo compromiso.

**Clasificación:** **defecto funcional confirmado** — no es una decisión de diseño documentada en ningún lado (no hay comentario ni commit que la explique), es una consecuencia de dos algoritmos de fechado independientes sobre el mismo dato que nunca se reconcilian.

**Impacto actual:** confirmado con datos reales de la cuenta activa de Diego (caso GYM, nov-2025 a feb-2026 ya ocurrido).
**Riesgo de escalabilidad:** crece con cada compra fija pagada a crédito en cuotas — no depende del volumen total de datos, depende de la frecuencia de este patrón de uso específico.

---

### Causa raíz C — Escrituras remotas sin confirmación visible de éxito, para datos con doble representación

**Qué es:** `Objetivo` y `Configuración (cierre de tarjeta)` se guardan simultáneamente en `localStorage` (por dispositivo) y en `config_usuario` (remoto), sin relación de derivación formal entre ambas copias. La escritura remota (`guardarConfigRemota()`) se dispara sin `await` y cualquier error se atrapa con `console.warn`, nunca visible para el usuario.

**Manifestaciones concretas:**
- Si la escritura remota falla, el dato queda correcto solo en el dispositivo local. En el siguiente `syncData()` completo, si existe una fila remota, esta sobrescribe el valor local sin comparar cuál es más reciente — el cambio del usuario puede revertirse silenciosamente.
- `config_usuario` tiene dos rutas de escritura (migración inicial + `upsert` posterior) sin ningún control de versión (`updated_at` se escribe pero nunca se compara) — sin mecanismo de resolución si dos dispositivos escriben casi al mismo tiempo.
- `resumen_open` (preferencia de UI) es la única entidad sin ninguna copia remota — diverge por diseño entre dispositivos.

**Clasificación:** deuda técnica / limitación de diseño — no hay evidencia de que un fallo de escritura remota haya ocurrido en la práctica; es un mecanismo sin salvaguarda, no un defecto ya manifestado.

**Impacto actual:** no confirmado — pendiente de validar si esto ya afectó a Diego.
**Riesgo de escalabilidad:** aumenta con el número de dispositivos en uso simultáneo, no con el volumen de datos.

---

### Causa raíz D — Ausencia de mecanismos de coordinación ante concurrencia e interrupciones, a nivel de aplicación

**Qué es:** ningún flujo asíncrono de la aplicación (sincronización, guardado, borrado) tiene protección contra ejecutarse en paralelo consigo mismo, y ningún código reacciona a que el dispositivo pierda conectividad, pase a segundo plano, o a que la pestaña se cierre a mitad de una operación.

**Manifestaciones concretas:**
- Sin flag de "sincronización en curso" — dos `syncData()` disparados en rápida sucesión corren en paralelo, sin coordinación; gana el que termine último.
- `eliminarMovimiento()` no deshabilita ningún control mientras espera la respuesta del servidor, a diferencia de `guardarMovimiento()`.
- Sin comunicación entre pestañas/dispositivos abiertos simultáneamente con la misma sesión — cada uno mantiene su propio estado en memoria.
- Sin listeners de `visibilitychange`, `pagehide`, `beforeunload`, `online`/`offline` en todo el archivo — confirmado por búsqueda explícita.
- Sin Service Worker ni `manifest.json` — sin ningún dato cacheado disponible si el dispositivo pierde conectividad, pese a los meta tags de PWA para iOS ya presentes.

**Clasificación:** deuda técnica — ninguna de estas ausencias fue observada produciendo un problema real durante la auditoría; son huecos de robustez ante escenarios no cubiertos, no defectos ya manifestados.

**Impacto actual:** no confirmado.
**Riesgo:** relevante dado el uso principal por iPhone (según mi memoria del proyecto), donde el sistema operativo suspende pestañas/PWAs en segundo plano con mayor frecuencia que en desktop — condición que amplifica la probabilidad de interrupciones a mitad de una operación, sin que esto constituya evidencia de que ya haya ocurrido.

---

### Causa raíz E — Validación de datos exclusivamente en el cliente

**Qué es:** ninguna de las 4 tablas tiene `CHECK` constraints; no hay `UNIQUE` sobre `subcategorias_config(user_id, categoria, subcategoria)`; el parseo de montos ingresados por el usuario (`parseMonto()`) tiene una ambigüedad demostrada por ejecución real, y los valores inválidos de `monto` se silencian (`parseFloat(...)||0`) sin registro.

**Manifestaciones concretas:**
- La misma sintaxis de un punto sin coma (`"1.23"` vs `"1.234"`) se interpreta como magnitudes que difieren en 3 órdenes de magnitud, según la cantidad de dígitos que le siguen — verificado ejecutando la función real en Node.js.
- 3 filas duplicadas de una misma subcategoría, confirmadas en la cuenta de prueba — no reproducidas en la cuenta activa de Diego, pero sin ningún constraint que lo impida en general.
- Redondeo de cuotas sin reconciliación: la suma de las cuotas generadas puede ser hasta $0,99 menor que el monto original de la compra (verificado con el caso GYM real) — el sistema no compensa esta pérdida en ninguna cuota.

**Clasificación:** deuda técnica en el nivel de base de datos (ausencia de constraints); **defecto funcional de baja magnitud** en el redondeo de cuotas (produce un resultado numéricamente incorrecto, aunque trivial); ambigüedad de `parseMonto()` sin evidencia de haberse materializado en un error real.

**Impacto actual:** confirmado solo para el redondeo de cuotas (magnitud mínima). Sin evidencia de impacto real para la ambigüedad de parseo ni para duplicados en la cuenta activa.
**Riesgo:** crece con el volumen de compras en cuotas (redondeo) y con el volumen de entradas manuales de montos (ambigüedad de parseo) — no con el tamaño total de la base.

---

### Causa raíz F — Ausencia de documentación versionada del sistema

**Qué es:** el repositorio contiene únicamente `index.html` y `apple-touch-icon.png`. No existe ningún archivo que documente el esquema de datos (columnas, constraints, RLS, políticas), ninguna decisión de arquitectura, ni la razón detrás de ningún umbral de negocio (score de salud, detección de anomalías, límite de tarjeta).

**Manifestaciones concretas:**
- Todo el modelo de datos auditado en esta fase se reconstruyó exclusivamente por consulta directa a Supabase — no existe ningún artefacto en el repositorio que lo describa o permita reconstruirlo sin acceso a la base en vivo.
- La tabla `fijos_config` (huérfana, sin ningún consumidor en código, triggers, funciones ni vistas) solo es explicable leyendo el mensaje de un commit específico (`e2551e0`) — sin ese commit, no habría ninguna forma de saber por qué existe.
- Ningún comentario justifica los umbrales del score de salud (40%/85%, 0%/30%, etc.) ni de la detección de anomalías (1.5x, $5.000, 6 meses).
- Los mensajes de commit no siguen una convención uniforme (37% con prefijo tipo Conventional Commits, 63% en lenguaje libre) — sin que esto implique necesariamente menor calidad informativa.

**Excepción positiva:** el código sí tiene 34 marcadores de sección internos, consistentes, que actúan como mapa de navegación del archivo — la ausencia es de documentación *externa*, no de toda forma de documentación.

**Clasificación:** deuda técnica pura — no es un defecto funcional (la aplicación funciona sin esta documentación), afecta la capacidad de mantener, transferir o auditar el sistema sin acceso directo a Supabase y sin quien lo escribió presente.

**Impacto actual:** ya se manifestó como costo dentro de esta misma auditoría (cada hecho del modelo de datos tuvo que reconstruirse por consulta en vivo).
**Riesgo:** crece si el proyecto cambia de responsable, si se pierde acceso al proyecto de Supabase, o si pasa tiempo suficiente para que el razonamiento detrás de los umbrales de negocio se olvide.

---

### Causa raíz G — Superficie de hardening incompleta, en capas separadas (aplicación / plataforma / infraestructura)

**Qué es:** varios puntos de seguridad no son vulnerabilidades comprobadas sino ausencia de una capa adicional de defensa, y su origen está repartido entre código propio, configuración de Supabase, y configuración de Netlify.

**Manifestaciones, con origen explícito:**
- **Aplicación:** 2 de ~15 puntos de interpolación de datos de usuario en el DOM no usan `esc()` (`renderIngresos`, `renderTarjeta`) — sin evidencia de explotabilidad a través del flujo normal de la UI, ya que los campos involucrados provienen de `<select>` cerrados en el uso estándar. Dependencia de `supabase-js` sin versión fija (`@2`, resuelto hoy a `2.111.0` por ejecución real) y sin Subresource Integrity en ningún recurso externo.
- **Plataforma (Supabase):** el rol `anon` tiene los mismos grants amplios que `authenticated` sobre las 4 tablas (patrón por defecto de la plataforma, no configurado especialmente por este proyecto), incluyendo `TRUNCATE` — que no está cubierto por RLS, aunque sin evidencia de un canal de acceso público que lo exponga. Sin MFA habilitado para ninguna cuenta.
- **Infraestructura (Netlify/HTTP):** ausencia de cabeceras `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options` — verificado por `curl` real contra el deploy.

**Clasificación:** en su totalidad, **oportunidades de hardening**, no vulnerabilidades comprobadas. Ningún hallazgo de este grupo tiene una cadena de ataque completa demostrada (actor + prerrequisito + capacidad + impacto) sin depender de un compromiso de terceros no evidenciado (CDN/npm) o de una condición hoy no explotable por los canales públicos existentes (TRUNCATE vía PostgREST).

**Impacto actual:** ninguno confirmado.
**Riesgo:** condicional en todos los casos a un evento externo (compromiso de un CDN/paquete, deshabilitación accidental de RLS) que no fue observado ni tiene evidencia de probabilidad elevada.

---

### Causa raíz H — Ausencia de accesibilidad para tecnologías asistivas

**Qué es:** distinto de la experiencia de usuario general (que se verificó funcionando correctamente para uso visual/táctil estándar) — la cobertura de atributos de accesibilidad es baja, confirmada tanto por código como por el árbol de accesibilidad real del navegador.

**Manifestaciones concretas:**
- Solo 5 de 56+ elementos interactivos (botones, inputs, selects) tienen algún atributo `aria-*`/`role`.
- Ninguno de los 22 `<label>` del archivo usa `for=` — confirmado también por observación real: el campo de contraseña se identifica ante un lector de pantalla por su `placeholder`, no por su etiqueta visible.
- Las pestañas de "Ingresar"/"Registrarme" se exponen como elementos genéricos, no como controles semánticamente interactivos.
- `user-scalable=no` deshabilita el zoom por gesto.

**Clasificación:** deuda técnica de accesibilidad — no es un defecto que impida el uso de la app en su patrón de uso actual (sin reporte de uso de tecnologías asistivas), es una dimensión del producto no desarrollada.

**Impacto actual:** confirmado como ausencia (por ejecución real), sin evidencia de que afecte a un usuario real hoy.
**Riesgo:** no depende del crecimiento de datos, depende de si el perfil de usuarios de la app llega a incluir a alguien que dependa de tecnologías asistivas.

---

### Hallazgos puntuales, sin causa raíz compartida con otros

- **`eliminarObjetivo()` no pide confirmación**, a diferencia de `eliminarMovimiento()`/`eliminarSubcat()` — inconsistencia de patrón UX, con relación a integridad de datos (se pierde el objetivo en un solo toque). **Defecto funcional menor** — impacto actual confirmable en cualquier uso de esa función, sin necesidad de un escenario especial.
- **Anon key de Supabase duplicada** en dos lugares del archivo — deuda de mantenibilidad menor, sin implicancia de seguridad (la clave está diseñada para ser pública).
- **Orden de presentación no garantizado** para movimientos con la misma fecha (sin segunda clave de ordenamiento en la consulta) — afecta el orden visual en listas, no los montos agregados. Confirmado que el 91% de los movimientos reales de Diego comparten fecha con al menos otro. Deuda técnica de bajo impacto.
- **Secuencialidad de `cargarFijos()`/`cargarConfig()`** en cada sincronización — oportunidad de optimización puntual, sin relación con el volumen de datos.

---

## 3. Tabla de clasificación cruzada

| Causa raíz | Naturaleza dominante | Impacto actual | Riesgo de escalabilidad |
|---|---|---|---|
| A — Sin capa de cálculo compartida/pura | Deuda técnica + 1 defecto confirmado (`capacidadAhorro`) | **Sí**, cuantificado ($1.100.000) | Alto — crecimiento superlineal del trabajo |
| B — Asimetría de medio de pago (fijo vs. variable) | Defecto funcional confirmado | **Sí**, caso real (GYM) ya ocurrido | Medio — depende de frecuencia de uso de crédito en fijos |
| C — Escrituras remotas sin confirmación, doble representación | Deuda técnica / limitación de diseño | No confirmado | Bajo-medio — depende de nº de dispositivos |
| D — Sin coordinación de concurrencia/interrupciones | Deuda técnica | No confirmado | Medio — amplificado por uso móvil (iPhone) |
| E — Validación solo en cliente | Deuda técnica + 1 defecto menor (redondeo) | Confirmado solo en redondeo (magnitud mínima) | Bajo-medio |
| F — Sin documentación versionada | Deuda técnica | Sí (costo ya absorbido en esta auditoría) | Alto si cambia el responsable del proyecto |
| G — Hardening incompleto por capas | Oportunidad de hardening (no vulnerabilidad) | No confirmado | Condicional a eventos externos no evidenciados |
| H — Accesibilidad | Deuda técnica | Confirmado como ausencia, sin usuario afectado evidenciado | No depende de crecimiento de datos |
| `eliminarObjetivo()` sin confirmación | Defecto funcional menor | Sí, en cualquier uso | Constante, no crece |

---

## 4. Base para la Fase 3 — Diseño

Este diagnóstico deja identificados, sin proponer solución ni orden de abordaje:

1. **Un defecto funcional con impacto financiero cuantificado** (capacidad de ahorro, causa raíz A) — su efecto sobre el valor mostrado se produce cuando julio 2026 pase a integrar los meses cerrados que usa el algoritmo, según el comportamiento verificado del código.
2. **Un defecto funcional ya materializado con datos reales históricos** (doble representación de fijos en cuotas, causa raíz B).
3. **Un defecto menor de patrón de confirmación** (`eliminarObjetivo`), de corrección acotada.
4. **Seis causas raíz de deuda técnica** (A residual, C, D, E residual, F, H) que no producen hoy un resultado incorrecto pero condicionan la evolución, mantenibilidad y escalabilidad del sistema.
5. **Un conjunto de oportunidades de hardening** (causa raíz G) sin vulnerabilidad comprobada, repartidas entre código propio y decisiones de plataforma/infraestructura fuera del control directo del proyecto.
6. **Fortalezas confirmadas** (sección 1) que no requieren intervención y deben preservarse como restricciones de diseño en cualquier cambio futuro (en particular: RLS, integridad referencial, ausencia de materialización de derivados, manejo de error en la ruta de Movimiento).

Ningún punto de este documento fue priorizado entre sí — esa es la tarea de la Fase 3.
