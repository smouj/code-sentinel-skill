name: Code Sentinel
version: 2.1.0
description: Advanced code security scanner for detecting vulnerabilities in commits and PRs
maintainer: security-team
tags: [security, scanning, code, vulnerabilities, saastools]
dependencies: [git, semgrep, trufflehog, bandit, gitleaks, docker]
requires_network: true
---

# Code Sentinel

## Propósito

Code Sentinel realiza análisis estático profundo (SAST), detección de secretos y escaneo de dependencias en commits y pull requests de código. Se integra en pipelines de CI/CD para detectar:

- Credenciales hardcodeadas (API keys, tokens, contraseñas)
- Patrones de SQL injection, XSS, command injection
- Deserialización insegura y Functions inseguras
- Dependencias vulnerables (mediante análisis SBOM)
- Configuraciones incorrectas en Dockerfiles, manifiestos de K8s, IaC

**Casos de uso reales:**
- Bloqueo de merges de PR cuando se detectan vulnerabilidades de alta severidad
- Reportes de seguridad automatizados en comentarios de PR de GitHub/GitLab
- Escaneos completos de repositorio nocturnos para detección de deriva
- Verificación de cumplimiento (PCI-DSS, SOC2) con generación de evidencia
- Retroalimentación a desarrolladores via pre-commit hooks para remediación rápida

## Alcance

### Comandos

`sentinel scan` - Escanea archivos, directorios o commits específicos

```bash
# Scan staged changes
sentinel scan --staged

# Scan specific commit
sentinel scan --commit abc123

# Scan PR diff (when in CI)
sentinel scan --pr 42

# Scan entire repo
sentinel scan --path . --full

# Custom rules
sentinel scan --rules custom-rules/ --severity high,critical

# Output formats
sentinel scan --format sarif  # for GitHub Code Scanning
sentinel scan --format json   # for custom parsers
sentinel scan --format html   # for human review
```

`sentinel config` - Configura perfiles de escáner

```bash
# Set project security profile
sentinel config set profile webapp   # includes OWASP Top 10
sentinel config set profile mobile   # mobile-specific rules
sentinel config set profile api      # API-focused checks

# Configure ignore patterns
sentinel config set ignore "*.min.js,dist/*"

# Set severity thresholds
sentinel config set threshold high   # fail on high+ issues
```

`sentinel fix` - Aplica correcciones automáticas para problemas de bajo riesgo

```bash
# Interactive fix mode
sentinel fix --interactive

# Auto-fix specific rule
sentinel fix --rule hardcoded-secret --file src/config.py

# Dry-run first
sentinel fix --dry-run
```

`sentinel report` - Genera y gestiona reportes de seguridad

```bash
# Generate executive summary
sentinel report --type executive --output security-summary.html

# Export compliance evidence
sentinel report --compliance pci-dss --format pdf

# Compare scans (drift detection)
sentinel report --baseline scan-001.json --current scan-002.json
```

`sentinel daemon` - Ejecuta como servicio en segundo plano para escaneo en tiempo real

```bash
# Start daemon (daemon mode)
sentinel daemon --watch . --port 8080

# Connect IDE plugin to daemon
sentinel daemon --ide-integration
```

## Proceso de Trabajo

### 1. Preparación Pre-escaneo

```bash
# Clone target repository
git clone <repo-url>
cd <repo>

# Load configuration
sentinel config load .sentinel.yml  # project-specific config
sentinel config load ~/.sentinel/profiles/default.yml  # global config

# Verify toolchain
sentinel doctor  # checks semgrep, trufflehog, docker are installed
```

### 2. Seleccionar Objetivo de Escaneo

Por rango de commits:
```bash
# Scan commits since last release
sentinel scan --since v1.2.0

# Scan commits in current branch not in main
sentinel scan --branch feature/login --against main
```

Por tipo de archivo:
```bash
# Scan only Python secrets
sentinel scan --lang python --rule HARDCODED_SECRET

# Scan container images
sentinel scan --docker-image myapp:latest
```

### 3. Ejecutar Escaneo Multi-Motor

Sentinel ejecuta análisis secuencial:

1. **Detección de Secretos** (truffleHog + gitleaks)
   - Escanea en busca de AWS keys, GitHub tokens, Slack webhooks
   - Análisis de entropía para strings de alta entropía
   - Expresiones regulares para más de 200 formatos de secretos

2. **SAST** (semgrep con reglasets personalizados)
   - Reglas OWASP Top 10
   - Vulnerabilidades específicas por lenguaje
   - Reglas personalizadas de la organización

3. **SBOM + Verificación de Dependencias** (si existe package.json/requirements.txt)
   - Generar CycloneDX SBOM
   - Consultar bases de datos de vulnerabilidades (NVD, GitHub Advisory)
   - Buscar CVEs conocidos

