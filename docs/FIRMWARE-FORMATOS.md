# Formatos de firmware JieLi — referência técnica

Material de consulta levantado em 30/10/2026, lendo o código aberto da comunidade
(`kagaimiq/jl-misctools`, `kagaimiq/jl-uboot-tool`) e a documentação pública da
JieLi. **Nada aqui foi executado**: é especificação, não procedimento testado.

Aplicabilidade: família "new-fw" (AC693N+, AC7xxx). O JL7018F6 do H3S passa no
filtro do unpacker por família de chip, não por teste no H3S — ver §9.

---

## 1. Primitivas

| Primitiva | Definição | Ref |
|---|---|---|
| `jl_crc16` | CRC-16/CCITT-FALSE, poly `0x11021`, init `0`, **não refletido** | `jltech/crc.py:10` |
| `jl_crc32` (JieLi) | poly `0x104C11DB7`, init `0x26536734`, refletido | `jltech/crc.py:11` |
| `jl_enc_cipher(buf,off,n,key)` | XOR com keystream de LFSR: `key = (key<<1) ^ (0x1021 se key&0x8000)`, 1 byte/iter | `jltech/cipher.py:14` |
| `jl_sfc_cipher(...,base,key)` | blocos de 32 B, chave do bloco = `key ^ ((off+i-base)>>2)` | `jltech/cipher.py:38` |
| `jl_crc_cipher` | XOR com CRC da string `"孟黎我爱你，玉林"` (GB2312) — loaders antigos | `jltech/cipher.py:21` |
| `jl_rxgp_cipher` | LCG 16807 semeado `0x70477852` ("RxGp", série DV) | `jltech/cipher.py:31` |

Cifrar e decifrar são a **mesma operação** — é XOR de keystream, não blocochave.

---

## 2. JLFS v2 — o "sistema de arquivos" da flash

Entrada de **32 bytes**, little-endian:

| Off | Tipo | Campo | Notas |
|---|---|---|---|
| 0x00 | u16 | `crc_hdr` | `jl_crc16(bytes 2..31)`; **validado**, aborta se errado |
| 0x02 | u16 | `data_crc` | CRC16 dos dados |
| 0x04 | u32 | `offset` | sobrecarregado: offset dos dados, ou entry-point no `app_dir_head` |
| 0x08 | u32 | `size` | tamanho; em modo *data-after-header* inclui o header |
| 0x0C | u8 | `flags` | ver abaixo |
| 0x0D | u8 | `resvd` | `0xFF` |
| 0x0E | u16 | `index` | **0 = continua, ≠0 = última entrada** |
| 0x10 | 16s | `name` | ASCII no firmware, GB2312 no maker |

Flags que o código realmente testa:

| Flag | Significado |
|---|---|
| `0x02` | arquivo regular |
| `0x03` | diretório |
| `0x10` | área reservada (ex.: `key_mac`) — **pulada**, não vira arquivo |
| `0x40` | SPL comprimido |
| `0x81` | `app_dir_head` (só uma utilizável) |
| `0x82` | arquivo regular (lado SFC) |
| `0x83` | diretório (dumper recursa) |

Dois layouts físicos, ambos geráveis por `mkjlfs.py`:

- **tabela**: N headers contíguos, dados depois; `offset` relativo ao `--base`.
- **daisy-chain**: header + dados logo após; `size = len+32`, `offset` recebe
  `--dc-offset` (`0x20`).

Raiz do firmware: o BootROM procura um header de 32 B em
`[0, 0x1000, 0x10000, 0x80000, 0x100000, 0x180000]`, decifrado com chave fixa
`0xFFFF`. Campos: `crc_hdr, burner_size, vid[4], flash_size, fs_ver,
block_align, resvd, spec_opt, pid[16]`. **VID e PID ficam em claro** (o unpacker
os recopia do original após decifrar).

A primeira entrada do topo é o SPL (`uboot.boot`), cujo conteúdo é um BANKCB.

