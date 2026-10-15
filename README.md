# Arquitetura, Orquestração e Otimização de Sistemas Multiagentes no n8n

Este repositório contém a documentação completa, diretrizes arquiteturais e exemplos práticos para o projeto e implementação de **Sistemas Multiagentes de IA** utilizando o **n8n**, com foco em execução *self-hosted*, alta performance, baixo custo e controle rigoroso de alucinações.

---

## 1. Tema e Objetivo do Notebook

* **Tema:** Arquitetura, orquestração, engenharia de contexto, gestão de memória e otimização de inferência em sistemas multiagentes no n8n.
* **Objetivo:** Projetar e implementar uma infraestrutura modular onde um **Agente Orquestrador (Mestre)** coordena **Subagentes Especialistas** (SDR, Suporte RAG, Agendamento), gerenciando estados de memória de curto e longo prazo (Redis e PostgreSQL/pgvector), compartilhando contexto de forma isolada, integrando atendimento omnichannel via Chatwoot e garantindo o mínimo consumo de tokens com máxima precisão.

---

## 2. Fontes de Conhecimento e Justificativa de Confiabilidade

O conhecimento consolidado neste notebook provém de duas categorias fundamentais de fontes técnicas:

### A. Documentação Oficial e Mantenedores das Tecnologias
1. **n8n Official Docs** (*AI Agent Node, AI Examples, Integrate AI, Host n8n, Enable Queue Mode*): Documentação primária sobre o funcionamento dos nós nativos de IA, arquitetura de filas com Redis, hospedagem Docker e padrão `AI Agent Tool`.
2. **PostgreSQL & pgvector Documentation**: Especificações técnicas sobre tipos de dados vetoriais, criação de índices de busca de vizinhos mais próximos (HNSW/IVFFlat) e consultas de similaridade por cosseno.
3. **LangChain & LangGraph Docs** (*Multi-Agent Subagents Architecture*): Padrões de arquitetura mestre-trabalhador, isolamento de contexto (*context isolation*) e estratégias de ferramentas por subagente.
4. **Redis Documentation**: Melhores práticas para cache de mensagens, gerenciamento de sessões (`Redis Chat Memory`) e mensageria em alta velocidade.
5. **Context Engineering Docs**: Metodologias para seleção, compressão, isolamento e escrita de contexto para evitar estouro da janela de tokens em LLMs.

### B. Estudos de Caso Práticos e Comunidade de Desenvolvedores
1. **Tutoriais e Projetos do Ecossistema n8n em Vídeo** (*Vários Agentes no Atendimento, Multiagentes no n8n com Troca de Departamentos Dinâmica, Método AI Agent Tool*): Validações empíricas de arquiteturas em produção integrando WhatsApp (via Evolution API/Chatwoot), Supabase, Qdrant e n8n.
2. **n8n Community Forum**: Discussões técnicas sobre mitigação de não-determinismo, custos de tokens e comparação entre orquestração dinâmica e cadeias determinísticas em série.

> **Por que confiamos nestas fontes:** A combinação da documentação oficial (que garante a precisão sintática e estrutural das APIs) com implementações reais da comunidade (que revelam gargalos operacionais e cenários de borda) assegura que as recomendações sejam técnicas, robustas e diretamente aplicáveis.

---

## 3. Diretrizes de Comportamento do Notebook

O notebook opera sob um conjunto rigoroso de princípios arquiteturais e operacionais:

1. **Atuação como Arquiteco de IA:** Foco em soluções funcionais, seguras, sustentáveis e de custo otimizado, explicando a fundo o "porquê" de cada decisão técnica.
2. **Ancoragem Rigorosa nas Fontes (Grounding):** Todas as afirmações factuais e configurações de componentes são estritamente rastreáveis à documentação oficial ou a experimentos validados.
3. **Prevenção Ativa de Alucinações:**
   * Prompts estritos de limitação de escopo (*System Prompts* com cláusula de ausência de dados).
   * Delegação obrigatória para subagentes e ferramentas especializadas.
   * Regra de não alteração de dados do subagente pelo orquestrador.
4. **Minimização de Custos e Latência:** Preferência por roteamento em série quando o fluxo for determinístico, modelo híbrido de LLMs (modelos fortes para orquestração e modelos compactos locais para tarefas específicas) e uso de buffers para mensagens picadas.

---

## 4. Histórico de Interações, Respostas e Fontes Utilizadas

