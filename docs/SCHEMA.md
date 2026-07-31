# CFO Personal — Esquema de datos

**Última verificación:** 2026-07-31, contra el proyecto Supabase `fzumqhrpiexgkutybcrc`.

**Este documento es documentación del estado actual, no la fuente de verdad del esquema.** Si en algún momento hay una diferencia entre lo descripto acá y la base de datos real, **prevalece la base de datos**. El objetivo de este documento es facilitar el entendimiento y el mantenimiento del proyecto sin necesidad de acceso directo a Supabase — no reemplaza las migraciones ni el esquema real.

---

## Tablas

### `movimientos`

Registro de cada transacción financiera del usuario (gasto, ingreso o ahorro).

| Columna | Tipo | Nullable | Default |
|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` |
| `user_id` | uuid | NO | — |
| `fecha` | date | NO | — |
| `tipo_operacion` | text | NO | — |
| `categoria` | text | SÍ | — |
| `subcategoria` | text | SÍ | — |
| `detalle` | text | SÍ | — |
| `medio_pago` | text | SÍ | — |
| `tarjeta` | text | SÍ | — |
| `es_cuota` | boolean | SÍ | `false` |
| `cantidad_cuotas` | integer | SÍ | `1` |
| `ing_tipo` | text | SÍ | — |
| `monto` | numeric | NO | — |
| `created_at` | timestamptz | SÍ | `now()` |

**Constraints:** `movimientos_pkey` PRIMARY KEY (`id`); `movimientos_user_id_fkey` FOREIGN KEY (`user_id`) REFERENCES `auth.users(id)` ON DELETE CASCADE. Sin CHECK constraints — `tipo_operacion`, `medio_pago`, `es_cuota`/`cantidad_cuotas` no tienen ninguna restricción de valores a nivel de base de datos.

**Índices:** `movimientos_pkey` (único, `id`); `idx_mov_user_fecha` (`user_id`, `fecha`).

**Valores reales observados (cuenta activa, 630 filas, auditoría 2026-07-31):** `tipo_operacion` ∈ {Gasto, Ingreso, Ahorro}; `medio_pago` ∈ {Cuenta / Billetera, Crédito, Débito}; `ing_tipo` ∈ {Sueldo, Devoluciones, Intereses, Billetera, Ahorro, Movimiento interno} (la séptima opción del formulario, "Otro", no tiene datos). Estos son los valores que produce el cliente hoy — no están garantizados por ningún constraint.

---

### `subcategorias_config`

Catálogo de subcategorías del usuario, con el flag que determina si una subcategoría se trata como gasto fijo.

| Columna | Tipo | Nullable | Default |
|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` |
| `user_id` | uuid | NO | — |
| `categoria` | text | NO | — |
| `subcategoria` | text | NO | — |
| `es_fijo` | boolean | SÍ | `false` |
| `orden` | integer | SÍ | `0` |
| `created_at` | timestamptz | SÍ | `now()` |

**Constraints:** `subcategorias_config_pkey` PRIMARY KEY (`id`); FK `user_id` → `auth.users(id)` ON DELETE CASCADE. Sin UNIQUE sobre (`user_id`, `categoria`, `subcategoria`) — no hay ninguna restricción de base de datos que impida subcategorías duplicadas para un mismo usuario.

**Índices:** `subcategorias_config_pkey` (único, `id`); `idx_subcats_user` (`user_id`, `categoria`).

**Vínculo con `movimientos`:** el vínculo entre un movimiento y su subcategoría es por valor de texto (`categoria`+`subcategoria`, comparados en minúsculas por el cliente), no por clave foránea — no existe una relación formal a nivel de base de datos entre ambas tablas.

---

### `config_usuario`

Configuración por usuario: día de cierre de tarjeta y objetivo de ahorro. Relación 1:1 con el usuario (la PK es directamente `user_id`).

| Columna | Tipo | Nullable | Default |
|---|---|---|---|
| `user_id` | uuid | NO | — |
| `cierre_tc_dia` | integer | NO | `20` |
| `objetivo_monto` | numeric | SÍ | — |
| `objetivo_plazo` | integer | SÍ | — |
| `objetivo_mes_inicio` | text | SÍ | — |
| `objetivo_mes_fin` | text | SÍ | — |
| `updated_at` | timestamptz | NO | `now()` |

