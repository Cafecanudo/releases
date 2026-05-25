# OmniCam — Multi Presença

> Roteamento de webcam para múltiplas saídas de vídeo (HDMI/USB) com simulação de lag, glitch, gravação em loop e sistema de microfone com push-to-talk.

**Versão:** 1.0.0 LTS

![OmniCam](src/resources/omnicam_256.png)

---

## Visão geral

OmniCam é um app desktop para Windows que distribui o vídeo de uma única webcam para várias saídas físicas (monitores HDMI/USB) ou simuladores em janela. Cada saída pode exibir a câmera ao vivo ou uma gravação prévia, com efeitos opcionais de lag/glitch para simular instabilidade. Inclui sistema completo de microfone com roteamento por saída e push-to-talk (PTT).

Útil para apresentações multi-tela, demonstrações ao vivo onde se quer simular falhas de conexão, ou cenários em que a mesma webcam precisa aparecer em mais de uma destinação física simultaneamente.

---

## Tecnologias

- **C++17** + **Qt 6.11** (UI, eventos, settings)
- **GStreamer 1.0** (captura, pipeline de vídeo/áudio, sinks WASAPI)
- **CMake** (build)
- **Windows API** (WASAPI direto via `IMMDeviceEnumerator` pra resolver default devices)
- **MSVC 2022**

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                       MainWindow                            │
│  ┌────────────┐  ┌──────────────────────────────────────┐   │
│  │ InputPanel │  │  Grid 2x2 de MonitorWidget (slots)   │   │
│  │            │  │  ┌─────────┐ ┌─────────┐             │   │
│  │ Webcam     │  │  │ Slot 1  │ │ Slot 2  │             │   │
│  │ Microfone  │  │  ├─────────┤ ├─────────┤             │   │
│  │ PTT        │  │  │ Slot 3  │ │ Slot 4  │             │   │
│  └────────────┘  └─────────────────────────────────────-┘   │
│  ┌────────────────────────────────────────────────────-─┐   │
│  │ Alert bar (mic disabled, dismissable)                │   │
│  └─────────────────────────────────────────────────────-┘   │
│  Footer: checkboxes globais, configurações                  │
└─────────────────────────────────────────────────────────────┘

