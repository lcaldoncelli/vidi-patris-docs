# Planejamento Detalhado de Desenvolvimento por Épico

Quebra dos épicos definidos em [estrutura-projeto.md](estrutura-projeto.md) em passos de desenvolvimento rastreáveis, com cenários de teste mapeados para cada passo. Segue a [ordem de desenvolvimento](estrutura-projeto.md#ordem-de-desenvolvimento) já combinada: **video-engine → core → frontend**.

Cada passo é uma entrega independente e testável — não avançar para o próximo até os cenários de teste do atual estarem cobertos.

---

## 1. `vidi-patris-video-engine` (C++)

### 1.1 Setup do projeto e esqueleto do CLI
- [ ] Estrutura de projeto CMake, build limpo funcionando
- [ ] Parsing de subcomando (`extract-audio`, `stt`, `snapshots`, `analyze-frames`) sem lógica implementada ainda — cada um retorna "not implemented"

**Cenários de teste:**
- `video-engine --help` lista os 4 subcomandos com suas opções
- `video-engine` sem argumentos retorna uso/help, exit code de erro
- Subcomando inexistente (`video-engine foo`) retorna erro claro, exit code != 0
- `video-engine --version` reporta uma versão
- Build limpo (sem cache) em ambiente novo termina com sucesso

### 1.2 Subcomando `extract-audio`
- [ ] Integração com ffmpeg/libav
- [ ] Saída em PCM s16le, mono, 16kHz (`<dir>/audio.wav`)

**Cenários de teste:**
- Vídeo mp4 com áudio → gera WAV válido no formato exato (16kHz, mono, 16-bit PCM)
- Vídeo sem trilha de áudio → erro claro, sem crash, exit code != 0
- Arquivo de entrada inexistente ou corrompido → erro tratado, mensagem legível
- Vídeo com múltiplas trilhas de áudio → extrai a trilha definida (comportamento documentado, ex. primeira faixa)
- `--output-dir` inexistente → cria o diretório ou falha com mensagem clara (definir e testar o comportamento escolhido)
- Contêineres variados (mp4, mkv, mov, avi) → todos extraem corretamente
- Vídeo muito curto (<1s) e muito longo (ex. 2h) → extração íntegra em ambos os extremos

### 1.3 Subcomando `stt`
- [ ] Integração com whisper.cpp
- [ ] Saída `transcript.json` com segmentos e timestamps

**Cenários de teste:**
- WAV válido (16kHz mono) com fala clara em PT-BR → `segments` coerentes, timestamps monotonicamente crescentes
- WAV em silêncio total → `segments: []`, sem erro
- WAV fora do formato esperado (ex. 44.1kHz estéreo) → rejeitado com erro claro (ou resample automático — decidir e testar o comportamento escolhido)
- Áudio longo (>30min) → processa sem estourar memória, dentro de um orçamento de tempo aceitável
- Áudio com ruído de fundo/música sobreposta → degradação aceitável, sem crash
- Arquivo de entrada inexistente/corrompido → erro tratado

### 1.4 Subcomando `snapshots`
- [ ] Scene detection
- [ ] Fallback de amostragem fixa a cada N segundos
- [ ] Manifesto `manifest.json` com timestamp + arquivo de cada frame

**Cenários de teste:**
- Vídeo com cortes de cena nítidos → cortes detectados batem com verificação manual (amostra de referência)
- Vídeo sem cortes (cena única/estática) → cai no fallback de amostragem fixa
- `--interval-seconds` customizado → intervalo respeitado no fallback
- Vídeo muito curto (<1s) → pelo menos 1 frame capturado
- Resoluções/aspect ratios variados → frames salvos sem distorção
- `manifest.json` consistente: contagem e nomes de arquivo batem com o que foi salvo em disco
- `--output-dir` inexistente → mesmo comportamento definido em 1.2 (consistência entre subcomandos)

### 1.5 Subcomando `analyze-frames`
- [ ] Integração ONNX Runtime (NudeNet, detector de armas, checkpoint de violência)
- [ ] Saída `frame_detections.json`

**Cenários de teste:**
- Diretório com frames neutros → `frame_detections` vazio ou confidences baixas
- Frames com nudez conhecida (dataset/benchmark de teste) → categoria `nudity` detectada com confidence plausível
- Frames com arma visível → categoria `weapon` detectada
- Frames com violência (sangue, luta) → categoria `violence` detectada
- Diretório vazio → JSON vazio, sem erro
- Imagem corrompida dentro do diretório → pula o arquivo e loga, não derruba o processamento dos demais
- Tempo de inferência por frame dentro de um orçamento definido (ex. <200ms/frame em CPU) — teste de performance, não só correção
- Falsos positivos/negativos conhecidos dos modelos genéricos → registrado como limitação esperada, não tratado como bug de implementação

### 1.6 Empacotamento e distribuição
- [ ] Build/empacotamento via CMake, binário distribuível

**Cenários de teste:**
- Build reproduzível em máquina limpa (sem cache local)
- Binário roda sem dependências externas faltando no ambiente-alvo
- Execução dos 4 subcomandos em sequência sobre o mesmo vídeo de ponta a ponta (teste de integração do próprio video-engine, ainda sem o backend)

---

## 2. `vidi-patris-core` (Python/FastAPI)

> Pré-requisito: os 4 subcomandos do video-engine já validados isoladamente (seção 1).

### 2.1 Ingestão de vídeo
- [ ] Endpoint de upload, validação de formato/tamanho, storage local, criação de job

**Cenários de teste:**
- Upload de vídeo válido dentro do limite → job criado, resposta 2xx
- Upload de arquivo não-vídeo (ex. `.txt` renomeado para `.mp4`) → rejeitado com 4xx
- Upload acima do limite de tamanho → rejeitado com mensagem clara
- Upload interrompido/incompleto → não deixa arquivo parcial "sujo" no storage
- Uploads concorrentes → jobs isolados, sem cruzamento de dados

### 2.2 Orquestração do pipeline
- [ ] Invocação dos subcomandos do video-engine via subprocess, na sequência definida
- [ ] Parsing dos JSONs de saída de cada etapa
- [ ] Status granular do job por etapa

**Cenários de teste:**
- Pipeline feliz: `extract-audio` → `stt` em paralelo com `snapshots` → `analyze-frames`, todos os JSONs parseados, status avança etapa a etapa até `done`
- Falha em uma etapa (ex. `extract-audio` retorna erro) → job marcado `failed` com motivo, etapas seguintes não executam
- Timeout de uma etapa (vídeo grande travando) → processo é encerrado, job marcado `failed`/`timeout`, sem travar o worker
- Binário `video-engine` ausente/não executável → erro tratado na inicialização do backend, não falha silenciosa em runtime
- Retry de um job que falhou → idempotente, não duplica arquivos/artefatos em disco

### 2.3 Integração com LLM de análise semântica
- [ ] Envio de segmentos da transcrição para a API de terceiros
- [ ] Parsing da resposta estruturada (categoria/severidade/trecho/timestamp)

**Cenários de teste:**
- Transcript com trechos sensíveis conhecidos → categoria/severidade retornadas conforme esperado (usar mock determinístico no teste, não a API real)
- Transcript vazio (vídeo sem fala) → LLM não é chamado, ou é chamado e retorna vazio sem erro
- Erro/timeout da API do LLM → retry/backoff aplicado, job não trava indefinidamente
- Rate limit do free tier atingido → tratado (fila/backoff), não falha silenciosa
- Resposta fora do schema esperado (JSON malformado) → validação rejeita e loga, pipeline não quebra

### 2.4 Motor de scoring
- [ ] Combinação de sinais de texto (LLM) + imagem (video-engine) por categoria
- [ ] Aplicação do ruleset Brasil/Classind

**Cenários de teste:**
- Caso conhecido de severidade leve → score "10" (conforme tabela Classind documentada)
- Nenhum sinal detectado em nenhuma categoria → score "Livre"
- Sinal máximo em todas as categorias → score "18"
- Sinais conflitantes entre texto e imagem (ex. texto indica 18, imagem indica Livre) → regra de composição aplicada de forma determinística (documentar e testar a regra escolhida, ex. "pior caso vence")
- Severidade exatamente no limite de um threshold → comportamento determinístico (`>=` vs `>` testado explicitamente)

### 2.5 Persistência (SQLite)
- [ ] Modelo de dados: vídeos, jobs, resultados, cenas sinalizadas, score final

**Cenários de teste:**
- Job e resultado persistidos sobrevivem a restart do processo
- Consultas retornam dados consistentes com o que foi gravado
- Escrita concorrente de dois jobs simultâneos não corrompe dados

### 2.6 API de consulta
- [ ] Endpoints de status, resultado e timeline para o frontend

**Cenários de teste:**
- GET status de job existente → etapa atual correta
- GET status de job inexistente → 404
- GET resultado antes do job terminar → resposta apropriada (ex. 202/processando), não erro genérico
- Contrato de resposta estável e testado via schema (evita quebrar o frontend em mudanças futuras)

### 2.7 Fila assíncrona in-process
- [ ] Processamento em background sem bloquear o upload

**Cenários de teste:**
- Upload retorna resposta imediata mesmo com processamento pesado em andamento
- Múltiplos jobs enfileirados processam na ordem/política esperada
- Falha em um job não derruba o processo principal nem afeta outros jobs em andamento

> Ao final da seção 2, validar toda a automação via chamadas de API diretas (curl/Postman/testes de integração) — sem frontend — conforme combinado na ordem de desenvolvimento.

---

## 3. `vidi-patris-frontend` (React)

> Pré-requisito: API do backend já validada via testes de integração (seção 2).

### 3.1 Upload de vídeo
- [ ] Tela de envio com validação client-side e feedback de progresso

**Cenários de teste:**
- Upload de vídeo válido → progresso exibido, job criado
- Arquivo inválido (tipo/tamanho) → bloqueado no client antes do envio
- Cancelamento do upload pelo usuário → estado limpo, sem job órfão

### 3.2 Acompanhamento de status do job
- [ ] Polling do endpoint de status

**Cenários de teste:**
- Mudança de etapa no backend reflete na UI dentro do intervalo de polling
- Erro de rede durante o polling não trava a UI (retry/mensagem de erro)
- Job com status `failed` exibido de forma clara para o usuário

### 3.3 Visualização do score final
- [ ] Badge de faixa etária + breakdown por categoria

**Cenários de teste:**
- Badge exibe a faixa etária correta retornada pela API
- Breakdown por categoria bate com os dados da API
- Caso "Livre" (sem sinais) exibido corretamente, sem seções vazias quebrando o layout

### 3.4 Timeline navegável
- [ ] Cenas sinalizadas clicáveis, sincronizadas com o player

**Cenários de teste:**
- Clique em cena sinalizada pula o player para o timestamp correto
- Timeline sem cenas sinalizadas (vídeo "Livre") não quebra a UI
- Timestamp fora do range de duração do vídeo é tratado sem erro

### 3.5 Modo apresentação
- [ ] View otimizada para demos comerciais

**Cenários de teste:**
- Renderização correta em tela cheia/projeção
- Carrega dados de um job já processado sem exigir upload ao vivo durante a demo

---

## Como usar este documento

- Marcar os checkboxes conforme os passos são implementados — dá visibilidade do progresso por repositório.
- Nenhum passo é considerado "pronto" sem os cenários de teste correspondentes cobertos (automatizados ou, no mínimo, executados manualmente e registrados).
- Ordem entre repositórios é sequencial (video-engine → core → frontend); dentro de cada repositório, a ordem dos passos segue dependências reais (ex. `stt` depende de `extract-audio` já funcionar).
