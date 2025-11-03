# UNTDF Lab Policies

Repositorio de políticas de análisis y evaluación para la plataforma UNTDF Lab.

## 📋 Descripción

Este repositorio define políticas centralizadas que controlan cómo se analiza y evalúa el código de los estudiantes en la plataforma UNTDF Lab. Las políticas incluyen:

- ✅ Umbrales de calidad (cobertura, lint, duplicación, seguridad)
- ⚖️ Pesos de rúbricas (proceso, calidad, razonamiento)
- 🔍 Configuración de analizadores estáticos
- 📊 Niveles de telemetría y retención
- 📝 Configuración de micro-quizzes

## 🏗️ Estructura

```
untdf-lab-policies/
├── schemas/
│   └── policy.schema.json      # JSON Schema v7 de validación
├── org/
│   └── default.yaml            # Política organizacional (base)
├── courses/
│   ├── paradigmas-2025/
│   │   └── policy.yaml         # Política del curso Paradigmas
│   └── ...                     # Otros cursos
└── docs/
    ├── policy-resolution.md    # Algoritmo de resolución
    └── schema-reference.md     # Referencia completa
```

## 🔄 Resolución de políticas

El sistema usa un modelo jerárquico con merge:

```
Org Default → Course Policy → Assignment Override
```

Cada nivel sobrescribe campos del nivel anterior. Ver [`docs/policy-resolution.md`](./docs/policy-resolution.md) para detalles.

## 📝 Ejemplo de política

```yaml
version: "1.0.0"

telemetry:
  level: standard
  retentionDays: 90

thresholds:
  coverage: 75
  duplicacion: 5

weights:
  proceso: 0.20
  calidad: 0.60
  razonamiento: 0.20

analyzers:
  python:
    enabled: true
    tools: [flake8, black, bandit, radon]
    timeout: 300
```

## 🚀 Uso

### En backend (Python/Django)

```python
from policy_resolver import resolve_policy

# Resolver política para curso + asignación
policy = await resolve_policy(
    course_id="paradigmas-2025",
    assignment_id="tp2-poo-django",
    policy_ref="v1.0.0"  # Tag o commit SHA
)

# Usar umbrales
if coverage < policy["thresholds"]["coverage"]:
    add_feedback("Coverage insuficiente")

# Calcular score final
score = (
    process_score * policy["weights"]["proceso"] +
    quality_score * policy["weights"]["calidad"] +
    quiz_score * policy["weights"]["razonamiento"]
)
```

### En repos de alumnos

Los alumnos especifican el `policyRef` en `.untdf-lab.yaml`:

```yaml
assignment:
  id: tp2-poo-django
  policyRef: v1.0.0  # Fijar versión para reproducibilidad
```

## 🔐 Política de privacidad

Niveles de telemetría:

- **minimal:** Solo IDs hasheados, sin código ni métricas específicas
- **standard:** Métricas agregadas (coverage, lint count), sin código
- **full:** Análisis detallado (complejidad, funciones, rutas) sin código crudo

Ver campos `telemetry.excludePatterns` para excluir archivos sensibles.

## ✅ Validación

Toda política debe pasar validación contra [`schemas/policy.schema.json`](./schemas/policy.schema.json).

CI valida automáticamente:
- ✅ Conformidad con JSON Schema v7
- ✅ Pesos suman 1.0 (`weights.proceso + .calidad + .razonamiento == 1.0`)
- ✅ YAML syntax
- ✅ Campos requeridos presentes

## 📦 Versionado

- **Políticas:** Tags git semánticos (`v1.0.0`, `v1.1.0`)
- **Schema:** Campo `version` en JSON Schema

### Breaking changes

Incrementar major version del schema si:
- Se renombran campos requeridos
- Se cambia estructura de objetos anidados
- Se remueven campos usados por backend

## 🛠️ Desarrollo

### Agregar nuevo curso

1. Crear directorio `courses/<nombre-curso>/`
2. Copiar `org/default.yaml` como punto de partida
3. Ajustar umbrales, pesos, y analizadores
4. Commit y PR con validación de CI

### Modificar schema

1. Editar `schemas/policy.schema.json`
2. Incrementar `version` si breaking
3. Actualizar `docs/schema-reference.md`
4. Validar políticas existentes contra nuevo schema

### Testing local

```bash
# Instalar dependencias
npm install

# Validar todas las políticas
npm run validate:policies

# Ejecutar tests de resolución
npm test
```

## 📚 Documentación adicional

- [Algoritmo de resolución](./docs/policy-resolution.md)
- [Referencia completa del schema](./docs/schema-reference.md)
- [Ejemplos de overrides por asignación](./docs/examples/)

## 🤝 Contribuir

1. Fork del repo
2. Branch por feature: `git checkout -b feature/nueva-politica`
3. Validar: `npm run validate:policies`
4. Commit: `git commit -m "feat: add policy for algoritmos-2025"`
5. PR con descripción clara

## 📄 Licencia

MIT License — Ver [LICENSE](./LICENSE)

## 📧 Contacto

- **Instructor responsable:** matiasgel@untdf.edu.ar
- **Documentación:** https://untdf-lab.edu.ar/docs/policies
- **Issues:** https://github.com/untdf-lab/policies/issues

---

**Última actualización:** 2 noviembre 2025  
**Versión del schema:** 1.0.0
