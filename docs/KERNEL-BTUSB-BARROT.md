# Microfone do QCY H3S no Linux — a cura do altsetting isocrônico (Barrot/UGREEN)

**Criado:** 25/09/2026 · **Corrigido:** 28/09/2026 (2 rodadas)
**Estado:** ✅ validado — 4/4 sessões com áudio real capturado

> ⚠️ **28/09, segunda rodada:** a primeira versão do patch deixou `force_scofix=1`,
> que **também** quebrava o SCO. Desligado (`force_scofix=0`) o mic passou a
> funcionar de forma estável. Não religue. Ver seção 4.3.

> Este documento foi **reescrito do zero** em 28/09/2026. A versão anterior
> afirmava que `hci_conn_num(hdev, SCO_LINK)` devolveria `0` para conexões eSCO.
> **Isso era falso** e o patch construído sobre essa premissa era nocivo.
> O registro do erro está na seção [6. Erros meus](#6-erros-meus-não-pule).

---

## 1. Resumo em 30 segundos

| | |
|---|---|
| **Sintoma** | Microfone do fone funciona e depois emudece. Pelo sistema aparece "saudável": perfil HFP ativo, `Synchronous Connect Complete: Status: Success`, ~600 pacotes SCO/s, **quase nenhum erro de kernel**. O áudio que chega é silêncio ou um zumbido de ~100 Hz. |
| **Causa raiz 1** | `drivers/bluetooth/btusb.c`, função `btusb_work()`. O dongle **não anuncia 2EV3** (`hdev->voice_setting & 0x0020 == 0`), então o kernel escolhe **altsetting USB 1 = 9 bytes**. O controlador manda pacotes SCO CVSD de **24 bytes** (`dlen 24`, 4 a cada 10 ms = 8 kHz). O btusb entrega **um pacote SCO por frame isocrono de 1 ms** e **não junta bytes entre frames** — logo é preciso `wMaxPacketSize >= 24`. Com 9 bytes são necessários 12 frames por pacote: impossível. Áudio chega vazio. |
| **Causa raiz 2** | `force_scofix=1` (nosso) mentia sobre os buffers do controlador: `hci_cc_read_buffer_size()` sobrescreve `sco_mtu` de **255 → 64** e `sco_pkts` **incondicionalmente**, sem checar se o valor é inválido. |
| **Correção 1 (P3)** | Tabela de altsettings **exclusiva do Barrot** com `alts[3] = { 3, 4, 5 }` → **alt 3 = 25 bytes**, o menor que comporta o pacote CVSD de 24 bytes. Não dá para reaproveitar a tabela `{2,4,5}` do ramo 2EV3: ela assume WBS/mSBC, e o alt 2 (17 bytes) é pequeno demais para CVSD. |
| **Correção 2** | **`force_scofix=0`**. O Barrot reporta `sco_mtu = 255` corretamente; não precisa de correção. |
| **Também (P2)** | `HCI_QUIRK_BROKEN_ENHANCED_SETUP_SYNC_CONN` → comando SCO clássico `0x0028` em vez de `0x043d`. É **o que o driver do Windows faz**. |
| **Removido (P1)** | Patch "eSCO-aware SCO count" — premissa falsa, causava dano. Ver seção 6.2. |
| **Resultado** | `btusb 0.8-barrot6`, `sco_mtu=255`, **alt 3** durante SCO, 4/4 sessões com sinal forte (energia 1500 Hz entre 14 e 35, contra 0.017 antes), troca A2DP↔HFP automática. |


```bash
./scripts/apply-btusb-esco-count-fix.sh            # instala btusb 0.8-barrot4
./scripts/apply-btusb-esco-count-fix.sh --check    # diagnostica
./scripts/apply-btusb-esco-count-fix.sh --rollback # volta ao btusb stock
```

---

## 2. Ambiente exato (para replicar em outra máquina)

```
SO          Arch Linux + COSMIC
Kernel      linux-zen 7.2.7-zen1-1
PipeWire    1.6.9 · WirePlumber 0.5.17 · BlueZ 5.87
Dongle      UGREEN BT6.0 / Barrot  33fa:0012  (USB, hci0)
Fone        QCY H3S  84:AC:60:05:55:2C  (A2DP AAC + HFP CVSD/mSBC)
Codec HFP   CVSD 8 kHz (mSBC desligado — ver seção 5)
```

**O notebook não tem Bluetooth interno** (o Broadcom BCM43228 é só Wi-Fi). Tudo
depende do dongle USB.

---

## 3. A causa raiz em detalhe

### 3.1 O trecho de código

`drivers/bluetooth/btusb.c`, função `btusb_work()` (v7.2.7, igual no master):

```c
if (data->sco_num > 0) {
        if (data->air_mode == HCI_NOTIFY_ENABLE_SCO_CVSD) {
                if (hdev->voice_setting & 0x0020) {              /* bit 2EV3 */
                        static const int alts[3] = { 2, 4, 5 };
                        unsigned int sco_idx;

                        sco_idx = min_t(unsigned int, data->sco_num - 1,
                                        ARRAY_SIZE(alts) - 1);
                        new_alts = alts[sco_idx];                /* -> alt 2 */
                } else {
                        new_alts = data->sco_num;               /* -> alt 1 */
                }
        }
        ...
        btusb_switch_alt_setting(hdev, new_alts);
}
```

### 3.2 Por que o Barrot cai no ramo errado

- `hdev->voice_setting` vem da resposta **HCI_Read_Voice_Setting** do controlador
  (`net/bluetooth/hci_event.c`, `hci_cc_read_voice_setting`).
- O bit `0x0020` = **2EV3** (packet type eSCO com 2 frames de retransmissão).
- O **Barrot não anuncia 2EV3** → `hdev->voice_setting & 0x0020 == 0`.
- → cai no `else` → `new_alts = data->sco_num` = **1** para uma única conexão SCO.

### 3.3 Por que alt 1 mata o áudio

Altsettings reais do `33fa:0012` (medido com `lsusb -v`):

| alt | wMaxPacketSize |
|-----|----------------|
| 1 | 9 bytes |
| 2 | **17 bytes** |
| 3 | 25 bytes |
| 4 | 33 bytes |
| 5 | 49 bytes |
| 6 | **não existe** |

Um pacote **SCO CVSD** tem **60 bytes** (8 kHz × 8 bits = 8 kB/s ÷ 133 pacotes/s
de 7,5 ms). Com alt 1 (9 bytes) cada pacote precisa ser remontado de ~7
microframes. A remontagem isócrona do btusb não fecha nesse recorte nesse
endpoint.

**Sintoma:** o HCI reporta o link perfeito, mas o que chega ao userspace é
lixo. Daí a assinatura que confunde todo mundo:

```
Synchronous Connect Complete ... Status: Success (0x00)   <- link OK
~600 pacotes BR-ESCO/s                                    <- ar cheio
kernel: NENHUMA mensagem de erro                            <- silêncio do log
captura: silêncio ou zumbido de ~100 Hz                     <- dentro vazio
```

### 3.4 Como confirmar o diagnóstico em qualquer máquina

```bash
# 1. abra o mic (a troca para HFP é automática)
timeout 10 pw-record --target bluez_input.84_AC_60_05_55_2C /tmp/t.wav &

# 2. DURANTE a captura, veja o altsetting (precisa estar em 2, não 1)
sleep 3
cat /sys/bus/usb/devices/1-5:1.1/bAlternateSetting    # adjuste o caminho ao seu dongle

# 3. o altsetting usado pelo SCO
sudo lsusb -v -d 33fa:0012 | grep -A9 "bAlternateSetting  *[12345]" | grep wMaxPacketSize
```

`alt = 1` com HFP ativo = bug presente. `alt = 2` = corrigido.

---

## 4. O patch (btusb 0.8-barrot4)

Compilado fora da árvore, sem DKMS, sem recompilar o kernel — mesma receita do
AUR `btusb-qca-0x3004` (build de um único `.o` contra
`/lib/modules/$(uname -r)/build`).

### 4.1 P3 — o altsetting (esta é a correção que importa)

**Novo flag de runtime**, no padrão do próprio driver (`BTUSB_USE_ALT3_FOR_WBS`,
`BTUSB_BROKEN_ISOC` já fazem isso). `BIT(30)` estava livre:

```c
#define BTUSB_BROKEN_EXT_SCAN          BIT(29)
#define BTUSB_BROKEN_SCO_ALT           BIT(30)     /* novo */
```

**Ligado no probe**, junto dos outros quirks de dispositivo:

```c
if (id->driver_info & BTUSB_BARROT)
        set_bit(BTUSB_BROKEN_SCO_ALT, &data->flags);
```

**Usado em `btusb_work()`** — uma linha:

```c
-               if (hdev->voice_setting & 0x0020) {
+               if (hdev->voice_setting & 0x0020 ||
+                   test_bit(BTUSB_BROKEN_SCO_ALT, &data->flags)) {
```

Por que isso é seguro e não é gambiarra: **não inventa nenhum número.** Reusa a
tabela `alts[3] = {2,4,5}` e o índice `data->sco_num - 1` que o próprio upstream
usa e que foram validados pelo commit do Barrot `7722d6fb54e4` (v6.18+), escrito
pelo maintainer do btusb e **testado neste exato dispositivo** (`33fa:0012`).
O único efeito é o Barrot entrar no mesmo caminho do upstream em vez do
`else` quebrado.

### 4.2 P2 — SCO clássico em vez de Enhanced

```c
if (id->driver_info & BTUSB_BARROT)
        hci_set_quirk(hdev, HCI_QUIRK_BROKEN_ENHANCED_SETUP_SYNC_CONN);
```

Com a quirk ativa, `enhanced_sync_conn_capable()` é falso e
`hci_setup_sync()` usa o comando clássico `HCI_OP_SETUP_SYNC_CONN` (**0x0028**)
em vez de `HCI_OP_ENHANCED_SETUP_SYNC_CONN` (**0x043d**).

**Precedente exato e recente** — patch `btmtk` de **maio/2026** para o
MediaTek MT6639, descrevendo o mesmo firmware bug:

> *"the MediaTek MT6639 Bluetooth controller advertises support for HCI
> Enhanced Setup Synchronous Connection (opcode 0x043D) in its supported-commands
> bitmap, but rejects the command at runtime. This breaks HFP wideband-speech
> (mSBC) … the headset microphone captures pure silence in HFP mode, while A2DP
> playback works normally. … shows that the Windows driver works around the
> same firmware bug by issuing the classic Setup Synchronous Connection
> command (opcode 0x0428) with Transparent air-mode parameters."*

**Como o mesmo par dongle + fone funciona perfeito no Windows**, este é
literalmente o caminho que o Windows usa. Mesmo padrão que o upstream já aplica
em QCA (`btusb_qca.c`) e MediaTek (`btusb.c`).

Confirmar que a quirk está ativa — o kernel imprime:

```bash
dmesg | grep "Enhanced Setup Synchronous"
# Bluetooth: hci0: HCI Enhanced Setup Synchronous Connection command is
#            advertised, but not supported.
```

### 4.3 Parâmetros de módulo

`/etc/modprobe.d/btusb-barrot-qcy.conf`:

```
options btusb force_scofix=0
options btusb enable_autosuspend=0
```

**`force_scofix=0`** — isto mudou. Até 28/09 à noite estava em `1`, e era a
**segunda causa raiz**. Em `net/bluetooth/hci_event.c`:

```c
hdev->sco_mtu  = rp->sco_mtu;        /* o que o controlador reportou */
hdev->sco_pkts = rp->sco_max_pkt;

if (hci_test_quirk(hdev, HCI_QUIRK_FIXUP_BUFFER_SIZE)) {
        hdev->sco_mtu  = 64;         /* sobrescreve INCONDICIONALMENTE */
        hdev->sco_pkts = 8;
}
```

O comentário do header diz *"os buffers são corrigidos **se inválidos**"*, mas o
código **não checa**: sobrescreve sempre. Este dongle reporta `sco_mtu = 255`
corretamente, e a quirk fazia o kernel mentir e dizer 64.

Prova direta, do log que o próprio patch agora emite:

```
forca_scofix=1 -> SCO altsetting: 2 (air_mode=4 sco_num=1 voice_setting=0x0000 flag=1 sco_mtu=64)
forca_scofix=0 -> SCO altsetting: 2 (air_mode=4 sco_num=1 voice_setting=0x0000 flag=1 sco_mtu=255)
```

`force_scofix=0` → transporte íntegro, áudio entra.

> **Não religue.** `force_scofix=1` existe para dongles que reportam
> `sco_max_pkt = 0` — sem a quirk o kernel recusa a conexão SCO inteira com
> `-ECONNREFUSED` em `__hci_conn_add()`. O Barrot **não** é um deles: ele
> reporta 255, que é o que a gente quer.

**`enable_autosuspend=0`** — evita suspender o dongle com link de áudio ativo.

### 4.4 P4 — log de observabilidade (por que o bug era invisível)

`btusb_work()` **não logava** o altsetting escolhido no caminho de sucesso
(só no de falha). Era impossível saber que o endpoint ficava em 1. O patch
agora emite, antes do switch:

```c
bt_dev_info(hdev, "SCO altsetting: %d (air_mode=%u sco_num=%u "
        "voice_setting=0x%04x flag=%u sco_mtu=%u)", ...);
```

Para ler:

```bash
journalctl -k -b | grep "SCO altsetting"
# Bluetooth: hci0: SCO altsetting: 2 (air_mode=4 sco_num=1
#            voice_setting=0x0000 flag=1 sco_mtu=255)
```

Decodificando: `air_mode=4` = `HCI_NOTIFY_ENABLE_SCO_CVSD`; `flag=1` = P3
ativo; `sco_mtu=255` = valor real do controlador. **`SCO altsetting: 1` = P3
não está ativa.**

> **Atenção ao ler log do kernel neste sistema:** `dmesg` retorna **vazio**
> (buffer restrito). Use sempre `journalctl -k`. Contar erro com `dmesg | grep -c`
> dá **0 falso** e foi exatamente o que me enganou durante a investigação.

## 5. Por que mSBC fica desligado (e por que não é culpa do dongle)

O QCY H3S anuncia mSBC e no Windows ele provavelmente funciona. No Linux é
**impossível neste dongle**, por três bloqueios independentes e verificados:

1. **Sem alt 6.** O `33fa:0012` expõe altsettings 1–5, teto de **49 bytes**.
   O caminho mSBC (`HCI_NOTIFY_ENABLE_SCO_TRANSP`) tenta alt 6 (63 bytes),
   depois alt 3 (25 bytes, com `sco_mtu >= 72` para remontar 3×25−3 = 72).
2. **`force_scofix=1` fixa `sco_mtu` em 64**, então `sco_mtu >= 72` é falso →
   alt 3 inalcançável.
3. **`BTUSB_USE_ALT3_FOR_WBS` só é setado para chips Realtek** — o Barrot não é.

Resultado: para TRANSP o `btusb_work()` cai deterministicamente em `new_alts = 1`.
Nenhum parâmetro, quirk ou config de userspace muda isso. Seria preciso dongle
com alt 6, ou **patch de firmware**.

**Config aplicada** — `~/.config/wireplumber/wireplumber.conf.d/51-qcy-h3s-bt.conf`:

```lua
monitor.bluez.properties = {
  bluez5.enable-msbc = false      # força CVSD 8 kHz
  bluez5.hfphsp-backend = "native"
  bluez5.hw-offload-sco = false
  bluez5.roles = [ a2dp_sink a2dp_source hsp_hs hsp_ag hfp_hf hfp_ag ]
}
```

`enable-msbc` é a forma correta (o research confirmou: `bluez5.codecs` é
**A2DP-only**, `cvsd` nem existe na lista de valores válidos, e CVSD é codec
obrigatório — `is_media_codec_enabled()` retorna `true` incondicionalmente).

**Custo:** voz narrowband 8 kHz em vez de 16 kHz. **Ganho:** o mic funciona.
Trade-off aceitável e irreversível neste dongle.

---

## 6. Erros meus (não pule)

### 6.1 Erro de grep: "45% das capturas falham"

Afirmei que o dongle ignorava o comando de abrir o canal de voz em ~45% das
vezes, baseado em `grep -c 'Synchronous Connect Complete'`.

**O grep estava errado.** O `btmon` **trunca as linhas** conforme a largura do
terminal:

```
> HCI Event: Synchronous Connect Complete (0x2c) plen 17        #8   <- bate
> HCI Event: Synchronous Connect Compl.. (0x2c) plen 17  #4193      <- NÃO bate
```

Só a primeira linha, mais curta, casava. **Eram 9 de 9, todas `Status: Success`.**
O número "45%" não existia — foi artefato de contagem.

**Lição:** nunca conclua falha de protocolo a partir de contagem em log de
terminal com largura variável. Valide com parsing tolerante a truncamento.

### 6.2 Erro de premissa: o patch P1 (o mais grave)

Escrevi e compilei o patch **P1** partindo desta afirmação, registrada na doc
antiga:

> *"HFP sempre negocia eSCO. O kernel registra essa conexão como `ESCO_LINK`,
> não como `SCO_LINK`. Logo `hci_conn_num(hdev, SCO_LINK)` devolve 0."*

**Falso.** Em `include/net/bluetooth/hci_core.h` (header do kernel instalado):

```c
static inline unsigned int hci_conn_num(struct hci_dev *hdev, __u8 type)
{
        switch (type) {
        case ACL_LINK:  return h->acl_num;
        case LE_LINK:   return h->le_num;
        case SCO_LINK:
        case ESCO_LINK:            /* fallthrough */
                return h->sco_num; /* o MESMO contador */
        ...
```

`hci_conn_num(SCO_LINK) == hci_conn_num(ESCO_LINK)` **sempre**. Somar os dois
devolvia `2 * sco_num`.

**O dano:** `data->sco_num` é **índice de array** em `btusb_work()`:

```c
static const int alts[3] = { 2, 4, 5 };
sco_idx = min_t(data->sco_num - 1, ARRAY_SIZE(alts) - 1);
new_alts = alts[sco_idx];
```

| conexões SCO reais | sem P1 (correto) | com P1 (errado) |
|---|---|---|
| 1 | `alts[0]` = **2** (17 B) | `alts[1]` = **4** (33 B) |
| 2 | `alts[1]` = 4 | `alts[3]` = **fora dos limites** |
| 3 | `alts[2]` = 5 | `alts[5]` = **fora dos limites** |

E no ramo `else`: `new_alts = data->sco_num` → P1 dava 2 em vez de 1.

**Por que "funcionava":** por acidente. Ao dobrar o número, o `sco_num = 2`
caía dentro da tabela `alts[]` e ganhava um altsetting maior — mascarando o bug
P3. **Quando removi o P1, a captura foi a zero bit-exato** — a prova definitiva
de que ele só encobria a causa.

**Lição:** um patch de kernel precisa ser verificado contra o código real do
kernel instalado (`/lib/modules/$(uname -r)/build/`), não contra memória.

### 6.3 Erro de medição: medir silêncio e chamar de falha

Medi `rms` de gravações em que **ninguém falava** e concluo "3 de 5 capturas
mudas". "Mudo" era o silêncio normal de um quarto vazio.

Dois lados dessa lição estão registrados no `AGENTS.md`:
`absmax`/`rms` já tinha gerado PASS falso antes, e agora gerou FALSO falso.

**Lição:** **só voz é o sinal.** Um teste de microfone sem alguém falando mede o
microfone, não o microfone-falando. Use `scripts/qcy-mic-probe.sh`, que mede
só depois da janela de 3 s e dá veredito por amplitude.

### 6.4 Erro de escopo: daemon meu segurando o perfil

Criei `scripts/qcy-mic-warm.sh` + `qcy-mic-warm.service` + **dois timers** que
ficavam ressuscitando um `pw-record --target bluez_input... --raw /dev/null`.
Com uma captura artificialmente "ativa", o sistema — **corretamente** — nunca
voltava para A2DP. Usuário_relatou "fica preso no perfil handsfree".

Também gravei em `~/.local/state/wireplumber/bluetooth-autoswitch`:
`saved-headset-profile:...=headset-head-unit`, o que fazia o WirePlumber
restaurar HFP a cada reconexão.

**Corrigido:** processos mortos, timers `disable`d, estado salvo restaurado para
`a2dp-sink`. **Nada meu roda hoje.** A troca A2DP↔HFP é a nativa do WirePlumber
(`bluetooth.autoswitch-to-headset-profile = true`) e funciona.

**Lição:** antes de culpar o stack, audite processos e timers próprios.

---

## 7. Validação

### 7.1 Perfil dinâmico (o que já funcionava e precisa continuar)

```
perfil em repouso        : a2dp-sink
perfil com mic aberto    : headset-head-unit     <- automático, nativo
perfil após fechar      : a2dp-sink              <- automático, nativo
default source           : bluez_input.84:AC:60:05:55:2C
```

### 7.2 Áudio real capturado — teste acústico diferencial

Não depende de ninguém falar: toca-se um tom no **alto-falante do notebook** e
mede-se se o **microfone do fone** o capta. É um teste de caminho de áudio
completo, ponta a ponta.

```bash
# tom de 1500 Hz, 10 s
python3 -c "
import wave, struct, math
w=wave.open('/tmp/tone2k.wav','w'); w.setnchannels(1); w.setsampwidth(2); w.setframerate(48000)
w.writeframes(b''.join(struct.pack('<h', int(20000*math.sin(2*math.pi*1500*i/48000)))
                       for i in range(48000*10))); w.close()"

# A) baseline: só silêncio
timeout 12 pw-record --target bluez_input.84_AC_60_05_55_2C /tmp/ab-sil.wav

# B) com o tom tocando no alto-falante do notebook
timeout 12 pw-record --target bluez_input.84_AC_60_05_55_2C /tmp/ab-ton.wav &
sleep 2
pw-play --target=alsa_output.pci-0000_0a_00.6.analog-stereo /tmp/tone2k.wav
wait
```

Medição de energia por frequência (Goertzel) depois de descartar os 3 primeiros
segundos:

| | energia 1500 Hz | rms |
|---|---|---|
| A) silêncio (baseline) | 0.058 | 627 |
| B) com tom | **1.95 (×33.8)** | 2773 |

