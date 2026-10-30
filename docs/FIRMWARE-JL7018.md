# Firmware do H3S — o que dá, o que não dá, e o que protege

Estudo de 30/10/2026. **Nada aqui foi executado no fone** além de leitura pura
(`GET_TARGET_INFO`, `GET_ADV_INFO`, `GET 0x23`). Nenhum comando OTA foi enviado.

## 1. O hardware

O H3S usa **JieLi JL7018F6** — SoC de áudio Bluetooth da JieLi (珠海杰理).

Fontes (duas, independentes, ambas 2026):
- 爱集微 — <https://www.ijiwei.com/n/1041072> → "主板上搭载杰理科技JL7018F6蓝牙音频芯片"
- 我爱音频网 (52audio) — <https://www.52audio.com/archives/273005.html> → teardown completo do produto

Specs do chip que o 爱集微 publica e que importam para engenharia reversa:

| Item | Valor |
|---|---|
| SoC | **JL7018F6** (JieLi) — 32-bit dual-core DSP, 160 MHz |
| Bluetooth | **v6.0 + BR + EDR + BLE**, LE Audio BIS/CIS, AoA/AoD |
| Codecs | LDAC, LHDC, AAC, SBC |
| ANC | amostragem até 750 kHz, latência analógica 7,5 µs, 4 entradas de mic |
| Microfones no produto | **7** (2 extra frontais + 2 de feedback por lado) |
| Transdutores | 4 (40 mm + 13 mm por lado), crossover passivo |
| Bateria | 750 mAh, 3,8 V |

`JL7018` **não** aparece na lista de chips cobertos pelas ferramentas da
comunidade (que param em AC690–AC700N). Isso importa para §5.

## 2. O que o próprio fone conta (leitura pura, 30/10/2026)

`GET_TARGET_INFO` (op `0x03`) com máscara total devolveu só 9 atributos:

```
04 08 01 00 01 | 04 09 00 00 00 | 05 0a 1c 52 41 7c
11 0b <16 bytes> | 05 0d 01 10 02 1c | 08 11 00 84 ac 60 f8 a6 bf
02 12 00 | 03 13 06 00 | 05 15 00 00 00 00
```

| Attr | Nome | Valor | Leitura |
|---|---|---|---|
| 8 | `SUPPORT_DOUBLE_BACKUP` | `01 00 01` | **OTA em dois bancos** — mantém o firmware antigo |
| 9 | `MANDATORY_UPGRADE_FLAG` | `00 00 00` | atualização opcional, não forçada |
| 10 | `VID_AND_PID` | `1C 52 41 7C` | identidade do produto |
| 11 | `AUTH_KEY` | 16 bytes ASCII | **presente** (mascarado aqui de propósito) |
| 13 | `PROTOCOL_MTU` | `01 10 02 1C` | 272 / 540 |
| 17 | `CONNECT_BLE_ONLY` | `00 84 AC 60 F8 A6 BF` | flag + endereço |
| 18 | (sem nome no SDK) | `00` | |
| 19 | `DEV_SUPPORT_FUNC` | `06 00` | |
| 21 | (sem nome no SDK) | `00 00 00 00` | |

**O que ele NÃO devolve:** plataforma (3), versão de firmware (5), versão do
uBoot (7), protocolo (0), SDK type (6). Ou seja: não há como pedir "qual é a
sua versão de firmware" por esse canal. A versão que o app mostra vem daqui:

```
GET 0x30  ->  30 03 00 01 08      # 1.8
```

`GET_DEV_MD5` (op `0xD4`) respondeu **status=1 (erro), sem payload** — só
funciona em modo de atualização, não em operação normal.

O `VID_AND_PID` aparece também no ADV: `05 d6 1c 52 41 7c` no product message.
Mesma identidade em dois canais.

## 3. O protocolo de atualização (do APK, `com.jieli.jl_bt_ota`)

Op codes, direto de `constant/Command.java`:

| Op | Decimal | Nome |
|---|---|---|
| `0xD4` | 212 | `GET_DEV_MD5` |
| `0xE1` | 225 | `GET_DEVICE_UPDATE_FILE_INFO_OFFSET` |
| `0xE2` | 226 | `INQUIRE_DEVICE_IF_CAN_UPDATE` |
| `0xE3` | 227 | `ENTER_UPDATE_MODE` |
| `0xE4` | 228 | `EXIT_UPDATE_MODE` |
| `0xE5` | 229 | `SEND_FIRMWARE_UPDATE_BLOCK` |
| `0xE6` | 230 | `GET_DEVICE_REFRESH_FIRMWARE_STATUS` |
| `0xE7` | 231 | `REBOOT_DEVICE` |
| `0xE8` | 232 | `NOTIFY_UPDATE_CONTENT_SIZE` |

Fluxo (`BluetoothOTAManager`, estado `upgradeStep01..05`):

