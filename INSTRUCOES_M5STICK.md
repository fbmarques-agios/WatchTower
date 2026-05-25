# WatchTower no M5StickC Plus 1.1

Adaptacao do projeto [WatchTower](https://github.com/emmby/WatchTower) (transmissor
WWVB de 60 kHz) para rodar no **M5StickC Plus 1.1** (ESP32-PICO-D4, 4 MB flash).

## O que foi alterado em relacao ao projeto original

| Arquivo | Mudanca |
|---|---|
| `platformio.ini` | Novo env `m5stickc_plus` (board `m5stick-c` + variante `m5stack_stickc_plus`, particao `huge_app.csv`). Definido como `default_envs`. |
| `platformio.ini` | Adicionada a biblioteca `M5Unified` (so para o M5Stick) para usar o LCD. |
| `WatchTower.ino` | `PIN_ANTENNA` mudou de GPIO13 para **GPIO26** (GPIO13 e usado pelo LCD do M5Stick). |
| `WatchTower.ino` | `timezone` mudou para `"BRT3"` (horario de Brasilia, UTC-3, sem horario de verao). |
| `WatchTower.ino` | Adicionado suporte ao **LCD**: inicializa o display (e o backlight via AXP192) e mostra hora, data e IP na telinha. |
| `WatchTower.ino` | Constante `ANTENNA_DRIVE_LEVEL` (0-3): ajusta a forca de saida do GPIO da antena por software. |

O M5StickC Plus nao tem LED NeoPixel — o codigo ja trata isso (`#ifdef PIN_NEOPIXEL`).
O status agora aparece em 3 lugares: **LCD do M5Stick**, monitor serial e interface web.

## 1. Gravar o firmware

Conecte o M5StickC Plus no PC via USB-C e descubra a porta serial:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

Se der erro de permissao na porta, libere o acesso (some quando voce reconecta
o aparelho):

```bash
sudo chmod 666 /dev/ttyUSB0
```

Para resolver de forma permanente (precisa deslogar/relogar depois):

```bash
sudo usermod -aG dialout $USER
```

Grave (o PlatformIO detecta a porta automaticamente):

```bash
cd /home/fbmarques/Projetos/radio-tower
~/.platformio/penv/bin/pio run -e m5stickc_plus -t upload
```

Se a porta nao for detectada, force com:
`~/.platformio/penv/bin/pio run -e m5stickc_plus -t upload --upload-port /dev/ttyUSB0`

> Isto **substitui o ESP32 Marauder**. Para voltar ao Marauder, basta regravar
> ele pelo M5Burner.

## 2. Acompanhar pelo monitor serial (opcional)

```bash
~/.platformio/penv/bin/pio device monitor -e m5stickc_plus
```

(115200 baud) — mostra a conexao WiFi, o sync NTP e o sinal sendo transmitido.

## 3. Configurar o WiFi (primeira vez)

1. No primeiro boot, o M5Stick cria uma rede WiFi chamada **`WatchTower`**
   (a tela mostra "Configure o WiFi").
2. Conecte o celular nessa rede — abre um portal automatico.
3. Escolha sua rede WiFi de casa e digite a senha.
4. O aparelho reinicia, conecta e sincroniza a hora via NTP (`pool.ntp.org`).
5. Acesse **http://watchtower.local** (ou o IP mostrado na tela) para ver o
   painel web completo (hora, forma de onda WWVB, uptime, ultimo sync).

## 4. O que aparece no LCD

- **Cabecalho**: "WatchTower" + indicador "TX" em vermelho.
- Durante o boot: mensagens de status ("Iniciando", "Configure o WiFi",
  "Sincronizado", etc.).
- Em operacao: **data**, **hora grande** (atualiza a cada segundo) e o **IP**.

## 5. Montar a bobina (antena)

O firmware gera uma portadora de 60 kHz (onda quadrada 3,3 V) no **GPIO26**.
A "antena" e uma bobina ligada entre **G26 e GND** (header superior da placa).