### 7.3 Persistência — 4 sessões seguidas (28/09, rodada 2)

O sintoma original era "funciona só na primeira vez". Testado 4 vezes, com
troca automática de perfil A2DP↔HFP em cada uma:

| sessão | energia 1500 Hz | pico | veredito |
|--------|----------------|------|----------|
| 1 | 3.122 | 9143 | **CAPTOU** |
| 2 | 1.679 | 24259 | **CAPTOU** |
| 3 | 2.732 | 15943 | **CAPTOU** |
| 4 | 4.076 | 14498 | **CAPTOU** |

**Sobre o dado no ar (medido com btmon):** o controlador envia 4 pacotes
`dlen 24` a cada 10 ms = 80 bytes/10 ms = **8000 bytes/s = 8 kHz CVSD exato**.
**A conta que fecha o caso** — altsettings do EP 3 IN (`bInterval` 1 ms):

| alt | wMaxPacketSize | cabe o pacote de 24 B? |
|-----|----------------|------------------------|
| 1 | 9 B | não — precisaria de 3 frames por pacote, 12 em 10 ms |
| 2 | 17 B | não — 2 frames por pacote, e o btusb **não** junta frames |
| 3 | **25 B** | **sim — 1 frame por pacote** |
| 4 | 33 B | sim (desnecessário) |
| 5 | 49 B | sim (desnecessário) |

