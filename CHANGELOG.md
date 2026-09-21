# Changelog

Todas as mudanças notáveis deste projeto serão documentadas neste arquivo.

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/).

---

## [0.0.1-CONCEPT] — 2026-05-24

Versão conceitual. Arquitetura definida, protocolo projetado, código-fonte gerado, diagramas criados. Nenhuma funcionalidade real executável — serve como blueprint para implementação.

### Definido

- **Método de transferência:** cabo P2 stereo (audio out → line in)
- **Protocolo SoundBridge:** calibração (6 etapas), sync (sweep), header (MAGIC "SNBR"), data frames (62B com CRC8)
- **4 perfis de qualidade:** low (26 KB/s), medium (96 KB/s), high (240 KB/s), ultra (576 KB/s)
- **Codificação stereo:** canal L = bytes pares, canal R = bytes ímpares
- **3 modos de codificação:** amplitude (implementado), frequência FSK (esqueleto), híbrido (futuro)
- **Hardware referência:** ASUS ROG STRIX B660-F com codec ALC4080 (Line-In dedicado, 113 dB SNR)

### Adicionado — PC A (Python, Transmissor)

- `main.py` — CLI com argparse, perfis, `--calibrate-only`, `--info`
- `soundbridge_tx/config.py` — dataclass de configuração com 4 perfis
- `soundbridge_tx/checksum.py` — CRC32 (zlib) + CRC8 (poly 0x07)
- `soundbridge_tx/protocol.py` — build_header, build_data_frames
- `soundbridge_tx/encoder.py` — encode amplitude/frequency, generate sync/silence
- `soundbridge_tx/calibration.py` — sinal de calibração 6 etapas
- `soundbridge_tx/wav_writer.py` — gerador WAV usando stdlib (wave + struct)
- `requirements.txt` — documentação de que usa apenas stdlib

### Adicionado — PC B (C++20/Qt6, Receptor)

- `CMakeLists.txt` — Qt6 + PortAudio + FFTW3 (opcionais)
- `src/main.cpp` — entry point Qt
- `src/config/` — ReceiverConfig struct + perfis
- `src/checksum/` — CRC32 + CRC8
- `src/protocol/` — parse_header, parse_data_frame, sync_detector (RMS)
- `src/decoder/` — interface base + amplitude decoder + frequency decoder
- `src/audio/` — audio_capture (stub PortAudio) + wav_reader (16/24-bit)
- `src/calibration/` — CalibrationAnalyzer
- `src/ui/main_window` — janela principal Qt dark (paleta OmniCam)
- `src/ui/log_widget` — log colorido por nível (INFO/CALIB/WARN/ERROR/OK/DATA)
- `src/ui/progress_widget` — barra gradient laranja com stats
- `src/ui/config_dialog` — perfis + áudio + decodificação + recepção + debug + CLI preview
- `src/ui/calibration_dialog` — 6 etapas animadas + resultado

### Adicionado — Documentação

- `README.md` — documentação completa (visão geral, uso, protocolo, estrutura, hardware)
- `.gitignore`
- `docs/ROADMAP.md` — roteiro de implementação em 12 fases (layout-first)
- 7 diagramas draw.io (paleta OmniCam dark):
  - `01-arquitetura-geral` — visão macro PC A → Cabo → PC B
  - `02-fluxo-protocolo` — sequência temporal do WAV
  - `03-fluxo-calibracao` — 6 etapas de calibração
  - `04-estados-receiver` — máquina de estados (8 estados, 10 transições)
  - `05-estrutura-frame` — layout binário de header, data frame, calibração, modos
  - `06-pipeline-dados` — pipeline TX → RX completo
  - `07-layout-ui` — wireframe da interface Qt + componentes + paleta

### Adicionado — Mock Interativo

- Mock React (JSX) da interface do receptor com 3 telas interativas
- Paleta visual da família OmniCam/KeyboardMouseBridge aplicada

### Decisões Técnicas Registradas

- PC A usa apenas Python stdlib (sem numpy/sounddevice) — gera WAV que o usuário reproduz
- PC B usa C++20 / Qt6 — interface dark, não CLI
- Stereo duplica velocidade sem crosstalk em cabo curto
- Calibração opcional mas recomendada
- CRC8 por frame + CRC32 global para integridade
- Perfis prontos como atalhos (low/medium/high/ultra)
- Modo debug `--from-wav` planejado para testar sem cabo

### Alternativas Descartadas

- ggwave/GibberLink (~20 B/s) — extremamente lento
- minimodem (~600 B/s) — lento
- ESP32 Bluetooth Audio — codec SBC destrói dados (1.8-7h para 5MB)
- ESP32 BT HID — risco de bloqueio
- Dongle RF TX/RX — caro (R$150+)
- QR Codes — backup viável mas ~28 min

### Erros Corrigidos

- Estimativas iniciais de BT Audio estavam erradas (confusão B/s vs KB/s) — corrigido de "3-7 min" para "1.8-7 horas"

---

*Próxima versão: 0.1.0 — Fase 1+2 do roadmap (esqueleto Qt compilável + layout principal funcional)*
