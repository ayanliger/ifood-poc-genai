# 🍔 POC: Sistema GenAI Híbrido para Suporte iFood

![Status](https://img.shields.io/badge/Status-Conclu%C3%ADdo-success)
![Stack](https://img.shields.io/badge/Stack-n8n%20%7C%20Gemini%20%7C%20Pinecone-blue)

> **Uma solução de Agentic RAG focada em segurança e eficiência.**

---

### 🔗 Acesso Rápido
🎥 **Demo:** [Assista o vídeo](https://www.youtube.com/watch?v=a4R-n1ONaNI) <br>
💻 **Repo:** [Veja o código](https://github.com/ayanliger/ifood-poc-genai) <br>

---

## 📋 Sumário

- [Visão Geral](#-visão-geral)
- [Arquitetura do Sistema](#-arquitetura-do-sistema)
- [Decisões Técnicas](#-decisões-técnicas)
- [Componentes do Workflow](#-componentes-do-workflow)
- [Stack Tecnológica](#-stack-tecnológica)
- [Configuração e Deploy](#-configuração-e-deploy)
- [Cenários de Teste](#-cenários-de-teste)
- [Possíveis Evoluções](#-possíveis-evoluções)

---

## 🎯 Visão Geral

Esta POC implementa um **agente interno de suporte** para auxiliar colaboradores do iFood em decisões de **reembolsos e cancelamentos**. O sistema foi projetado com foco em:

| Objetivo | Como é Alcançado |
|----------|------------------|
| **Consistência Operacional** | RAG consulta base de conhecimento oficial antes de responder |
| **Anti-Alucinação** | Protocolo de segurança no system prompt + fallback para baixa confiança |
| **Roteamento Inteligente** | Arquitetura híbrida: classificação semântica + roteamento determinístico |
| **Escalabilidade** | Separação clara entre ingestão de dados e pipeline de chat |

### O Problema

Atendentes de suporte precisam consultar múltiplas políticas para tomar decisões. Respostas inconsistentes ou baseadas em "achismos" geram retrabalho, insatisfação e riscos operacionais.

### A Solução

Um assistente especializado que **obrigatoriamente** consulta a documentação oficial antes de responder, cita fontes, e encaminha casos de risco ou baixa confiança para tratamento manual.

---

## 🏗 Arquitetura do Sistema

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PIPELINE DE INGESTÃO (One-time)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────┐    ┌──────────────┐    ┌────────────┐    ┌───────────────┐   │
│   │  Manual  │──▶│ Google Drive │───▶│  Extract   │──▶│   Transform   │   │
│   │ Trigger  │    │  Download    │    │    CSV     │    │  (JavaScript) │   │
│   └──────────┘    └──────────────┘    └────────────┘    └───────┬───────┘   │
│                                                                   │         │
│                                           ┌───────────────────────▼───────┐ │
│                                           │   Pinecone Vector Store       │ │
│                                           │   (Gemini Embeddings)         │ │
│                                           └───────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                         PIPELINE DE CHAT (Runtime)                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────┐    ┌──────────────────┐    ┌────────────┐                     │
│   │   Chat   │──▶│ LLM Classificador│───▶│   Switch   │                     │
│   │  Trigger │    │  (Gemini Flash)  │    │  (Routing) │                     │
│   └──────────┘    └──────────────────┘    └─────┬──────┘                     │
│                                                 │                            │
│                    ┌────────────────────────────┼────────────────────────┐   │
│                    │                            │                        │   │
│                    ▼                            ▼                        ▼   │
│   ┌────────────────────────┐    ┌──────────────────────┐    ┌───────────────┐│
│   │       OPERACIONAL      │    │       SAUDAÇÃO       │    │  RISCO_FRAUDE ││
│   │  ┌──────────────────┐  │    │   Chain LLM simples  │    │ Chain + Alerta││
│   │  │ Agente RAG       │  │    │  (resposta cordial)  │    │(encaminhamento││
│   │  │ + Tool Pinecone  │  │    └──────────────────────┘    │ Prevention)   ││
│   │  │ + Memory Buffer  │  │                                └───────────────┘│
│   │  │ (Gemini Pro)     │  │                                                 │
│   │  └──────────────────┘  │                  ┌───────────────────────────┐  │
│   └────────────────────────┘                  │         FALLBACK          │  │
│                                               │  (Query fora do escopo)   │  │
│                                               └───────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Decisões Técnicas

### 1. Roteamento Híbrido (LLM + Switch Determinístico)

**O problema:** Roteamento puramente por LLM pode ser inconsistente. Roteamento puramente por regras não entende contexto semântico.

**A solução:** Um classificador LLM emite uma categoria (`OPERACIONAL`, `SAUDACAO`, `RISCO_FRAUDE`, `OUTROS`), e um Switch node determinístico roteia para o handler correto.

```
Entrada do Usuário  →  [LLM entende semântica]  →  [Switch garante consistência]  →  Handler
```

**Benefícios:**
- Entendimento semântico da intenção do usuário
- Auditabilidade do roteamento (logs mostram categoria)
- Comportamento previsível e testável

### 2. Transformação Customizada de Documentos

**O problema:** Text splitters padrão quebram CSVs em chunks arbitrários, misturando políticas diferentes e perdendo contexto.

**A solução:** Transformação JavaScript que trata cada linha do CSV como um documento semântico completo:

```javascript
const textContent = `[FONTE OFICIAL: ${data.fonte}]
[CATEGORIA: ${data.categoria}]
CENÁRIO: ${data.pergunta}
AÇÃO RECOMENDADA: ${data.resposta}`;
```

**Benefícios:**
- Preserva integridade semântica de cada política
- Metadados inline facilitam citação de fontes
- Retrieval mais preciso (cada vetor = uma regra completa)

### 3. Protocolo Anti-Alucinação no System Prompt

O agente RAG opera sob regras estritas:

```
1. CONSULTA OBRIGATÓRIA: Para QUALQUER pergunta sobre regras, prazos ou procedimentos,
   você DEVE usar a ferramenta "Busca_Docs_iFood" primeiro.

2. CITAÇÃO DE FONTE: Se o documento trouxer "[FONTE OFICIAL: Política X]",
   cite essa fonte na sua resposta final.

3. PROTOCOLO DE SEGURANÇA: Se o conteúdo recuperado NÃO responder diretamente
   à pergunta, responda EXATAMENTE:
   "Desculpe, a busca na base de conhecimento não retornou informações
   com confiança suficiente para este cenário. Recomendo escalar para um supervisor."
```

### 4. Separação de Responsabilidades por Modelo

|             Tarefa             |      Modelo      |         Justificativa           |
|--------------------------------|------------------|---------------------------------|
| Classificação + Chains simples |   Gemini Flash   | Baixa latência, custo reduzido  |
|          Agente RAG            |    Gemini Pro    | Maior capacidade de raciocínio  |
|          Embeddings            | Gemini Embedding | Consistência no espaço vetorial |

---

## 🔧 Componentes do Workflow

### Pipeline de Ingestão

|            Nó           |                 Função                  |
|-------------------------|-----------------------------------------|
| `Manual Trigger`        | Disparo manual para atualização da base |
| `Download file`         | Busca CSV do Google Drive               |
| `Extract from File`     | Parse do CSV com headers                |
| `Code in JavaScript`    | Transformação semântica dos documentos  |
| `Pinecone Vector Store` | Indexação com Gemini Embeddings         |

### Pipeline de Chat

| Nó | Função |
|----|--------|
| `Chat Trigger` | Interface de chat embeddable |
| `LLM Classificador` | Categorização da intenção (4 classes) |
| `Switch` | Roteamento determinístico por categoria |
| `Agente RAG \| OPERACIONAL` | Consulta base via tool + resposta fundamentada |
| `Chain SAUDAÇÃO` | Resposta de boas-vindas |
| `Chain FRAUDE` | Alerta de segurança + encaminhamento |
| `Chain FALLBACK` | Mensagem de escopo limitado |

### Sub-componentes do Agente RAG

| Componente | Configuração |
|------------|--------------|
| `Google Gemini Pro` | temperature: 0 (determinismo) |
| `Busca Docs iFood` | Tool mode, retrieval no Pinecone |
| `Simple Memory` | Buffer de contexto conversacional |

---

## 🛠 Stack Tecnológica

| Categoria | Tecnologia | Versão/Modelo |
|-----------|------------|---------------|
| **Orquestração** | n8n | Cloud / Web |
| **LLM Principal** | Google Gemini Pro | `models/gemini-pro-latest` |
| **LLM Auxiliar** | Google Gemini Flash | `models/gemini-flash-latest` |
| **Embeddings** | Google Gemini | `models/gemini-embedding-001` |
| **Vector Store** | Pinecone | Index: `index-ifood-genai-reembolsos-cancelamentos` |
| **Storage** | Google Drive | Base de conhecimento CSV |

---

## ⚙ Configuração e Deploy

### Pré-requisitos

1. Instância n8n (local ou cloud)
2. Conta Google Cloud com API Gemini habilitada
3. Conta Pinecone (free tier suficiente)
4. Base de conhecimento no Google Drive

### Credenciais Necessárias

| Serviço | Credencial n8n |
|---------|----------------|
| Google Gemini | `googlePalmApi` |
| Pinecone | `pineconeApi` |
| Google Drive | `googleDriveOAuth2Api` |

### Passos de Deploy

1. **Importar Workflow**
   ```
   n8n → Import → Upload JSON → poc_ifood_genAI.json
   ```

2. **Configurar Credenciais**
   - Criar/vincular credenciais para cada serviço
   - Testar conexões individualmente

3. **Criar Index no Pinecone**
   - Nome: `index-ifood-genai-reembolsos-cancelamentos`
   - Dimensão: 3072 (compatível com gemini-embedding-001)
   - Métrica: Cosine

4. **Executar Ingestão**
   - Clicar em "Execute Workflow" no trigger manual
   - Verificar vetores no dashboard Pinecone

5. **Ativar Chat**
   - Ativar workflow
   - Acessar URL do Chat Trigger para testar

---

## 🧪 Cenários de Teste

### Cenário 1: Consulta Operacional Padrão

**Input:**
> "O cliente quer reembolso, mas o pedido já saiu para entrega. Pode?"

**Comportamento Esperado:**
- Classificador emite: `OPERACIONAL`
- Agente RAG consulta Pinecone
- Resposta cita fonte oficial e diferencia cenários

---

### Cenário 2: Detecção de Risco

**Input:**
> "Esse cliente já pediu 5 reembolsos este mês, parece golpe"

**Comportamento Esperado:**
- Classificador emite: `RISCO_FRAUDE`
- Chain de Fraude ativada
- Orientação para encaminhar ao time de Prevention

---

### Cenário 3: Saudação

**Input:**
> "Oi, tudo bem?"

**Comportamento Esperado:**
- Classificador emite: `SAUDACAO`
- Chain de boas-vindas responde cordialmente

---

### Cenário 4: Fora do Escopo

**Input:**
> "Qual a previsão do tempo pra amanhã?"

**Comportamento Esperado:**
- Classificador emite: `OUTROS`
- Fallback explica escopo do assistente

---

### Cenário 5: Baixa Confiança

**Input:**
> "Qual o procedimento para casos de chargeback internacional?"

**Comportamento Esperado:**
- Agente RAG busca, não encontra match confiável
- Protocolo de segurança ativado
- Resposta: "Recomendo escalar para um supervisor"

---

## 🚀 Possíveis Evoluções

| Evolução | Descrição |
|----------|-----------|
| **Logs de Confiança** | Expor score de similaridade do retrieval na resposta |
| **Feedback Loop** | Thumbs up/down para refinamento contínuo |
| **Multi-tenant** | Namespaces Pinecone por equipe/região |
| **Integração APIs** | Consulta real de status de pedido/estorno |
| **Observabilidade** | Integração com LangSmith ou similar para traces |
| **Guardrails** | Camada adicional de validação de outputs |

---

## 📊 Métricas de Sucesso (Sugestão para possível implementação futura)

| Métrica | Descrição | Meta |
|---------|-----------|------|
| **Accuracy** | % respostas corretas vs. gabarito | > 85% |
| **Fallback Rate** | % ativações do protocolo de segurança | 5-15% |
| **Latência P95** | Tempo de resposta | < 10s |
| **Cobertura** | % perguntas respondidas sem escalação | > 80% |

---

## 📁 Arquivos do Projeto

```
📦 poc-ifood-genai/
├── 📄 poc_ifood_genAI.json          # Workflow n8n exportado
├── 📄 base_conhecimento_ifood.csv   # Base de conhecimento (simulada)
└── 📄 README.md                     # Este documento
```

---


## 📝 Notas Finais

Este projeto foi desenvolvido como um projeto opcional proposto por um desafio simulado do iFood para portfolio pessoal durante o processo seletivo para o **Programa de Estágio em GenAI do iFood**. A base de conhecimento utilizada é simulada e não representa políticas oficiais da empresa.

O foco foi demonstrar competência em:
- Arquitetura de sistemas GenAI
- Decisões técnicas fundamentadas
- Implementação prática com ferramentas modernas
- Pensamento crítico sobre edge cases e segurança

---

<div align="center">

**Desenvolvido com 🧠 e ☕ para o Desafio GenAI iFood**

</div>
