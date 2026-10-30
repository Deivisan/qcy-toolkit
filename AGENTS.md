# AGENTS.md — QCY Toolkit (QCY H3S, Linux + Windows)

## Visão Geral
Projeto de engenharia reversa e ferramentas para fones QCY (H3S testado).
Repo: **`qcy-toolkit`** (<https://github.com/Deivisan/qcy-toolkit>, renomeado de
`qcy-ble-linux` em 25/10/2026 — o GitHub redireciona o nome antigo).

- Linux: `bin/qcy-ctl` + `scripts/` + CLI Bun (ANC, volume, música, game, LDAC).
- Windows: LDAC 990 kbps funcional via Alternative A2DP Driver — ver
  `docs/WINDOWS-LDAC.md`.

## Windows (estado em 25/10/2026)
- Windows 11 Pro 26H2 (build 26300), dongle BARROT 6.0 `33fa:0012` (UGREEN CM748).
- **LDAC 990 kbps / 96 kHz / 24-bit OK** com Alternative A2DP Driver 1.8.3.1
  (trial até 01/11/2026; licença ~US$10).
- O LDAC só aparece no driver depois de habilitar **no app QCY do celular**
  (gatilho de firmware). Verificação objetiva:
  `HKLM\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices\{Capability,Current}\<mac>`
  — `Codec` é bitmask (1=SBC, 2=AAC, 4=LDAC).
- **SPP não funciona no Windows**: porta COM6 abre mas `ConnectionCount=0`;
  socket RFCOMM direto dá `WSA 10049/10050`. Os comandos `FF 03 23 01 01` etc.
  **não chegam no fone** — e provavelmente nunca foram validados no Linux
  (o próprio README dizia "validação auditiva pendente"). O controle real do app
  é RCSP/GATT com handshake.
- Barrot está na lista de adaptadores problemáticos do fabricante do driver
  (capa SBC/AAC <200 kbps). Se picotar em LDAC: cair para 660/330 ou trocar por
  dongle Realtek RTL8761B.

## Stack Atual
> ⚠️ **Atualizado 25/09/2026.** O AGENTS.md original descrevia um ambiente que
> **não existe mais**. Realidade verificada nesta data:

- **CLI:** Binário C (`bin/qcy-spp-raw`) + wrapper bash (`bin/qcy-ctl`)
- **TypeScript/Bun:** `src/cli/ble.ts` delegando para `bin/qcy-ctl`
- **Hardware:** QCY H3S (`84:AC:60:05:55:2C`) + dongle UGREEN Barrot `33fa:0012` (`hci0`)
- **SO:** **Arch Linux** (não é mais CachyOS) + **COSMIC** (não é mais KDE Plasma)
- **Kernel:** **`linux-zen 7.2.7-zen1-1`** (não é mais `linux-cachyos 7.0.11`)
- **Áudio:** PipeWire + WirePlumber 0.5.17, BlueZ 5.87
- **Runtimes:** Bun 1.3.x, Node 26, Python 3.14, Rust

### Kernel btusb é item obrigatório, não opcional
O mic **depende** de um `btusb` patcheado (`0.8-barrot4`) em
`/lib/modules/<kernel>/updates/dkms/btusb.ko`. Ele é perdido a cada troca de
kernel/distro e foi justamente o que ficou ausente por 3 meses (ver
`TENTATIVAS-SEM-SUCESSO.md` item 17).

**Causa raiz (corrigida 28/09/2026):** o Barrot `33fa:0012` não anuncia 2EV3,
então `btusb_work()` escolhia **altsetting USB 1 (9 bytes)** para pacotes SCO
CVSD de **60 bytes** → remontagem isócrona quebrada → `corrupted SCO` e áudio
mudo **com o HCI reportando o link como perfeito** (`Status: Success`, ~600
pkt/s, zero erro de kernel). O patch força o Barrot a usar a tabela de
altsettings do próprio upstream → **alt 2 (17 bytes)**.

```bash
./scripts/apply-btusb-esco-count-fix.sh            # (re)instala
./scripts/apply-btusb-esco-count-fix.sh --check    # confere
cat /sys/module/btusb/version                      # tem que dizer 0.8-barrot4
```

**Segunda causa raiz,também nossa:** `force_scofix=1` fazia o kernel
sobrescrever `sco_mtu` de 255 → 64 (o código sobrescreve
incondicionalmente, apesar do comentário dizer "se inválido"). **Não
religue.** O Barrot reporta 255 corretamente. Prova: o log do driver
mostra `sco_mtu=64` (quebrado) vs `sco_mtu=255` (funcional).

**Como saber se está tudo certo** (não confie só no version):
```bash
timeout 10 pw-record --target bluez_input.84_AC_60_05_55:2C /tmp/t.wav &
sleep 3; cat /sys/bus/usb/devices/1-5:1.1/bAlternateSetting   # tem que ser 2
```
`alt = 1` com HFP ativo = **mic vai ficar mudo**.

Detalhes completos, erros do caminho errado e receita para outras máquinas:
**`docs/KERNEL-BTUSB-BARROT.md`**.

