# Resolución de políticas — UNTDF Lab

## Visión general

El sistema de políticas del UNTDF Lab usa un modelo jerárquico de herencia con merge:

```
Política Org (default) → Política Curso → Override Asignación
```

Cada nivel puede sobrescribir campos del nivel anterior, permitiendo flexibilidad pedagógica mientras se mantienen estándares organizacionales.

## Estructura del repositorio

```
untdf-lab-policies/
├── schemas/
│   └── policy.schema.json      # JSON Schema v7 de validación
├── org/
│   └── default.yaml            # Política organizacional base
├── courses/
│   ├── paradigmas-2025/
│   │   └── policy.yaml         # Política del curso de Paradigmas
│   ├── bases-datos-2025/
│   │   └── policy.yaml         # Política del curso de Bases de Datos
│   └── ...
└── docs/
    ├── policy-resolution.md    # Este archivo
    └── schema-reference.md     # Referencia completa del schema
```

## Algoritmo de resolución

### Paso 1: Cargar política organizacional

```python
org_policy = load_yaml("org/default.yaml")
validate_schema(org_policy, "schemas/policy.schema.json")
```

### Paso 2: Cargar política de curso (si existe)

```python
course_policy = load_yaml(f"courses/{course_id}/policy.yaml")
validate_schema(course_policy, "schemas/policy.schema.json")
```

### Paso 3: Merge curso sobre org

```python
merged_policy = deep_merge(org_policy, course_policy)
```

El merge usa las siguientes reglas:
- **Objetos:** merge recursivo campo por campo
- **Arrays:** reemplazar completo (no merge de elementos)
- **Primitivos:** valor del curso sobrescribe org

Ejemplo:
```yaml
# org/default.yaml
thresholds:
  coverage: 70
  duplicacion: 10

# courses/paradigmas-2025/policy.yaml
thresholds:
  coverage: 75  # Sobrescribe

# Resultado merged:
thresholds:
  coverage: 75      # Del curso
  duplicacion: 10   # Del org (heredado)
```

### Paso 4: Aplicar override de asignación (si existe)

```python
if assignment_id in merged_policy.get("overrides", {}):
    override = merged_policy["overrides"][assignment_id]
    final_policy = deep_merge(merged_policy, override)
else:
    final_policy = merged_policy
```

### Paso 5: Validación final

```python
validate_schema(final_policy, "schemas/policy.schema.json")
validate_weights_sum(final_policy["weights"])  # Debe sumar 1.0
```

## Caching

### Estrategia de cache

Cada política resuelta se cachea con:
- **Key:** `{course_id}:{assignment_id}:{policy_ref}`
- **TTL:** Configurable (default: 24 horas)
- **Invalidación:** Por `policy_ref` (commit SHA o tag)

```python
cache_key = f"{course_id}:{assignment_id}:{policy_ref}"
cached = redis.get(cache_key)

if cached and not_expired(cached):
    return cached["policy"]

# Fetch from GitHub
policy = resolve_policy(course_id, assignment_id, policy_ref)
redis.setex(cache_key, ttl=86400, value=policy)
return policy
```

### ETags

Usar GitHub API ETags para validación eficiente:

```python
headers = {"If-None-Match": cached_etag}
response = github.get_file(path, headers=headers)

if response.status == 304:  # Not Modified
    return cached_policy
else:
    new_policy = parse_yaml(response.content)
    cache_etag = response.headers["ETag"]
    return new_policy
```

## Policy References

Las asignaciones deben especificar un `policyRef` para reproducibilidad:

```yaml
# En repo de alumno: .untdf-lab.yaml
assignment:
  id: tp2-poo-django
  policyRef: v1.2.0  # Tag del repo de políticas
```

Si no se especifica `policyRef`, usar `main` branch (⚠️ no reproducible).

## Fallback y resiliencia

### Fallback en caso de error

```python
try:
    course_policy = load_course_policy(course_id)
except (FileNotFoundError, NetworkError):
    logger.warning(f"Course policy not found for {course_id}, using org default")
    course_policy = {}

try:
    merged = deep_merge(org_policy, course_policy)
except MergeError:
    logger.error("Policy merge failed, using org default only")
    merged = org_policy
```

### Validación estricta

Si una política no pasa validación de schema:
1. Loggear error con detalles
2. Fallar el análisis (no continuar con política inválida)
3. Notificar a instructores

```python
if not validate_schema(policy):
    raise PolicyValidationError(
        f"Policy for {course_id} failed schema validation",
        errors=validation_errors
    )
```

## Ejemplos de uso

### Caso 1: Curso sin política propia

```python
course_id = "algoritmos-2025"  # Sin policy.yaml
assignment_id = "tp1-sorting"

# Resultado: solo org/default.yaml
policy = resolve_policy(course_id, assignment_id)
assert policy["thresholds"]["coverage"] == 70  # De org
```

### Caso 2: Curso con política, sin override

```python
course_id = "paradigmas-2025"  # Tiene policy.yaml
assignment_id = "tp4-logico"

# Resultado: org merged con course
policy = resolve_policy(course_id, assignment_id)
assert policy["thresholds"]["coverage"] == 75  # De course
assert policy["weights"]["calidad"] == 0.60    # De course
```

### Caso 3: Override de asignación específica

```python
course_id = "paradigmas-2025"
assignment_id = "tp1-introduccion-python"  # Tiene override

# Resultado: org → course → override
policy = resolve_policy(course_id, assignment_id)
assert policy["thresholds"]["coverage"] == 60      # De override
assert policy["quiz"]["questionsCount"] == 4       # De override
assert policy["weights"]["calidad"] == 0.60        # De course (sin override)
```

## Versionado

- **Schema:** Semver (`policy.schema.json` version)
- **Políticas:** Tags git (`v1.0.0`, `v1.1.0`, etc.)
- **Breaking changes:** Major version bump en schema

Ejemplo de breaking change:
```yaml
# v1.0.0
weights:
  proceso: 0.2
  calidad: 0.6
  razonamiento: 0.2

# v2.0.0 (breaking: rename campo)
weights:
  process: 0.2      # Renombrado
  codeQuality: 0.6  # Renombrado
  reasoning: 0.2    # Renombrado
```

## Testing

Validar políticas en CI:

```bash
# Validar todos los YAML contra schema
npm run validate:policies

# Verificar que pesos suman 1.0
npm run validate:weights

# Smoke test de resolución
npm run test:resolution
```

Ver `/.github/workflows/validate-policies.yml` para implementación.

---

**Última actualización:** 2 noviembre 2025  
**Autor:** UNTDF Lab Platform Team
