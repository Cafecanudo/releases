# SoundBridge 🔊 → 🎧

**Transferência de arquivos entre dois PCs usando apenas um cabo de áudio.**

Sem rede, sem Wi-Fi, sem pendrive, sem Bluetooth. Os bytes viram som no PC transmissor,
atravessam um cabo P2 estéreo, e voltam a ser o arquivo original no PC receptor.

---

## Como funciona

O SoundBridge usa **áudio analógico** como meio de transmissão. O transmissor (PC A) codifica o
arquivo em um sinal **OFDM** (várias subportadoras de áudio em paralelo, como um modem de linha
telefônica ou o Wi-Fi), com correção de erros. O receptor (PC B) captura esse áudio pela entrada
de linha, decodifica e remonta o arquivo, validando com CRC32.

```
PC A  (transmissor)          cabo P2 estéreo           PC B  (receptor)
┌────────────────┐          ┌──────────────┐          ┌────────────────┐
│ Python (CLI)   │  saída    │              │  entrada │ C++ / Qt6 (GUI)│
│ gera o áudio   │ ───────►  │  )))   )))   │ ───────► │ captura e      │
│ OFDM do arquivo│  de áudio │              │  de linha│ decodifica     │
└────────────────┘          └──────────────┘          └────────────────┘
```

|  | PC A (transmissor) | PC B (receptor) |
|---|---|---|
| **Linguagem** | Python 3.10+ (só stdlib + numpy) | C++17 / Qt 6 |
| **Interface** | linha de comando | interface gráfica |
| **Papel** | gera o áudio (WAV) ou toca ao vivo | captura da entrada de linha e decodifica |
| **Conexão** | saída de áudio (fone/line-out) | entrada de linha (line-in) |

---

## Velocidade

A velocidade depende da **modulação** escolhida (quanto mais densa, mais rápida — e menos robusta):

| Modulação | Velocidade | 10 MB em | Observação |
|---|---|---|---|
| QPSK | ~4 KB/s | ~42 min | mais robusta |
| 16-QAM | ~8 KB/s | ~21 min | equilíbrio |
| **64-QAM** | **~12 KB/s** | **~14 min** | **recomendada (teto robusto)** |
| 256/1024-QAM | ~16-20 KB/s | — | experimentais (ver nota) |
| **`--zip`** | **até 100×+** | **segundos** | quando os dados são compressíveis |

> **256-QAM e 1024-QAM (experimentais):** funcionam bem em **arquivos pequenos e médios**. No cabo,
> o combo 256-QAM + FEC leve (`--fec r23`/`r34`) transferiu arquivos de até ~200 KB com sucesso, e é
> cerca de 3× mais rápido que o 64-QAM para esses tamanhos. Em arquivos grandes (vários MB) as
> modulações densas ficam marginais — uma transmissão longa acumula ruído suficiente para corromper
> os dados. Para arquivos grandes, use o **64-QAM** (robusto). Para arquivos pequenos/médios onde a
> velocidade importa, o combo 256-QAM + `--fec r34` é a opção mais rápida.

O **64-QAM** é a opção recomendada: rápida e confiável no cabo. O `--zip` é a maior alavanca quando
o arquivo comprime bem (texto, logs, código).

---

## Instalação

### PC A — transmissor (Python)

Precisa de **Python 3.10+** e **numpy**. Nenhuma compilação.

```bash
pip install numpy sounddevice
```

(`sounddevice` só é necessário para tocar ao vivo com `--play`; para gerar um arquivo WAV, basta o numpy.)

### PC B — receptor (C++ / Qt6, Windows)

Precisa de:
- **Qt 6.5+** (testado com Qt 6.11.1, MSVC 2022 64-bit)
- **Visual Studio 2022 Build Tools** (MSVC, C++17)
- **CMake 3.20+**
- **vcpkg** com **PortAudio**

**1. Instalar o PortAudio via vcpkg:**
```bash
git clone https://github.com/microsoft/vcpkg.git
cd vcpkg
./bootstrap-vcpkg.bat
./vcpkg install portaudio:x64-windows
```

**2. Compilar (CLion ou linha de comando):**

No CLion, configure as opções de CMake:
```
-DCMAKE_PREFIX_PATH=C:/Qt/6.11.1/msvc2022_64
-DCMAKE_TOOLCHAIN_FILE=<caminho>/vcpkg/scripts/buildsystems/vcpkg.cmake
-DVCPKG_TARGET_TRIPLET=x64-windows
```

A flag `-DVCPKG_TARGET_TRIPLET=x64-windows` é obrigatória (sem ela o CMake não acha o PortAudio). O
`portaudio.dll` é copiado automaticamente para a pasta do executável ao compilar.

---

## Como usar

### Conexão física