4. **Escaneo de IaC** (checkov/terrascan si se detectan archivos IaC)
   - Terraform, CloudFormation, K8s, Dockerfile security

### 4. Triaje y Deduplicación

```bash
# Interactive triage
sentinel triage --interactive  # mark as false-positive, accept risk, etc.

# Auto-deduplicate
sentinel dedupe --strategy hash  # merge identical findings across commits
```

### 5. Generación de Reportes

La salida incluye:
- **Resumen**: Conteo por severidad, categoría, archivo
- **Hallazgos**: Números de línea, fragmentos de código, IDs CWE, sugerencias de corrección
- **Evidencia**: Hash de commit, autor, marca de tiempo
- **Remediación**: Cambios de código específicos, referencias OWASP, puntajes CVSS

Ejemplo de salida JSON:
```json
{
  "scan_id": "sentinel-2024-03-15-abc123",
  "repo": "myorg/myapp",
  "commit": "abc123def456",
  "findings": [
    {
      "severity": "critical",
      "rule": "hardcoded-aws-secret",
      "file": "src/aws.py",
      "line": 42,
      "code": "AWS_SECRET = 'AKIAIOSFODNN7EXAMPLE'",
      "confidence": "high",
      "cwe": "CWE-798",
      "fix": "Use AWS Secrets Manager or environment variable",
      "exploitability": 9.1
    }
  ]
}
```

### 6. Ganchos de Integración

**GitHub Actions** (`.github/workflows/sentinel.yml`):
```yaml
- name: Run Code Sentinel
  uses: code-sentinel/action@v2
  with:
    sarif-output: true
    fail-on-severity: high
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v2
  with:
    sarif_file: sentinel-results.sarif
```

**GitLab CI** (`.gitlab-ci.yml`):
```yaml
sentinel_scan:
  script:
    - sentinel scan --pr $CI_MERGE_REQUEST_IID --format sarif
  artifacts:
    paths: [sentinel-results.sarif]
    expire_in: 1 week
```

## Reglas de Oro

1. **Nunca desactivar el escaneo** – Siempre ejecutar sentinel en PRs dirigidos a main/staging
2. **Tratar los secretos como críticos** – Cualquier credencial hardcodeada debe bloquear el merge
3. **Corregir en el código fuente, no en la configuración** – Si una regla genera falsos positivos, mejora la regla, no la lista de ignorados
4. **Revisar hallazgos dentro de 4 horas** – Los hallazgos de seguridad tienen SLA basado en severidad:
   - Crítico: 1 hora
   - Alto: 4 horas
   - Medio: 24 horas
   - Bajo: 1 semana
5. **Mantener reglas personalizadas como código** – Almacenar reglas específicas de la organización en `security/rules/` y versionarlas
6. **Firmar todos los commits** – Forzar commits con firma GPG para prevenir manipulación
7. **Rotar si se filtra un secreto** – Si sentinel encuentra una credencial expuesta, rotar inmediatamente, sin importar la antigüedad
8. **Escanear todo** – Incluir scripts, tests, docs (los secretos a menudo se esconden allí)

## Ejemplos

### Ejemplo 1: Hook Pre-commit para Proyecto Python

`.git/hooks/pre-commit`:
```bash
#!/bin/bash
# Code Sentinel pre-commit hook

echo "🔍 Running Code Sentinel on staged changes..."

# Only scan staged Python/JS files
STAGED_FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(py|js|ts)$')
if [ -z "$STAGED_FILES" ]; then
  exit 0
fi

# Run sentinel
sentinel scan --files "$STAGED_FILES" --format json --output /tmp/sentinel-precommit.json

# Check for failures
if jq -e '.findings[] | select(.severity | test("critical|high"))' /tmp/sentinel-precommit.json > /dev/null; then
  echo "❌ Critical/High vulnerabilities detected. Commit blocked."
  sentinel report /tmp/sentinel-precommit.json --format pretty
  exit 1
fi

echo "✅ No blocking vulnerabilities found."
exit 0
```

### Ejemplo 2: Automatización de Comentarios en PR

GitHub Action step:
```yaml
- name: Comment PR with findings
  run: |
    sentinel scan --pr ${{ github.event.pull_request.number }} --format json > results.json
    
    CRITICAL=$(jq '[.findings[] | select(.severity=="critical")] | length' results.json)
    HIGH=$(jq '[.findings[] | select(.severity=="high")] | length' results.json)
    
    if [ "$CRITICAL" -gt 0 ] || [ "$HIGH" -gt 0 ]; then
      gh pr comment ${{ github.event.pull_request.number }} \
        --body "🚨 **Code Sentinel found $CRITICAL critical and $HIGH high severity vulnerabilities.**\n\nRun \`sentinel scan --pr ${{ github.event.pull_request.number }}\` locally to view details."
    fi
```