---

## 3. Containers

### 3.1 `.ufw` — o arquivo de atualização

```
0x00  header 0x40 B          <- jl_enc_cipher(chave fixa 0xFFFF)
0x40  N entradas de 0x50     <- mesma cifra, mesma chave
...   dados
```

Header `<HHIHHI48s`:

| Off | Tipo | Campo |
|---|---|---|
| 0x00 | u16 | `crc_hdr` = CRC16 dos outros 62 B |
| 0x02 | u16 | `crc_lista` = CRC16 da tabela **já decifrada** |
| 0x04 | u32 | `img_size` |
| 0x08 | u16 | `numents` |
| 0x0A | u16 | `wa3` (desconhecido) |
| 0x0C | u32 | `wa4` (desconhecido) |
| 0x10 | 48s | `chipname` (ASCII, NUL-terminated) |

Entrada `<HHHHIII44s16s`: `etype, eindex, data_crc, wa1, offset, size, size2,
wa2[44], name[16]`.

Regras verificadas no unpacker:

1. `etype == 0` ⇒ a entrada é o `flash.bin`; dados em `offset + offskew`.
2. CRC dos dados é **avisado e ignorado** ("data crc mismatch but who cares?").
3. CRC de header e de lista **não** são ignorados: header inválido = arquivo
   rejeitado.

Conclusão: o `.ufw` é invólucro. O conteúdo útil é o `flash.bin` cru, que já é a
imagem de flash que o BootROM espera (com o header de 32 B na posição 0).

### 3.2 `.fwsc` — variante

20 blocos de 0x30 da qual se aproveitam 0x2F → header de 0x3B0. Resto idêntico,
`offskew = 0x10`. Mesmo parser.

### 3.3 `.bfu` — o que o `isd_download` come (big-endian)

| Off | Tipo | Campo | Valor |
|---|---|---|---|
| 0x00 | 8s | magic | `"JL_UDFIR"` |
| 0x08 | u32 BE | header size | pós-padding |
| 0x0C | u32 BE | CRC16 do header | |
| 0x10 | u32 BE | data offset | `0x200` |
| 0x14 | u32 BE | data size | |
| 0x18 | u32 BE | CRC16 dos dados | |
| 0x1C | u32 BE | load address | default 0 |
| 0x20 | u32 BE | run address | default 0 |
| 0x24 | ASCII | name/type | zero-padded até múltiplo de 16 |
| 0x200 | — | dados | alinhamento 512 |

`name/type` conhecidos: `JL_AC690X.BIN`, `JL_AC692X.BIN`, `BT_CFG.BIN` (AC69xx);
`CODE`, `RESOURCE`, `FLASH`, `AUDLOGO`, `ALL` (AC5xxx). O próprio autor do
`mkbfu.py` não sabe a diferença entre `FLASH` e `ALL`.

### 3.4 `.ota` — o container do app QCY (não é JieLi)

O QCY Audio **fabrica** pacotes `.ota` em Kotlin para tom e tema. Layout, direto
de `com/qcy/audio/common/subutil/OTAFile.java` e `OTAUpgrade.java`:

```
18 B big-endian:  vid(u16) pid(u16) version(u16) len(u32) imageId(u32) crc32(u32)
16 B little-end:  magic 0xABCD6543, fwVersion=1, 0x1000, 0xFFFFFFFF
16 B little-end:  len, len, 0, 0
 4 B little-end:  crc32 dos 16 B anteriores
   ... payload
 4 B little-end:  crc32 do payload
```

`imageId` (`ImageType.java`): `TONE = 144`, `THEME0..THEME6 = 145..151`.
Defaults do construtor: `pid=28725`, `vid=1902`, `version=0x1001`.

Isso não tem relação com o `.ufw`: é um formato **do app**, para mandar tom e
tema ao fone por outro caminho.

---

## 4. BANKCB — o que o mask-ROM carrega em SDRAM