Ligue a **saída de áudio do PC A** (fone/line-out) à **entrada de linha do PC B** (line-in) com um
cabo P2 estéreo macho-macho. A entrada de **microfone** funciona, mas é menos confiável (o controle
automático de ganho distorce o sinal) — prefira a entrada de linha quando disponível.

### Passo 1 — Preparar o receptor (PC B)

1. Abra o **SoundBridge Receiver**.
2. Clique em **Configurações** (⚙).
3. Em **Codificação**, escolha o modo **OFDM**, a **banda** (22 kHz recomendado) e a **modulação**
   (64-QAM recomendado). Cada campo tem um ícone de ajuda (ⓘ) explicando as opções.
4. Em **Áudio**, selecione o dispositivo de **entrada** (a entrada de linha conectada ao cabo).
5. Clique em **SALVAR**.
6. Clique em **INICIAR**. O receptor fica aguardando o sinal.

> As configurações do receptor precisam bater com as do transmissor (banda, modulação, estéreo).
> Se a configuração do app não bater com a do Windows, um aviso oferece abrir o painel de som para
> ajustar — depois é só clicar INICIAR de novo.

### Passo 2 — Transmitir (PC A)

O transmissor tem dois modos: **gerar um arquivo WAV** (para tocar depois) ou **tocar ao vivo** na
placa de som (`--play`).

**Tocar ao vivo (recomendado):**
```bash
python -m sim.make_wav --in arquivo.zip --qam64 --stereo --band-high 22000 --play --device N
```
(`--device N` é o índice da saída de áudio; veja como listar abaixo)

**Gerar um WAV (para tocar manualmente depois):**
```bash
python -m sim.make_wav --in arquivo.zip --qam64 --stereo --band-high 22000 --out saida.wav
```
Depois reproduza `saida.wav` em qualquer player, com o cabo conectado à saída.

**Listar os dispositivos de saída:**
```bash
python -m sim.make_wav --list-devices
```

> **Não sabe o índice do `--device`?** Use `--play` **sem** o `--device`. O transmissor lista os
> dispositivos disponíveis e pergunta qual usar antes de enviar. Ex:
> `python -m sim.make_wav --in arquivo.zip --qam64 --stereo --band-high 22000 --play`

### Passo 3 — Acompanhar

No receptor, a barra de progresso avança conforme o arquivo chega. Ao terminar, ele valida o CRC e
salva o arquivo na pasta de saída configurada.

---

## Opções do transmissor

| Opção | O que faz |
|---|---|
| `--in ARQUIVO` | o arquivo a transmitir (ou uma pasta, veja abaixo) |
| `--out ARQUIVO` | gera um WAV em vez de tocar ao vivo |
| `--play` | toca ao vivo na placa de som |
| `--device N` | índice da saída de áudio (com `--play`) |
| `--stereo` | usa os dois canais (2× mais rápido; recomendado) |
| `--band-high N` | frequência máxima da banda em Hz (recomendado: 22000) |
| `--qam16` / `--qam64` | modulação 16-QAM / 64-QAM (padrão: QPSK) |
| `--qam256` / `--qam1024` | modulações experimentais (ver nota abaixo) |
| `--fec MODO` | correção de erro: `r12` (padrão, robusto, arquivos grandes), `r23`/`r34` (mais leve/rápido, arquivos pequenos-médios), ou `none` |
| `--resync N` | re-sincronização a cada N blocos: `off`/`10`/`25`/`5` (padrão 10) |
| `--parity N` | blocos de recuperação: `off`/`8`/`16`/`32` (padrão 16) |
| `--zip` | comprime os dados antes de enviar (o receptor descomprime) |
| `--name "NOME"` | nome do arquivo salvo no receptor (aceita subpasta: `docs/a.txt`) |
| `--text "..."` | envia um texto direto, sem arquivo |
| `--copymemory` | o receptor copia o conteúdo para a área de transferência |
| `--profile NOME` | usa um perfil de recepção (define a pasta de destino no receptor) |
| `--in /pasta` | envia todos os arquivos de uma pasta, um após o outro |
| `--gap N` | intervalo em segundos entre arquivos (com `--in /pasta`) |
| `--verbose` | mostra detalhes de cada etapa |

### Exemplos

```bash
# Arquivo com nome e destino por perfil
python -m sim.make_wav --in relatorio.pdf --qam64 --stereo --band-high 22000 \
    --name "docs/relatorio.pdf" --profile trabalho --play --device N

# Texto direto para a área de transferência do receptor
python -m sim.make_wav --text "chave: abc123" --copymemory --qam64 --stereo \
    --band-high 22000 --play --device N

# Comprimir (ótimo para texto/código)
python -m sim.make_wav --in log.txt --zip --qam64 --stereo --band-high 22000 --play --device N

# Uma pasta inteira
python -m sim.make_wav --in ./projeto --qam64 --stereo --band-high 22000 --play --device N
```