```
1. GET_TARGET_INFO            -> identidade + se suporta MD5
2. (opcional) RcspAuth.startAuth(...)   -> challenge-response, se isUseAuth
3. 0xE1  pergunta onde está o "update file flag" dentro do arquivo
4. 0xE2  manda os bytes do flag (tamanho vem do fone) -> responde 0..7 (0 = ok)
5. 0xE3  entra em update mode (fone reinicia no loader)
6. 0xE8  informa tamanho total; 0xE5 lê blocos (offset + len, sem CRC por bloco)
7. 0xE6  consulta resultado; 0xE7 reboot; reconexão
```

Detalhe importante do passo 6: o **fone puxa** os blocos. O app responde a cada
`0xE5` com `[offset:4B][len:2B]` + payload cru. Não vi assinatura nem CRC por
bloco no parâmetro — `FirmwareUpdateBlockParam` monta só offset e tamanho.

### O APK não conhece o formato do arquivo

Isso é o achado mais útil da leitura do código: `BluetoothOTAManager.b(offset,
len, cb)` só faz `System.arraycopy(this.M, offset, out, 0, len)` — copia bytes
cegos do buffer do arquivo. `thread/ReadFileThread.java` também só lê o arquivo
inteiro em memória. `util/FileUtil.java` é utilitário genérico (não parseia
nada). Não há header, magic, `.ufw`, checksum nem tabela de partição em lugar
nenhum do APK.

Consequência prática: **a semântica do arquivo é 100% do lado do fone**. O app
não sabe se o `.UFW` está certo — pergunta ao dispositivo onde está o flag
(`0xE1`), devolve os bytes (`0xE2`) e deixa ele decidir. O formato do `.UFW`
nunca esteve no QCY Audio; ele está revertido em código aberto, ver §5.

## 4. É assinado? Não.

Não existe verificação criptográfica de assinatura em **nenhum** ponto do
caminho de OTA. Nenhuma ocorrência de RSA/ECDSA/verify no pacote `jl_bt_ota`.
O que existe é isto, em ordem de força:

### 4.1 Autenticação RCSP (challenge-response) — o único gate real

`RcspAuth` manda um desafio aleatório gerado em nativo
(`getRandomAuthData`, em `libjl_ota_auth.so`, que também exporta
`getEncryptedAuthData` e `setLinkKey`). O fone responde e o app valida com
`isValidAuthData`: aceita resposta de **5 bytes** (`02 xx ...`) ou de
**17 bytes** (`02 xx ...` + payload). Bateu → app manda `getAuthOkData()`
(`02 70 ...`). Erros: `SUB_ERR_AUTH_DEVICE = 20481`.
Obs.: o tamanho exato do desafio não está no Java — é decidido dentro do `.so`.

O detalhe que muda tudo: **a chave está no próprio fone**.
`ATTR_TYPE_AUTH_KEY` (11) é lido por BLE sem autenticação nenhuma. É um
segredo que o dispositivo entrega a quem perguntar — e a função de cifra está
em 15 KB de `.so` ARM64.

Ou seja: tecnicamente contornável, mas **não testei o handshake** e não vou
testar sem você mandar. Risco de lockout em auth falha não é zero.

### 4.2 Identidade do arquivo e pré-requisitos — um único ponto de checagem

O fone diz onde, dentro do arquivo, estão os bytes de identidade (op `0xE1` →
`UpdateFileOffsetResponse{updateFileFlagOffset, updateFileFlagLen}`), o app
devolve esses bytes (op `0xE2` → `InquireUpdateParam.updateFileFlagData`) e o
fone decide. O retorno de `0xE2` é o mapa de rejeição inteiro
(`BluetoothOTAManager`, `upgradeStep02`):

| Result | Erro | Significado |
|---|---|---|
| **0** | — | pode atualizar |
| 1 | 16386 `DEVICE_LOW_VOLTAGE` | bateria baixa |
| 2 | 16387 `CHECK_UPGRADE_FILE` | **arquivo não confere** (flag errado) |
| 3 | 16396 `UPGRADE_FILE_VERSION_SAME` | versão igual/mais nova |
| 4 | 16397 `TWS_NOT_CONNECT` | TWS desconexo |
| 5 | 16398 `HEADSET_NOT_IN_CHARGING_BIN` | exige estar no estojo |
| 6 | 16403 `OTA_IN_PROGRESS` | já tem update rodando |
| 7 | 16404 `DEVICE_IN_DOUBLE_CONNECTION` | dois celulares conectados |

**É por isso que cross-flash de outro modelo QCY falha** (result 2) — e é o que
se patcheia nos bytes do flag, cujo tamanho o próprio fone informa.

Note que o result 3 bloqueia downgrade: firmware modificado precisa carregar
versão **maior**, não é opção apagar o número.