CameraPreview (tee → multiple appsinks → push para cada slot)
MicCapture     (wasapi2src → level → valve → wasapi2sink dinâmico)
```

**Componentes principais:**
- `CameraPreview` — captura webcam via GStreamer, distribui via `tee` para o input panel + cada slot ativo. Suporta gravação concorrente em arquivo MP4.
- `MonitorWidget` — um por slot. Renderiza CAM (ao vivo) ou VIDEO (gravação em loop). Tem efeitos próprios de glitch/lag.
- `MicCapture` — pipeline de áudio único e dinâmico. Roteia microfone para uma saída específica (ou default Windows pra simulador). Suporta efeitos (packet loss + bit crush).
- `HardwareDetector` — descobre webcams, monitores, microfones e saídas de áudio.
- `AppSettings` — persistência em INI (Qt `QSettings`).
- `GlitchEffects` — algoritmos de efeitos visuais (RGB shift, slices, etc).

---

## Features

### 🎥 Roteamento de vídeo
- 1 webcam → até 4 saídas físicas simultâneas (monitores HDMI, USB capture, etc.)
- Simulador em janela (mesmo PC) para teste sem hardware adicional
- Detecção automática de monitores conectados
- Hot-plug bidirecional: detecta conexão/desconexão em tempo real
- Slot fantasma: monitor salvo no config mas não conectado aparece com `DESCONECTADO` (apenas botão de remover habilitado)
- Persistência: monitor e slot são salvos e restaurados automaticamente
- Auto-incremento de nome para simuladores ("Simulador 1", "Simulador 2"…)
- **Câmera exclusiva**: apenas 1 slot pode estar em modo CAM por vez (configurável)

### 🎬 Modos por slot
- **CAM**: transmite a webcam ao vivo
- **VIDEO**: reproduz uma gravação em loop, com substituição perfeita por outra a qualquer momento
- **GO LIVE**: abre janela física no monitor de destino e começa a transmitir
- **IN LIVE**: indicador visual piscando (vermelho) quando transmitindo

### 🎞️ Gravação
- Botão GRAVAR no painel de webcam
- Salva como `omnicam_<camera>_<WxH>_<yyyyMMdd_HHmmss>.mp4` com metadados embedados (resolução nativa, FPS, codec)
- Dialog informativo com dicas pré-gravação (dismissable)
- Loop em VIDEO se mantém sincronizado entre repetições

### ⚡ Simulação de lag/glitch
- **P. Glitch**: pausa o frame ("trava") temporariamente
- **Sim. Lag**: combinação de FPS drop + slices RGB + lag burst
- **Sim. Mic**: efeitos no microfone (packet loss + bit crush) — independente do Sim. Lag, mas acoplado por padrão
- **Lag Periódico** (footer): trigger automático em intervalos aleatórios
- **Trigger CAM→VIDEO** durante o lag (transição imperceptível entre live e gravação)
- Configurável: %FPS, frequência de glitch, intensidade, candidates de bit depth

### 🎙️ Microfone (Plano B — single sink dinâmico)
- Regra de transmissão: áudio só é enviado quando **exatamente 1 monitor está em CAM+LIVE**
- 0 monitores ativos → mute
- 2+ monitores ativos → mute total (mecanismo de segurança) + alerta visual + ícone 🎤 nos slots afetados
- Roteamento automático: monitor físico → seu device de áudio HDMI; simulador → default Windows
- Voice Activity Detection (VAD): bezel verde quando voz é detectada (só com PTT ativo)
- Detecção de mics em runtime: clique no combo reescaneia automaticamente
- Persistência do mic selecionado

### 🎤 PTT (Push-to-talk)
- Atalho global no app: **F12**
- Botão na UI sincronizado com o atalho
- Sem delay (controle via valve, não via stop/start de pipeline)
- Modo **"Ativo enquanto fala"**: clica uma vez no menu e mic continua ativo enquanto VAD detecta voz; timeout configurável (default 15s) sem voz → desliga automaticamente
- Lock desativa em: timer expira, F12 novamente, click no botão, mudança de fonte (CAM→VIDEO), saída de LIVE, múltiplos canais ativos
- Dialog de configuração acessível em 2 lugares (menu PTT + settings no footer)

### 🚨 Sistema de alertas
- Barra reutilizável acima do footer (Info/Warning/Error)
- Botão "Não mostrar novamente" persistente
- Overlay 🎤 nos monitores afetados quando mic é desativado por segurança

### 💾 Persistência
Tudo salvo em INI (`omnicam.config`):
- Câmera selecionada
- Microfone selecionado
- Layout de monitores (slot, conector, nome)
- Config de transition lag (FPS, % glitch, mic effects min/max)
- Config PTT (timeout)
- Alertas dismissed
- Checkboxes do footer (Downscale, Desativar Sim. Lag, Lag Periódico, Câmera Exclusiva)

---

## Build

### Requisitos

- **Windows 10/11** (x64)
- **Qt 6.11** (`C:/Qt/6.11.0/msvc2022_64`)
- **GStreamer 1.0** instalação completa (`C:/gstreamer/1.0/msvc_x86_64`)
  - Download: https://gstreamer.freedesktop.org/download/
  - Plugins necessários: `wasapi2`, `appsrc/appsink`, `videoconvert`, `level`, `valve`
- **Visual Studio 2022** ou Build Tools
- **CMake 3.20+**

### Compilação

```bash
git clone <repo>
cd omnicam

cmake -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

O executável fica em `build/Release/omnicam.exe`.

### Variáveis de ambiente

Se GStreamer não estiver no caminho padrão, defina:

```bash
set GSTREAMER_1_0_ROOT_MSVC_X86_64=C:\path\to\gstreamer\1.0\msvc_x86_64
```

---

## Uso rápido

