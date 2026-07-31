# CFO Personal — Decisiones de negocio y arquitectura

**Última actualización:** 2026-07-31.

Este documento registra decisiones de negocio y arquitectura que **no pueden reconstruirse leyendo únicamente el código o la base de datos** — quedaron establecidas en conversación con Diego durante la Fase 2.2 (Auditoría) y la Fase 3 (Diseño) del proyecto. Para cada una se indica **la regla**, **el estado de implementación actual** (a la fecha de este documento) y **el bloque de trabajo** que la implementa o corrige, cuando aplica. Cuando el repositorio no permite reconstruir el motivo histórico de una decisión, se indica explícitamente — no se infieren justificaciones que no están documentadas.

---

## 1. Fórmula única de "Disponible del mes"

**Regla:** `Disponible = ingresos + uso de ahorro + movimientos internos − gastos fijos − gastos variables que impactan ese mes − ahorro aportado`. Es el dato financiero más usado de la aplicación — toda funcionalidad que represente dinero disponible debe usar exactamente esta definición, salvo regla de negocio explícita en contrario.

**Estado de implementación (2026-07-31):** implementada correctamente en `renderInicio()` y `renderHistorial()`. Implementada de forma **incompleta** en `capacidadAhorro()` — omite el término de ahorro aportado. La discrepancia quedó cuantificada con datos reales de la cuenta activa ($1.100.000, movimiento del 2026-07-07); su efecto sobre el valor mostrado se produce cuando julio 2026 pase a integrar los meses cerrados que usa el algoritmo.

**Corrige:** Bloque A (no implementado a la fecha de este documento).

---

## 2. Invariante de cuotas: redondeo y reconciliación

**Regla:** la suma de las cuotas generadas por una compra en cuotas debe ser exactamente igual al monto original del movimiento — con precisión a centavos en las cuotas intermedias y reconciliación del residuo en la última cuota.

**Estado de implementación (2026-07-31):** no implementada. `buildCuotas()` calcula cada cuota de forma independiente por redondeo a peso entero (`Math.round(monto/cuotas)`), sin ningún ajuste posterior. Verificado con datos reales: una compra de $47.979,99 en 3 cuotas produce 3 cuotas de $15.993, cuya suma ($47.979) es $0,99 menor al monto original.

**Corrige:** Bloque A (no implementado a la fecha de este documento).

---

## 3 y 4. Tratamiento de medios de pago: criterio de caja para variables, criterio de compromiso para fijos

**Regla:**
- Un **gasto variable** (no fijo) descuenta de Disponible únicamente si el dinero efectivamente salió de la cuenta ese mes — incluye todo medio de pago que represente esa salida real (hoy: Cuenta/Billetera y Débito) y excluye los que no (hoy: Crédito, hasta que se paga el resumen de la tarjeta).
- Un **gasto fijo** descuenta de Disponible en el momento en que se asume el compromiso, independientemente del medio de pago — porque representa una obligación que el usuario sabe que va a pagar sí o sí, se haya efectivizado el pago o no.

Esta distinción responde a cómo Diego piensa el flujo mensual: "cobro el sueldo → descuento los compromisos fijos que sé que voy a pagar sí o sí → lo que queda es la plata realmente disponible para administrar durante el mes."

**Estado de implementación (2026-07-31):**
- Gastos fijos: implementado correctamente — `getFijosParaMes()` no filtra por medio de pago, coherente con el criterio de compromiso.
- Gastos variables por Cuenta/Billetera: correcto.
- Gastos variables por Crédito: correctamente excluidos.
- Gastos variables por **Débito: incorrectamente excluidos** — hoy no descuentan de Disponible pese a representar salida real de dinero. Confirmado que la cuenta activa de Diego no tiene movimientos a Débito reales al momento de esta auditoría; la asimetría es real mecánicamente, sin manifestación en los datos actuales.

**Corrige:** Bloque B.1 (no implementado a la fecha de este documento).

---

## 5. Invariante de no duplicación (caso excepcional: fijo pagado en cuotas de tarjeta)

**Regla:** un mismo gasto nunca puede impactar dos veces el Disponible del mes, bajo ninguna circunstancia. Un gasto fijo pagado excepcionalmente en cuotas de tarjeta sigue siendo, conceptualmente, un fijo — se reconoce como compromiso una única vez, en el mes de la compra (coherente con la regla 3/4), y las cuotas que aparecen después en la pestaña Tarjeta son únicamente informativas de seguimiento de deuda, sin volver a descontar de Disponible.

Este es explícitamente un caso excepcional en el uso real de Diego — los gastos fijos normalmente se pagan por Cuenta/Billetera o Débito, nunca con tarjeta de crédito. La regla no debe generalizar el tratamiento de todos los fijos, solo cubrir esta excepción sin romper el flujo normal.

**Estado de implementación (2026-07-31):** violada. Caso real documentado: un fijo (GYM) pagado en 3 cuotas de tarjeta en noviembre de 2025 se contó correctamente completo ese mes, y se volvió a contar en diciembre de 2025 (sin pago real ese mes) porque el mecanismo de "referencia al mes anterior" trajo el mismo monto de nuevo.

**Corrige:** Bloque B.2 (no implementado a la fecha de este documento).

---

## 6. Umbrales del motor de cálculo

Los siguientes valores están hardcodeados en `index.html`. Se documenta su valor y el comportamiento que implementan actualmente. **No se encontró, durante la auditoría, ningún comentario en el código, documento ni commit que justifique por qué se eligió cada valor específico** — se registra tal cual, sin inferir una justificación que no existe.

### Score de salud financiera (`calcScore`)

Combina 4 factores, cada uno con un tope de 25 puntos (suma máxima 100), por interpolación lineal entre dos umbrales:

| Factor | 25 puntos (mejor caso) | 0 puntos (peor caso) |
|---|---|---|
| % de fijos sobre el ingreso | ≤ 40% | ≥ 85% |
| % de dependencia de ahorro | ≤ 0% | ≥ 30% |
| Diferencia entre % de gasto pagado y % de días transcurridos del mes | ≤ 0 puntos porcentuales | ≥ 25 puntos porcentuales |
| Tendencia de gasto diario vs. promedio de 3 meses previos | ≤ 0% | ≥ 30% |

Sin ingresos registrados, el tercer factor otorga 10 puntos fijos. Sin datos suficientes de tendencia, el cuarto factor otorga 15 puntos fijos.

### Detección de anomalías en gastos fijos (`getAnomalias`)

Se marca un gasto fijo como anómalo cuando: el monto pagado es ≥ 1,5 veces el promedio histórico de esa subcategoría, **y** la diferencia absoluta contra ese promedio es ≥ $5.000. El promedio histórico se calcula sobre hasta 6 meses previos con pago registrado, y requiere un mínimo de 2 meses de historial para activarse.

### Límite de referencia de tarjeta

$800.000 — usado tanto para la alerta de "alto consumo" como para el cálculo de porcentaje de uso mostrado en la pestaña Tarjeta.

**Estado de implementación:** los tres se usan tal cual están documentados arriba, sin cambios previstos por ningún bloque de trabajo de la Fase 3 — no forman parte del roadmap actual.

---

## Cómo mantener este documento

Cada vez que un bloque de trabajo listado como "no implementado a la fecha de este documento" pase a implementarse, actualizar la sección correspondiente: cambiar el estado de implementación y quitar la referencia al bloque pendiente. Si aparece una nueva decisión de negocio no reconstruible desde el código durante una implementación futura, agregarla acá siguiendo el mismo formato (regla → estado actual → bloque que la implementa).