```
  G26 ---(bobina)--- GND
```

Enrole o fio numa bobina (**nunca** um fio reto de G26 a GND). Quanto mais
voltas, mais forte o campo: com fio comum mire em **50+ voltas**; com fio
esmaltado fino da para por 100-200. O alcance e de poucos centimetros —
encoste o relogio na bobina.

### Forca de saida — `ANTENNA_DRIVE_LEVEL` (0 a 3)

No topo do `WatchTower.ino`, a constante `ANTENNA_DRIVE_LEVEL` controla a
forca do sinal:

| Nivel | Corrente | Uso |
|---|---|---|
| 0 | ~5 mA  | Mais fraco, mais seguro |
| 1 | ~10 mA | Intermediario |
| 2 | ~20 mA | Padrao — forca normal de um GPIO (sem resistor, OK para teste) |
| 3 | ~40 mA | Maximo — **so com resistor ~100 ohm em serie** |

Sem resistor, use no maximo o nivel 2. Para o nivel 3:

```
  G26 ---[ resistor ~100 ohm ]---(bobina)--- GND
```

Depois de mudar a constante, recompile e regrave (secao 7).

## 6. Sincronizar o relogio

So funciona com relogios que recebem a banda **WWVB (EUA, 60 kHz)**. Relogios
que so pegam DCF77 (Europa) ou JJY (Japao) **nao** sincronizam com este sinal.

### Relogio alvo confirmado: Citizen H874 (manual em `p.pdf`)

O Citizen H874 e multibanda e recebe WWVB (estacao C, Fort Collins). Pontos
importantes do manual:

- **Estacao escolhida pelo fuso horario**: fora do Japao o H874 nao escolhe a
  estacao sozinho. Configure o relogio num fuso dos EUA para ele escutar WWVB.
  O **UTC-3 (Brasilia)** ja e mapeado para "Estados Unidos"/WWVB — fuso correto
  e hora correta ao mesmo tempo.
- **Antena do relogio fica na posicao das 9 horas** — encoste esse lado na bobina.
- **Recepcao manual**: coroa na posicao 0 -> segurar o botao inferior direito (A)
  por 2 s -> ponteiro vai para "RX" -> deixar parado sobre a bobina (2 a 30 min).
- **Resultado**: coroa na posicao 0 -> apertar o botao A -> ponteiro mostra
  "OK" ou "NO".
- **Horario de verao**: deixar em "STD MA" (horario padrao / manual), porque o
  firmware transmite sempre "sem horario de verao".
- E Eco-Drive (solar): carregue o relogio antes; com carga baixa nao ha recepcao.

A transmissao de um quadro completo de hora/data leva 60 segundos.

## 7. Recompilar depois de editar o codigo

```bash
cd /home/fbmarques/Projetos/radio-tower
~/.platformio/penv/bin/pio run -e m5stickc_plus            # so compila
~/.platformio/penv/bin/pio run -e m5stickc_plus -t upload  # compila e grava
```

## Observacoes

- **Legal**: a potencia de uma bobina de poucos cm de alcance e desprezivel;
  fica muito abaixo de qualquer limite de dispositivo de baixa potencia.
- **Fuso horario**: o sinal WWVB carrega UTC + flags de horario de verao dos EUA.
  Com `timezone = "BRT3"` o firmware encoda "sem horario de verao", entao um
  relogio configurado em UTC-3 fixo mostra a hora de Brasilia corretamente.
- **HackRF / SDR / Evil Crow RF nao servem**: WWVB esta em 60 kHz. O HackRF
  opera de 1 MHz a 6 GHz e os chips CC1101 do Evil Crow RF de 300-928 MHz —
  ambos muito acima. Sinais de relogio radiocontrolado (40-77,5 kHz) exigem
  acoplamento magnetico por bobina; o ESP32 + bobina e a abordagem correta.