### 4.3 Pré-condições operacionais

Os results 1, 4, 5, 6 e 7 acima não são proteções de segurança: são proteções
**contra brick acidental** (bateria, conexão dupla, TWS fora do estojo).

### 4.4 Integridade

Só do APK (verificado):

* `GET_DEV_MD5` (op `0xD4`) — MD5 do firmware do fone. O app guarda em cache
  (`DeviceStatusCache.updateDeviceMD5`) para saber se mudou depois do update.
  **Não é comparado com o arquivo local**: é relatório, não autenticação. O SDK
  pergunta se o device suporta (`TargetInfoResponse.isSupportMD5`) antes de usar.
  No H3S em modo normal essa op responde `status=1` sem payload.
* Nenhum checksum/CRC no caminho `0xE5` (bloco) nem no `0xE8` (tamanho).

Da documentação de terceiros (não verificável daqui, não confirmado no H3S):

* `UPDATA_PARM` / `UPDATA_FLAG_ADDR` na flash — tabela de controle de update com
  CRC16 do resultado, gravada após a atualização. Protege contra corrupção de
  flash, não contra firmware malicioso. Fonte: wiki da comunidade `jielie`.
* Flash write-protect e "multi-algorithm authorization" (多算法授权) — existem
  na toolchain oficial (`isd_download.exe`, parâmetros de flash), configurados
  por FLASH ID. Não há como ler essa configuração por BLE; só com dump da flash.

### 4.5 Resumo honesto

| Pergunta | Resposta |
|---|---|
| Assinatura criptográfica? | **Não** (zero ocorrências de RSA/ECDSA/verify/certificate em `jl_bt_ota`) |
| Cifra do firmware no arquivo? | Sim, mas não protege nada: o envelope `.UFW` usa chave **fixa `0xFFFF`** (pública, em `jl-misctools`). O conteúdo da flash é que é cifrado por chipkey — ver §5 |
| O que impede gravar firmware alheio? | Flag de identidade (tamanho informado pelo fone) + ordem de versão + auth opcional |
| Isso é forte? | Não. Flag e versão são bytes no arquivo; auth é challenge-response com a chave entregue pelo próprio device |
| O que realmente não dá para contornar remotamente | Ausência de um arquivo de firmware válido para gravar |

## 5. O container `.UFW` está documentado em código aberto

Isso muda o tom do estudo: o formato do arquivo de atualização **não é segredo**.
A comunidade já o reverteu. Fonte: `kagaimiq/jl-misctools`
(≤MIT, C + Python), arquivo `firmware/fwunpack_newfw.py`, que declara cobrir
"AC693N e acima, AC7xxx" — a família do JL7018.

### 5.1 Layout do `.UFW`

```
0x00  header 0x40 bytes            <- "embaralhado" com jl_enc_cipher(key=0xFFFF)
0x40  tabela de N entradas, 0x50 cada   <- mesma cifra, mesma chave 0xFFFF
...   dados das entradas
```

Header (`struct.unpack_from('<HHIHHI48s')`):

| Off | Campo | Nota |
|---|---|---|
| 0x00 | `hdrcrc` u16 | CRC16 dos outros 62 bytes |
| 0x02 | `listcrc` u16 | CRC16 da tabela de entradas |
| 0x04 | `imgsize` u32 | |
| 0x08 | `numents` u16 | número de entradas |
| 0x0C | `chipname` 48 B | nome do chip em ASCII |

Entrada (`'<HHHHIII44s16s'`): `etype`, `eindex`, `edcrc` (CRC16 do dados),
`ewa1`, `eoffset`, `esize`, `esize2`, `ewa2`, `ename` (44 B), reservado.
`etype == 0` é o `flash.bin` — a imagem inteira da flash.

Duas coisas importantes:

1. **A "cifra" do container usa chave constante `0xFFFF`**, que está publicada
   no código aberto. Não é segredo; é embaralhamento. `jl_enc_cipher` é um XOR
   com um LFSR de 16 bits (polinômio `0x1021`), e cifrar = decifrar.
2. **CRC16 valida integridade, não origem.** Qualquer um monta um `.UFW`
   coerente com CRC certo.

### 5.2 O que está cifrado de verdade: o conteúdo da flash

Dentro do `flash.bin`, a região de app é embaralhada com `jl_sfc_cipher`
(blocos de 32 bytes, chave por bloco derivada da chipkey + `offset>>2`).

Mas — e isso é decisivo — **a chipkey viaja dentro do próprio firmware**: ela
está codificada no `isd_config.ini` embutido, e `jltech/chipkeybin.py`
(`chipkeybin_decode`) a decodifica. O `fwunpack_newfw.py` faz exatamente isso
(linha ~242: "Here's the chipkey!") e usa a chave para decifrar a área de app.