> **Dica (git-bash no Windows):** caminhos com barra inicial em `--name` são convertidos pelo shell.
> Use `//x`, `MSYS_NO_PATHCONV=1`, ou um caminho relativo sem barra inicial.

---

## Perfis de recepção

O receptor pode ter **perfis** que definem a pasta de destino. Configure em **Configurações →
Perfis de Recepção** (adicionar/editar/remover). Cada perfil tem um nome e uma pasta base.

Ao transmitir com `--profile trabalho`, o arquivo é salvo na pasta base desse perfil. Se o nome do
arquivo tiver subpasta (`--name "docs/a.txt"`), a árvore é criada dentro da pasta base. Se o perfil
não existir no receptor, ele usa a pasta padrão e avisa.

---

## Modo WAV (para testes, sem cabo)

O receptor pode decodificar um WAV direto do disco, sem precisar do cabo — útil para testar.

1. **Configurações → Debug → marcar "Modo WAV"**.
2. Selecione o `.wav` gerado pelo transmissor (o app mostra sample rate, canais, duração).
3. **SALVAR** e **INICIAR** — o app decodifica o arquivo pelo mesmo pipeline.

---

## Preparar o PC transmissor (Windows)

Para o áudio chegar íntegro ao cabo, desative qualquer processamento de som no PC A. **Sem isso, o
sistema operacional pode alterar o sinal antes de sair pela placa.**

**1. Desativar efeitos do driver de áudio:**
`Win+R` → `mmsys.cpl` → aba **Reprodução** → o dispositivo de saída → **Propriedades** →
**Aprimoramentos** → marcar **"Desativar todos os efeitos sonoros"**.

**2. Desativar o som espacial:**
Botão direito no ícone de som → **Som espacial** → **Desativado**.

**3. Desativar softwares de áudio do fabricante:**
Se houver algum painel de áudio de fabricante (equalizador, "surround", "bass boost", etc.),
desative os efeitos. Qualquer processamento no caminho do áudio distorce a codificação.

**4. Conferir o sample rate:**
`mmsys.cpl` → **Reprodução** → dispositivo → **Propriedades** → **Avançado** → **Formato padrão**
deve ser **48000 Hz** (o SoundBridge opera a 48 kHz).

> **Robustez:** na prática, o SoundBridge decodifica mesmo com alguns desses efeitos ligados (o FEC
> corrige). Mas desativá-los dá a melhor margem — recomendado para a primeira transmissão.

---

## Protocolo (resumo técnico)

O sinal é OFDM a 48 kHz: um símbolo de sincronização, um símbolo de estimativa de canal (LTF), um
**header** de 96 bits protegido por CRC-24, e os símbolos de dados. Os dados passam por um código
convolucional (FEC r=1/2) com interleaver e paridade, e são organizados em blocos com CRC próprio.

O **header** carrega o tamanho, o CRC32 do conteúdo, e os parâmetros de FEC e modulação. O
**conteúdo** é `[tamanho dos metadados][metadados JSON][dados]` — os metadados carregam o nome, o
perfil, e as flags (zip, clipboard, etc.), de forma extensível sem mudar o formato do sinal.

No estéreo, os blocos são divididos entre os dois canais (dobrando a velocidade). O receptor
remonta os blocos por número de sequência, recupera perdas pela paridade quando possível, e valida
o CRC32 final antes de salvar.

Detalhes completos do formato estão em `ROADMAP.md` e `SoundBridge-HANDOFF.md`.

---

## Estrutura do projeto

```
soundbridge/
├── README.md
├── ROADMAP.md                    # roteiro, achados de engenharia, formato de linha
├── SoundBridge-HANDOFF.md        # contexto completo para retomar o projeto
├── DECISOES-ESTRATEGICAS.md      # decisões e o porquê
├── FREEZE-AMPLITUDE.md           # o modo amplitude v1 (congelado)
├── tx/                           # transmissor (Python)
│   └── soundbridge_tx/ofdm/      # OFDM + FEC (modem, fec, tx, tx_stereo, ...)
├── sim/                          # oráculo / CLI Python
│   ├── make_wav.py               # CLI do transmissor
│   ├── decode_wav.py             # decodificador de referência (arquivo ou --capture)
│   ├── rx.py, fec_rx.py          # decodificador OFDM em Python
│   └── evm_signal.py, measure_evm.py   # medição de qualidade do canal
├── src/                          # receptor (C++ / Qt6)
│   ├── audio/                    # captura, WAV, enumeração de devices
│   ├── decoder/                  # OfdmDecoder, FecDecoder, BlockAssembler
│   ├── protocol/                 # sync, header
│   ├── core/                     # config, settings, logger
│   └── ui/                       # MainWindow, ConfigDialog, ...
└── tools/                        # geradores (sequências OFDM, vetores FEC)
```

