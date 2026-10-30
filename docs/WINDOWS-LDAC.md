# LDAC no Windows 11 — QCY H3S

**Status: ✅ funcionando (25/10/2026)** — LDAC **990 kbps / 96 kHz / 24-bit**.

| Item | Valor |
|------|-------|
| SO | Windows 11 Pro 26H2 (build 26300) |
| Dongle | BARROT Bluetooth 6.0 `33fa:0012` (UGREEN CM748), driver `11.8.41.844` |
| Driver A2DP | Alternative A2DP Driver `1.8.3.1` (Luculent Systems) |
| Fone | QCY H3S `84:AC:60:05:55:2C` |
| Codec ativo | LDAC — Dual Channel, 96 kHz, 24 bit, 990 kbps |

---

## Por que precisa de driver

O Windows só negocia **SBC** e **AAC** no A2DP. LDAC é licenciado pela Sony e a
Microsoft não inclui no sistema. O **Alternative A2DP Driver** substitui o driver
A2DP inbox e adiciona LDAC, aptX, aptX HD e aptX LL.

- Site: <https://www.bluetoothgoodies.com/a2dp/>
- Trial: 7 dias. Licença perpétua ~US$ 10 (ou ~US$ 13 com AAC).
- Trial expirado: o fone conecta mas **não sai som** até comprar ou desinstalar.
- Suporta oficialmente até Windows 11 **25H2**; funcionou no 26H2 (build 26300).

## Passo a passo

1. **Habilitar LDAC no fone pelo app oficial QCY (Android/iOS).**
   É um gatilho de firmware: sem ele o fone anuncia só SBC+AAC e a opção LDAC
   fica **cinza** no driver. Não tem como fazer isso só pelo PC (ver seção SPP).
2. Instalar o Alternative A2DP Driver (MSI, pede UAC).
3. `AltA2dpConfig.exe` → selecionar o **QCY H3S** → **Device Driver =
   Alternative A2DP Driver** → **Install**.
4. **CODEC Type = LDAC**, marcar **96 kHz**, **24 bit**, **Dual Channel** e
   **Encode Quality = HQ (High)** (990 kbps).
5. Clicar **(Re)connect device** — as configurações só valem na reconexão.

> Deixar "Automatically reduce the encoding quality when the radio quality is
> poor" (ABR) marcado é recomendado: se o rádio piorar, ele cai para 660/330 em
> vez de picotar.

## Verificação objetiva (registro do driver)

O driver expõe tudo em:

```
HKLM\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices
├── Capability\<mac>   -> o que o FONE anuncia
└── Current\<mac>      -> o que está sendo USADO agora
```

`Codec` é bitmask: **1 = SBC, 2 = AAC, 4 = LDAC** (aptX em diante nos outros bits).

```powershell
$base = 'HKLM:\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices'
Get-ItemProperty "$base\Capability\000084ac6005552c" | Select Codec, LdacChannelMode, LdacSamplingFrequency
Get-ItemProperty "$base\Current\000084ac6005552c"    | Select Codec, LdacEqmid, LdacSamplingFrequency, LdacSampleFormat
```

Valores observados no teste:

| Chave | Valor | Significado |
|-------|-------|-------------|
| `Capability\...\Codec` | `0x3` | antes: só SBC+AAC (LDAC cinza) |
| `Capability\...\Codec` | `0x7` | depois do gatilho no app: SBC+AAC+**LDAC** |
| `Current\...\Codec` | `4` | LDAC em uso |
| `Current\...\LdacEqmid` | `0` | HQ = 990 kbps (`2` = MQ = 330) |
| `Current\...\LdacSamplingFrequency` | `4` | 96 kHz |
| `Current\...\LdacSampleFormat` | `2` | 24 bit |

## Pegadinha: o dongle Barrot

O fabricante do driver lista os adaptadores **BARROT 5.4/6.0** (vendidos como
UGREEN CM748) como problemáticos: o adaptador/driver limita SBC/AAC a **menos de
200 kbps** e pode não sustentar bitrates altos (aptX HD ~600, LDAC ~660/990).

No teste com o `33fa:0012`:

- Em AAC o driver reportou **140–185 kbps** e mostrou os avisos
  *"high rate of delayed Bluetooth packet transmission"* e
  *"unusually long delay ... often caused by a bug in the Bluetooth adapter"*.
- Em **LDAC 990 kbps** os avisos não voltaram após a reconexão, mas o dongle
  continua sendo o elo fraco.

Se aparecer picote ou o bitrate cair sozinho (ABR):

1. Baixar para **SQ/HQ (660)** ou **MQ (330)**;
2. aproximar o fone do PC / evitar USB 3.0 e Wi-Fi 2.4 GHz perto do dongle;
3. trocar por um dongle **Realtek RTL8761B** (ex.: TP-Link UB500) — barato e
   conhecido por segurar LDAC melhor que o Barrot.

## Reverter

- **Só o fone:** Device Driver = **Windows Standard Driver** → Install.
- **Tudo:** desinstalar "Alternative A2DP Driver" em Apps → restaura o driver
  inbox do Windows.

## SPP no Windows não funciona (e provavelmente nunca funcionou no Linux)

Tentativas feitas nesta sessão:

- Porta `COM6` ("Serial Padrão por link Bluetooth") abre, mas
  `ConnectionCount = 0` → o RFCOMM nunca conecta de fato e os bytes não chegam.
- `socket(AF_BTH, SOCK_STREAM, BTHPROTO_RFCOMM)` + `connect()` no canal 1
  retorna `WSA 10049/10050` mesmo com o rádio ligado.
- Nenhuma resposta a `FF 03 FE 01 30` (request de versão) e o comando de ANC
  (`FF 03 0C 01 04`) não teve efeito audível.

Conclusão: o canal de controle real do app QCY **não é** o SPP "cru" que o
`bin/qcy-ctl` usava. É RCSP/GATT (com handshake) — próximo alvo do toolkit.

---

## Estado 29/10/2026 - A2DP cai para HFP e driver AltA2DP falha

Sintoma: audio do H3S sai em **Hands-Free (HFP)** em vez de A2DP, e o
microfone do fone fica aberto como endpoint de captura ativo ("microfone ta
aberto? ta apenas handsfree pq").

Causa raiz medida (nao teorica):

1. `HKLM\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices\Current\<mac>`
   - `Codec=0`, `Opened=0`, **`Error=1`**. `Capability.Codec=3` (SBC+AAC),
     ou seja o SDP negocia, mas o driver **nao abre o stream**.
2. Apagar as chaves `Current` e `Capability` do device e reiniciar o servico
   `AltA2DP` regenera tudo - e ele falha igual (`Error=1`). Nao e corrupcao
   de registro.
3. Com o A2DP sem abrir, o Windows cai no HFP: endpoints
   `Headset (QCY H3S Hands-Free)` ficam `Active` (render **e** capture) e o
   endpoint A2DP `Fones de ouvido (QCY H3S)` fica `Unplugged`.

Isto coincide com a lista de adaptadores problematicos do fabricante do
driver: o BARROT `33fa:0012` esta nela.

Comandos que funcionam para ressuscitar o link classico (ACL) quando o fone
esta ligado e o BLE responde mas nada conecta:

```powershell
Restart-Service BTAGService -Force
Start-Service BluetoothUserService_<sid>      # fica Stopped e o A2DP nao sobe sem ele
python C:\Users\Deivi\AppData\Local\Temp\opencode\bt_service.py   # BluetoothSetServiceState toggle A2DP/HFP
```

Diagnostico BLE rapido: `BleakScanner.discover()` deve achar
`84:AC:60:F8:A6:BF`. Se retornar 0 dispositivos, o radio esta travado -
destravar com `python .../radio_toggle.py` (WinRT `Radio.SetStateAsync`),
nao com PowerShell (WinRT nao funciona no pwsh 7 aqui).

Decisao pendente: se apos reboot limpo o `Error=1` persistir, remover
`oem2.inf` (`pnputil /delete-driver oem2.inf /uninstall`) devolve o A2DP
nativo do Windows (AAC/SBC), ao custo do LDAC.

Backups salvos no Desktop: `AltA2DP-Current-backup.reg`,
`AltA2DP-Devices-backup.reg`.