Ou seja: a chipkey não é barreira para quem tem o arquivo — é barreira só para
quem tem **dump cru de flash sem arquivo**. E `recrypt.py` permite reembaralhar
uma região com outra chave (cifrar = decifrar aqui), então editar conteúdo e
fechar de novo é mecanicamente possível.

`bruteforce.py` existe para as famílias antigas (BR17–BR21) onde a chipkey vinha
num `chipkey.bin` de 32 bytes estranhamente codificado — nas novas ela já vem
no `isd_config.ini`.

### 5.3 Ferramentas, honestamente

| Ferramenta | Serve para | Limite |
|---|---|---|
| `jl-misctools/fwunpack_newfw.py` | abrir `.UFW`, extrair `flash.bin`, listar JLFS | incompleto (não trata áreas reservadas) |
| `jl-misctools/mkbfu.py` | **criar `.bfu`** byte-exato igual ao `bfumake.exe` do SDK | precisa saber o nome/tipo da imagem |
| `jl-misctools/mkjlfs.py` | montar JLFSv2 (o "sistema de arquivos" da flash) | |
| `jl-misctools/recrypt.py` / `bruteforce.py` | trocar/recuperar chave SFC | só famílias antigas validadas |
| `jl-uboot-tool` | dump/flash cru via **USB UBOOT** | cobertura real pequena (chips antigos); JL7018 não listado; exige solda nos pinos |
| `kagaimiq/jielie` | base de conhecimento das famílias (BR17…BR36) | referência, não opera nada |
| Toolchain oficial (`fw_add`, `ufw_maker`, `isd_download`) | gerar/gravar `.UFW` oficial | SDK sob NDA |

Nenhuma dessas ferramentas fala com o JL7018 pelo rádio. Todas exigem arquivo
em mãos ou acesso físico.

## 6. O que é possível, por nível de acesso

| Nível | Precisa | Dá para fazer |
|---|---|---|
| **BLE (o que temos)** | nada além do fone | ler identidade, chave de auth, config, botões, EQ, ANC |
| **BLE + `.UFW`** | um `.ufw` de QCY com JL7018 | abrir com `fwunpack_newfw`, comparar versões, achar o flag, repackar |
| **BLE + `.UFW` + edição** | o mesmo arquivo (chipkey sai de dentro dele) | editar conteúdo, reembaralhar com `recrypt.py`, fechar `.UFW` |
| **USB UBOOT** | solda nos pinos / PCB exposta | dump e flash cru — se a família for suportada |
| **SPI direto** | dessoldar a flash (se externa) | dump total; ainda cifrado pelo chipkey |

Não temos (ainda): arquivo `.UFW` do H3S, dump de flash, nem acesso físico.

## 7. Riscos, sem eufemismo

1. **Brick de aplicação ≠ brick total.** O loader/UBOOT é separado do firmware
   de aplicação. Com `SUPPORT_DOUBLE_BACKUP = 1`, o fone tem dois bancos. Isso
   aumenta muito a chance de recuperação — mas não é garantia.
2. **Auth falhada** tem código de erro próprio e retry 2x no SDK. Não sabemos o
   que o fone faz depois (lockout? nada?). Não testar às cegas.
3. **Update mode exige reboot** (`0xE3`). No Windows com AltA2DP isso derruba
   A2DP e pode repetir o episódio de HFP que já custou tempo.
4. **Flag de identidade patchado errado** = fone aceita o arquivo e escreve
   firmware incompatível. É o cenário que o double bank existe para salvar.
5. **Repackar um `.UFW` é fácil; fazer o conteúdo funcionar é o problema.** CRC
   do envelope não é barreira. Chipkey, mapa de flash e o layout do app é que
   são. Um `.ufw` com CRC certo e conteúdo errado passa no `0xE2`? Talvez — e aí
   o double bank é a única rede.

## 8. Próximos passos (quando você mandar)

Em ordem de custo/benefício, **nenhum deles toca no código que está em uso**:

1. Procurar `.UFW` do H3S ou de outro QCY com JL7018 (4pda, XDA, grupos de
   firmware QCY). Sem arquivo, tudo o resto é teoria.
2. Com um `.UFW`: rodar `fwunpack_newfw.py`, ler o `chipname` do header, e
   comparar dois arquivos de versões diferentes — o diff mostra o que muda.
3. Só então: testar `0xE1` (pergunta o offset do flag) com o fone real. É
   **leitura** e não muda estado — o passo mais informativo e mais barato.
4. `0xE2` (inquire) **não** mandar sem arquivo: ele inicia a validação e, com
   flag errado, produz erro no fone sem necessidade.
5. Se um dia houver `.UFW`: extrair o `flash.bin`, procurar a assinatura JLFS e
   mapear as regiões de recurso (UI, EQ, tons) — mexer em recurso é bem menos
   arriscado que mexer em código.
