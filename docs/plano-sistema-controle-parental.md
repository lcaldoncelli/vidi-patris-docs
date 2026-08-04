# Sistema de Avaliação de Controle Parental em Vídeos

## Sumário

Sistema que analisa vídeos automaticamente (áudio + imagem) e gera um **score de controle parental** — indicando presença e intensidade de violência, conteúdo sexual, linguagem imprópria e temas sensíveis, com timestamps das cenas sinalizadas.

O pipeline combina:
- **Speech-to-text local** (faster-whisper/whisper.cpp) da trilha de áudio, seguido de análise semântica da transcrição via LLM de terceiros (free tier/baixo custo) para identificar linguagem imprópria, temas de violência/sexo/drogas mencionados na fala.
- **Análise de frames de vídeo** via modelos open-source locais (NudeNet, detector de armas, checkpoint de violência) para detectar cenas com nudez, violência física, armas, etc.
- Um **motor de scoring** que combina os sinais de texto e imagem, aplicando o ruleset de classificação de um país (Brasil/Classind como primeiro) para gerar uma faixa etária indicativa (Livre/10/12/14/16/18) com breakdown por categoria e timeline navegável.

O projeto está dividido em duas fases principais: uma **v0** enxuta, monolítica e local-first (STT e visão computacional rodando localmente, com custo de infraestrutura próximo de zero — a única dependência de API de terceiros paga fica na análise semântica de texto via LLM), com foco em gerar demos convincentes para angariação de clientes; e uma **v1**, que escala e otimiza essa base (GPU dedicada, fine-tuning do modelo de violência, fila robusta) e evolui o scoring para um ruleset plugável por país.

---

## Fase v0 — Prova de conceito e angariação de clientes

**Objetivo:** validar a ideia rapidamente e ter demos visualmente convincentes para reuniões comerciais, com o menor esforço de engenharia possível.

### Escopo técnico
- **STT:** `faster-whisper` ou `whisper.cpp` rodando local (CPU ou GPU), com timestamps por segmento — grátis, sem custo por chamada, em vez de Whisper API/Deepgram/AssemblyAI.
- **Análise de texto:** LLM via API de terceiros (não local), priorizando opções com plano gratuito ou baixo custo (ex. Gemini free tier, ou créditos gratuitos de Claude/GPT), com prompt estruturado retornando JSON com categoria, severidade (0-5), trecho e timestamp.
- **Análise de frames:** modelos open-source locais — `NudeNet` (nudez/sexo) + detector de armas pré-treinado (YOLO open-weights) + checkpoint público de violência (ponto de partida, ex. treinado em RWF-2000) — em vez de Google Cloud Vision/AWS Rekognition.
- **Scoring:** composto ponderado por categoria (intensidade máxima + frequência + concentração temporal). Mapeamento de severidade → faixa etária brasileira (Classind) embutido como tabela fixa no próprio monolito (sem abstração de "país plugável" ainda — isso fica para v1).
- **Backend:** API monolítica simples (FastAPI/Node), fila in-process (sem SQS/Celery) e **SQLite local** (sem banco gerenciado) — sem separação de serviços; objetivo é validar viabilidade técnica com custo de infraestrutura próximo de zero.
- **Frontend:** upload de vídeo → score final + timeline clicável das cenas sinalizadas.

### Entregáveis
- [ ] Pipeline funcional ponta a ponta (upload → score)
- [ ] 5-10 vídeos processados como cases de demonstração
- [ ] Interface simples de visualização do score e das cenas sinalizadas
- [ ] Material de apresentação comercial baseado nos resultados
- [ ] Tabela de mapeamento severidade → faixa etária (Livre/10/12/14/16/18) do Brasil, aplicada no motor de score

### Limitações aceitas nesta fase
- Custo concentrado na chamada de LLM por texto (STT e visão computacional já rodam local, sem custo por chamada)
- Dependência de API de terceiros apenas na análise semântica de texto (LLM) — STT e visão computacional já rodam local
- Detecção de violência ainda limitada (fraqueza conhecida também dos modelos open-source prontos)

---

## Fase v1 — Produtização e escala

**Objetivo:** reduzir custo por vídeo, aumentar precisão (especialmente em violência) e suportar volume real de clientes.

### Escopo técnico
- **GPU dedicada / otimização de throughput** para `faster-whisper` em volume — já que STT nasce local desde a v0, o foco aqui é escalar, não migrar de API.
- **Fine-tuning do modelo de violência:** evoluir o checkpoint open-source usado na v0 (ex. treinado em RWF-2000) com dataset próprio, para cobrir o gap conhecido dos modelos genéricos.
- **Orquestração de pipeline:** fila robusta (Celery/SQS) com workers dedicados, em vez de chamadas síncronas.
- **Cache por hash de vídeo:** evita reprocessar conteúdo repetido (filmes/séries populares).
- **Scoring configurável por país:**
  - Separação conceitual entre (a) **sinais brutos** por categoria (saída do pipeline STT+visão: severidade 0-5, frequência, timestamps) e (b) **ruleset de classificação por país**, que traduz esses sinais numa faixa etária "oficial-like".
  - Ruleset como dado configurável (YAML/JSON) por país — Brasil (Classind) como primeiro, com estrutura pronta para adicionar outros (ex. MPAA/EUA, BBFC/UK) sem mudar código.