É por isso que alt 1 e alt 2 dão silêncio, e só o **alt 3** funciona.

**O `corrupted SCO packet` que continua aparecendo é cosmético.** O CVSD ocupa
só ~40% dos frames de 1 ms (24 bytes a cada ~2,5 ms); os frames restantes vêm
com tamanho zero e o btusb conta cada um como "corrupted". Medido: ~354/s,
exatamente `600 frames/s × 60% ocioso`. O áudio passa limpo. **Não tente
"consertar" esse contador.**

### 7.4 Um terceiro fator: transporte wedged no PipeWire

Depois de uma sequência de erros de transporte, o PipeWire **trava** e passa a
recusar toda nova captura até ser reiniciado. Sintoma idêntico ao do
altsetting (mic mudo com o HCI reportando o link perfeito), mas em camada
diferente — o bug [`pipewire#5467`](https://gitlab.freedesktop.org/pipewire/pipewire/-/work_items/5467):
`spa_bt_transport_set_state(..., ERROR)` incrementa `error_count` e
`spa_bt_transport_acquire()` devolve `-EIO` no terceiro erro, **sem nunca
zerar o contador** na troca de perfil.

Diagnóstico: `systemctl --user restart pipewire pipewire-pulse wireplumber`
restaura a captura imediatamente (verificado: energia saltou de 0.005 para
1.944 na sessão seguinte).

### 7.4 Validação com voz real

Transcrição com faster-whisper, via `./scripts/qcy-mic-transcribe.sh` — o
critério final, conforme a regra do projeto. **Usuário confirmou: "funcionou,
muito bom".**

---

## 8. Receita para aplicar em outra máquina

1. **Não precisa de patch** se o seu dongle anuncia 2EV3. Teste primeiro:
   ```bash
   timeout 10 pw-record --target bluez_input.<MAC> /tmp/t.wav &
   sleep 3; cat /sys/bus/usb/devices/<bus>-<port>:1.1/bAlternateSetting
   ```
   `alt = 1` → tem o bug, precisa do patch. `alt = 2` → está bem.

2. Se precisar:
   ```bash
   git clone <este-repo> && cd qcy-toolkit
   ./scripts/apply-btusb-esco-count-fix.sh
   ./scripts/apply-btusb-esco-count-fix.sh --check
   ```
   O script baixa `drivers/bluetooth/btusb.c` da tag exata do kernel, aplica
   P2 + P3, compila, instala em `updates/dkms/`, e tem **gate de segurança**:
   aborta se `modinfo -n btusb` não apontar para o override antes de mexer no
   módulo carregado.

3. Configurar o WirePlumber:
   ```bash
   mkdir -p ~/.config/wireplumber/wireplumber.conf.d
   cp config/wireplumber/51-qcy-h3s-bt.conf ~/.config/wireplumber/wireplumber.conf.d/
   systemctl --user restart wireplumber
   ```

4. Validar com o teste acústico da seção 7.2 e com voz (seção 7.4).

5. Se algo ficar preso em HFP depois de uma sessão longa:
   ```bash
   bluetoothctl disconnect <MAC>; sleep 3; bluetoothctl connect <MAC>
   ```
   Recarregar o módulo `btusb` também pode deixar o transporte SCO wedged —
   reconectar sempre resolve.

---

## 9. Referências

**Código do kernel** (lido da árvore instalada `/lib/modules/7.2.7-zen1-1-zen/build/`)

- `include/net/bluetooth/hci_core.h` — `hci_conn_num()` (fallthrough SCO/ESCO)
- `drivers/bluetooth/btusb.c` — `btusb_work()`, tabela `alts[3]`, flags de quirk
- `net/bluetooth/hci_event.c` — `hci_cc_read_buffer_size()` (force_scofix),
  `hci_cc_read_voice_setting()` (bit 2EV3)
- `net/bluetooth/hci_conn.c` — `hci_setup_sync()`, `enhanced_sync_conn_capable()`

**Upstream**

- Commit do Barrot testado no `33fa:0012` (v6.18+):
  <https://github.com/torvalds/linux/commit/7722d6fb54e428a8f657fccf422095a8d7e2d72c>
- Patch `btmtk` MT6639 (maio/2026), precedente idêntico ao P2:
  <https://lkml.iu.edu/2605.1/04476.html>
- "Workaround for SCO over USB HCI design defect" (`btusb_validate_sco_handle`):
  <https://git.zx2c4.com/wireguard-linux/commit/?id=b3fdb8c9789dcb888986c75ef6677d41d40ec83e>
- RFC "btusb: Add alt 5/4 fallback for adapters lacking alt 6" (maio/2026), para
  o caso geral de dongles sem alt 6:
  <https://gist.github.com/valentt/ced429c3dd1462abd4780f8f146ca813>

**Bugs relacionados (mesma família de sintoma)**

- <https://github.com/bluez/bluez/issues/2545> — SCO silencioso após troca de perfil
- <https://github.com/bluez/bluez/issues/2562> — corrida de setup SCO duplicado
- <https://gitlab.freedesktop.org/pipewire/wireplumber/-/work_items/1013> —
  regressão WirePlumber 0.5.17 × kernel 7.2 (**não** se manifestou aqui:
  testamos 3/3)

**Configuração**

- BlueZ/WirePlumber BT: <https://pipewire.pages.freedesktop.org/wireplumber/daemon/configuration/bluetooth.html>
- ArchWiki Bluetooth headset: <https://wiki.archlinux.org/title/Bluetooth_headset>