### Ejemplo 3: Script de Triaje de Falsos Positivos

`scripts/triage-findings.sh`:
```bash
#!/bin/bash
SCAN_ID=$1
MARK_AS=$2  # "false-positive", "accept-risk", "wontfix"

# Get all findings
sentinel findings list --scan $SCAN_ID --format json > /tmp/findings.json

# Loop through and mark
jq -c '.[]' /tmp/findings.json | while read -r finding; do
  ID=$(echo "$finding" | jq -r '.id')
  sentinel findings update $ID --status $MARK_AS --reason "Reviewed by $USER"
done

echo "Triage complete for scan $SCAN_ID"
```

## Verificación

```bash
# 1. Verify installation
sentinel --version  # should output 2.1.0
sentinel doctor     # checks all dependencies

# 2. Test scan on known vulnerable code
curl -s https://raw.githubusercontent.com/OWASP/CheatSheetSeries/master/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.md > test_vuln.py
sentinel scan test_vuln.py
# Should detect SQL injection patterns

# 3. Verify GitHub integration (if configured)
sentinel test-github --repo myorg/testrepo
# Should post test comment to PR

# 4. Check SARIF output validity
sentinel scan . --format sarif > results.sarif
jq '.' results.sarif  # should parse without error
```

## Reversión

Si un escaneo bloquea incorrectamente un merge o genera falsos positivos:

1. **Anulación temporal** (para despliegues urgentes):
```bash
# Bypass sentinel check for specific commit
git commit --no-verify -m "Urgent fix"
# OR set environment variable
SENTINEL_SKIP=true git push
```

2. **Revertir actualización de regla de sentinel** (si un cambio reciente de regla causó problemas):
```bash
# List recent config changes
sentinel config history

# Restore previous config
sentinel config restore --revision abc123 --file .sentinel.yml

# Disable problematic rule set
sentinel config disable-rules --rule-id XSS-001
```

3. **Limpiar datos de escaneo corruptos**:
```bash
# Remove specific scan results
sentinel scans delete --id sentinel-2024-03-15-xyz789

# Reset local cache (forces fresh scan)
rm -rf ~/.cache/sentinel/
```

4. **Revertir estado del demonio**:
```bash
sentinel daemon stop
# Restore daemon config from backup
cp ~/.sentinel/backups/daemon-config-2024-03-14.yml ~/.sentinel/daemon.yml
sentinel daemon start
```

## Dependencias y Requisitos

**Sistema:**
- Python 3.9+ (para semgrep, scripts personalizados)
- Docker (para escaneo de contenedores)
- Git 2.30+

**Paquetes de Python** (instalados via `pip install codesentinel`):
- semgrep>=1.50.0
- truffleHog>=3.0.0
- gitleaks>=8.18.0
- pyyaml, jsonschema, tabulate

**Opcional:**
- trivy (para escaneo de imágenes de contenedor)
- checkov (para IaC)
- bandit (específico de Python)
- npm audit (JavaScript)

**Variables de entorno:**
- `SENTINEL_API_KEY` – Para feed de reglas comerciales (por defecto: lee de `~/.sentinel/api-key`)
- `SENTINEL_CONFIG_PATH` – Sobrescribir ubicación de configuración (por defecto: `.sentinel.yml`)
- `SENTINEL_CACHE_DIR` – Directorio de caché (por defecto: `~/.cache/sentinel/`)
- `SENTINEL_GITHUB_TOKEN` – Para acceso a la API de GitHub (para comentarios en PR, datos de advisory)

## Solución de Problemas

**Problema**: `sentinel scan` se cuelga en repositorio grande
**Solución**: Excluir directorios con `sentinel config set ignore "node_modules,build,dist"` o usar `--max-files 1000`

**Problema**: Falsos positivos en fixtures de pruebas
**Solución**: Marcar directorio como tests en config: `sentinel config set test-dirs "test,tests,__test__"`

**Problema**: Archivo SARIF rechazado por GitHub
**Solución**: Validar esquema SARIF: `sentinel validate-sarif results.sarif` y asegurar versión 2.1.0

**Problema**: No se detectan secretos en archivo vulnerable conocido
**Solución**: Actualizar patrones regex: `sentinel rules update --source official` y verificar que `trufflehog` sea >= 3.0

**Problema**: Alto uso de memoria en monorepo
**Solución**: Usar escaneo incremental: `sentinel scan --changed-since origin/main --workers 2` (limitar workers)

**Problema**: Fallo en escaneo Docker con permiso denegado
**Solución**: Agregar usuario al grupo docker: `sudo usermod -aG docker $USER` y reiniciar sesión```