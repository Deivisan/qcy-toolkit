# QCY Toolkit — Linux + Windows (QCY H3S e compatíveis)

Ferramentas para tirar o máximo de fones QCY (H3S testado) **sem depender do app Android**.

**Status (2026-10-25):** ✅ **LDAC 990 kbps no Windows 11** ([docs/WINDOWS-LDAC.md](./docs/WINDOWS-LDAC.md)) · ✅ **Mic OpenWhispr + BrowserOS** (A2DP música + HFP ao gravar) · ✅ **Protocolo completo documentado** (72 Cmd IDs, 25 UUIDs, 2 rotas de transporte) · ⚠️ **SPP no Windows não conecta** (controle real é RCSP/GATT)

**Protocolo (engenharia reversa):** [analysis/QCY-H3S-PROTOCOL-COMPLETO.md](./analysis/QCY-H3S-PROTOCOL-COMPLETO.md) — capturado de 3 versões do APK oficial (682/689/715).

Microfone / voz: [docs/MIC-SETUP-FINAL.md](./docs/MIC-SETUP-FINAL.md) — `./scripts/qcy-install-mic-setup.sh`

Windows / qualidade máxima: [docs/WINDOWS-LDAC.md](./docs/WINDOWS-LDAC.md) — LDAC 990 kbps / 96 kHz / 24-bit via Alternative A2DP Driver.

Firmware (o que dá / o que não dá): [docs/FIRMWARE-JL7018.md](./docs/FIRMWARE-JL7018.md) — SoC JieLi JL7018F6, OTA RCSP, container `.UFW` e onde estão os bloqueios reais. Formatos binários (JLFS, `.ufw`/`.bfu`, chipkey, UBOOT): [docs/FIRMWARE-FORMATOS.md](./docs/FIRMWARE-FORMATOS.md).

- GATT vendor (0000a001/00001001) **não é exposto** pelo BlueZ neste dispositivo.
- Todo o controle real acontece por **SPP/RFCOMM** (UUID 00001101, canal 1).
- Framing 0xFF idêntico ao que mapeamos do APK oficial + Quicky + Jieli RCSP.

---

## Requisitos

- Bluetooth conectado (cabo USB **removido** — o cabo desliga o rádio BT interno do fone).
- `bluez-libs` instalado (para compilar o sender se necessário).
- Binário pronto: `bin/qcy-spp-raw` (já compilado e testado).

---

## Uso rápido (mais simples)

```bash
# ANC
./bin/qcy-ctl anc on
./bin/qcy-ctl anc trans
./bin/qcy-ctl anc off

# Game / Low Latency
./bin/qcy-ctl game on
./bin/qcy-ctl game off

# Volume
./bin/qcy-ctl volume 70

# LDAC (cuidado — costuma reiniciar o fone)
./bin/qcy-ctl ldac off

# Música
./bin/qcy-ctl music play
./bin/qcy-ctl music next

# Bateria (usa bluetoothctl padrão)
./bin/qcy-ctl battery
```

---

## Via Bun / TypeScript (CLI antigo atualizado)

```bash
bun run src/cli/ble.ts battery
bun run src/cli/ble.ts anc on
bun run src/cli/ble.ts latency on
bun run src/cli/ble.ts volume 60 60
bun run src/cli/ble.ts music next
```

O cliente agora delega automaticamente para o sender SPP que funciona.

---

## Como o controle realmente funciona

- `bin/qcy-spp-raw` (C) abre socket RFCOMM direto com BlueZ (`AF_BLUETOOTH`, `BTPROTO_RFCOMM`).
- Envia pacotes no formato exato que o fone espera:
  ```
  0xFF  <body_len>  <cmd>  <param_count>  [params...]
  ```
- Comandos principais já mapeados e testados em envio:
  - `0x0C` — ANC / Transparency (simples)
  - `0x17` — ANC avançado (modo + subScene + noiseValue)
  - `0x09` — Low Latency / Game Mode
  - `0x08` — Volume
  - `0x23` — LDAC
  - `0x04` — Music control
  - etc.

---

## Compilando o sender (se precisar)

```bash
gcc -o bin/qcy-spp-raw scripts/qcy-spp-raw.c -lbluetooth
```

---

## Estado atual (resumo técnico)

- Cabo USB = áudio estável (UAC1 + HID). Não mexer.
- Bluetooth A2DP = funciona normalmente.
- Controle de features (ANC, Game, Volume, LDAC...) = **funcionando via SPP**.
- Validação auditiva dos comandos enviados ainda pendente do usuário (ANC realmente muda o ruído? Transparency funciona? Game tem efeito? Volume obedece?).

---

## Próximos passos (após seu feedback)

1. Confirmar quais comandos surtem efeito real no hardware.
2. Treinar ANC avançado (0x17), GameConfig (0x4A), LDAC ON (com aviso), EQ, etc.
3. Adicionar leitura de respostas/notificações pelo mesmo canal SPP.
4. Integrar em daemon / TUI / GUI se desejado.
5. Documentar tudo que foi validado.

---

**Dica importante:** sempre remova o cabo antes de tentar controle. Com o cabo plugado o rádio Bluetooth interno do fone é desativado pelo firmware.

Qualquer dúvida ou relatório do que você observou nos comandos enviados: é só falar.
