# QCY H3S Microphone Issue - Complete Report

**Date:** June 14, 2026  
**Repository:** `qcy-toolkit` (renomeado de `qcy-ble-linux` em 25/10/2026)  
**Hardware:** UGREEN BT6.0 (USB ID `33fa:0012` - Barrot BR8554) + QCY H3S (MAC `84:AC:60:05:55:2C`)

---

## Current Real Status (Important Note)

**Before the Barrot SCO handle patch**, the microphone had already worked to some degree (HFP profiles appeared and audio could be captured, even if unstable).

After we applied the `btusb` DKMS patch for Barrot (init byte fix + SCO handle bypass), the situation became worse or at best inconsistent. The microphone is currently **not being properly recognized/used** by the system.

The user’s main complaint is clear:  
> “Meu sistema não consegue detectar o microfone do meu fone. Ele continua não sendo disponível para uso.”

---

## Main Observed Symptoms

- A2DP (music) works well.
- `headset-head-unit` and `headset-head-unit-cvsd` profiles appear and can be activated.
- `bluez_input.84:AC:60:05:55:2C` node exists in PipeWire.
- However, the input source stays mostly **SUSPENDED** and does not deliver usable microphone audio.
- Repeated kernel errors:

```log
Bluetooth: hci0: SCO packet for unknown connection handle XXXX
```

Handles seen: 2934, 375, 3547, 3510, 3037, 0, etc (highly variable).

This suggests the driver is dropping SCO packets because it cannot associate them with a known synchronous connection.

**Critical context:** The SCO handle bypass patch we added may have been too narrow or introduced regression. We should consider reverting or heavily revising it.

---

## Histórico de Descobertas (em ordem)

### Problemas Identificados e Status

1. **Init Byte Corruption**
   - Dongle Barrot envia 1 byte extra no evento HCI.
   - Causava "Unexpected continuation: 1 bytes".
   - Patch aplicado via DKMS (`btusb_barrot_urb_quirk()` + correção de ordem).
   - Status: Parece resolvido (zero ocorrências nos logs recentes).

2. **Conflito ofono + hsphfpd**
   - Interferiam com o backend nativo do WirePlumber.
   - Ambos desabilitados e maskeados.
   - Status: Resolvido.

3. **Configuração bluetoothd fragmentada**
   - `main.conf` e `bluetooth.conf` conflitando + chaves inválidas.
   - Unificado e limpo.
   - Status: Resolvido.

4. **SCO Handle Validation / Pacotes SCO desconhecidos (Problema Atual)**
   - Erro recorrente: "SCO packet for unknown connection handle".
   - Patch parcial em `btusb_validate_sco_handle()` já existe, mas aparentemente insuficiente.
   - Status: **Ainda não resolvido**. Esta é a principal hipótese atual.

---

## O que Já Foi Feito

- DKMS module custom (`btusb-barrot`) versão 0.8 compilado e carregado.
- Patch inclui:
  - `btusb_barrot_urb_quirk()`
  - `btusb_is_barrot_dongle()`
  - Bypass parcial em `btusb_validate_sco_handle()`
- Configurações limpas de WirePlumber e bluetoothd.
- Múltiplos fresh pairing realizados.
- Testes de gravação com `pw-cat` e `arecord`.

---

## Arquivos Relevantes

- `QCY-H3S-BLUETOOTH-ISSUE-REPORT.md` ← este arquivo
- `/usr/src/btusb-barrot-1.0/btusb.c` → versão patchada
- `/etc/bluetooth/main.conf`
- `/etc/wireplumber/wireplumber.conf.d/50-bt-hfp-fix.conf`
- `ESTADO_ATUAL.md`
- `KERNEL-BARROT-PATCH-PLAN.md`

---

## Recomendações para Próximos Agentes / Humanos

**Estado real:** Ainda não resolvemos o problema do microfone. Tudo que temos são hipóteses fortes baseadas em logs. O erro "SCO packet for unknown connection handle" é o sintoma mais consistente.

**Próximos passos sugeridos (não execute sem pensar):**

1. Capturar um `btmon -w /tmp/hfp-sco.btmon` completo enquanto tenta usar o microfone (importante: capture durante uma chamada ou gravação).
2. Analisar se o handle que aparece nos erros do dmesg é consistente ou realmente varia aleatoriamente.
3. Tornar o bypass de SCO handle **mais agressivo** (aceitar qualquer handle > 0 quando for Barrot).
4. Investigar se precisamos chamar `btusb_add_sco_handle()` ou manipular a tabela de conexões SCO manualmente no driver.
5. Testar se o problema persiste com outro dongle Bluetooth (mesmo que o usuário não queira trocar, para isolamento).
6. Verificar logs do PipeWire/WirePlumber em nível debug (`WIREPLUMBER_DEBUG=3`) durante uso do microfone.

**Regra importante (AGENTS.md):**
- NUNCA mate BrowserOS automaticamente quando o usuário reclamar que o microfone está picotando ou inaudível.
- O usuário já declarou que o problema de microfone é crônico neste hardware (UGREEN + kernel CachyOS) e "tem nada a ver" com BrowserOS.

---

**Última atualização:** 14 de Junho de 2026  
**Autor:** DevSan  
**Status:** Aberto - Hipótese principal = falha no gerenciamento de SCO handles pelo driver btusb para este dongle Barrot.

Este arquivo deve ser lido por qualquer agente ou pessoa que for continuar trabalhando no problema.

