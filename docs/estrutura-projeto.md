# Estrutura de Projeto

Detalhamento dos repositórios, do contrato entre módulos e dos épicos de cada um. Complementa o resumo em [plano-sistema-controle-parental.md](plano-sistema-controle-parental.md).

## Repositórios (multi-repo)

- **[`vidi-patris-docs`](https://github.com/lcaldoncelli/vidi-patris-docs)** — repositório dedicado à documentação de arquitetura e planejamento (este arquivo, o plano geral, o planejamento de épicos). Decisões cross-repo vivem aqui, não em nenhum dos repos de código.
- **[`vidi-patris-core`](https://github.com/lcaldoncelli/vidi-patris-core)** — o **backend**: API (Python/FastAPI), orquestração do pipeline, integração com LLM de terceiros, motor de scoring, persistência.
- **[`vidi-patris-video-engine`](https://github.com/lcaldoncelli/vidi-patris-video-engine)** — **C++**: decodificação de vídeo, STT local (whisper.cpp), scene detection, inferência de visão computacional (ONNX Runtime: NudeNet, detector de armas, checkpoint de violência). Exposto como binário CLI, sem servidor próprio em v0. **Restrição de design: sem acesso a rede/APIs externas** — todos os subcomandos operam 100% local, só leem/escrevem arquivo em disco.
- **[`vidi-patris-frontend`](https://github.com/lcaldoncelli/vidi-patris-frontend)** — **React**: upload, acompanhamento de status, visualização de score e timeline de cenas.

Os 4 repositórios já existem no GitHub (privados), com scaffolding mínimo de cada stack (FastAPI, CMake/C++, Vite+React+TS) — ainda sem lógica de negócio implementada.

## Contrato entre backend e video-engine (v0)

Em vez de uma única invocação que processa o vídeo inteiro de ponta a ponta, o binário `video-engine` expõe **subcomandos independentes**, cada um com entrada/saída isoladas em arquivo. Isso deixa a orquestração do backend mais previsível: cada etapa vira um passo de job rastreável com status próprio, em vez de um "processamento" opaco de caixa-preta.

### Subcomandos

1. **`video-engine extract-audio --input <video> --output-dir <dir>`**
   Extrai a trilha de áudio do vídeo e salva em `<dir>/audio.wav` no formato **PCM s16le, mono, 16kHz** — o formato exigido diretamente pelo whisper.cpp, evitando reamostragem/decodificação extra antes do `stt`. Sem compressão (FLAC/Opus/MP3): o áudio é gerado por nós via ffmpeg, não preservado de uma fonte externa, e o STT precisa de PCM bruto de qualquer forma — comprimir só adicionaria um passo de decodificação sem ganho real (o WAV já fica pequeno: ~1,9MB/min).

2. **`video-engine stt --input <audio.wav> --output <transcript.json>`**
   Roda STT local (whisper.cpp) sobre um arquivo de áudio e escreve um JSON com todos os segmentos de texto:
   ```json
   { "segments": [{"start": 0.0, "end": 3.2, "text": "..."}] }
   ```

3. **`video-engine snapshots --input <video> --output-dir <dir> [--interval-seconds N]`**
   Captura frames do vídeo (scene detection ou amostragem fixa a cada N segundos) e salva as imagens em `<dir>/`, junto com um manifesto:
   ```json
   { "frames": [{"timestamp": 12.4, "file": "frame_00124.jpg"}] }
   ```

4. **`video-engine analyze-frames --input-dir <dir> --output <frame_detections.json>`**
   Roda a inferência de visão computacional (NudeNet, detector de armas, checkpoint de violência) sobre os frames salvos em um diretório de snapshots e escreve:
   ```json
   { "frame_detections": [{"timestamp": 12.4, "category": "nudity|weapon|violence", "confidence": 0.87, "label": "..."}] }
   ```

### Orquestração no backend

O backend chama os subcomandos como subprocessos independentes: `extract-audio` → `stt`, em paralelo com `snapshots` → `analyze-frames` (áudio e vídeo são trilhas independentes). Cada etapa atualiza o status do job (`extracting_audio`, `running_stt`, `capturing_snapshots`, `analyzing_frames`, `scoring`, `done`), permitindo ao frontend mostrar progresso granular por etapa em vez de um único spinner "processando".

Todos os 4 subcomandos do `video-engine` (incluindo `analyze-frames`) rodam totalmente offline, sem chamada a API externa — os modelos de visão computacional (NudeNet, detector de armas, checkpoint de violência) são embarcados via ONNX Runtime no próprio binário. A **única** chamada a serviço de terceiros de todo o pipeline é a análise semântica de texto (LLM), feita pelo backend (`vidi-patris-core`) diretamente sobre a transcrição gerada pelo `stt` — o `video-engine` nunca acessa rede.

Esse contrato permite desenvolver os dois repositórios em paralelo desde o início, cada um mockando a saída/entrada do outro.

## Ordem de desenvolvimento

Apesar do contrato acima permitir paralelismo, a prioridade de implementação é **sequencial**, para reduzir retrabalho de integração:

1. **`vidi-patris-video-engine` primeiro** — implementar e validar cada subcomando isoladamente via linha de comando (`extract-audio`, `stt`, `snapshots`, `analyze-frames`), sem o backend envolvido. Cada etapa é testável sozinha (arquivo de entrada → arquivo de saída), o que dá confiança na engine antes de qualquer integração.
2. **`vidi-patris-core` em seguida** — integrar a orquestração desses subcomandos (invocação via subprocess) e expor isso como API. Validar essa automação com chamadas de API diretas (curl/Postman/testes de integração), ainda sem frontend.
3. **`vidi-patris-frontend` por último** — construído sobre uma API já validada e estável do backend, evitando retrabalho por mudanças de contrato em cascata.

Detalhamento passo a passo de cada épico nessa ordem, com cenários de teste mapeados, em [planejamento-epicos.md](planejamento-epicos.md).

## Épicos — `vidi-patris-core` (backend, Python/FastAPI)

1. Ingestão de vídeo — upload, validação de formato/tamanho, storage local, criação de job.
2. Orquestração do pipeline — invocar os subcomandos do video-engine (`extract-audio`, `stt`, `snapshots`, `analyze-frames`) na sequência correta, capturar/parsear o JSON de saída de cada etapa, tratar falhas e timeout por etapa, e atualizar o status granular do job.
3. Integração com LLM de análise semântica — enviar segmentos da transcrição para API de terceiros (free tier/baixo custo), receber categoria/severidade/trecho/timestamp estruturados.
4. Motor de scoring — combinar sinais de texto (LLM) + sinais de imagem (video-engine) por categoria; aplicar o ruleset Brasil/Classind (severidade → faixa etária).
5. Persistência — modelo de dados em SQLite (vídeos, jobs, resultados, cenas sinalizadas, score final).
6. API de consulta — endpoints para o frontend (status do job com granularidade por etapa, resultado do score, timeline).
7. Processamento assíncrono in-process — fila leve (ex. BackgroundTasks do FastAPI) para não bloquear o upload.

## Épicos — `vidi-patris-video-engine` (C++)

1. Decodificação e extração de mídia (ffmpeg/libav) → subcomando `extract-audio`.
2. Scene detection / amostragem de frames (detecção de corte de cena ou amostragem fixa a cada N segundos como fallback simples de v0) → subcomando `snapshots`.
3. STT local (integração com whisper.cpp, transcrição com timestamps) → subcomando `stt`.
4. Inferência de visão computacional (ONNX Runtime C++: NudeNet, detector de armas, checkpoint de violência) → subcomando `analyze-frames`.
5. Serialização da saída de cada subcomando no formato JSON contratado com o backend.
6. Interface CLI com subcomandos independentes (`extract-audio`, `stt`, `snapshots`, `analyze-frames`), cada um com entrada/saída por arquivo — sem API de rede em v0.
7. Build/empacotamento (CMake, binário distribuível consumido pelo backend).

## Épicos — `vidi-patris-frontend` (React)

1. Upload de vídeo — tela de envio com validação client-side e feedback de progresso.
2. Acompanhamento de status do job — polling do endpoint de status.
3. Visualização do score final — badge de faixa etária (Livre/10/12/14/16/18) + breakdown por categoria.
4. Timeline navegável — cenas sinalizadas clicáveis, sincronizadas com o player de vídeo.
5. Modo apresentação — view otimizada para demos comerciais (alinhado ao entregável "material de apresentação comercial" já previsto na v0).