**Constraints:** `config_usuario_pkey` PRIMARY KEY (`user_id`); FK `user_id` → `auth.users(id)` ON DELETE CASCADE.

**Índices:** `config_usuario_pkey` (único, `user_id`) — no tiene índices adicionales; no los necesita, dado que el acceso es siempre por PK.

**Nota de diseño (Fase 2.2, Causa raíz C):** este es uno de los dos datos (junto con el Objetivo, que vive en las mismas columnas) con doble representación — también se guarda en `localStorage` del dispositivo. Ver `DECISIONS.md`.

---

### `fijos_config`

| Columna | Tipo | Nullable | Default |
|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` |
| `user_id` | uuid | NO | — |
| `nombre` | text | NO | — |
| `categoria` | text | NO | — |
| `subcategoria` | text | NO | — |
| `orden` | integer | SÍ | `0` |
| `created_at` | timestamptz | SÍ | `now()` |

**Constraints:** `fijos_config_pkey` PRIMARY KEY (`id`); FK `user_id` → `auth.users(id)` ON DELETE CASCADE.

**Índices:** `fijos_config_pkey` (único, `id`) — sin índices adicionales.

**⚠️ Tabla sin consumidor activo, confirmado en la auditoría 2026-07-31.** No existe ninguna referencia a `fijos_config` en `index.html`, ni ningún trigger, función o vista en el esquema `public` que la use. Historial confirmado por `git log`: el commit `aea3abf` ("Fijos configurables por usuario desde Supabase") introdujo su uso; el commit `e2551e0` ("Subcategorias_config unificado + fijos opcionales insight + sin tabs duplicados") reemplazó, línea por línea, cada llamada a `fijos_config` por la llamada equivalente a `subcategorias_config` (que agregó la columna `es_fijo` en su lugar). La tabla no fue eliminada de Supabase — sus filas permanecen sin ningún consumidor desde ese commit en adelante. Ninguna decisión fue tomada sobre su destino durante la Fase 3 — queda pendiente si corresponde eliminarla o conservarla.

---

## Seguridad de acceso (RLS)

Las 4 tablas tienen Row Level Security **habilitado** (`rowsecurity = true`, verificado 2026-07-31). Políticas activas:

| Tabla | Política | Comando | Regla |
|---|---|---|---|
| `movimientos` | `users_own_data` | ALL (una sola política cubre las 4 operaciones) | `auth.uid() = user_id` (lectura y escritura) |
| `subcategorias_config` | `subcats_own` | ALL | `auth.uid() = user_id` (lectura y escritura) |
| `fijos_config` | `fijos_own` | ALL | `auth.uid() = user_id` (lectura y escritura) |
| `config_usuario` | `config propia - select/insert/update/delete` | 4 políticas separadas, una por comando | `auth.uid() = user_id` en cada una |

**Grants de rol:** el rol `anon` tiene los mismos privilegios de tabla que `authenticated` sobre las 4 tablas (SELECT/INSERT/UPDATE/DELETE/TRUNCATE/REFERENCES/TRIGGER) — patrón por defecto de Supabase, sin configuración especial de este proyecto. La protección real depende enteramente de que RLS permanezca habilitado. Ver `DECISIONS.md` (Bloque Hardening — Infraestructura) para la decisión de reducir estos grants.

---

## Relación con `auth.users`

Todas las tablas referencian `auth.users(id)` vía foreign key con `ON DELETE CASCADE` — al eliminar un usuario, se eliminan automáticamente todas sus filas en las 4 tablas. Verificado en la auditoría: 0 filas huérfanas en ninguna de las 4 tablas respecto a `auth.users`. No existe ninguna tabla propia de la aplicación para usuarios — la identidad y autenticación son gestionadas enteramente por Supabase Auth.

---

## Objetos adicionales del esquema `public`

Confirmado en la auditoría: no existe ningún trigger, función (`pg_proc`) ni vista (`information_schema.views`) definidos en el esquema `public`. Toda la lógica de negocio vive en el cliente (`index.html`), no en la base de datos.