### Regras de Contributors
- **NUNCA** religar `bluez5.enable-msbc` — não é limitação do dongle, é
  topologia: o `33fa:0012` não tem alt 6 (teto alt 5 = 49 bytes) e
  `BTUSB_USE_ALT3_FOR_WBS` só existe para chips Realtek, então mSBC cairia
  deterministicamente em alt 1. Codec negocia, áudio chega mudo. Ver
  `docs/KERNEL-BTUSB-BARROT.md` seção 5.
- **Validar áudio por TRANSCRIÇÃO**, nunca por `absmax`/`rms`. Já gerou PASS
  falso (item 19) **e** FALSO falso (item 22 — medir microfone sem ninguém
  falar). Use `./scripts/qcy-mic-transcribe.sh`, ou o teste acústico
  diferencial da doc (tom no alto-falante do notebook medido no mic do fone).
- **NUNCA** deixar daemon/timer próprio segurando captura de microfone. Já
  prendeu o perfil em HFP por dias (item 23). A troca A2DP↔HFP é nativa do
  WirePlumber (`bluetooth.autoswitch-to-headset-profile = true`) e funciona.
  Se precisar reverter algum dia: `bluetoothctl disconnect; connect`.
- **`dmesg` retorna VAZIO neste sistema.** Use `journalctl -k -b`.
  `dmesg | grep -c "corrupted SCO"` dá 0 falso — foi o que me enganou.
- **Nunca confiar em `grep -c` de log do btmon** para contar eventos de
  protocolo — o btmon trunca linhas conforme a largura do terminal e já
  fabricou um "45% de falha" que não existia (item 20).
- **Antes de patchar o kernel, ler o código instalado** em
  `/lib/modules/$(uname -r)/build/`. Um patch anterior partiu de uma premissa
  falsa sobre `hci_conn_num()` e causou dano real (item 21).
- Advice de KDE neste repo está obsoleto — o DE é **COSMIC**. Os erros do
  `cosmic-settings-daemon` no journal são de textura/tema, não de áudio.



## Histórico de Commits (últimos 15)
```
79090d3 feat(mic): config completa QCY H3S — OpenWhispr, BrowserOS, mSBC, autoswitch
302aeb9 docs: rewrite QCY H3S report with honest current status
0fbc537 docs: create comprehensive QCY H3S Bluetooth issue report for agents
910a663 feat: integrate proven SPP/RFCOMM control channel (bin/qcy-spp-raw + bin/qcy-ctl wrapper)
8dbd5f6 feat(BLE): implementação completa do protocolo QCY via BlueZ D-Bus
564d572 docs: correção do relatório de travamento - segundo incidente pós-reboot
4ed13a2 docs: relatório completo de investigação do travamento 2026-06-12 ~20:54
9e6db4a docs: add QCY H3S black product image and fix gitignore to allow docs/images
65d5bbd feat: initial release — CLI BLE + protocol docs
8dbd5f6 feat(BLE): implementação completa do protocolo QCY via BlueZ D-Bus
58b8542 feat(RE): análise estática do APK QCY 4.0.7-689 e captura BLE
51bc742 feat(Core): estrutura inicial bun+typescript implementada
```

## Documentação Capturada (via APK Reverse Engineering)

| Arquivo | Conteúdo | Status |
|---------|----------|--------|
| `analysis/QCY-H3S-PROTOCOL-COMPLETO.md` | Tabela completa de Cmd IDs (72 IDs), UUIDs GATT (25), modos ANC (2 rotas), formatação de pacote, diferenças entre builds 682/689/715. **Gerado via análise de smali**. | ✅ Concluído |
| `docs/MIC-SETUP-FINAL.md` | Configuração WirePlumber + OpenWhispr + BrowserOS para mic. | ✅ Concluído |
| `TENTATIVAS-SEM-SUCESSO.md` | Histórico de falhas e lições aprendidas (anti-repetição). | ✅ Concluído |
| `ESTADO_ATUAL.md` | Status ao vivo (15/06/2026): A2DP OK, mic via mSBC/HFP, controle SPP OK. | ⚠️ Atualização pendente (último commit 2 meses) |
| `docs/MIC-QUALITY.md` | Qualidade de áudio/mic. | ✅ Concluído |
| `docs/FIRMWARE-JL7018.md` | Viabilidade de firmware: SoC JL7018F6, OTA RCSP, se é assinado, container `.ota`, como o arquivo oficial se obtém. | ✅ Concluído (30/10/2026) |
| `docs/FIRMWARE-FORMATOS.md` | Formatos binários JieLi: JLFS v2, `.ufw`/`.fwsc`/`.bfu`, BANKCB, chipkey/key file, protocolo UBOOT, receita de repack. | ✅ Referência (30/10/2026) |
| `QCY-H3S-BLUETOOTH-ISSUE-REPORT.md` | Histórico. | ⚠️ Desatualizado |

## Protocolos Capturados (engineering reverse)

