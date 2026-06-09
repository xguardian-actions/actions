<div align="center">
  <img src="XGuardian.png" alt="XGuardian Logo" width="200"/>

  # XGuardian Security Action

  **Análises de segurança automatizadas direto do seu pipeline GitHub Actions**

  ![Versão](https://img.shields.io/badge/versão-v26.6.1-purple)
  ![GitHub Marketplace](https://img.shields.io/badge/GitHub-Marketplace-blue?logo=github)
  ![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-2088FF?logo=github-actions&logoColor=white)
</div>

---

## Tipos de scan suportados

| Tipo | Mecanismo | Credencial na plataforma |
|------|-----------|--------------------------|
| **SAST** | Nativo XGuardian — análise estática de código-fonte | PAT do XGuardian |
| **SCA** | Nativo XGuardian — análise de dependências (roda junto com SAST) | PAT do XGuardian |
| **DAST Web** | Nativo XGuardian — análise dinâmica de apps web com login por formulário | PAT do XGuardian |
| **DAST API** | Nativo XGuardian — análise dinâmica de APIs REST autenticadas via Bearer token | PAT do XGuardian |
| **Container** | Nativo XGuardian — scan de imagens de container (arquivo `.tar` via Grype) | PAT do XGuardian |

> **SAST + SCA** rodam juntos no mesmo job (upload ZIP único). **DAST Web**, **DAST API** e **Container** são sempre chamadas separadas.

---

## Tópicos

- [Requisitos do sistema](#requisitos-do-sistema)
- [Credenciais e Secrets](#credenciais-e-secrets)
- [Parâmetros de configuração](#parâmetros-de-configuração)
- [Exemplos de uso](#exemplos-de-uso)
- [Outputs disponíveis](#outputs-disponíveis)
- [Integração com notificações](#integração-com-notificações)
- [Perguntas Frequentes](#perguntas-frequentes)
- [Suporte](#suporte)

---

## Requisitos do sistema

- **Runner:** Ubuntu (`ubuntu-latest`)
- **Dependências:** `curl`, `jq`, `zip` — instaladas automaticamente durante a execução
- **Permissões:** acesso de leitura ao código do repositório
- **Tempo de execução:** variável conforme tamanho do projeto e tipos de scan habilitados

---

## Credenciais e Secrets

### Autenticação na plataforma XGuardian (todos os scan types)

| Secret | Descrição | Obrigatório |
|--------|-----------|:-----------:|
| `XGUARDIAN_TOKEN` | PAT (Personal Access Token) gerado na plataforma XGuardian | ✅ |

Gere o PAT nas configurações da sua conta no XGuardian e armazene como `XGUARDIAN_TOKEN` nos secrets do GitHub.

### Autenticação no alvo (somente DAST)

Esses campos são as credenciais do **site ou API que será escaneado** — não da plataforma XGuardian.

**DAST Web** — autenticação por formulário de login:

| Secret | Descrição | Quando usar |
|--------|-----------|-------------|
| `SITE_USER` | Usuário para login no site alvo | Quando `auth_exist: true` |
| `SITE_PASSWORD` | Senha para login no site alvo | Quando `auth_exist: true` |

**DAST API** — autenticação via Bearer token:

| Secret | Descrição | Quando usar |
|--------|-----------|-------------|
| `API_BEARER_TOKEN` | Bearer token de autenticação da API alvo | Quando o alvo é uma API REST autenticada |

Para adicionar secrets ao seu repositório: **Settings → Secrets and variables → Actions → New repository secret**

---

## Parâmetros de configuração

### Configurações da Aplicação

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `app_name` | Nome da aplicação no XGuardian | — |
| `team_id` | ID(s) da(s) equipe(s) no formato JSON | `[1]` |
| `languages` | Linguagens no formato JSON array | `["JavaScript"]` |
| `description` | Descrição da aplicação | `"Aplicação criada através do GitHub Actions - XGuardian"` |

### SAST

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `sast` | Ativa o scan SAST | `"false"` |
| `policy_sast` | ID da política SAST | `"0"` |

### SCA

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `sca` | Ativa o scan SCA | `"false"` |
| `policy_sca` | ID da política SCA | `"0"` |

### DAST

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `dast` | Ativa o scan DAST | `"false"` |
| `policy_dast` | ID da política DAST | `"0"` |
| `site_url` | URL completa do site ou API a ser analisada | `""` |

**DAST Web** — autenticação por formulário de login:

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `auth_url` | URL da página de login | `""` |
| `logout_url` | URL da página de logout | `""` |
| `auth_exist` | Indica se o site requer autenticação por formulário | `false` |
| `user_login` | Usuário para login no site alvo | `""` |
| `password_login` | Senha para login no site alvo | `""` |

**DAST API** — autenticação via Bearer token:

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `api_token` | Bearer token de autenticação da API alvo | `""` |

### Container

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `container` | Ativa o scan Container | `"false"` |
| `container_image` | Nome e tag da imagem Docker a ser escaneada. Ex: `minha-app:latest`. A action executa `docker save` internamente e faz o upload do `.tar` para o XGuardian. | `""` |

### Configurações Adicionais

| Parâmetro | Descrição | Padrão |
|-----------|-----------|--------|
| `translate` | Traduz o relatório para português brasileiro | `"false"` |
| `exclude` | Diretórios/arquivos a excluir do scan | `""` |
| `pdf` | Gera relatório PDF detalhado | `"false"` |
| `scan_directory` | Diretório específico para análise | `"."` |
| `get_scan_id` | Busca o ID do scan após o upload | `"false"` |
| `save_vulns` | Salva vulnerabilidades no banco | `"false"` |
| `pipeaction` | Ação na pipeline quando vulnerabilidades encontradas (`warn`, `fail`, `noAction`) | `"noAction"` |
| `is_development` | Usa URLs do ambiente de desenvolvimento | `"false"` |

---

## Exemplos de uso

### 1. SAST + SCA

```yaml
name: XGuardian — SAST + SCA
on:
  push:
    branches: [main]

jobs:
  code-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: XGuardian SAST + SCA
        uses: xguardian-actions/actions@main
        with:
          token: ${{ secrets.XGUARDIAN_TOKEN }}
          app_name: ${{ github.event.repository.name }}
          team_id: "[1]"
          languages: '["JavaScript", "TypeScript"]'
          sast: "true"
          sca: "true"
          get_scan_id: "true"
```

### 2. DAST Web (autenticação por formulário)

```yaml
name: XGuardian — DAST Web
on:
  schedule:
    - cron: "0 2 * * 1"  # toda segunda às 02h

jobs:
  dast-web:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: XGuardian DAST Web
        uses: xguardian-actions/actions@main
        with:
          token: ${{ secrets.XGUARDIAN_TOKEN }}
          app_name: "meu-website"
          dast: "true"
          site_url: "https://meu-site.com"
          auth_url: "https://meu-site.com/login"
          logout_url: "https://meu-site.com/logout"
          auth_exist: true
          user_login: ${{ secrets.SITE_USER }}
          password_login: ${{ secrets.SITE_PASSWORD }}
          get_scan_id: "true"
```

### 3. DAST API (Bearer token)

```yaml
name: XGuardian — DAST API
on:
  schedule:
    - cron: "0 3 * * 1"  # toda segunda às 03h

jobs:
  dast-api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: XGuardian DAST API
        uses: xguardian-actions/actions@main
        with:
          token: ${{ secrets.XGUARDIAN_TOKEN }}
          app_name: "minha-api"
          dast: "true"
          site_url: "https://api.meu-site.com"
          api_token: ${{ secrets.API_BEARER_TOKEN }}
          get_scan_id: "true"
```

### 4. Container scan

```yaml
name: XGuardian — Container
on:
  push:
    branches: [main]

jobs:
  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build da imagem Docker
        run: docker build -t minha-app:${{ github.sha }} .

      - name: XGuardian Container Scan
        uses: xguardian-actions/actions@main
        with:
          token: ${{ secrets.XGUARDIAN_TOKEN }}
          app_name: ${{ github.event.repository.name }}
          container: "true"
          container_image: "minha-app:${{ github.sha }}"
          get_scan_id: "true"
```

> A action executa `docker save minha-app:{sha} -o minha-app.tar` internamente e faz o upload do `.tar` para o XGuardian via S3.

### 5. Security gate em Pull Request

```yaml
name: XGuardian — PR Security Gate
on:
  pull_request:
    branches: [main, develop]
    paths-ignore:
      - "**.md"
      - "docs/**"

jobs:
  security-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: XGuardian SAST + SCA
        id: xguardian
        uses: xguardian-actions/actions@main
        with:
          token: ${{ secrets.XGUARDIAN_TOKEN }}
          app_name: ${{ github.event.repository.name }}
          sast: "true"
          sca: "true"
          pipeaction: "warn"
          get_scan_id: "true"

      - name: Comentar resultado no PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Análise de Segurança XGuardian

              - **App ID:** ${{ steps.xguardian.outputs.app_id }}
              - **Scan ID:** ${{ steps.xguardian.outputs.scan_id }}
              - **Versão:** ${{ steps.xguardian.outputs.scan_version }}

              [Ver resultados completos](${{ steps.xguardian.outputs.scan_url }})`
            })
```

### 6. Pipeline completo

```yaml
name: XGuardian — Pipeline Completo
on:
  push:
    branches: [main]

jobs:
  code-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: XGuardian SAST + SCA
        uses: xguardian-actions/actions@main
        with:
          token: ${{ secrets.XGUARDIAN_TOKEN }}
          app_name: ${{ github.event.repository.name }}
          sast: "true"
          sca: "true"
          get_scan_id: "true"

  container-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: XGuardian Container Scan
        uses: xguardian-actions/actions@main
        with:
          token: ${{ secrets.XGUARDIAN_TOKEN }}
          app_name: ${{ github.event.repository.name }}
          container: "true"
          container_image_url: "ghcr.io/${{ github.repository }}:latest"
          get_scan_id: "true"
```

---

## Outputs disponíveis

| Output | Descrição | Disponibilidade |
|--------|-----------|-----------------|
| `app_id` | ID da aplicação no XGuardian | Quando `get_scan_id: "true"` |
| `scan_id` | ID do scan executado | Quando `get_scan_id: "true"` |
| `scan_url` | URL para visualizar os resultados | Quando `get_scan_id: "true"` |
| `scan_version` | Nome + SHA do commit | Sempre disponível |

> Para acessar os outputs, adicione um `id` ao step e defina `get_scan_id: "true"`.

```yaml
- name: XGuardian Scan
  id: xguardian
  uses: xguardian-actions/actions@main
  with:
    # ...
    get_scan_id: "true"

- name: Usar outputs
  run: |
    echo "App ID: ${{ steps.xguardian.outputs.app_id }}"
    echo "Scan ID: ${{ steps.xguardian.outputs.scan_id }}"
    echo "Resultados: ${{ steps.xguardian.outputs.scan_url }}"
```

---

## Integração com notificações

### Microsoft Teams

```yaml
- name: Notificar no Microsoft Teams
  uses: aliencube/microsoft-teams-actions@v0.8.0
  with:
    webhook_uri: ${{ secrets.MS_TEAMS_WEBHOOK_URI }}
    title: "Scan de Segurança XGuardian"
    summary: "Resultados do scan concluído"
    theme_color: "0076D7"
    sections: |
      [{
        "activityTitle": "Scan Finalizado",
        "activitySubtitle": "${{ github.repository }} - ${{ github.ref_name }}",
        "facts": [
          { "name": "Status", "value": "${{ job.status }}" },
          { "name": "App ID", "value": "${{ steps.xguardian.outputs.app_id }}" },
          { "name": "Scan ID", "value": "${{ steps.xguardian.outputs.scan_id }}" }
        ],
        "potentialAction": [{
          "@type": "OpenUri",
          "name": "Ver Resultados",
          "targets": [{ "os": "default", "uri": "${{ steps.xguardian.outputs.scan_url }}" }]
        }]
      }]
```

### Slack

```yaml
- name: Notificar no Slack
  uses: rtCamp/action-slack-notify@v2
  env:
    SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
    SLACK_TITLE: "XGuardian Security Action"
    SLACK_MESSAGE: |
      *Scan concluído*
      • Repositório: ${{ github.repository }}
      • Branch: ${{ github.ref_name }}
      • Resultados: ${{ steps.xguardian.outputs.scan_url }}
    SLACK_COLOR: ${{ job.status == 'success' && 'good' || 'danger' }}
```

---

## Perguntas Frequentes

### Quanto tempo leva um scan?

| Tipo | Tempo estimado |
|------|----------------|
| SAST | 2–10 minutos |
| SCA | 1–5 minutos |
| DAST Web | 10–60 minutos |
| DAST API | 5–30 minutos |
| Container | 5–20 minutos |

### Posso usar em repositórios privados?

Sim. O XGuardian Security Action funciona em repositórios públicos e privados. As credenciais são transmitidas de forma segura via secrets do GitHub.

### O que fazer se o scan falhar?

1. Verifique os logs do workflow para identificar o step exato da falha
2. Problemas comuns:
   - **PAT inválido ou expirado:** gere um novo token nas configurações do XGuardian
   - **URL DAST inacessível:** confirme que o site/API está no ar e acessível pelo runner
   - **Credenciais do alvo inválidas:** verifique `user_login`/`password_login` (DAST Web) ou `api_token` (DAST API)
   - **container_image_url inválida:** verifique o formato e a acessibilidade da imagem
   - **Timeout no scan ID:** o scan pode ainda estar em processamento
3. Se o problema persistir, abra uma issue no repositório

### Como faço para o build falhar quando há vulnerabilidades críticas?

```yaml
with:
  pipeaction: "fail"
```

### Posso combinar SAST, SCA e Container no mesmo job?

SAST + SCA rodam juntos (mesmo upload ZIP). Container é uma chamada separada e deve ser um job independente — conforme o [Exemplo 6 — Pipeline completo](#6-pipeline-completo).

---

## Modo de desenvolvimento

Para testar com o ambiente de desenvolvimento do XGuardian:

```yaml
with:
  is_development: "true"
```

---

## Suporte

Para dúvidas ou problemas, entre em contato com a equipe XGuardian:

- **Email:** suporte@xmartsolutions.com
- **Issues:** [github.com/xguardian-actions/actions/issues](https://github.com/xguardian-actions/actions/issues)