Tabela de headers de 16 B: `crc16(u16) + bank_id(u16) + size(u32) + load_addr(u32)
+ data_off(u32) + data_crc(u32)`. O `bank_id` do banco 0 é a **quantidade total**
de bancos; os demais têm `id = i-1`. Máx. `0xFFFF` B por banco.

`debank.py` quebra a chave de um BANKCB por força bruta de 16 bits, validando
CRC do header e dos dados. É o que envolve o SPL.

---

## 5. chipkey e key file

- **chipkey** = chave de confidencialidade da área de app (SFC). Sai do
  `isd_config.ini` embutido no firmware: 32 B + CRC16 → `chipkeybin_decode`
  (soma dos 16 primeiros bytes vira limiar; bit *i* = `(data[16+i] ^ data[15-i]) <
  limiar`).
- **sensível**: a chipkey é **OTP e unidirecional** — `nova = nova & atual`, só
  apaga bits. Comando `WRITE_KEY` (0xFC12) exige VPP 5000. **Errar essa gravação
  inutiliza o aparelho de forma irreversível.** `READ_KEY` (0xFC09) é livre.
- **key file** = transporte da chipkey em 72 caracteres ASCII: 64 hex (16 B
  cifrados + 16 B de chave AES-ECB) + 8 hex de CRC32. Payload decifrado =
  `"<chipkey:04x>a000\0"`. Serve para alimentar o `isd_download` num chip cuja
  chipkey ≠ 0xFFFF. **Não existe conceito de write-protect nesse formato.**

Proteção de flash, segundo a doc oficial: `FLASH_WRITE_PROTECT=YES` (escrita,
não leitura), `OTP_CFG_SIZE=512`, multi-algoritmo por `Authorization_ID` em OTP.
Não há "read-out protection" documentada.

---

## 6. UBOOT — o caminho de hardware

Transporte: o chip em download mode aparece como **Mass Storage** com interface
SCSI custom: `opcode(u16 BE) + subopcode + args`, padding `0xFF` até 16 B.
**Sem autenticação, sem challenge-response.**

Entrada no modo: `USB_KEY = 0x16EF` enviado serialmente em D-/D+ (clock/data),
repetido até o chip puxar as linhas a GND 1–2 ms como ACK. Chips recentes têm
`ISP_KEY` como alternativa.

| Comando | Opcode | Efeito |
|---|---|---|
| erase setor/bloco/chip | `0xFB00/0xFB01/0xFB02` | 4 k / 64 k / total |
| write flash | `0xFB04` | `addr(4 BE)+len(2 BE)+0x00+crc16(2 LE)` |
| read flash | `0xFD05` | dump |
| mem read / write | `0xFB06` / `0xFD07` | RAM |
| mem jump | `0xFB08` | executa com arg 16-bit |
| read key | `0xFC09` | chipkey |
| online device | `0xFC0A` | 
| read id | `0xFC0B` | flash ID |
| run app | `0xFC0C` | reset |
| flash crc16 | `0xFC0E` | |
| write key | `0xFC12` | **OTP, irreversível** |
| usb buff | `0xFC14` | |

VID do bootloader: `4c4a`. PID é por família (`"BR35UBOOT1.00"` etc.).

Cobertura do `jl-uboot-tool` (`data/chips.yaml`): BR17–BR34 "working/seems to
work", DV15 ok, **BR36 (AC700N) = "n/a, no loaders yet"**. `JL7018` não aparece
em nenhuma linha da tabela.

---

## 7. Doc oficial pública da JieLi (não é NDA)

