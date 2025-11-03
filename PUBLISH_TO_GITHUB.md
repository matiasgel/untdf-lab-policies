# Instrucciones para publicar untdf-lab-policies en GitHub

## Opción 1: Crear repo en organización `untdf-lab` (recomendado)

Si tienes acceso a una organización GitHub para UNTDF Lab:

```bash
# 1. Crear repo en GitHub (vía web o gh CLI)
gh repo create untdf-lab/untdf-lab-policies \
  --public \
  --description "Políticas de análisis y evaluación para UNTDF Lab" \
  --source=/home/matiasgel/desarrollo/paradigmas/repos/untdf-lab-policies

# 2. Push inicial
cd /home/matiasgel/desarrollo/paradigmas/repos/untdf-lab-policies
git remote add origin https://github.com/untdf-lab/untdf-lab-policies.git
git push -u origin main
```

## Opción 2: Crear repo en cuenta personal

```bash
# 1. Crear repo en tu cuenta personal
gh repo create untdf-lab-policies \
  --public \
  --description "Políticas de análisis y evaluación para UNTDF Lab" \
  --source=/home/matiasgel/desarrollo/paradigmas/repos/untdf-lab-policies

# O manualmente en GitHub web, luego:
cd /home/matiasgel/desarrollo/paradigmas/repos/untdf-lab-policies
git remote add origin https://github.com/TU_USERNAME/untdf-lab-policies.git
git push -u origin main
```

## Opción 3: Crear manualmente en GitHub web

1. Ir a https://github.com/new
2. Repository name: `untdf-lab-policies`
3. Description: "Políticas de análisis y evaluación para UNTDF Lab"
4. Public
5. **NO** inicializar con README, .gitignore o license (ya los tienes localmente)
6. Crear repositorio
7. Ejecutar:

```bash
cd /home/matiasgel/desarrollo/paradigmas/repos/untdf-lab-policies
git remote add origin https://github.com/TU_USERNAME/untdf-lab-policies.git
git push -u origin main
```

## Verificar CI/CD

Después del push, GitHub Actions ejecutará automáticamente el workflow `validate-policies.yml`:

1. Ir a https://github.com/<org o user>/untdf-lab-policies/actions
2. Verificar que el workflow "Validate Policies" se ejecutó
3. Debe estar verde ✅

Si falla, revisar los logs para ver qué validación falló.

## Siguiente paso: MCP Validation

Una vez publicado, ejecutar validación MCP desde el backend o extensión VS Code:

```python
# Ejemplo de validación MCP
from mcp_tools import github

# Verificar que el repo existe
repos = github.list_repos(org="untdf-lab")  # o tu username
assert "untdf-lab-policies" in [r.name for r in repos]

# Verificar que el schema existe
schema_file = github.get_file(
    repo="untdf-lab-policies",
    path="/schemas/policy.schema.json"
)
assert schema_file is not None
assert len(schema_file.content) > 1000  # Schema tiene contenido

# Verificar que el CI pasó
workflow_runs = github.get_workflow_runs(
    repo="untdf-lab-policies",
    workflow="validate-policies.yml"
)
latest_run = workflow_runs[0]
assert latest_run.status == "success"
```

## Post-publicación

1. Agregar badge de CI al README:
   ```markdown
   ![Validate Policies](https://github.com/untdf-lab/untdf-lab-policies/workflows/Validate%20Policies/badge.svg)
   ```

2. Crear tag para la versión inicial:
   ```bash
   git tag -a v1.0.0 -m "Initial release: org policy + Lab Prog y Lenguajes 2025"
   git push origin v1.0.0
   ```

3. Actualizar tracking `T1.1.md` con:
   - URL del repo
   - URL del CI run
   - Marcar entregables como completos
   - Estado de tarea: 🟢 Done

---

**Generado:** 2 noviembre 2025 23:00 UTC-3