### Entregáveis
- [ ] GPU dedicada / otimização de throughput para faster-whisper em volume
- [ ] Fine-tuning do modelo de detecção de violência (evoluindo o checkpoint open-source da v0)
- [ ] Pipeline assíncrono escalável com fila e workers
- [ ] Sistema de cache de vídeos já processados
- [ ] Painel de configuração de pesos de scoring, incluindo o ruleset plugável por país

### Pontos de atenção
- **LGPD:** retenção de vídeo/transcrição envolvendo imagem de terceiros é sensível — mapear política de retenção e anonimização desde o início da fase.
- Definir métricas de qualidade (precisão/recall por categoria) para validar o modelo próprio antes de substituir a API de moderação de imagem.
- O score do sistema é **indicativo/aproximado**, não substitui classificação oficial — classificadores humanos avaliam contexto e relevância narrativa de forma qualitativa, algo que o pipeline automatizado só aproxima.

---

## Estrutura de Projeto

O projeto é dividido em **3 repositórios** (multi-repo, não monorepo):
- `vidi-patris-core` — **backend** (Python/FastAPI): orquestração do pipeline, integração com LLM, motor de scoring, persistência.
- `vidi-patris-video-engine` — **C++**: STT local (whisper.cpp), scene detection, inferência de visão computacional (ONNX Runtime). Exposto como binário CLI, sem servidor próprio em v0.
- `vidi-patris-frontend` — **React**: upload, acompanhamento de status, visualização de score e timeline de cenas.

Detalhamento completo (contrato entre módulos, épicos de cada repositório) em [estrutura-projeto.md](estrutura-projeto.md).

---

## Especificação técnica: Brasil (Classind)

**Órgão responsável e documento-base:** Sedigi (Secretaria Nacional de Direitos Digitais) / Ministério da Justiça e Segurança Pública (MJSP), "Guia Prático de Classificação Indicativa" (5ª edição, nov/2025, Portaria MJSP nº 1.048/2025). Essa edição inclui pela 1ª vez a faixa de 6 anos e critérios de interatividade para apps/IA — não relevante para vídeo, mas bom contextualizar.

**Faixas etárias:** Livre, 6, 10, 12, 14, 16, 18.

**3 eixos temáticos centrais**, cada um avaliado por frequência, relevância, contexto, intensidade e importância — não é presença binária:
- Violência
- Sexo e Nudez
- Drogas

### Tabela-resumo aproximada por faixa etária x eixo

> ⚠️ **Ressalva importante:** esta tabela é uma **aproximação não-oficial**, reconstruída a partir de fontes secundárias (imprensa, Wikipédia, glossário de bilheteria) — os PDFs oficiais do Guia Prático são digitalizados/escaneados, sem camada de texto, e não puderam ser lidos via OCR neste ambiente (poppler não instalado). **Antes de fixar estes thresholds em produção, é obrigatório validar linha a linha contra o Guia Prático oficial (5ª edição).**

| Faixa | Violência | Sexo e Nudez | Drogas |
|---|---|---|---|
| Livre | Ausente ou muito leve, sem realismo (ex. cartoon) | Ausente | Ausente |
| 10 | Leve, não gráfica, sem sangue explícito | Nudez não erótica, contexto não sexual | Menção/insinuação, sem uso explícito |
| 12 | Moderada, alguma tensão/ameaça, pouco sangue | Insinuação sexual leve, nudez parcial | Uso ocasional, sem ênfase ou incentivo |
| 14 | Mais intensa, cenas de luta/ação com consequência | Cenas sexuais sugeridas, sem explicitação | Uso mais recorrente, ainda sem detalhamento de consumo |
| 16 | Forte, com sangue/ferimentos explícitos | Cenas sexuais mais explícitas, nudez frontal | Uso explícito, podendo detalhar efeitos |
| 18 | Extrema, gráfica, gore, crueldade | Sexo explícito | Uso explícito associado a contextos extremos (produção, tráfico) |

**Fontes consultadas:**
- gov.br/mj — página oficial "Guia de Classificação" (Classind)
- Agência Brasil (17/11/2025) — matéria sobre a 5ª edição do guia
- Wikipédia — "Sistema de Classificação Indicativa Brasileiro"
- Central de Ajuda Ingressar — resumo prático de critérios por faixa etária

---

## Próximos passos sugeridos
1. Validar arquitetura do v0 com um protótipo mínimo (1 vídeo, pipeline manual)
2. Definir taxonomia final de categorias e níveis de severidade do score
3. Rodar os primeiros 5-10 vídeos de demonstração
4. Levantar requisitos legais (LGPD) em paralelo, antes de escalar para v1
5. Validar a tabela de critérios brasileira (Classind) contra o PDF oficial (5ª edição) antes de fixar thresholds de score em código — hoje reconstruída via fontes secundárias.
