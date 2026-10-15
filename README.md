# 🤖 Automação Multiagentes com n8n para Corretora de Seguros e Consórcios

Este repositório contém a documentação e os guias de arquitetura para a implementação de um **Sistema Multiagentes de Atendimento Omnichannel**, orquestrado no **n8n** e alimentado por modelos locais via **Ollama** (*Llama 3.2 3B* e *BGE-M3*), banco de dados vetorial **PostgreSQL + pgvector**, cache **Redis** e transbordo humano via **Chatwoot**.

---

## 📌 1. Tema e Objetivo do Notebook

* **Tema**: Arquitetura, orquestração, engenharia de contexto e otimização de sistemas multiagentes com n8n em infraestrutura *self-hosted*.
* **Objetivo**: Projetar e implementar uma arquitetura de atendimento automatizado para corretoras de seguros e consórcios (automação `WF-001-INBOUND`), definindo o papel do Agente Supervisor e de subagentes especializados (Seguros, Consórcios, Sinistros, Reboque 24h, 2ª Via, RAG), estratégias de memória de curto/longo prazo, buffer de mensagens para o WhatsApp, transcrição local de áudio com Whisper/Speaches, guardrails de segurança (LGPD/Prompt Injection) e transbordo humano *Human-in-the-Loop*.

---

## 📚 2. Fontes Utilizadas e Justificativa de Confiabilidade

O projeto baseia-se em 34 fontes técnicas rigorosamente selecionadas e validadas:

1. **Documentação Oficial do n8n (`n8n Docs`, `AI Agent Nodes`, `Queue Mode`, `Host n8n`)**:
   * *Motivo da confiança*: Fonte primária mantida pelos desenvolvedores da plataforma, garantindo precisão quanto às APIs, nó `AI Agent`, padrões de fila com Redis e suporte a sub-workflows.
2. **Documentação Oficial do PostgreSQL & pgvector**:
   * *Motivo da confiança*: Especificações oficiais da extensão vetorial open-source, fundamentando a criação de índices HNSW e buscas por similaridade de cosseno.
3. **Artigos Técnicos e Estudos de Caso de Engenharia (n8n Blog, HatchWorks AI, Chronexa, ZenML, Medium/Towards AI)**:
   * *Motivo da confiança*: Análises de produção sobre padrões de orquestração (*Supervisor-Worker*, *Routing by Branch*), resiliência com *Auto-fixing Parsers* e otimização de custos e latência para LLMs.
4. **Diagrama de Arquitetura (`Mermaid-preview.png`) e Especificação de Seguros em Markdown**:
   * *Motivo da confiança*: Arquitetura de referência do sistema inbound da corretora, mapeando detalhadamente cada etapa da pipeline de dados.

---

## 🎯 3. Diretrizes de Comportamento do Notebook

* **Fundamentação Estrita nas Fontes (*Source-Grounded*)**: Todas as soluções técnicas propostas são estritamente rastreáveis à documentação oficial e às boas práticas de engenharia de software fornecidas nas fontes.
* **Abordagem Defensiva para LLMs Locais**: Em vez de expor modelos compactos (como o *Llama 3.2 3B*) a loops complexos e instáveis de decisão ReAct, adota-se a classificação estruturada de intenção em JSON seguida de roteamento determinístico via nó *Switch Router* (limiar de confiança de 80%).
* **Segurança e Privacidade (*Self-Hosted & LGPD*)**: Foco em componentes 100% locais (Ollama, Speaches AI, PostgreSQL, Redis) para proteção de dados sensíveis de segurados, com sanitização de PII e guardrails perimetrais.
* **Resiliência e Idempotência**: Garantia de não-duplicação de mensagens via travas no Redis (`SETNX`) e tratamento grácil de falhas com fallback para atendimento humano.

---

## ❓ 4. Histórico de Perguntas, Respostas e Fontes

