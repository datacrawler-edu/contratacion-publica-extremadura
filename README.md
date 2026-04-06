# Contratación Pública de Extremadura

Base de datos de contratación pública de la Comunidad Autónoma de Extremadura. Datos extraídos de fuentes oficiales, procesados y listos para reutilizar.

Explora los datos en el portal web: **[datosextremadura.com](https://datosextremadura.com)**

---

## Descarga

La base de datos se distribuye como archivo SQLite (`.db`) adjunto en cada [Release](https://github.com/datacrawler-edu/contratacion-publica-extremadura/releases).

| Archivo | Descripción | Tamaño aprox. |
|---|---|---|
| `transparencia_extremadura.db` | Base de datos SQLite completa | ~450 MB |

### Cómo descargar

```bash
# Desde la línea de comandos (última versión)
gh release download --repo datacrawler-edu/contratacion-publica-extremadura --pattern "*.db"
```

O descarga manualmente desde la pestaña [Releases](https://github.com/datacrawler-edu/contratacion-publica-extremadura/releases).

---

## Contenido

### Tablas principales

#### `contrato` — 114.851 filas
Contratos adjudicados y licitaciones publicadas.

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | INTEGER | Identificador interno |
| `id_original` | TEXT | ID en la plataforma de origen |
| `url` | TEXT | URL del contrato en PLACSP |
| `expediente` | TEXT | Número de expediente |
| `organo_id` | INTEGER | FK → `organo_contratante.id` |
| `empresa_adj_id` | INTEGER | FK → `empresa.id` (NULL si no adjudicado) |
| `estado_code` | TEXT | FK → `cat_estado.code` |
| `conjunto_code` | TEXT | `menores` o `licitaciones` |
| `tipo_contrato_code` | TEXT | FK → `cat_tipo_contrato.code` |
| `procedimiento_code` | TEXT | FK → `cat_procedimiento.code` |
| `objeto` | TEXT | Descripción del contrato |
| `cpv` | TEXT | Código CPV (clasificación europea) |
| `nuts` | TEXT | Código NUTS (clasificación geográfica) |
| `presupuesto_sin_iva` | REAL | Presupuesto base de licitación (€) |
| `presupuesto_con_iva` | REAL | Presupuesto con IVA (€) |
| `importe_adj_sin_iva` | REAL | Importe adjudicado sin IVA (€) |
| `importe_adj_con_iva` | REAL | Importe adjudicado con IVA (€) |
| `num_ofertas` | REAL | Número de ofertas recibidas |
| `fecha_publicacion` | TEXT | Fecha de publicación (ISO 8601) |
| `fecha_adjudicacion` | TEXT | Fecha de adjudicación (ISO 8601) |
| `score_calidad` | REAL | Score de calidad del dato (0–100) |

#### `empresa` — 19.286 filas
Empresas adjudicatarias, deduplicadas y cruzadas con el BORME.

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | INTEGER | Identificador interno |
| `nombre` | TEXT | Nombre oficial |
| `nif` | TEXT | NIF/CIF (cuando disponible) |
| `borme_id` | INTEGER | ID en el Registro Mercantil (BORME) |
| `nif_estado` | TEXT | `verificado`, `deducido`, `sin_nif` |
| `confianza` | INTEGER | Nivel de confianza del cruce BORME (0–3) |

#### `organo_contratante` — 1.184 filas
Entidades públicas que contratan.

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | INTEGER | Identificador interno |
| `nombre` | TEXT | Nombre del órgano |
| `nif` | TEXT | NIF del órgano |
| `dir3` | TEXT | Código DIR3 |
| `nuts` | TEXT | Código NUTS de ubicación |

#### `persona` — 177.483 filas
Personas con cargos en empresas adjudicatarias (extraído del BORME).

| Columna | Tipo | Descripción |
|---|---|---|
| `id` | INTEGER | Identificador interno |
| `nombre` | TEXT | Nombre completo |
| `borme_person_id` | INTEGER | ID en el BORME |

#### `cargo_empresa` — 336.553 filas
Relación persona–empresa con cargo y período.

| Columna | Tipo | Descripción |
|---|---|---|
| `empresa_id` | INTEGER | FK → `empresa.id` |
| `persona_id` | INTEGER | FK → `persona.id` |
| `cargo` | TEXT | Cargo (Administrador, Consejero, etc.) |
| `fecha_desde` | TEXT | Fecha de inicio del cargo |
| `fecha_hasta` | TEXT | Fecha de fin (o último filing conocido) |
| `fuente` | TEXT | Siempre `BORME` |

### Tablas de resumen (pre-calculadas)

| Tabla | Descripción |
|---|---|
| `empresa_resumen` | Stats por empresa: n_contratos, total_adj, score_medio, n_organos_distintos |
| `organo_resumen` | Stats por órgano: n_contratos, total_adj, n_empresas, concentración |
| `persona_resumen` | Stats por persona: n_empresas, n_cargos, red de órganos conectados |
| `metadata_estadisticas` | KPIs globales y rankings (top10 adjudicatarios, evolución anual, etc.) |

### Tablas de catálogo

| Tabla | Descripción |
|---|---|
| `cat_estado` | Estados del contrato (ADJ, RES, LIC, …) |
| `cat_procedimiento` | Procedimientos (Abierto, Negociado, Menor, …) |
| `cat_tipo_contrato` | Tipos (Obras, Servicios, Suministros, …) |
| `cat_cpv` | Catálogo CPV 2008 completo (9.454 códigos) |

---

## Ejemplos de consultas

```sql
-- Top 10 empresas por importe adjudicado
SELECT e.nombre, e.nif, er.n_contratos, er.total_adj
FROM empresa e
JOIN empresa_resumen er ON er.empresa_id = e.id
ORDER BY er.total_adj DESC
LIMIT 10;

-- Contratos menores por órgano en 2024
SELECT o.nombre, COUNT(*) AS n, SUM(c.importe_adj_sin_iva) AS total
FROM contrato c
JOIN organo_contratante o ON o.id = c.organo_id
WHERE c.conjunto_code = 'menores'
  AND c.fecha_adjudicacion LIKE '2024%'
GROUP BY o.id
ORDER BY total DESC;

-- Personas con cargos en más de 5 empresas adjudicatarias
SELECT p.nombre, COUNT(DISTINCT ce.empresa_id) AS n_empresas
FROM persona p
JOIN cargo_empresa ce ON ce.persona_id = p.id
JOIN empresa e ON e.id = ce.empresa_id
JOIN empresa_resumen er ON er.empresa_id = e.id
WHERE er.n_contratos > 0
GROUP BY p.id
HAVING n_empresas > 5
ORDER BY n_empresas DESC;

-- Evolución anual del importe adjudicado
SELECT strftime('%Y', fecha_adjudicacion) AS anio,
       COUNT(*) AS n_contratos,
       SUM(importe_adj_sin_iva) AS total_adj
FROM contrato
WHERE fecha_adjudicacion >= '2018-01-01'
GROUP BY anio
ORDER BY anio;
```

---

## Fuente de los datos

- **Contratos y licitaciones**: [Plataforma de Contratación del Sector Público (PLACSP)](https://contrataciondelestado.es) — conjunto de datos de Extremadura
- **Datos registrales**: [BORME](https://www.boe.es/diario_borme/) (Boletín Oficial del Registro Mercantil) — cargos directivos de empresas
- **Clasificaciones**: CPV 2008 (Vocabulario Común de Contratos Públicos), NUTS (Nomenclatura de Unidades Territoriales Estadísticas)

Los datos son de dominio público. Su uso está sujeto a las condiciones de reutilización de la información del sector público.

---

## Cobertura y limitaciones

- **Período**: 2018–presente (44 contratos anteriores a 2018 con fechas no fiables, excluidos de los análisis)
- **Ámbito**: Comunidad Autónoma de Extremadura únicamente
- **Empresas**: 36.709 nombres originales deduplicados a 19.286 entidades únicas mediante normalización de NIF y similitud de nombre
- **BORME**: El cruce con el Registro Mercantil cubre ~70% de las empresas con NIF verificado. `fecha_hasta` en `cargo_empresa` refleja el último filing conocido, no necesariamente la fecha real de cese
- **Sin garantía de exhaustividad**: Los datos reflejan lo publicado en PLACSP. Contratos no publicados o publicados con errores no están recogidos

---

## Actualización

Los datos se actualizan mensualmente. La fecha de la última actualización está disponible en la tabla `metadata_carga`.

```sql
SELECT * FROM metadata_carga ORDER BY fecha_carga DESC LIMIT 5;
```

---

## Licencia

Los datos originales son de dominio público (fuentes oficiales del Estado español).  
Este repositorio se publica sin restricciones adicionales — úsalos libremente.

---

## Contacto

[@edu_seo_scraper](https://x.com/edu_seo_scraper) en X/Twitter  
Portal web: [datosextremadura.com](https://datosextremadura.com)
