Fluxo de automação e autocura (Self-Healing) com **Agentes AI** no GitHub. Guia dividindo no cenário **Pessoal (KIND Local)** e em uma **Organização**.

---

**Diferenças: Cenário de Estudo (Pessoal) vs. Real (Organização)**

| Funcionalidade | Cenário Estudo (Local) | Cenário Real (Organização) |
| --- | --- | --- |
| **Cluster Kubernetes** | **KIND (Kubernetes in Docker)** rodando localmente na sua máquina. | Cluster gerenciado na nuvem (**EKS, GKE, AKS**). |
| **Acesso ao Cluster** | O GitHub Actions não alcança seu `localhost` diretamente sem um túnel (ex: `ngrok`) ou um **Runner Auto-hospedado (Self-Hosted Runner)** rodando localmente. | O GitHub Actions conecta via **OIDC (OpenID Connect)** e IP público seguro/VPN do cluster cloud. |
| **Aprovação de Ações** | Feita manual na Issue ativando a action por comentário ou rótulo (Labels). | **GitHub Environments** com "Required Reviewers" bloqueando o deploy automaticamente. |
| **Gatilho de Observabilidade** | Script simulador ou cURL enviando um webhook para criar a Issue. | **Alertmanager/Datadog** integrado via Webhook abrindo a Issue automaticamente ao falhar um pod. |

---

### Passo a Passo de Execução e Códigos

#### Passo 1: Preparar o Cluster e o Runner

* **Pessoal / Estudo:** Como o GitHub Actions é executado nos servidores da Microsoft na nuvem, ele não consegue acessar o seu KIND local. A solução mais simples é configurar um **Self-Hosted Runner** no seu computador.
* Vá no repositório no GitHub: **Settings** > **Actions** > **Runners** > **New self-hosted runner**.
* Execute os comandos fornecidos pelo GitHub no seu terminal para registrar o runner localmente.


* **Organização:** Ignora essa etapa local. O cluster estará na nuvem e o GitHub usará os *GitHub-hosted runners* padrão configurados com autenticação AWS/GCP/Azure via OIDC.

---

#### Passo 2: Configurar Secrets e Credenciais

Como você está na tela de configurações do seu repositório:

1. Vá em **Settings** > **Secrets and variables** > **Actions** (não use a aba Dependabot).
2. Adicione as seguintes Secrets:
* **Pessoal / Estudo:**
* `KUBE_CONFIG`: O conteúdo base64 do seu file `~/.kube/config` (rode `cat ~/.kube/config | base64` no terminal para pegar).
* `OPENAI_API_KEY`: Sua chave da API da OpenAI (ou do LLM de sua preferência).
* `GH_PAT`: Um Personal Access Token do GitHub com escopo de `repo`.


* **Organização:**
* Configurado em nível de Organização (**Organization Secrets**) para compartilhar entre repositórios, utilizando OIDC / IAM roles em vez de tokens de longa duração.





---

#### Passo 3: O Agente de IA em Python

Crie um arquivo chamado `agent.py` na raiz do seu repositório de suporte. Esse script consultará uma base de conhecimento simples, analisará a causa raiz e decidirá se aplica a correção direta (Infra) ou se abre um PR/pede autorização.