### 🟢 Pergunta 1: Visão Geral da Arquitetura e Componentes da Infraestrutura
* **O que foi respondido**: Detalhamento do fluxo inbound (`Mermaid-preview.png`), justificativa do uso do Supervisor Classificador com *Switch Router*, divisão da memória entre Redis (curto prazo/sessão) e PostgreSQL `pgvector` com índice HNSW (longo prazo/RAG), engenharia de contexto (Isolate, Select, Compress, Write) e otimização da infraestrutura Docker.
* **Fontes primárias**: `Mermaid-preview.png`, *"Arquitetura e Implementação de Sistemas Multi-Agentes com RAG no n8n..."*, *"AI Agent | Nodes | n8n Docs"*, *"GitHub - pgvector/pgvector"*, *"Enable queue mode | Deploy | n8n Docs"*.

---

### 🟢 Pergunta 2: Como Estruturar o Prompt do Agente Supervisor?
* **O que foi respondido**: Elaboração do prompt de sistema em 5 blocos (Papel, Taxonomia de Intenções, Regras de Avaliação de Confiança, Schema JSON Obrigatório e Exemplos *Few-Shot*). Recomendação de temperatura baixa (`0.0 - 0.1`) e validação do JSON via nó *Code* para direcionamento no *Switch Router*.
* **Fontes primárias**: *"n8n LLM Nodes: How to Get Reliable Structured..."*, *"Auto-fixing Output Parser | Nodes - n8n Docs"*, *"n8n AI Agent Node: Production Multi-Agent Patterns (2026)"*.

---

### 🟢 Pergunta 3: Como Implementar o Buffer de Mensagens?
* **O que foi respondido**: Estruturação de mecanismo para consolidar rajadas de mensagens fragmentadas no WhatsApp. Uso do Redis com `RPUSH` na chave `buffer:whatsapp:<id>`, trava de idempotência com `SETNX` na chave `lock:whatsapp:<id>`, pausa temporal de 5 a 15 segundos no nó *Wait*, e nó *Code* em JavaScript para concatenar os textos em um único payload.
* **Fontes primárias**: *"Message buffer system with Redis for efficient processing"*, *"Multi Agent Solutions in n8n for Reliable AI Agent Orchestration"*.

---

### 🟢 Pergunta 4: Como Tratar Mensagens de Áudio no n8n?
* **O que foi respondido**: Pipeline de conversão assíncrona: o nó *Switch* identifica mídia do tipo `audio`, baixa o binário e faz uma requisição HTTP POST multipart/form-data para o container `speaches` (`ghcr.io/speaches-ai/speaches:0.8.3-cpu` na rota `/v1/audio/transcriptions` usando Whisper em português). O texto transcrito é injetado no campo `message_text` e entra normalmente no buffer e no supervisor.
* **Fontes primárias**: *"Code integrations | Workflow automation with n8n"*, *"WhatsApp AI agent that understand text, image, audio"*.

---

### 🟢 Pergunta 5: Como Integrar o Chatwoot para Transbordo Humano?
* **O que foi respondido**: Arquitetura *Human-in-the-Loop* bidirecional. O transbordo é acionado por confiança inferior a 80%, solicitação expressa do usuário ou risco alto. O n8n define a flag `bot_active = false` no Redis, executa uma requisição HTTP para a API do Chatwoot inserindo etiquetas (ex: `#humano`), e bloqueia interações automáticas da IA no início do fluxo através de um nó *If*. Um Webhook do Chatwoot no evento `conversation_resolved` reativa o bot (`bot_active = true`).
* **Fontes primárias**: *"Webhook and Fluent Support: Automate Workflows with n8n"*, *"Set a human fallback for AI workflows (n8n Docs)"*, *"Arquitetura e Implementação de Sistemas Multi-Agentes com RAG..."*.

---

## 🔗 5. Link de Acesso Público ao Notebook

O notebook público contendo todas as fontes, arquivos e histórico interativo pode ser acessado através do link abaixo:

👉 **[Acessar Notebook Público no Gemini Notebook / NotebookLM]([https://notebooklm.google.com/](https://notebook.google.com/notebook/f80184cf-f7aa-4325-abd5-6a1e684d66bb?authuser=2))**

---

*Documentação gerada para apoio à implantação no repositório GitHub da automação.*
