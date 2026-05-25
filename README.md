# WatchTower — Fork para M5StickC Plus 1.1

> Transmissor WWVB caseiro: faz o seu relógio radiocontrolado pegar a hora certa **sem precisar do sinal real** de Fort Collins (EUA).

Este é um fork do projeto [emmby/WatchTower](https://github.com/emmby/WatchTower), adaptado para rodar no **M5StickC Plus 1.1** (ESP32-PICO-D4, 4 MB de flash). Foi montado e testado no Brasil, com foco em sincronizar o **Citizen H874** (Eco-Drive multibanda) — mas funciona com qualquer relógio que receba a banda WWVB (60 kHz).

> 📄 A versão original em inglês deste README foi escrita para o build com Adafruit QT Py ESP32 + H-bridge DRV8833. Para esse caminho, consulte o [README do projeto upstream](https://github.com/emmby/WatchTower/blob/main/README.md). Este README cobre o caminho **M5StickC Plus**.

---

## Como funciona, em poucas palavras

```
NTP (pool.ntp.org)
   ↓
ESP32 mantém o relógio interno (UTC)
   ↓
A cada segundo, calcula o bit WWVB daquele segundo do quadro de 60 s
   ↓
Modula em largura de pulso uma portadora de 60 kHz no GPIO26
   ↓
Bobina ligada ao pino gera um campo magnético de 60 kHz
   ↓
O relógio (Citizen, Casio multibanda…) capta o campo na própria
antena de ferrite interna e decodifica hora + data + ano
```

É **acoplamento magnético de campo próximo** — não é uma "antena de rádio" no sentido comum. Alcance: poucos centímetros a algumas dezenas, dependendo da bobina.

---

## O que este fork adiciona ao upstream

| Mudança | Por que |
|---|---|
| Novo ambiente PlatformIO `m5stickc_plus` | board `m5stick-c` + variante `m5stack_stickc_plus`, partição `huge_app.csv` (3 MB de app, sem OTA) para caber em 4 MB de flash |
| Suporte ao LCD via M5Unified | Mostra cabeçalho, status de boot, data, hora e IP na telinha (240×135) |
| Pino da antena em **GPIO26** (era 13 no upstream) | GPIO13 no M5Stick é usado pelo LCD interno |
| Constante `ANTENNA_DRIVE_LEVEL` (0–3) | Ajusta por software a corrente de saída do GPIO — permite testar bobina sem resistor em série |
| Timezone padrão `BRT3` (Brasília, sem horário de verão) | Brasil mapeia para a estação WWVB nos relógios multibanda |
| Documentação em português + receitas para o **Citizen H874** | Esta README + procedimentos específicos do calibre |
| Git hooks (`pre-commit` build, `pre-push` build + testes) | Mesmo padrão do projeto irmão `soc-bot-abasp` |

O env original do projeto (`adafruit_qtpy_esp32`) está preservado em `platformio.ini` e continua funcionando — quem tiver a placa Adafruit não perde nada.

---

## Hardware

### Versão mínima (para teste)

- **M5StickC Plus 1.1** (ESP32-PICO-D4, com LCD e WiFi embutidos)
- **Bobina** ligada entre `G26` e `GND` (qualquer fio, 10–30 voltas para começar — ver seção sobre antena)
- Cabo USB-C para alimentar e gravar
- (Opcional) Cabos jumper macho-macho para a conexão

### Versão definitiva (planejada)

- **Amplificador ponte-H** — TB6612FNG ou DRV8833 entre o GPIO e a bobina (empurra o sinal com muito mais força)
- **Antena de ferrite de rádio AM** (bastão ~100 mm com bobina já enrolada de fábrica)
- **Placa ilhada** para a montagem soldada permanente
- **Caixa "torre" impressa em 3D** (o repositório upstream tem o STL para a versão QT Py — uma caixa específica para o M5Stick precisa ser projetada à parte)
- Ferro de solda

---

## Software necessário

- [PlatformIO Core](https://platformio.org/) (instalado em `~/.platformio/penv/`)
- Git

Toda a toolchain do ESP32 é baixada automaticamente pelo PlatformIO na primeira compilação.

---

## Clonando e configurando

```bash
git clone https://github.com/fbmarques-agios/WatchTower.git
cd WatchTower

# Ativa os hooks de git deste repositório (build pre-commit, build + testes pre-push)
git config core.hooksPath .githooks
```

---

## Compilação e gravação

A porta serial do M5StickC Plus normalmente aparece como `/dev/ttyUSB0`. Se ela estiver bloqueada por permissão, libere temporariamente com:

```bash
sudo chmod 666 /dev/ttyUSB0
```

(Para resolver de vez, `sudo usermod -aG dialout $USER` e relogue.)

```bash
PIO=~/.platformio/penv/bin/pio

# Só compilar
$PIO run -e m5stickc_plus

# Compilar e gravar
$PIO run -e m5stickc_plus -t upload

# Monitor serial (115200 baud)
$PIO device monitor -e m5stickc_plus

# Rodar os testes nativos (Unity, no PC)
$PIO test -e native
```

A gravação **substitui** o firmware que estiver na placa (por exemplo, o ESP32 Marauder). Para voltar, basta regravar via M5Burner.

---

## Primeira configuração (WiFi)

1. No primeiro boot, o M5Stick mostra **"Configure o WiFi"** na tela e cria uma rede WiFi chamada **`WatchTower`** (aberta).
2. Conecte o **celular** nessa rede — um portal automático abre.
3. Selecione a sua rede WiFi de casa e digite a senha.
4. O M5Stick reinicia, conecta, e sincroniza a hora via NTP (`pool.ntp.org`).
5. A telinha passa a mostrar data, hora e IP. O painel web fica em `http://watchtower.local` (ou pelo IP mostrado na tela).

A configuração de WiFi fica salva — só faz isso uma vez (até trocar de rede).

---

## O que aparece no LCD

```
┌──────────────────────────────┐
│ WatchTower               TX  │  ← cabeçalho ciano + indicador vermelho
│ ──────────────────────────── │
│ dom 24/05/2026               │  ← data (branco)
│ 14:32:18                     │  ← hora grande (verde, atualiza por segundo)
│ 192.168.0.170                │  ← IP do M5Stick na sua rede (ciano)
└──────────────────────────────┘
```

Durante o boot, em vez da data/hora aparecem mensagens de status como `Iniciando…`, `Configure o WiFi rede: WatchTower`, `Conectando…`, `Sincronizado! Transmitindo…`.

---

## A antena de transmissão (bobina)

A "antena" do WatchTower é uma **bobina** ligada entre **G26 e GND** do M5Stick. Quanto mais voltas e melhor o núcleo magnético, mais forte o campo e maior a chance de o relógio decodificar.

### Versão simples (para o primeiro teste)

Pegue qualquer fio (jumper, fio de cabo velho), raspe as duas pontas até aparecer o cobre, e enrole **10 a 30 voltas** numa caneta, lápis ou pilha AA. Ligue uma ponta em `G26`, a outra em `GND`.

```
  G26 ───(bobina)─── GND
```

⚠️ **Nunca** ligue um fio reto de G26 a GND. Isso é um curto e estressa o pino. Sempre enrolado.

### Versão melhor (com bastão de ferrite)

Bobina **com núcleo de ferrite** é o divisor de águas — concentra muito mais o campo magnético.

Atalho barato: qualquer **rádio AM/MW velho** tem dentro uma "antena de ferrite" — um bastão com fio de cobre enrolado, com 2-4 fios saindo. É uma bobina pronta. Ligue as duas pontas da bobina maior (a com mais voltas) em `G26` e `GND`.

Ou compre nova no Mercado Livre por R$ 10–30: pesquise por `bastão de ferrite com bobina` ou `antena ferrite rádio AM`.

### Constante `ANTENNA_DRIVE_LEVEL` (0–3)

No topo de [`WatchTower.ino`](WatchTower.ino), a constante `ANTENNA_DRIVE_LEVEL` define a força de saída do GPIO:

| Nível | Corrente | Uso |
|---|---|---|
| 0 | ~5 mA  | Mais fraco, mais seguro |
| 1 | ~10 mA | Intermediário |
| 2 | ~20 mA | Padrão de um GPIO do ESP32 |
| 3 | ~40 mA | Máximo — **só com resistor ~100 Ω em série** OU com bobina de alta indutância (como a de ferrite) |

Sem resistor, fique no máximo no nível 2 com bobina improvisada. A bobina de ferrite tem indutância suficiente para limitar a corrente sozinha — com ela, o nível 3 é seguro permanentemente.

Após mudar a constante, recompile e regrave (`pio run -e m5stickc_plus -t upload`).

---

## Painel web

Quando o M5Stick está conectado ao WiFi, acesse **http://watchtower.local** (ou o IP mostrado na tela) pelo navegador.

O painel mostra:

- Hora atual (com fuso e timezone)
- Data por extenso
- Janela de transmissão (os 60 bits do quadro WWVB, com a posição atual)
- Último sync NTP
- Uptime do dispositivo

---

## Relógios compatíveis

Funciona com qualquer relógio que receba a banda **WWVB (EUA, 60 kHz)** — tipicamente os "multibanda" 5 ou 6 (que pegam várias estações pelo mundo).

| Marca / linha | Banda WWVB? |
|---|---|
| Casio Waveceptor / G-Shock Multiband 6 | ✅ |
| Citizen multibanda (atomic timekeeping) | ✅ |
| Junghans Mega Solar | ✅ |
| Seiko Astron (radiocontrolados) | ✅ |
| Relógios só DCF77 (Europa) | ❌ frequência diferente (77,5 kHz) |
| Relógios só JJY (Japão) | ❌ frequência diferente (40 ou 60 kHz mas formato diferente) |

### Citizen H874 (testado neste fork)

O **Citizen H874** é um calibre Eco-Drive (alimentado por luz) com recepção radiocontrolada multibanda. Recebe **5 estações** em 4 regiões do mundo — uma delas é a estação **WWVB (Fort Collins, EUA)**, que é a que este projeto transmite.

#### Configurações necessárias no relógio

1. **Fuso horário em UTC-3** (Brasília). No H874, o fuso é selecionado pela posição do **ponteiro dos segundos** num modo específico:
   - Coroa na posição 1
   - Gire a coroa até o ponteiro dos segundos apontar para a posição **57** (= UTC-3)
   - Empurre a coroa para a posição 0

   Esta é a posição correta porque o H874 escolhe a estação pelo fuso horário, e os fusos americanos (incluindo o UTC-3 do Brasil) ficam mapeados para a estação WWVB.

2. **Horário de verão em "STD MA"** (horário padrão / manual). Como o firmware transmite sempre "sem horário de verão" (consistente com a regra atual do Brasil), o "STD MA" garante que o relógio ignore o flag de DST e mostre sempre o horário padrão.

3. **Estado de carga OK**. O Eco-Drive precisa estar bem carregado — se o ponteiro dos segundos andar 1 vez a cada 2 segundos, a recepção **não roda**. Deixe o relógio na luz forte por algumas horas antes de testar.

#### Antena interna fica nas 9 horas

A antena de ferrite do H874 fica na **lateral esquerda do mostrador**, na posição das 9 horas. Para o melhor acoplamento, deite o relógio sobre a sua bobina de transmissão com o lado das 9 horas alinhado com a bobina.

#### Procedimento de recepção manual

1. Coroa na posição 0.
2. **Segure o botão A** (inferior direito) por **2 a 3 segundos** e solte.
3. O ponteiro dos segundos deve **pular para a posição "RX"** e parar de andar normalmente. Se ele continuar marcando a hora, a recepção **não começou** — tente segurar por mais tempo.
4. **Coloque o relógio sobre a bobina, lado das 9 horas alinhado**, e **não mova**. A recepção leva de 2 a 30 minutos. Quando termina, o ponteiro dos segundos volta ao normal.
5. **Confira o resultado**: coroa na posição 0, **toque rápido no botão A** (apertar e soltar). O ponteiro dos segundos vai apontar para **OK** (sucesso) ou **NO** (falha).

---

## Problemas comuns

### O relógio sincronizou só a hora, a data continua errada

Isso quase sempre é **posição de referência do calendário desalinhada** — o relógio recebeu a data correta, mas o ponteiro do dia está num "zero" deslocado. Solução no próprio relógio (manual H874, pág. 9):

1. Coroa na posição 0.
2. **Segure o botão B** (superior direito) por **10 segundos ou mais**. Os ponteiros vão se mover para as posições de referência salvas em memória.
3. Confira se cada um está na posição correta:

| Indicador | Deve apontar para |
|---|---|
| Fase da lua | Lua cheia |
| Ponteiro de função (sub-mostrador) | "S" (Sunday / domingo) |
| Indicação da data | Meio caminho entre 31 e 1 |
| Ponteiros das horas, minutos, segundos | 0h 00min 0s (todos para o 12) |
| Ponteiro das 24 horas | 24 |

4. Se tudo certo, aperte B uma vez para confirmar.
5. Se algum estiver errado: puxe a coroa para a posição 2, aperte B para selecionar o alvo a corrigir (sequência: fase da lua → ponteiro de função/data → horas/min/24h → segundos), gire a coroa para acertar cada um, empurre a coroa para 0 e aperte B para confirmar.

Depois disso, refaça a recepção manual.

### O resultado dá NO

O sinal está fraco demais para o relógio decodificar. Em ordem de impacto:

1. **Bobina ruim** — poucas voltas ou sem núcleo de ferrite. Resolva com mais voltas (50+) ou trocando para uma bobina de ferrite de rádio AM.
2. **Posição** — o ferrite interno do relógio (nas 9 horas) precisa estar perto e alinhado com a sua bobina. Tente girar o relógio.
3. **Carga baixa** no Eco-Drive — recarregue na luz forte.
4. **Drive level baixo** — suba o `ANTENNA_DRIVE_LEVEL` (lembrando do limite de corrente sem amplificador).

### Pino do M5Stick estressado

Se você ficou no `ANTENNA_DRIVE_LEVEL = 3` com bobina de fio comum por horas, o GPIO pode ter degradado. Sinais: o sinal sumiu mesmo recompilando, ou a transmissão ficou intermitente. Solução: trocar o pino (existem outros GPIOs livres no header — basta mudar `PIN_ANTENNA` no código) ou trocar o M5Stick.

### A telinha fica preta

Se o firmware não estiver inicializando o LCD (bug, build errado), confira pelo monitor serial se o ESP32 está bootando. Tela preta sem ESP32 também responder = M5Stick travado ou descarregado. Tela preta com serial ativo = bug no display init — abra uma issue.

---

## Estrutura do projeto

```
.
├── WatchTower.ino              # Firmware: setup, loop, encoder WWVB, LCD
├── platformio.ini              # Envs de build (m5stickc_plus, adafruit_qtpy_esp32, native)
├── customJS.h                  # JavaScript customizado injetado no ESPUI
├── test/test_native/           # Testes Unity (rodam no PC)
├── test/mocks/                 # Mocks de Arduino/ESP/WiFi/ESPUI usados pelos testes
├── enclosure/                  # STL e .f3d da caixa (upstream, formato para QT Py)
├── docs/                       # Imagens usadas pelo README original
├── .githooks/                  # pre-commit e pre-push (build + testes)
├── CLAUDE.md                   # Guia para Claude Code (arquitetura, gotchas)
└── README.md                   # Este arquivo
```

### Ambientes de build (`platformio.ini`)

| Env | Placa / variante | Para que serve |
|---|---|---|
| `m5stickc_plus` *(padrão)* | `m5stick-c` + variante `m5stack_stickc_plus`, partição `huge_app.csv` | Este fork |
| `adafruit_qtpy_esp32` | `adafruit_qtpy_esp32`, partição `default_8MB.csv` | Build original do upstream |
| `native` | host (Linux/macOS/Win) | Testes Unity do encoder WWVB |

---

## Hooks de git

Configurados em [`.githooks/`](.githooks/):

- **pre-commit** — compila `m5stickc_plus` (cacheado, ~15 s) para pegar erro de sintaxe antes de gravar o commit.
- **pre-push** — compila + roda os 5 testes nativos antes de subir.

Ative em qualquer clone com:

```bash
git config core.hooksPath .githooks
```

Os hooks pulam graciosamente se o PlatformIO não estiver instalado.

---

## Status do projeto

Este fork ainda está em evolução. O **firmware** está estável e validado (5/5 testes nativos passam, build limpo, transmissão WWVB conferida no log serial). A **antena de teste com fio comum** funciona de forma intermitente. Próximos passos:

- [ ] Receber e ligar a antena de ferrite (pendente)
- [ ] Receber o amplificador TB6612FNG e a placa ilhada (pendente)
- [ ] Montagem soldada definitiva (pendente)
- [ ] Projetar uma torre 3D específica para o M5Stick (pendente)
- [ ] Abrir Pull Request opcional ao [upstream](https://github.com/emmby/WatchTower) com o env do M5Stick

---

## Créditos

- Projeto original: [emmby/WatchTower](https://github.com/emmby/WatchTower) — toda a lógica de codificação WWVB, esquema H-bridge e caixa 3D vêm dele. Este fork apenas adapta para outro hardware e adiciona o LCD.
- Fork e adaptação para M5StickC Plus / Brasil: [@fbmarques-agios](https://github.com/fbmarques-agios).
- Bibliotecas: WiFiManager (tzapu), ESPUI (emmby fork), AsyncWebServer (ESP32Async), M5Unified (M5Stack), Adafruit NeoPixel.

---

## Licença

MIT — veja [LICENSE](LICENSE). Mesmos termos do upstream.

> ⚖️ Sobre legalidade: o WWVB real opera com licença federal nos EUA. A regulamentação da FCC isenta transmissores de 60 kHz desde que o campo elétrico seja menor que 40 μV/m a 300 metros — e uma bobina de poucos centímetros de alcance fica ordens de grandeza abaixo disso. Para uso doméstico no Brasil, a potência aqui envolvida está bem abaixo de qualquer limiar regulatório de dispositivo de baixa potência.