| Recurso | Onde |
|---|---|
| Protocolo UBOOT v1.1.2 (`0xAA55`, cmd `0xC0`–`0xC5`, `JL_SU_CMD_SDK_ID_ERROR`) | `github.com/Jieli-Tech/fw-Bootloader/blob/main/doc/` |
| `isd_config.ini` (`PID`, `VID`, `UFW_ELEMENT`, `DOWNLOAD_MODEL`, `AUTH_CODE`, `EOFFSET`) | `doc.zh-jieli.com/Tools/zh-cn/dev_tools/toolchains/ini_cfg.html` |
| flash params (`FLASH_WRITE_PROTECT`, `OTP_CFG_SIZE`) | `doc.zh-jieli.com/Tools/zh-cn/dev_tools/post_build/flash_params/` |
| multi-algoritmo (`Authorization_ID` 64–999, `.authrun`) | `doc.zh-jieli.com/Tools/zh-cn/mass_prod_tools/multi_auth/` |
| RCSP-OTA (`CmdReadFileOffset` devolve `{offset,len}` do updateFileFlag) | `doc.zh-jieli.com/SDKWiki/.../3-jl-rcsp-ota.html` |
| SDKs AC701N completos | `gitlab.zh-jieli.com/soundbox/novisualization/ac701n_soundbox_sdk` (zip responde 200 sem login) |

`AUTH_CODE=0` é o **default** no `isd_config.ini` — o campo existe, mas nasce
desligado.

Um dump de terceiro mostra que flash nova é legível por UBOOT: ElectronicCats
publicou 4 MB byte-exact de um AC707N/BR35 (sha256 `5a91b1…f8e`), "in its
on-flash (SFC-scrambled) form". Ou seja: **legível, mas cifrado**.

---

## 8. Receita unpack → editar → repack (NÃO EXECUTADA)

1. `python3 fwunpack_newfw.py firmware.ufw` → `top/`, `files/`, `decrypted.bin`,
   `jlfw.yaml`. Se der "Could not locate the base offset", não é new-fw.
2. Conferir no `jlfw.yaml`: `chip-key`, `base-offset`, `entry-point`,
   `spl.compressed`.
3. Editar:
   - **tons** (menor risco): `tonecfgmk.py tone.yaml tone.cfg` → substituir
     `files/tone.cfg`;
   - **UI/textos/imagens**: `ui_resource/menu.res/ru2unpack.py` / `ru2pack.py`;
   - **subárvore JLFS**: `mkjlfs.py --align 8` (tabela) ou `--root-daisychain
     --dc-offset 0x20` (lado SFC).
4. Re-cifrar a área alterada: `recrypt.py dec.bin enc.bin -1 0xFFFF <start>
   <end>` (topo) ou `-1 0x<chipkey>` (app/resource). `end` **inclusivo**,
   alinhado a 32 B.
5. Recompor o header de flash (`crc_hdr`, e `flash_size`/`block_align` se o
   tamanho mudou) — **não automatizado**.
6. Recompor o container: `.bfu` via `mkbfu.py --name ...`; `.ufw` **não existe
   ferramenta** — seria script novo montando `<HHIHHI48s` + `N×0x50`, cifrando
   com `0xFFFF` e recalculando `crc_lista` sobre a lista decifrada.
7. Auto-validar: reabrir o pacote com o próprio unpacker e comparar CRCs.
8. Gravar (só com hardware): `jluboottool.py "write 0x0 flash.bin" "reset"`.

Riscos embutidos: (a) `index ≠ 0` marca fim da lista — errar faz o parser ler
lixo; (b) CRCs de header não toleram erro; (c) `WRITE_KEY` é irreversível;
(d) entry-point tem de casar com o `app_dir_head`.

---

## 9. O que isso NÃO garante

- Nenhum desses scripts foi rodado num arquivo do H3S — não existe arquivo.
- `JL7018F6` não está em `chips.yaml` do `jl-uboot-tool`. O paralelo com
  `AC701N/BR28` ("seems to work") é **inferência**, não fato.
- O protocolo RCSP OTA não está implementado em nenhuma dessas ferramentas. Elas
  trabalham com arquivo local e USB. **Como o fone recebe os blocos (`0xE5`) não
  está coberto por nenhum código aberto** — só pelo SDK oficial e pelo APK.