### Transportes (2 rotas)
1. **SPP/RFCOMM** — UUID `00001101` — já funciona via `bin/qcy-spp-raw`
2. **GATT Write** — Service `0000A001`, Char `0000ae00` — identificado via APK reverse, **não validado no hardware ainda**

### Cmd IDs principais (validados)
| Cmd | Formato | Comando |
|-----|---------|---------|
| `0x04` | `[0x04 0x01 act]` | `music play/pause/next/prev` |
| `0x08` | `[0x08 0x03 L R 0x00]` | `volume L R` |
| `0x09` | `[0x09 0x01\|0x02]` | `game on/off` |
| `0x17` | `[0x17 0x03 mode subSence noiseValue]` | ANC avançado (GATT fallback) |
| `0x0C` | `[0x0C 0x01\|0x00\|0x03\|0x04]` | `anc on/off/out/trans` |
| `0x23` | `[0x23 0x01\|0x00]` | `ldac on/off` |
| `0x20` | `[0x20 ...]` | EQ multi-banda (pendente) |
| `0xFE` | `[0xFE 0x01 cmdID]` | REQUESTDATA — leitura de status |

### UUIDs GATT identificados
- Service: `0000A001-0000-1000-8000-00805f9b34fb`
- Write Char: `0000ae00-0000-1000-8000-00805F9B34FB`
- CCCD/Notify: `00002902-0000-1000-8000-00805f9b34fb`
- Jieli RCSP (v689+): `e49a25e0` / `e49a28e1`
- Custom various: 5f78df94, 9d84b9a3, 6c53db25, etc.

## Pending / To-Do

### Alta prioridade
1. **Validar comandos GATT** — testar escrita nos UUIDs `0000ae00`/`0000A001` via `gatttool` ou `bletool` quando SPP não disponível
2. **Implementar controle EQ** — CMDIDs 0x20/0x22/0x46/0x47 (presets + custom 10 bandas)
3. **ANC avançado 0x17** — validar `mode + subSence + noiseValue` no hardware real
4. **Leitura de respostas** — implementar canal RX no SPP (atualmente só TX)
5. **TUI/GUI** — criar interface front-end para o CLI Bun existente

### Média prioridade
6. **KEYFUN (0x2B)** — mapear funções dos botões físicos do fone
7. **Bateria pela GATT** — usar char `00002a06` em vez de `bluetoothctl`
8. **CMDIDs pendentes** — `0x06`, `0x0A`, `0x11`, `0x1E`, `0x1F`, `0x32`, `0x3A`, `0x3B`, `0x3D`, `0x3E`, `0x3F`, `0x43`, `0x45`, `0xFE` (não mapeados completamente)

### Baixa prioridade
9. **Perfil EasyEffects** — criar perfil DSP usando `~/.config/easyeffects/` sem mexer no firmware
10. **SPACE_AUDIO / ENV_ADAPTATION / INEAR_SENSITIVITY** — comandos ainda não mapeados completamente
11. **TWS_ENABLE / LED_SWITCH / LED_EFFECT** — testar configs extras
12. **Diferenças entre builds 682/689/715** — validar quais mudanças de protocolo são relevantes para o Linux

## Ferramentas e Habilidades Disponíveis
- `bin/qcy-spp-raw` — C sender via RFCOMM (compilado, testado)
- `bin/qcy-ctl` — wrapper bash delegando para spp-raw
- `src/cli/ble.ts` — CLI Bun TypeScript, delegando automaticamente para `qcy-ctl`
- `scripts/qcy-install-mic-setup.sh` — setup WirePlumber/OpenWhispr/BrowserOS
- `scripts/qcy-mic-diagnose.sh` — diagnóstico de mic
- `context7` — para documentação de bibliotecas externas
- `firecrawl` — para busca/web scraping quando necessário
- `tavily` — busca web para validações externas

## Regras de Contribuição / Development
- **CLI first, Bun first** — todo novo código deve ser CLI-first; prefira Bun/TypeScript quando possível
- **Minúsculas** — arquivos e comandos em Linux são case-sensitive
- **Autonomia total** — o agente DevSan opera sem pedir permissão
- **tmux é rei** — use tmux para sessões (mt, dev, sys)
- **Commit messages** — seguir padrão do repo: `feat/fix/docs/refactor/test/chore: descrição curta`
- **Branch** — main é a branch ativa; features new podem ser feitas em branches temporárias
- **Push** — confirmar que `git status` está limpo (ou apenas `easyeffects_source` untracked) antes de push
- **Teste** — `bun test` executa testes; mas sem testes automatizados para hardware BLE, validação é manual

## Próximos Passos (Roadmap)
1. Commit da documentação QCY-H3S-PROTOCOL-COMPLETO.md
2. Atualizar ESTADO_ATUAL.md com status atual (08/2026)
3. Validar 2-3 comandos GATT via `bluetoothctl` + `gatttool` fallback
4. Adicionar suporte EQ ao `qcy-ctl` ou wrapper Bun
5. Criar TUI minimalista (ncurses/bun) para controle rápido
6. Documentar diferenças protocolo build 682→689→715
7. Post-mortem: o que funcionou, o que não, próximos 3 meses