1. **Adicionar saída**: clique direito num slot vazio → escolha um monitor real ou "Simulador"
2. **Selecionar câmera**: botão `SELECIONAR` no painel de webcam
3. **Ligar câmera**: botão `LIGAR`
4. **Ativar slot**: clique `CAM` no slot
5. **Ir ao ar**: clique `GO LIVE` → output window abre no monitor de destino
6. **Falar**: aperta `F12` (ou click+hold no botão PTT) → áudio transmite

Para gravar:

1. `GRAVAR` no painel → dialog de dicas → confirma
2. Faça os movimentos necessários
3. `PARAR` quando terminar → arquivo MP4 salvo em `<temp>/omnicam_*.mp4`

---

## Logs

O app emite logs no stdout úteis pra depurar produção:

```
==============================================
  OmniCam 1.0.0-LTS
  Multi Presença
  Build: Release | Max monitors: 4
==============================================
[MainWindow] camera restaurada: EMEET SmartCam S600
[MainWindow] monitor restaurado: hdmi-1 no slot 0
[CameraPreview] iniciada: 3840x2160 @30fps MJPEG (EMEET SmartCam S600)
[Monitor 0] IN LIVE
[MicCapture] iniciado, target='{0.0.0.00000000}.{952770e1-...}'
[PTT] ativado
[PTT] desativado
[Monitor 0] off-air
[MicCapture] parado
[MainWindow] monitor desconectado: hdmi-1 no slot 0
[MainWindow] monitor reconectado: hdmi-1 no slot 0
[CameraPreview] gravando: C:\Temp\omnicam_EMEET_3840x2160_20260522_111530.mp4
[CameraPreview] gravacao finalizada (180s)
```

Erros e warnings em `stderr` (BUS errors do GStreamer, falhas de pipeline, etc.).

---

## Atalhos

| Tecla | Ação |
|---|---|
| `F12` | Push-to-talk (mic) |
| `Ctrl+F1..F8` | P. Glitch / Sim. Lag por slot (configurável) |

---

## Arquitetura interna (notas técnicas)

### Pipeline de áudio (Plano B)

```
wasapi2src → audioconvert → audioresample → level → valve → audioconvert → wasapi2sink
                                              │
                                              ↓
                                       VAD (poll bus)
```

- 1 sink WASAPI por vez (evita conflito de clock entre múltiplos sinks)
- `valve` na cadeia para mute/unmute instantâneo (sem reinicializar pipeline)
- Probe de buffer após o `level` aplica efeitos (packet loss + bit crush) condicionalmente
- VAD via `level` element + poll do bus, com suporte a `GST_VALUE_LIST`, `GST_VALUE_ARRAY` e `G_VALUE_BOXED` (fallback pra `GValueArray`)

### Resolução do default Windows

`wasapi2sink` sem device explícito não usa confiavelmente o default. Solução: consultar via `IMMDeviceEnumerator::GetDefaultAudioEndpoint(eRender, eConsole)` e setar o GUID explicitamente.

### Slot fantasma

Quando um monitor salvo no INI não está conectado no startup, criamos um `MonitorWidget` "ghost" com:
- `connector_type = "ghost"`
- Botões desabilitados
- Overlay "DESCONECTADO" centralizado (label filho do `bezel`, não do `screen` — `screen` tem `WA_NativeWindow` que sobrescreve QPainter)
- Opacity 0.7 via `QGraphicsOpacityEffect`
- Reposicionamento via `QTimer::singleShot(0, ...)` pra esperar layout calcular

### Hot-plug

Timer de 2s (`pollDisconnectedMonitors`) que detecta:
- Monitor desconectado: força `off-air`, fecha output window, marca como ghost
- Monitor reconectado: atualiza info, remove ghost state

Skipa simuladores (não desconectam).

---

## Limitações conhecidas

- Áudio só transmite com **1 monitor** em CAM+LIVE. Quando 2+, mute total (intencional, é safety).
- Atalho PTT funciona apenas com app focado (sem global hotkey por enquanto).
- Gravação fixa em formato MP4 com codec da câmera (sem transcoding).

---

## Licença

Proprietário. Todos os direitos reservados.
