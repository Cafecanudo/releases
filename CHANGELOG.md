# Changelog

Todas as mudanças notáveis do OmniCam serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/)
e este projeto adere ao [Versionamento Semântico](https://semver.org/lang/pt-BR/).

---

## [1.0.0-LTS] - 2026-05-22

Primeira versão estável (LTS) do OmniCam. Aplicação completa de roteamento de
vídeo multi-monitor com captura de áudio, gravação, simulador e empacotamento
para distribuição via instalador Windows.

### Adicionado

#### Captura e roteamento de vídeo
- Captura de webcam via GStreamer com suporte a Media Foundation e DirectShow
- Decodificação MJPEG em tempo real (`jpegdec`) com fallback para câmeras RAW
- Roteamento simultâneo de até 4 monitores (físicos ou simuladores)
- Hot-plug bidirecional: detecta conexão/desconexão de monitores em tempo real
- Slot fantasma com overlay "DESCONECTADO" quando monitor físico cai
- Modo CAM (câmera ao vivo) e VIDEO (arquivo) por slot independente
- Trigger CAM → VIDEO com glitch effects de transição
- Janela de simulador redimensionável, com taskbar e fechamento via X

#### Áudio
- Captura de microfone via WASAPI2
- VAD (Voice Activity Detection) com indicador visual (bezel verde nos monitores)
- Plano B de áudio: apenas 1 slot CAM+LIVE transmite áudio por vez
- Seleção persistente de microfone entre sessões
- Detecção automática de saída de áudio (default ou específica)
- Recuperação automática em caso de `AUDCLNT_E_DEVICE_INVALIDATED`
- Buffer de áudio ajustado (500ms) para evitar drops em PCs lentos

#### PTT (Push-to-Talk)
- Acionamento por tecla `F12` ou botão na interface
- Modo "Ativo enquanto fala" (keep-active): mantém PTT ligado durante fala detectada,
  desliga após timeout configurável (padrão 15s)
- Countdown visual no botão durante keep-active
- Diálogo de configuração `PttConfigDialog` (timeout 1-600s)
- Desativação automática quando 0 ou múltiplos canais LIVE detectados

#### Efeitos visuais
- Pause Glitch: congela frame atual com efeito visual aleatório
- Sim.Lag: simulação de lag periódico com janela configurável e chance %
- Sim.Mic: bit crush + packet loss no microfone, atrelado ao Sim.Lag
- Trigger CAM→VIDEO automático: alterna fonte após N segundos de fala detectada
- Loop trigger: dispara VIDEO quando voz para por X segundos
- Efeitos configuráveis via dialog `LagSettingsDialog`

#### Gravação
- Gravação H.264 via `x264enc` (com fallback para `openh264enc` e `mfh264enc`)
- Saída em MP4 com metadata da câmera (resolução, fps, modelo)
- Diálogo de dicas com tempo aceitável/recomendado/ótimo (até 10min)
- Checkbox "Não mostrar novamente" persistente

#### Interface
- Janela frameless customizada (sem barra do Windows)
- Header com ícone, nome do app, descrição/versão e botões Min/Max/Close
- Drag-to-move no header, double-click para maximizar/restaurar
- Resize via QSizeGrip no canto inferior direito
- Ícone laranja personalizado (`omnicam.ico` + PNG 256×256)
- Alert bar com mensagens dismissíveis (persistente por chave)
- Overlay 🎤 vermelho em monitor quando microfone fica indisponível
- Footer com configurações e atalhos rápidos

#### Sistema e logging
- Logger com rotação diária (`omnicam_YYYY-MM-DD.log`)
- Captura de `std::cout` / `std::cerr` para arquivo + console
- Thread-safe via QMutex
- Log do GStreamer separado (`gstreamer.log`) com nível 3
- Contador de uptime exibido ao encerrar (Xh Ym Zs)
- Banner de inicialização com versão, build type, log path e modo GStreamer
- Configurações e logs em `%APPDATA%/OmniCam/OmniCam/`

#### Empacotamento e deploy
- Script `deploy.py` automatiza coleta de DLLs Qt + GStreamer + MSVC redist
- Inno Setup (`omnicam.iss`) gera instalador `setup.exe` para Windows
- Suporte a pasta portable, ZIP e instalador (configuráveis via CLI)
- Modo `--gst-full-bin` (default): copia todas as DLLs do GStreamer
- Modo `--no-gst-full-bin`: seletivo, para distribuições menores
- `diagnose.bat` incluído para troubleshooting de plugins GStreamer
- `gst-inspect-1.0.exe` empacotado para diagnóstico em campo
- Registry do GStreamer regenerado a cada inicialização (evita cache stale)
- Variáveis de ambiente configuradas automaticamente em modo deploy

### Alterado

- Configurações e logs migrados de `applicationDirPath()` para
  `QStandardPaths::AppDataLocation` (resolve permissões em Program Files)
- Atalho do PTT exposto no texto do botão: "Pressione para falar (F12)"
- `OutputWindow` do simulador usa `Qt::Window` (visível na taskbar) em vez de `Qt::Tool`
- Versão bumped de `0.0.1-SNAPSHOT` para `1.0.0-LTS`
- `add_executable(omnicam WIN32 ...)` no CMake: remove console ao executar fora do IDE
- Logs verbose removidos: `detectWebcams`, `detectMicrophones`, `detectAudioOutputs`
- Logs adicionados em eventos relevantes: PTT, mic start/stop, monitor live/off-air
- `Lag Periódico` não dispara se não houver CAM disponível nem vídeo carregado

### Corrigido

- Access violation `0xffffffffffffffff` ao fechar o app (signals residuais
  do `CameraPreview::stopped` atingindo monitores em destruição)
- Slot ghost: posicionamento do overlay "DESCONECTADO" centralizado no bezel
- CAM sem vídeo: clique no botão CAM agora congela limpo (sem glitch effects)
- Sim. Mic: desabilitado corretamente quando slot está em modo VIDEO
- Sim. Mic: atrelado a Sim. Lag (ativam/desativam juntos)
- Bezel verde: só aparece com PTT ativo (não mais com voz detectada apenas)
- Resize: drag pelo grip não move mais a janela ao soltar o mouse
- Recursos Qt: aliases corretos (`:/omnicam.ico` em vez de `:/src/resources/omnicam.ico`)
- Ícone do header: carrega corretamente do `omnicam_256.png` embarcado

### Build e dependências

- **Qt 6.11** (módulos Core, Gui, Widgets)
- **GStreamer 1.x** (plugins good, bad, ugly, libav)
- **Visual C++ Redistributable 2015-2022** (incluído no instalador)
- Compilação via **CLion** com generator **Ninja**
- **Inno Setup 6** para gerar o instalador
- **Python 3.10+** apenas para o script de deploy (stdlib, sem dependências pip)

### Plataformas suportadas

- Windows 10 (1809+) x64
- Windows 11 x64
- Recomendado: GPU integrada (Intel UHD, AMD Vega) ou superior
- Mínimo: 4 GB RAM, CPU dual-core 2 GHz
- Recomendado: 8 GB RAM, CPU quad-core 3 GHz (para 2+ slots ativos simultaneamente)

---

[1.0.0-LTS]: https://github.com/SEU-USUARIO/omnicam/releases/tag/v1.0.0-LTS
