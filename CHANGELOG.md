# Changelog

Todas as mudanças notáveis do OmniCam serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/)
e este projeto adere ao [Versionamento Semântico](https://semver.org/lang/pt-BR/).

---

## [1.0.0-beta.2] - 2026-05-26

> **Status:** Pre-release (beta). Não recomendado para uso em produção.

Segunda iteração com sistema de atualização online, visualizador de logs e
refinos diversos. Consolida todo o trabalho desde a 1.0.0-LTS.

### Adicionado

#### Sistema de atualização online
- Verificação automática de updates no startup contra
  [Cafecanudo/releases](https://github.com/Cafecanudo/releases)
- `UpdateChecker` consulta `releases/latest` (apenas versões estáveis no aviso automático)
- `UpdateDialog` exibe notificação de nova versão com release notes formatadas (markdown)
- Botões: **Atualizar agora**, **Lembrar depois**, **Pular esta versão**
- `UpdateProgressDialog` mostra progresso de download bloqueante com cancelamento
- `UpdateOrchestrator` coordena todo o fluxo: check → dialog → download → executa setup → fecha app
- Após instalação, pergunta se reabre o app automaticamente
- "Pular esta versão" persiste no settings; ressurge quando há versão ainda mais nova
- `SemverVersion` parser e comparador: respeita hierarquia `alpha < beta < rc < estável`

#### Listagem de versões
- Nova entrada **"Sobre / Versões"** no menu de configurações
- `AvailableVersionsDialog` lista todas as releases (estáveis + pre-releases)
- Tabela com Versão, Data, Tipo (Estável / 🧪 Beta), Ação
- Indicador `●` na versão atualmente instalada
- Botão "Baixar" em cada versão permite rollback ou teste de pre-release
- Link "Ver no GitHub" para a página de releases

#### Visualizador de logs in-app
- Novo diálogo `LogViewerDialog` acessível pelo footer (botão de monitor)
- ComboBox lista todos os arquivos de log da pasta `%APPDATA%/OmniCam/OmniCam/logs/`
- Modo "Tempo real" (default): mostra log do dia atual com refresh automático (1s)
- Scroll inteligente: rola automaticamente quando há novo conteúdo,
  preserva posição quando o usuário rola para ler trechos anteriores
- Botão "Abrir pasta" abre o Windows Explorer no diretório de logs
- Botão "Atualizar lista" re-escaneia a pasta
- Inclui também o `gstreamer.log` (debug nativo do GStreamer)
- Janela maximizável com barra de título nativa, visível na taskbar
- Fonte monoespaçada (Consolas 9pt) para leitura confortável

#### Sistema de publicação automatizada
- `deploy.py` refatorado com suporte a canais: `--channel final|beta|alpha`
- Consulta GitHub Releases para auto-incrementar suffix (`beta.1`, `beta.2`, ...)
- Atualiza `OMNICAM_VERSION_SUFFIX` no CMakeLists automaticamente
- Build híbrido: verifica que `omnicam.exe` está atualizado antes de empacotar
  (compara timestamps com fontes e CMakeLists)
- Empacota tudo no repo público (`omnicam-dist/build/`)
- Sincroniza `CHANGELOG.md` e `README.md` com o repo público
- Gera setup.exe via Inno Setup com versão correta no nome
- Cria release via GitHub API e faz upload do asset automaticamente
- `extract_changelog_section` com fallback inteligente (full version → base → "Em desenvolvimento")
- Confirmação interativa antes de publicar
- Flag `--no-publish` para gerar setup local sem publicar
- Flag `--no-gst-full-bin` para modo seletivo de DLLs

#### Configuração e infraestrutura
- Arquivo `.deploy_config` (gitignored) com token e URL do repo de releases
- Variável `OMNICAM_RELEASES_REPO` no CMakeLists exposta via `config::releases_repo`
- Repo público `Cafecanudo/releases` armazena CHANGELOG e releases
- `omnicam.iss` aceita `MyAppVersion` e `MySourceDir` via parâmetros externos (`/D`)

### Alterado

- Janela voltou a usar a **barra de título nativa do Windows**
  (removido `FramelessWindowHint`)
- Removidos botões customizados Min/Max/Close — agora usa os nativos do sistema
- Removido drag-to-move manual e `QSizeGrip` — resize por qualquer borda nativo
- CMakeLists passou a usar `OMNICAM_VERSION_BASE` + `OMNICAM_VERSION_SUFFIX`
  separados em vez de versão única no `project()`
- `config.hpp.in` usa `@OMNICAM_FULL_VERSION@` (suporta sufixos como `beta.3`)
- Banner de inicialização mostra versão completa (`1.0.0-beta.2`) no log
- Logs do GStreamer redirecionados para arquivo separado (`gstreamer.log`)
- Registry do GStreamer regenerado a cada inicialização (evita cache stale)

### Corrigido

#### Captura de microfone (continuação da 1.0.0-LTS)
- **Recuperação automática em `AUDCLNT_E_DEVICE_INVALIDATED`** (`hr: 0x88890004`):
  detecta o erro no bus do GStreamer e reinicializa a captura automaticamente
- **Fallback em device ID stale**: quando o ID gravado não existe mais,
  re-detecta micros disponíveis e cai pro default
- **"Can't record audio fast enough"**: buffer aumentado de 200ms para 500ms,
  com `low-latency=false`. Resolve drops em PCs com CPU alta

### Build e dependências

- Adicionado módulo `Qt6::Network` (HTTP, JSON, download de releases)
- Novos arquivos: `SemverVersion`, `UpdateChecker`, `UpdateDownloader`,
  `UpdateOrchestrator`, `UpdateDialog`, `UpdateProgressDialog`,
  `AvailableVersionsDialog`, `LogViewerDialog`
- `omnicam.iss` parametrizado via `/DMyAppVersion=` e `/DMySourceDir=`

---

## [1.0.0-LTS] - 2026-05-22

> **Status:** Release final estável (Long Term Support).

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

[1.0.0-beta.2]: https://github.com/Cafecanudo/releases/releases/tag/v1.0.0-beta.2
[1.0.0-LTS]: https://github.com/Cafecanudo/releases/releases/tag/v1.0.0-LTS