---

## Solução de problemas

**O CMake não acha o PortAudio**
Falta a flag `-DVCPKG_TARGET_TRIPLET=x64-windows`, ou o `CMAKE_TOOLCHAIN_FILE` não aponta para o vcpkg.

**`STATUS_DLL_NOT_FOUND` (0xC0000135) ao abrir o app**
O `portaudio.dll` não foi encontrado. Ele é copiado ao compilar; se você moveu o `.exe`, copie o
`portaudio.dll` junto (está em `vcpkg/installed/x64-windows/bin/` ou `debug/bin/`).

**Aviso "Configuração incompatível com o Windows" ao clicar INICIAR**
Esperado. O Windows exige que o app use o mesmo sample rate configurado no painel de som. Clique em
"Abrir Config do Windows", ajuste o sample rate para 48000 Hz, e clique INICIAR de novo.

**Canais L e R desbalanceados**
Quase sempre é efeito de áudio no PC transmissor. Veja "Preparar o PC transmissor" e desative os
aprimoramentos e o som espacial. Para isolar, teste com loopback no mesmo PC (saída → entrada): se
balancear, o problema é o transmissor.

**Ruído alto quando os dois PCs estão ligados na tomada**
Loop de terra entre as tomadas (zumbido de 50/60 Hz). O SNR normalmente ainda basta para decodificar.
Se atrapalhar, alimente um dos PCs pela bateria (notebook) ou use um isolador de áudio.

**Modo WAV recusa o arquivo**
Formatos suportados: PCM 16/24/32-bit ou Float 32-bit. Arquivos comprimidos (MP3/ADPCM em container
WAV) não funcionam. Exporte como "WAV PCM".

---

## Como funciona por dentro (camada física)

O SoundBridge combina várias técnicas de comunicação digital para transmitir dados por áudio de
forma confiável. Em resumo, cada peça e seu papel:

- **FFT / IFFT — o motor (tempo ↔ frequência).** A Transformada de Fourier converte entre o sinal de
  áudio (amostras no tempo) e os dados organizados por frequência. A IFFT monta o som no transmissor;
  a FFT o separa de volta no receptor. É o que torna o OFDM possível e rápido o suficiente para
  rodar em tempo real.

- **OFDM — a estrutura (subportadoras paralelas).** Em vez de uma portadora rápida, divide a banda
  em ~200 subportadoras lentas em paralelo (como o Wi-Fi ou o ADSL). Cada uma carrega um pouco dos
  dados. Se algumas frequências forem ruins, só aquelas se perdem — o resto chega. Robusto a canais
  irregulares como o áudio.

- **QAM — a modulação (bits → pontos).** Cada grupo de bits vira um ponto num plano (amplitude +
  fase). QPSK = 2 bits/ponto, 16-QAM = 4, 64-QAM = 6, 256/1024-QAM = 8/10. Mais bits por ponto =
  mais rápido, mas os pontos ficam mais próximos e o ruído confunde mais. O 64-QAM é o equilíbrio.

- **Pilotos — correção de fase fina.** Subportadoras com valores conhecidos, espalhadas no sinal.
  O receptor as usa como referência para corrigir, símbolo a símbolo, a distorção de fase que o
  canal introduz.

- **FEC — correção de erros e perdas (Viterbi + interleaver + paridade).** Envia redundância para o
  receptor corrigir erros sozinho, sem retransmitir. O **código convolucional + Viterbi** corrige
  erros espalhados; o **interleaver** transforma rajadas de erro em erros espalhados (que o Viterbi
  corrige); a **paridade** (estilo RAID) recupera blocos inteiros perdidos. É o que faz o arquivo
  chegar íntegro apesar do ruído do cabo.

- **Resync — anti-drift.** Os dois PCs têm relógios levemente diferentes, e o desalinhamento
  acumula ao longo da transmissão. O transmissor insere marcas de sincronização periódicas; o
  receptor as usa para se re-alinhar e zerar o desvio acumulado. Sem isso, transmissões longas
  quebrariam no meio.

- **CRC — verificação final.** Uma "impressão digital" (CRC32) do arquivo, calculada no transmissor
  e recalculada no receptor. Se batem, o arquivo está garantidamente íntegro (byte a byte); se não,
  o receptor sabe que algo corrompeu e não entrega dados errados.

A qualidade do canal de áudio foi medida com uma métrica chamada **EVM** (o quanto os pontos QAM
chegam deslocados do ideal), que guiou as escolhas do projeto — como usar a entrada de linha e o
64-QAM como padrão. Detalhes completos em `ROADMAP.md`.

---

## Licença

Projeto acadêmico / pessoal.