```python
import os
import sys
import json
from openai import OpenAI
import subprocess

# Chaves de ambiente
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
ISSUE_BODY = os.getenv("ISSUE_BODY", "")
ISSUE_NUMBER = os.getenv("ISSUE_NUMBER")
REPO_NAME = os.getenv("GITHUB_REPOSITORY")

client = OpenAI(api_key=OPENAI_API_KEY)

# 1. Base de Conhecimento Simples (Simulação de Vector DB / RAG)
KNOWLEDGE_BASE = {
    "ImagePullBackOff": "Erro de imagem inexistente ou sem permissão. Correção: Ajustar a tag da imagem no Deployment.",
    "CrashLoopBackOff": "Erro de aplicação ao iniciar (ex: variável de ambiente faltando). Correção de infra: Ajustar ConfigMap ou Secret.",
    "OOMKilled": "Aplicação estourou o limite de memória. Correção de infra: Aumentar os limits do Kubernetes."
}

def analyze_issue(issue_text):
    prompt = f"""
    Você é um agente de DevOps/SRE. Analise o erro abaixo e a base de conhecimento.
    Base de conhecimento: {json.dumps(KNOWLEDGE_BASE)}
    Erro reportado: {issue_text}

    Responda no formato JSON com as chaves:
    - "categoria": "INFRA" ou "APLICACAO"
    - "gravidade": "ALTA" ou "BAIXA"
    - "diagnostico": "Explicação sucinta"
    - "acao_recomendada": "Comando ou mudança a fazer"
    """
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        response_format={"type": "json_object"}
    )
    return json.loads(response.choices[0].message.content)

def main():
    print("🤖 Agente SRE iniciando análise...")
    analysis = analyze_issue(ISSUE_BODY)
    print(f"Resultado: {analysis}")

    categoria = analysis.get("categoria")
    gravidade = analysis.get("gravidade")
    acao = analysis.get("acao_recomendada")
    diagnostico = analysis.get("diagnostico")

    # Decisão de Execução
    if categoria == "INFRA" and gravidade == "BAIXA":
        print("🛠️ Erro de Infraestrutura de baixa gravidade detectado. Aplicando correção automática no cluster...")
        # Exemplo: Aplicando fix via kubectl no cluster local
        # subprocess.run(["kubectl", "apply", "-f", "k8s/fix.yaml"])
        print(f"Sucesso! Ação executada: {acao}")
    else:
        print("⚠️ Ação requer aprovação manual ou ajuste no código da aplicação.")
        # Salva o resultado para o GitHub Actions ler e comentar na Issue
        with open("agent_output.txt", "w") as f:
            f.write(f"**Análise do Agente SRE:**\n- **Categoria:** {categoria}\n- **Gravidade:** {gravidade}\n- **Diagnóstico:** {diagnostico}\n- **Ação Sugerida:** {acao}\n\n*Por favor, responda com '/aprovar' para aplicar a solução.*")

if __name__ == "__main__":
    main()

```

---

#### Passo 4: O Workflow do GitHub Actions

Crie o arquivo `.github/workflows/sre-agent.yml`.

```yaml
name: SRE Auto-Healing Agent

on:
  issues:
    types: [opened, labeled]

jobs:
  analyze-and-heal:
    # Pessoal / Estudo: Usa o runner local rodando na sua máquina que acessa o KIND
    runs-on: self-hosted 
    
    # Organização: Usaria runners nativos do GitHub
    # runs-on: ubuntu-latest

    steps:
      - name: Checkout do Repositório
        uses: actions/checkout@v4

      - name: Configurar Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Instalar Dependências
        run: |
          pip install openai

      - name: Configurar Kubeconfig (Conexão ao KIND)
        env:
          KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
        run: |
          mkdir -p ~/.kube
          echo "$KUBE_CONFIG" | base64 -d > ~/.kube/config

      - name: Executar Agente SRE
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          ISSUE_BODY: ${{ github.event.issue.body }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          GITHUB_REPOSITORY: ${{ github.repository }}
        run: python agent.py

      - name: Comentar na Issue com o Diagnóstico
        if: always()
        uses: peter-evans/create-or-update-comment@v4
        with:
          issue-number: ${{ github.event.issue.number }}
          body-path: 'agent_output.txt'

```

---

#### Passo 5: Testando a Execução (Simulação)

1. **Abra uma Issue no Repositório:**
* **Título:** `Pod crashando no cluster KIND`
* **Corpo:** `O pod order-service apresentou o erro ImagePullBackOff após a última atualização.`


2. **Resultado Esperado:**
* O **Self-hosted Runner** detecta a Issue.
* O script `agent.py` consulta a OpenAI e identifica que é um problema de imagem (`APLICACAO` / `ALTA`).
* Como é um erro de aplicação/alta gravidade, ele não aplica mudanças de infra diretamente; em vez disso, comenta na Issue informando a causa e pedindo aprovação ou sugerindo o ajuste no repositório correspondente.