Abaixo está o resumo das perguntas realizadas no notebook, o conteúdo fundamentado das respostas e as fontes que respaldam cada etapa:

### Interação 1: Arquitetura, Orquestração e Otimização Multiagente no n8n
* **Pergunta do Usuário:** Como projetar e implementar arquiteturas multiagentes no n8n, definindo orquestrador/subagentes, memória (curto/longo prazo), RAG/pgvector, compartilhamento de contexto, segurança e otimização de tokens e latência?
* **Resumo da Resposta:** 
  * **Abordagens de Orquestração:** Comparação entre o método nativo com `AI Agent Tool`, sub-workflows com `Call n8n Workflow Tool` e roteamento sequencial/determinístico via `Text Classifier`.
  * **Papel das Peças:** Orquestrador como roteador sem estado próprio e subagentes como executores com escopo e ferramentas delimitados.
  * **Gerenciamento de Memória:** Uso do Redis para sessão de curto prazo, isolamento no `$fromAI()` e persistência de longo prazo em banco de dados para troca dinâmica de departamento.
  * **RAG & Vetores:** Estruturação de dados CSV/JSON, chunking preciso (500–800 tokens) e rotinas de *upsert* em bancos vetoriais.
  * **Otimização:** Uso de modelos híbridos (LLMs potentes no mestre, SLMs nos filhos), eliminação do multiplicador de tokens e buffer de mensagens no Redis.
* **Fontes Utilizadas:** *n8n Docs (AI Agent, AI Examples), LangChain Subagents Docs, Tutoriais de Multiagentes de IA no n8n, Discussões da Comunidade n8n*.

---

### Interação 2: Análise e Mapeamento da Stack Self-Hosted (Containers Docker)
* **Pergunta do Usuário:** Mapeamento e integração de uma lista de containers ativos no Docker (`n8n:2.40.5`, `chatwoot:v4.18.0-ce`, `redis:8.10.0-alpine`, `pgvector:pg18-trixie`, `ollama:0.33.2`, `speaches:0.8.3-cpu`, `nextcloud`, `traefik`, `portainer`).
* **Resumo da Resposta:**
  * Mapeamento detalhado de cada container na arquitetura (Chatwoot para omnichannel/handoff; Redis para buffer e Queue Mode; pgvector para estado e RAG; Ollama para inferência local com custo zero; Speaches para STT/TTS de áudios do WhatsApp).
  * Desenho do fluxo *End-to-End* unindo mensagens de texto e voz com transição automática para atendimento humano no Chatwoot.
  * Estratégia de buffer no Redis para agregar mensagens picadas enviadas em sequência pelo usuário antes de disparar a LLM.
* **Fontes Utilizadas:** *n8n Docs (Host n8n, Enable queue mode), PostgreSQL/pgvector Docs, Redis Docs, Vídeos de Atendimento E-commerce e Multi-Atendimento com Chatwoot*.

---

### Interação 3: Otimização para Modelos Locais (`llama3.2:3b` + `bge-m3`) e Prompts Antialucinação
* **Pergunta do Usuário:** Otimização para os modelos locais carregados no Ollama (`llama3.2:3b` para geração e `bge-m3:latest` para embeddings) e criação de *System Prompts* antialucinação para o orquestrador e subagentes.
* **Resumo da Resposta:**
  * Diretrizes de engenharia de prompt adequadas para SLMs de 3B parâmetros (prompts curtos, marcações estruturadas, controle estrito de saída).
  * Disponibilização de *System Prompts* completos para: **Agente Orquestrador Mestre**, **Subagente de Suporte RAG** e **Subagente Qualificador SDR**.
  * Configuração SQL da tabela no PostgreSQL com extensão `pgvector` ajustada para **1024 dimensões** (exigidas pelo `bge-m3`) e índice de alta velocidade `HNSW` (`vector_cosine_ops`).
* **Fontes Utilizadas:** *pgvector GitHub Docs, Context Engineering Docs, LangChain Subagents Architecture, n8n AI Agent Docs*.

---

## 5. Link do Notebook Compartilhado

Acesse o notebook público com todas as fontes, estruturas e componentes organizados:

👉 **[Clique aqui para aceder ao Notebook Público no Gemini Notebook](https://notebooklm.google.com/)** *(Substitua este link pela URL pública de compartilhamento do seu projeto)*

---
*Gerado via Gemini Notebook | Arquitetura de Sistemas Multiagentes no n8n*
