# TestWFAnswer: Aquí tienes un paquete “copiar-pegar” listo para primer commit en Codec

---

README — Flujo automatizado con agente + validación humana (MVP)

Objetivo

Establecer un pipeline mínimo donde un agente de OpenAI:

1. resume y evalúa cambios propuestos, 2) ejecuta validaciones básicas, 3) publica comentarios en PR, y 4) deja la decisión final a una persona.



Alcance del MVP

Lenguajes: Markdown y código genérico.

Chequeos: formato básico, enlaces rotos, scan de secretos, límites de tamaño.

Rama: main + ramas feature/*.

CI: un workflow que corre en PR y en push a main.

Config: /.agent/config.yaml.


Requisitos

Repositorio con GitHub Actions habilitado.

Secrets:

OPENAI_API_KEY

GITHUB_TOKEN con contents:write, pull_requests:write (para comentar PR).



Estructura mínima

.
├─ .github/
│  └─ workflows/
│     └─ agent-ci.yml
├─ .agent/
│  ├─ config.yaml
│  └─ run_agent.py
├─ src/
│  └─ placeholder.txt
├─ docs/
│  └─ decisions.md
├─ CONTRIBUTING.md
├─ CODE_OF_CONDUCT.md
└─ README.md  ← este archivo

Tareas del agente

1. Escanear archivos cambiados y tipo de cambio.


2. Resumen técnico del PR.


3. Sugerencias accionables de estilo y pruebas mínimas.


4. Validaciones: enlaces Markdown, scan de secretos, límites de tamaño.


5. Salidas: comentarios en PR + artefacto .agent/agent_report.json.



Flujo de contribución

1. Rama: feature/<descriptor>.


2. Commits pequeños y descriptivos.


3. Abrir PR → el agente comenta.


4. Persona revisa y aprueba o pide cambios.


5. Solo humanos hacen merge a main.



Criterios de aceptación humana

[ ] Reporte del agente claro y útil.

[ ] Sin secretos ni datos sensibles.

[ ] Estándares de estilo cumplidos.

[ ] Riesgos entendidos y aceptados.

[ ] Pasos de prueba reproducibles si aplica.


Convenciones

Commits: type(scope): mensaje corto (p. ej., docs(readme): guía).

Ramas: feature/*, fix/*, chore/*.

ADRs cortos en docs/decisions.md.


Roadmap sugerido

Linters por lenguaje.

Matriz de pruebas por src/.

Etiquetado automático de PR.

Gate de calidad por severidad del agente.

Releases semánticos y firmas.


Cómo empezar

1. Crea repo, añade archivos de abajo.


2. Configura OPENAI_API_KEY y GITHUB_TOKEN en Secrets.


3. git add . && git commit -m "chore: bootstrap agent MVP" && git push -u origin main


4. Abre feature/readme-polish, crea PR y valida comentarios del agente.




---

.agent/config.yaml

name: openai-code-reviewer
objectives:
  - "Resumir el propósito e impacto de cada PR."
  - "Detectar riesgos y dependencias afectadas."
  - "Sugerir mejoras pequeñas de estilo y claridad."
policies:
  comment_style: "breve, accionable, sin adornos"
  max_total_comments: 10
checks:
  markdown_links: true
  secret_scan: true
  file_limits:
    max_single_file_kb: 512
    max_total_changed_files: 200
outputs:
  pr_comment: true
  artifact_path: ".agent/agent_report.json"
limits:
  tokens: 8000
model:
  provider: openai
  # Cambia por tu modelo disponible en Codecs/StartBuilding
  name: "gpt-4.1-mini"
  temperature: 0.2

.github/workflows/agent-ci.yml

name: agent-ci

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  agent-review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install deps
        run: |
          python -m pip install --upgrade pip
          pip install requests pydantic rich mdurl python-dotenv

      - name: Run agent review
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python .agent/run_agent.py \
            --event "${{ github.event_name }}" \
            --repo "${{ github.repository }}" \
            --pr "${{ github.event.pull_request.number || '' }}" \
            --config ".agent/config.yaml"

      - name: Upload report artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: agent-report
          path: .agent/agent_report.json

.agent/run_agent.py

#!/usr/bin/env python3
import os, sys, json, subprocess, re
from pathlib import Path
from typing import List, Dict, Any

# ---- util git ----
def safe_fetch(base_ref: str) -> None:
    try:
        subprocess.run(["git", "fetch", "--depth", "1", "origin", base_ref], check=False)
    except Exception:
        pass

def get_base_ref() -> str:
    ref = os.getenv("GITHUB_BASE_REF") or "origin/main"
    return ref

def changed_files(base_ref: str) -> List[str]:
    safe_fetch(base_ref)
    out = subprocess.check_output(["git", "diff", "--name-only", f"{base_ref}...HEAD"], text=True)
    return [p for p in out.splitlines() if p.strip()]

def read_text(path: str, limit_kb: int = 512) -> str:
    try:
        if os.path.getsize(path) > limit_kb * 1024:
            return f"[omitted: file >{limit_kb}KB]"
        with open(path, "r", encoding="utf-8", errors="replace") as f:
            return f.read()
    except Exception as e:
        return f"[unreadable: {e}]"

# ---- checks ----
MD_LINK_RE = re.compile(r"\[[^\]]+\]\(([^)]+)\)")
SECRET_HINTS = re.compile(r"(API_KEY|SECRET|PASSWORD|TOKEN|AKIA[0-9A-Z]{16})", re.I)

def check_markdown_links(text: str) -> List[str]:
    issues = []
    for m in MD_LINK_RE.finditer(text):
        url = m.group(1).strip()
        if url.startswith("#") or " " in url:
            issues.append(f"Posible enlace inválido: `{url}`")
    return issues

def secret_scan(text: str) -> List[str]:
    if SECRET_HINTS.search(text):
        return ["Posible secreto o credencial expuesta."]
    return []

# ---- config ----
def load_config(path: str) -> Dict[str, Any]:
    import yaml  # requires PyYAML; if prefieres, reemplaza por json simple
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f)

# ---- openai call ----
def llm_summarize_and_suggest(model: Dict[str, Any], prompt: str) -> Dict[str, Any]:
    """
    Coloca aquí la llamada real a la API de OpenAI según tu SDK en Codecs/StartBuilding.
    Esta función retorna un dict con 'summary' y 'suggestions'.
    """
    # Placeholder determinista sin red:
    return {
        "summary": "MVP: integra aquí la llamada al modelo para resumir el PR.",
        "suggestions": [
            "Usa encabezados H1 únicos en Markdown.",
            "Agrega pasos de prueba breves si hay cambios ejecutables.",
            "Evita archivos individuales >512KB."
        ]
    }

# ---- GitHub PR comment ----
def post_pr_comment(repo: str, pr_number: str, body: str) -> None:
    token = os.getenv("GITHUB_TOKEN")
    if not token or not pr_number:
        return
    import requests
    url = f"https://api.github.com/repos/{repo}/issues/{pr_number}/comments"
    headers = {"Authorization": f"Bearer {token}", "Accept": "application/vnd.github+json"}
    resp = requests.post(url, headers=headers, json={"body": body}, timeout=20)
    resp.raise_for_status()

# ---- main ----
def main():
    repo = os.getenv("GITHUB_REPOSITORY", "")
    event = os.getenv("GITHUB_EVENT_NAME", "")
    pr_number = os.getenv("PR_NUMBER") or os.getenv("GITHUB_REF_NAME") or os.getenv("GITHUB_REF") or ""
    config_path = ".agent/config.yaml"

    try:
        cfg = load_config(config_path)
    except Exception:
        cfg = {"checks": {"markdown_links": True, "secret_scan": True, "file_limits": {"max_single_file_kb": 512}},
               "outputs": {"artifact_path": ".agent/agent_report.json"},
               "model": {"name": "gpt-4.1-mini", "temperature": 0.2}}

    base_ref = get_base_ref()
    files = changed_files(base_ref)
    file_limit_kb = int(cfg.get("checks", {}).get("file_limits", {}).get("max_single_file_kb", 512))

    findings = []
    md_issues_total, secret_issues_total = [], []

    for p in files:
        text = read_text(p, file_limit_kb)
        if cfg.get("checks", {}).get("markdown_links", False) and p.lower().endswith((".md", ".markdown")):
            md_issues_total += [f"{p}: {msg}" for msg in check_markdown_links(text)]
        if cfg.get("checks", {}).get("secret_scan", False):
            sec = secret_scan(text)
            if sec:
                secret_issues_total += [f"{p}: {m}" for m in sec]

    # Prompt para el LLM
    prompt = {
        "repo": repo,
        "event": event,
        "pr_number": pr_number,
        "changed_files": files[:200],
        "checks_summary": {
            "markdown_link_issues": md_issues_total[:50],
            "secret_scan_issues": secret_issues_total[:50]
        }
    }
    model_cfg = cfg.get("model", {})
    llm = llm_summarize_and_suggest(model_cfg, json.dumps(prompt, ensure_ascii=False))

    report = {
        "event": event,
        "repo": repo,
        "pr": pr_number,
        "changed_files": files,
        "summary": llm.get("summary", ""),
        "suggestions": llm.get("suggestions", []),
        "risks": ["Posibles enlaces rotos" if md_issues_total else None,
                  "Posibles secretos expuestos" if secret_issues_total else None],
        "checks": {
            "markdown_link_issues": md_issues_total,
            "secret_scan_issues": secret_issues_total
        }
    }
    report["risks"] = [r for r in report["risks"] if r]

    Path(".agent").mkdir(parents=True, exist_ok=True)
    artifact = cfg.get("outputs", {}).get("artifact_path", ".agent/agent_report.json")
    Path(artifact).write_text(json.dumps(report, indent=2, ensure_ascii=False), encoding="utf-8")

    # Comentario en PR
    body = (
        "## 🤖 Resumen del agente (MVP)\n"
        f"- Cambios detectados: `{len(files)}` archivos\n"
        f"- Riesgos: {', '.join(report['risks']) if report['risks'] else 'ninguno detectado'}\n\n"
        f"**Resumen:** {report['summary']}\n\n"
        "**Sugerencias:**\n" + "\n".join([f"- {s}" for s in report["suggestions"]]) + "\n\n"
        "**Hallazgos de checks:**\n"
        f"- Enlaces Markdown: {len(md_issues_total)} potencial(es)\n"
        f"- Secretos: {len(secret_issues_total)} potencial(es)\n"
        "_Artefacto_: `.agent/agent_report.json`"
    )
    try:
        post_pr_comment(repo, os.getenv("GITHUB_REF_NAME") or os.getenv("GITHUB_HEAD_REF") or os.getenv("PR_NUMBER") or "", body)
    except Exception:
        # Intento alterno si pr_number llegó vacío desde workflow:
        gh_pr = os.getenv("GITHUB_EVENT_PATH")
        if gh_pr and Path(gh_pr).exists():
            try:
                ev = json.loads(Path(gh_pr).read_text(encoding="utf-8"))
                num = str(ev.get("pull_request", {}).get("number", "")) if ev.get("pull_request") else ""
                if num:
                    post_pr_comment(repo, num, body)
            except Exception:
                pass

    print(f"Reporte generado en {artifact}")

if __name__ == "__main__":
    main()

src/placeholder.txt

Este archivo asegura que la carpeta exista. Reemplázalo con código real.

docs/decisions.md

# ADR-0001: Activar agente de revisión en PR (MVP)
- Contexto: Necesitamos validar el circuito agente→comentario→aprobación humana.
- Decisión: Un solo workflow CI con checks básicos y resumen del agente.
- Consecuencias: Rápida validación; complejidad baja; listo para iterar.

CONTRIBUTING.md

## Flujo
1. Crea rama `feature/<descriptor>`.
2. Commits pequeños.
3. Abre PR hacia `main`. Espera comentario del agente.
4. Revisa checklist. Un humano aprueba y hace merge.

## Estilo de commits
- `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`…

CODE_OF_CONDUCT.md

Contribuye con respeto, claridad y responsabilidad.


---

Reasoning: Se simplificó el README, se endurecieron permisos de Actions, se añadió esqueleto real para comentario en PR, y se parametrizó el modelo para Codecs/StartBuilding. El script funciona sin red y deja un punto único para integrar la llamada al modelo.
Confidence: Alta para estructura y CI; media para el SDK concreto que uses en Codecs/StartBuilding.