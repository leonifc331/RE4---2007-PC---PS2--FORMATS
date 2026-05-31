# Resident Evil 4 — Documentação técnica do formato `.AEV`

> Base de engenharia reversa: ELF debug `SLPS_000.00`, com símbolos e registros STABS.  
> Plataforma analisada: PS2 / RE4 debug.  
> Endianness: **little-endian**.  
> Finalidade do arquivo: **Scenario Atari Data** — volumes/áreas de interação do cenário, gatilhos, portas, mensagens, saves, ladder, hide, jumps, eventos e callbacks.

---

## 1. Resumo estrutural

O `.AEV` é carregado por `SceAtInit__FP13SCE_AT_HEADERT0` em `0x002956B8`. O loader valida:

- assinatura `"AEV"`;
- versão `0x0104`;
- contador de entradas em `header + 0x06`;
- primeira entrada em `header + 0x10`;
- stride de cada entrada `.AEV`: **0xA0 bytes**.

O mesmo loader também aceita um segundo bloco irmão, `ITA`, com versão `0x0105` e stride `0xB0`, usado para item-set data. `.ITA` usa a mesma base `SCE_AT_UNIT`, mas payload `SCE_AT_DATA_ITEM` maior.

```text
AEV file
├─ SCE_AT_HEADER      0x10 bytes
└─ SCE_AT_DATA[num]   num * 0xA0 bytes
   ├─ SCE_AT_UNIT     0x60 bytes
   └─ payload union   0x40 bytes
```

---

## 2. Tipos primitivos

| Tipo | Tamanho | Endian | Observação |
|---|---:|---|---|
| `uint8` / `sint8` | 1 | n/a | byte |
| `uint16` / `sint16` | 2 | little | PS2 RE4 usa little-endian |
| `uint32` / `sint32` | 4 | little | também usado para ponteiros runtime |
| `float32` | 4 | little | IEEE-754 |
| `tagVec` | 0x10 | little | `{ float x, y, z, w; }` |
| pointer | 4 | little | ponteiro MIPS EE runtime; preservar, não recalcular às cegas |

---

## 3. Header `.AEV` — `SCE_AT_HEADER`

Tamanho confirmado por debug: **0x10 bytes**.

```c
struct SCE_AT_HEADER {
    char     id[4];   // "AEV\0" no arquivo .AEV
    uint16_t ver;     // 0x0104
    uint16_t num;     // quantidade de SCE_AT_DATA
    uint32_t pad1;    // normalmente 0 / reservado
    uint32_t pad2;    // normalmente 0 / reservado
};
```

| Offset | Tipo | Nome debug | Descrição |
|---:|---|---|---|
| `0x00` | `char[4]` / `uint32` | `id` | assinatura. Para `.AEV`: ASCII `41 45 56 00` = `AEV\0`. |
| `0x04` | `uint16` | `ver` | versão. Loader exige `0x0104`; caso contrário exibe `SceAt DATA IS OLD VERSION`. |
| `0x06` | `uint16` | `num` | quantidade de entradas `SCE_AT_DATA`. |
| `0x08` | `uint32` | `pad1` | reservado. O loader principal não usa diretamente. |
| `0x0C` | `uint32` | `pad2` | reservado. O loader principal não usa diretamente. |

Cálculo do tamanho mínimo:

```text
file_size_min = 0x10 + num * 0xA0
```

---

## 4. Entrada `.AEV` — `SCE_AT_DATA`

Tamanho confirmado por debug: **0xA0 bytes**.

O tipo `SCE_AT_DATA` é composto por:

```text
0x00..0x5F  SCE_AT_UNIT   // parte comum de todo trigger
0x60..0x9F  union payload // dados específicos por id/tipo
```

O debug identifica a base comum como `SCE_AT_UNIT` e o payload como union de 0x40 bytes.

---

## 5. Parte comum — `SCE_AT_UNIT` / primeiros 0x60 bytes

Tamanho confirmado por debug: **0x60 bytes**.

```c
struct SCE_AT_UNIT {
    uint32_t      tag;            // 0x00
    AREA_HIT_DATA area;           // 0x04
    uint8_t       be_flg;         // 0x34
    uint8_t       id;             // 0x35
    uint8_t       no;             // 0x36
    uint8_t       hit_type;       // 0x37
    uint8_t       trg_type;       // 0x38
    uint8_t       target_type;    // 0x39
    uint8_t       task_level;     // 0x3A
    uint8_t       trg_type_bak;   // 0x3B
    void*         pParam;         // 0x3C
    void*         pFunc;          // 0x40
    uint8_t       priority;       // 0x44
    uint8_t       kind;           // 0x45
    uint8_t       waiting_type;   // 0x46
    uint8_t       waiting_no;     // 0x47
    sint8_t       hit_dir_ang;    // 0x48
    sint8_t       hit_open_ang;   // 0x49
    uint8_t       act_type;       // 0x4A
    uint8_t       waiting_etc_id; // 0x4B
    cModel*       pParent;        // 0x4C
    sint16_t      parts_no;       // 0x50
    uint8_t       country;        // 0x52, cFlag<SCEAT_COUNTRY>
    uint8_t       act_color;      // 0x53
    uint8_t       pad2[8];        // 0x54
    uint32_t      pad32;          // 0x5C
};
```

| Offset | Tipo | Nome debug | Descrição operacional |
|---:|---|---|---|
| `0x00` | `uint32` | `tag` | Tag/list node usado pelo sistema OT interno (`sceAtGetOtAddr`, `ClearOTagR`). Em edição de arquivo, preservar ou zerar; o loader reorganiza listas em runtime. |
| `0x04` | `AREA_HIT_DATA` | `area` | Volume/área de colisão do trigger. Tamanho `0x30`. |
| `0x34` | `uint8` | `be_flg` | Flags de ativação. Bit `0x01` = enable/ativo (`SceAtSetEnable`, `SceAtCheckEnable`). Bit `0x08` = usa rotação/matriz local ao transformar área no `sceAtGetArea`. Outros bits devem ser preservados. |
| `0x35` | `uint8` | `id` | ID do handler. Indexa `sceAtFunc_tbl`. Valores válidos observados: `0..21`. |
| `0x36` | `uint8` | `no` | Número lógico do AEV. Usado como índice de bit em `SceAtSetHitFlg`, `SceAtSetExecFlg`, `SceAtPtr`. Em `.ITA`, o loader adiciona/marca `0x80` nesse campo. |
| `0x37` | `uint8` | `hit_type` | Flags/modo de hit. Bit `0x01` seleciona vetor/posição alternativa no check; outros bits participam de validação direcional/estado. Preservar bits desconhecidos. |
| `0x38` | `uint8` | `trg_type` | Flags de trigger/execução. Ver tabela de bits abaixo. |
| `0x39` | `uint8` | `target_type` | Máscara de alvo aceita pelo trigger. O check usa `target_type & current_mask`; se der zero, ignora a entrada. |
| `0x3A` | `uint8` | `task_level` | Nível de task/cenário para execução (`SCE_LEVEL`), usado principalmente em `exec`/`skey`. |
| `0x3B` | `uint8` | `trg_type_bak` | Backup temporário de `trg_type`, usado por `SceAtDataSet_exec`/`SceAtDataReset`. |
| `0x3C` | `void*` / `uint32` | `pParam` | Ponteiro/parâmetro genérico. Em `exec`, é o argumento passado ao callback/evento. |
| `0x40` | `void*` / função | `pFunc` | Ponteiro de função/callback. Em arquivos, preservar; criar endereços inválidos pode crashar. |
| `0x44` | `uint8` | `priority` | Prioridade/list bucket. Usado na inicialização para organizar OT e no ActionButton como `SCE_PRIORITY`. |
| `0x45` | `uint8` | `kind` | Subtipo/variação do trigger. Também pode ser usado como prioridade secundária em exec/eventos. |
| `0x46` | `uint8` | `waiting_type` | Tipo de espera/condição. |
| `0x47` | `uint8` | `waiting_no` | Número/ID da condição de espera. |
| `0x48` | `sint8` | `hit_dir_ang` | Ângulo de direção para validação de hit. Usado em checks angulares. |
| `0x49` | `sint8` | `hit_open_ang` | Abertura/tolerância angular. O código converte para faixa angular durante `sceAtHitCheck`. |
| `0x4A` | `uint8` | `act_type` | `ACTION_TYPE` mostrado no botão de ação. |
| `0x4B` | `uint8` | `waiting_etc_id` | ID auxiliar de condição/espera. |
| `0x4C` | `cModel*` | `pParent` | Modelo pai. Se não nulo, `sceAtGetArea` transforma a área pela matriz do modelo/parte. |
| `0x50` | `sint16` | `parts_no` | Índice de parte/bone do modelo pai. `-1` geralmente indica matriz do modelo inteiro. |
| `0x52` | `uint8` | `country` | `cFlag<unsigned char, SCEAT_COUNTRY>`. Flags: USA/JPN. |
| `0x53` | `uint8` | `act_color` | Cor/variação visual do ActionButton (`SCEAT_ACT_COL`). |
| `0x54` | `uint8[8]` | `pad2` | padding/reservado; preservar. |
| `0x5C` | `uint32` | `pad32` | padding/reservado; preservar. |

### Bits confirmados / úteis de `trg_type` (`0x38`)

| Bit | Máscara | Função observada |
|---:|---:|---|
| 0 | `0x01` | habilita/display class ligado ao ActionButton; o check atribui código interno `6`. |
| 1 | `0x02` | habilita/display class alternativo; o check atribui código interno `4`. |
| 2 | `0x04` | habilita/display class alternativo; o check atribui código interno `0`. |
| 3 | `0x08` | exige interação/botão de ação antes de executar o handler. Sem esse bit, pode executar automaticamente ao tocar/entrar na área. |
| 7 | `0x80` | one-shot/auto-disable após execução bem sucedida. O código chama `SceAtSetEnable(no, false)` quando o handler retorna sucesso. |

Bits não listados devem ser preservados.

---

## 6. Área de detecção — `AREA_HIT_DATA`

Tamanho confirmado por debug: **0x30 bytes**. Fica em `SCE_AT_DATA + 0x04`.

```c
struct AREA_HIT_DATA {
    uint8_t  be_flag; // 0x00
    uint8_t  type;    // 0x01
    uint16_t pad;     // 0x02
    union {           // 0x04, tamanho 0x2C
        AREA_XZ4      xz4;
        AREA_CYLINDER cylinder;
        AREA_EYE      eye_trigger;
    } u;
};
```

### Tipos de área (`AREA_HIT_DATA.type`)

| Valor | Nome prático | Estrutura usada | Descrição |
|---:|---|---|---|
| `0` | inválido/nenhum | — | Mensagem debug: `AREA_HIT_DATA : AREA_TYPE[%d] invalid.` |
| `1` | `xz4` / quad XZ | `AREA_XZ4` | área poligonal no plano XZ, com piso/altura. |
| `2` | `cylinder` | `AREA_CYLINDER` | cilindro vertical usando centro XZ, raio, piso e altura. |
| `3` | `eye_trigger` | `AREA_EYE` | área cônica/visual com posição XZ, ângulos e abertura. |

### Layout absoluto dentro da entrada `.AEV`

Como `AREA_HIT_DATA` começa em `record + 0x04`, os offsets abaixo são relativos ao início do `AREA_HIT_DATA`; entre parênteses está o offset absoluto dentro de `SCE_AT_DATA`.

| Offset área | Offset record | Tipo | Campo | Descrição |
|---:|---:|---|---|---|
| `0x00` | `0x04` | `uint8` | `be_flag` | flags da área. Preservar. |
| `0x01` | `0x05` | `uint8` | `type` | tipo da área: `1`, `2`, `3`. |
| `0x02` | `0x06` | `uint16` | `pad` | padding/reservado. |
| `0x04` | `0x08` | union | payload | varia por tipo. |

### `AREA_XZ4` — type `1`

Tamanho: **0x2C bytes**.

```c
struct AREA_XZ4 {
    float floor;      // +0x04
    float height;     // +0x08
    float radius;     // +0x0C, auxiliar/fallback
    float xz[4][2];   // +0x10, quatro pontos X/Z
};
```

| Offset área | Offset record | Tipo | Campo |
|---:|---:|---|---|
| `0x04` | `0x08` | `float` | `floor` / Y mínimo |
| `0x08` | `0x0C` | `float` | `height` / altura |
| `0x0C` | `0x10` | `float` | `radius` / auxiliar |
| `0x10` | `0x14` | `float` | `xz[0].x` |
| `0x14` | `0x18` | `float` | `xz[0].z` |
| `0x18` | `0x1C` | `float` | `xz[1].x` |
| `0x1C` | `0x20` | `float` | `xz[1].z` |
| `0x20` | `0x24` | `float` | `xz[2].x` |
| `0x24` | `0x28` | `float` | `xz[2].z` |
| `0x28` | `0x2C` | `float` | `xz[3].x` |
| `0x2C` | `0x30` | `float` | `xz[3].z` |

### `AREA_CYLINDER` — type `2`

Tamanho: **0x2C bytes**.

```c
struct AREA_CYLINDER {
    float floor;      // +0x04
    float height;     // +0x08
    float radius;     // +0x0C
    float xz[2];      // +0x10, centro X/Z
    uint8_t pad[24];  // +0x18
};
```

| Offset área | Offset record | Tipo | Campo |
|---:|---:|---|---|
| `0x04` | `0x08` | `float` | `floor` |
| `0x08` | `0x0C` | `float` | `height` |
| `0x0C` | `0x10` | `float` | `radius` |
| `0x10` | `0x14` | `float` | `xz[0]` / center X |
| `0x14` | `0x18` | `float` | `xz[1]` / center Z |
| `0x18` | `0x1C` | `uint8[24]` | padding/reservado até `0x2F` |

### `AREA_EYE` / `eye_trigger` — type `3`

Tamanho: **0x2C bytes**.

```c
struct AREA_EYE {
    float floor;      // +0x04
    float height;     // +0x08
    float radius;     // +0x0C
    float xz[2];      // +0x10
    float ang_x;      // +0x18
    float ang_y;      // +0x1C
    float pad00;      // +0x20
    float open_ang;   // +0x24
    float pad[2];     // +0x28
};
```

| Offset área | Offset record | Tipo | Campo |
|---:|---:|---|---|
| `0x04` | `0x08` | `float` | `floor` |
| `0x08` | `0x0C` | `float` | `height` |
| `0x0C` | `0x10` | `float` | `radius` |
| `0x10` | `0x14` | `float` | `xz[0]` / origem X |
| `0x14` | `0x18` | `float` | `xz[1]` / origem Z |
| `0x18` | `0x1C` | `float` | `ang_x` |
| `0x1C` | `0x20` | `float` | `ang_y` |
| `0x20` | `0x24` | `float` | `pad00` |
| `0x24` | `0x28` | `float` | `open_ang` |
| `0x28` | `0x2C` | `float[2]` | `pad` |

---

## 7. Payload union — `record + 0x60`, tamanho 0x40

O payload começa sempre no offset **`0x60`** da entrada. O `id` em `0x35` define qual handler interpreta esse bloco.

### Visão geral dos payloads no debug

| Payload debug | Tamanho | Uso principal |
|---|---:|---|
| `data[64]` | 0x40 | acesso bruto |
| `door` | 0x20 | porta / transição de sala |
| `normal` | 0x40 | lista runtime de modelos em contato |
| `flg` | 0x04 | set/check de flags |
| `mes` | 0x0C | mensagem / câmera / SE |
| `jump` | 0x0C | destino simples de jump |
| `shd_disp` | 0x04 | shadow/display flag |
| `damage` | 0x10 | dano por área |
| `scr_at` | 0x1C | SAT/EAT/script collision link |
| `cam_ctrl` | 0x20 | controle de câmera; struct existe, mas handler direto não aparece na tabela principal |
| `field` | 0x30 | field info |
| `skey` | 0x0C | scenario key / callback |
| `ladder` | 0x20 | escada |
| `use` | 0x04 | use id |
| `hide` | 0x40 | esconderijo/hide |
| `pos_jump` | 0x20 | teleporte/jump com ângulo |
| `save` | 0x04 | terminal de save |
| `ada_wire` | 0x40 | Ada wire / hookshot |

---

## 8. Tabela de handlers — `sceAtFunc_tbl`

Tabela encontrada em `0x0036A160`. O `id` (`record + 0x35`) indexa essa tabela. IDs `>= 0x16` são rejeitados pelo check principal.

| `id` | Handler | Payload provável | Flag exclusiva da tabela | Observação |
|---:|---|---|---:|---|
| `0` | `sceAtFunc_normal` | `normal` | `0` | trigger normal; registra modelos tocando. |
| `1` | `sceAtFunc_door` | `door` | `1` | porta / room jump / transição. |
| `2` | `sceAtFunc_exec` | usa campos comuns `pParam/pFunc/task_level` | `0` | executa callback/evento. |
| `3` | `sceAtFunc_item` | `SCE_AT_DATA_ITEM` em `.ITA` | `1` | item set data; normalmente vem do bloco `ITA`, não do `.AEV` puro. |
| `4` | `sceAtFunc_flg` | `flg` | `0` | manipula flag de cenário/sistema. |
| `5` | `sceAtFunc_mes` | `mes` | `1` | mensagem/event text/câmera/SE. |
| `6` | `sceAtFunc_normal` | `normal` | `1` | variação normal com gating exclusivo. |
| `7` | `sceAtFunc_normal` | `normal`/`jump` | `0` | struct `jump` existe, mas tabela aponta normal. |
| `8` | `sceAtFunc_save` | `save` | `1` | terminal/save. |
| `9` | `sceAtFunc_shd_disp` | `shd_disp` | `0` | display/shadow flag; possui reverse quando sai da área. |
| `10` | `sceAtFunc_damage` | `damage` | `0` | dano em área. |
| `11` | `sceAtFunc_scr_at` | `scr_at` | `0` | ativa/desativa SAT/EAT/script atari. |
| `12` | `sceAtFunc_normal` | `cam_ctrl`/normal | `0` | `cam_ctrl` existe no debug, mas handler principal é normal. |
| `13` | `sceAtFunc_field_info` | `field` | `0` | informação de campo. |
| `14` | `sceAtFunc_stoop` | sem payload próprio | `0` | ação de abaixar/stoop. |
| `15` | `sceAtFunc_skey` | `skey` | `1` | scenario key/callback. |
| `16` | `sceAtFunc_ladder` | `ladder` | `1` | escada. |
| `17` | `sceAtFunc_use` | `use` | `0` | uso/interação genérica por `use_id`. |
| `18` | `sceAtFunc_hide` | `hide` | `0` | esconderijo. |
| `19` | `sceAtFunc_pos_jump` | `pos_jump` | `0` | teleporte/jump posicional. |
| `20` | `sceAtFunc_normal` | `normal` | `0` | variação normal. |
| `21` | `sceAtFunc_ada_wire` | `ada_wire` | `0` | wire/hookshot da Ada. |

A “flag exclusiva” é o segundo dword da entrada da tabela. Quando ela é `1`, o check principal trata o handler como uma ação que não deve competir livremente com outra ação já executada no mesmo passe.

---

## 9. Payloads detalhados

Todos os offsets desta seção são **relativos ao início do payload**, ou seja: offset absoluto = `record_offset + 0x60 + offset_payload`.

### 9.1 `SCE_AT_DATA_DOOR` — payload `door`, id `1`

Tamanho: **0x20 bytes**.

```c
struct SCE_AT_DATA_DOOR {
    float   next_pos_x;    // +0x00
    float   next_pos_y;    // +0x04
    float   next_pos_z;    // +0x08
    float   next_ang_y;    // +0x0C
    uint8_t next_stage_no; // +0x10
    uint8_t next_room_no;  // +0x11
    uint8_t key_id;        // +0x12
    uint8_t key_flg;       // +0x13
    void*   pExitFunc;     // +0x14
    uint8_t next_part_no;  // +0x18
    sint8_t key_se;        // +0x19
    uint8_t open_se;       // +0x1A
    uint8_t fade_eff;      // +0x1B
    void*   pExitParam;    // +0x1C
};
```

| Abs | Tipo | Campo | Descrição |
|---:|---|---|---|
| `0x60` | `float` | `next_pos_x` | posição X após transição. |
| `0x64` | `float` | `next_pos_y` | posição Y após transição. |
| `0x68` | `float` | `next_pos_z` | posição Z após transição. |
| `0x6C` | `float` | `next_ang_y` | rotação Y após transição. |
| `0x70` | `uint8` | `next_stage_no` | stage destino. |
| `0x71` | `uint8` | `next_room_no` | sala destino. |
| `0x72` | `uint8` | `key_id` | chave/item/condição de porta. |
| `0x73` | `uint8` | `key_flg` | flag associada à chave/porta. |
| `0x74` | `void*` | `pExitFunc` | callback de saída. Preservar se não estiver recriando lógica. |
| `0x78` | `uint8` | `next_part_no` | parte/variação do destino. |
| `0x79` | `sint8` | `key_se` | som quando chave/condição falha ou é usada. |
| `0x7A` | `uint8` | `open_se` | som de abertura. |
| `0x7B` | `uint8` | `fade_eff` | `DOOR_EFF_TYPE`: normal/fade/black. |
| `0x7C` | `void*` | `pExitParam` | parâmetro para `pExitFunc`. |

### 9.2 `SCE_AT_DATA_NORMAL` — payload `normal`, ids `0`, `6`, `7`, `12`, `20`

Tamanho: **0x40 bytes**.

```c
struct SCE_AT_DATA_NORMAL {
    cModel* pModel[16]; // +0x00, 16 ponteiros
};
```

Esse bloco é usado como work/runtime. `sceAtFunc_normal` grava ponteiros de modelos que tocaram o trigger. Em arquivo, normalmente deve ser zerado ou preservado se o dado original vier preenchido.

### 9.3 `SCE_AT_DATA_FLG` — payload `flg`, id `4`

Tamanho: **0x04 bytes**.

```c
struct SCE_AT_DATA_FLG {
    uint8_t  flg_id;  // +0x00
    uint8_t  flg_act; // +0x01
    uint16_t flg_no;  // +0x02
};
```

| Abs | Tipo | Campo | Descrição |
|---:|---|---|---|
| `0x60` | `uint8` | `flg_id` | grupo/tipo de flag. |
| `0x61` | `uint8` | `flg_act` | ação sobre a flag: set/clear/check, conforme handler. |
| `0x62` | `uint16` | `flg_no` | número da flag. |

### 9.4 `SCE_AT_DATA_MES` — payload `mes`, id `5`

Tamanho: **0x0C bytes**.

```c
struct SCE_AT_DATA_MES {
    sint16_t mes_type; // +0x00
    sint16_t mes_no;   // +0x02
    uint8_t  cam_no;   // +0x04
    uint8_t  se_type;  // +0x05
    uint16_t se_no;    // +0x06
    uint8_t  attr;     // +0x08
    uint8_t  padd[3];  // +0x09
};
```

| Abs | Tipo | Campo | Descrição |
|---:|---|---|---|
| `0x60` | `sint16` | `mes_type` | tipo/categoria da mensagem. |
| `0x62` | `sint16` | `mes_no` | ID da mensagem. |
| `0x64` | `uint8` | `cam_no` | câmera associada. |
| `0x65` | `uint8` | `se_type` | tipo de sound effect. |
| `0x66` | `uint16` | `se_no` | ID do sound effect. |
| `0x68` | `uint8` | `attr` | atributos da mensagem. |
| `0x69` | `uint8[3]` | `padd` | padding. |

### 9.5 `SCE_AT_DATA_JUMP` — payload `jump`

Tamanho: **0x0C bytes**.

```c
struct SCE_AT_DATA_JUMP {
    float dest_pos_x; // +0x00
    float dest_pos_y; // +0x04
    float dest_pos_z; // +0x08
};
```

A struct existe no debug, mas a tabela principal não aponta para um handler dedicado `jump`; em mods, preserve quando `id` original usar esse layout.

### 9.6 `SCE_AT_DATA_SHD_DISP` — payload `shd_disp`, id `9`

Tamanho: **0x04 bytes**.

```c
struct SCE_AT_DATA_SHD_DISP {
    uint16_t shd_no;   // +0x00
    uint8_t  disp_flg; // +0x02
    uint8_t  set_flg;  // +0x03
};
```

### 9.7 `SCE_AT_DATA_DAMAGE` — payload `damage`, id `10`

Tamanho: **0x10 bytes**.

```c
struct SCE_AT_DATA_DAMAGE {
    uint32_t dmg_timer; // +0x00
    uint8_t  dmg_type;  // +0x04
    uint8_t  dmg_ctrl;  // +0x05
    uint8_t  pad[2];    // +0x06
    uint32_t dmg_vol;   // +0x08
    float    dmg_ang;   // +0x0C
};
```

### 9.8 `SCE_AT_DATA_SCR_AT` — payload `scr_at`, id `11`

Tamanho: **0x1C bytes**.

```c
struct SCE_AT_DATA_SCR_AT {
    scSat*   pSat;      // +0x00
    scSat*   pEat;      // +0x04
    uint8_t  set_flg;   // +0x08
    uint8_t  pad[3];    // +0x09
    uint32_t sat_attr;  // +0x0C
    uint32_t eat_attr;  // +0x10
    uint32_t ctrl_flag; // +0x14
    uint32_t sat_flag;  // +0x18
};
```

Esse payload controla vínculos com SAT/EAT/script atari. Ponteiros devem ser preservados; criar ponteiros arbitrários normalmente quebra o jogo.

### 9.9 `SCE_AT_DATA_CAM_CTRL` — payload `cam_ctrl`

Tamanho: **0x20 bytes**.

```c
struct SCE_AT_DATA_CAM_CTRL {
    tagVec   pos;       // +0x00
    float    ang_y;     // +0x10
    uint8_t  pos_set;   // +0x14
    uint8_t  type;      // +0x15
    uint8_t  padd[2];   // +0x16
    float    radius;    // +0x18
    float    out_range; // +0x1C
};
```

A struct existe no debug; no `sceAtFunc_tbl` principal o índice `12` aponta para `normal`, então este layout pode ser usado por ferramenta/debug/editor ou por fluxo não direto.

### 9.10 `SCE_AT_DATA_FIELD_INFO` — payload `field`, id `13`

Tamanho: **0x30 bytes**.

```c
struct SCE_AT_DATA_FIELD_INFO {
    uint32_t id;      // +0x00, FIELD_ID
    cModel*  pParent; // +0x04
    uint8_t  free[40];// +0x08
};
```

`FIELD_ID` conhecido:

| Valor | Nome |
|---:|---|
| `0` | `FIELD_IN_ROOM` |
| `1` | `FIELD_WINDOW` |
| `2` | `FIELD_DOOR` |
| `3` | `FIELD_ROBO` |
| `4` | `FIELD_IN_MAX` |

### 9.11 `SCE_AT_DATA_SKEY` — payload `skey`, id `15`

Tamanho: **0x0C bytes**.

```c
struct SCE_AT_DATA_SKEY {
    void*   pParam;     // +0x00
    void*   pFunc;      // +0x04
    uint8_t task_level; // +0x08
    uint8_t kind;       // +0x09
    uint8_t pad[2];     // +0x0A, implícito por alinhamento
};
```

### 9.12 `SCE_AT_DATA_LADDER` — payload `ladder`, id `16`

Tamanho: **0x20 bytes**.

```c
struct SCE_AT_DATA_LADDER {
    tagVec  pos;     // +0x00
    float   ang_y;   // +0x10
    sint8_t height;  // +0x14
    uint8_t type;    // +0x15
    uint8_t pos_set; // +0x16
    uint8_t cam_no;  // +0x17
    uint8_t cam_no2; // +0x18
    uint8_t cam_no3; // +0x19
    uint8_t padd[2]; // +0x1A
};
```

### 9.13 `SCE_AT_DATA_USE` — payload `use`, id `17`

Tamanho: **0x04 bytes**.

```c
struct SCE_AT_DATA_USE {
    uint32_t use_id; // +0x00
};
```

### 9.14 `SCE_AT_DATA_HIDE` — payload `hide`, id `18`

Tamanho: **0x40 bytes**.

```c
struct SCE_AT_DATA_HIDE {
    tagVec        pos;       // +0x00
    uint8_t       type;      // +0x10
    uint8_t       pos_set;   // +0x11
    uint8_t       area_set;  // +0x12
    uint8_t       status;    // +0x13
    AREA_XZ4      hide_area; // +0x14, 0x20 bytes efetivos dentro do struct debug
    void*         func;      // +0x34
    uint8_t       cam_no;    // +0x38
    uint8_t       padding[3];// +0x39
};
```

Observação: no debug o campo `hide_area` aparece como tipo interno derivado de área, com 0x20 bytes nessa posição. O payload total continua 0x40.

### 9.15 `SCE_AT_DATA_POS_JUMP` — payload `pos_jump`, id `19`

Tamanho: **0x20 bytes**.

```c
struct SCE_AT_DATA_POS_JUMP {
    tagVec  dest_pos;   // +0x00
    float   dest_ang_y; // +0x10
    uint8_t pos_set;    // +0x14
    uint8_t padding[3]; // +0x15
};
```

### 9.16 `SCE_AT_DATA_SAVE` — payload `save`, id `8`

Tamanho: **0x04 bytes**.

```c
struct SCE_AT_DATA_SAVE {
    uint32_t term_no; // +0x00
};
```

### 9.17 `SCE_AT_DATA_ADA_WIRE` — payload `ada_wire`, id `21`

Tamanho: **0x40 bytes**.

```c
struct SCE_AT_DATA_ADA_WIRE {
    tagVec  pos0;       // +0x00
    tagVec  pos1;       // +0x10
    tagVec  pos2;       // +0x20
    float   ang;        // +0x30
    uint8_t WireId;     // +0x34
    uint8_t pos_set[3]; // +0x35
    uint8_t pad[8];     // +0x38, restante do union
};
```

---

## 10. Enums e constantes relevantes

### `SCE_PRIORITY`

```c
enum SCE_PRIORITY {
    SCE_PRIO_0 = 0,  SCE_PRIO_1 = 1,  SCE_PRIO_2 = 2,  SCE_PRIO_3 = 3,
    SCE_PRIO_4 = 4,  SCE_PRIO_5 = 5,  SCE_PRIO_6 = 6,  SCE_PRIO_7 = 7,
    SCE_PRIO_8 = 8,  SCE_PRIO_9 = 9,  SCE_PRIO_10 = 10, SCE_PRIO_11 = 11,
    SCE_PRIO_12 = 12, SCE_PRIO_13 = 13, SCE_PRIO_14 = 14, SCE_PRIO_15 = 15,
    SCE_PRIO_DEF_0 = 0,
    SCE_PRIO_DEF_1 = 1,
    SCE_PRIO_DEF_2 = 2,
    SCE_PRIO_ACT = 5,
    SCE_PRIO_ACT_2 = 6,
    SCE_PRIO_ACT_3 = 7,
    SCE_PRIO_GET = 8,
    SCE_PRIO_ATTACK = 11,
    SCE_PRIO_ATTACK_2 = 12
};
```

### `SCE_LEVEL`

```c
enum SCE_LEVEL {
    SCE_NO_TASK = 0,
    SCE_LEVEL_EV = 7,
    SCE_LEVEL00 = 8,
    SCE_LEVEL01 = 9,
    SCE_LEVEL02 = 10,
    SCE_LEVEL03 = 11,
    SCE_LEVEL04 = 12,
    SCE_LEVEL05 = 13,
    SCE_LEVEL06 = 14,
    SCE_LEVEL07 = 15,
    SCE_LEVEL08 = 16,
    SCE_LEVEL09 = 17,
    SCE_LEVEL10 = 18,
    SCE_LEVEL11 = 19,
    SCE_LEVEL_ANY = 20
};
```

### `SCEAT_COUNTRY`

```c
enum SCEAT_COUNTRY {
    SCEAT_COUNTRY_USA = 0,
    SCEAT_COUNTRY_JPN = 1
};
```

### `SCEAT_ACT_COL`

```c
enum SCEAT_ACT_COL {
    ACTCOL_NORMAL = 0,
    ACTCOL_DOOR = 1,
    ACTCOL_HOOKSHOT = 2
};
```

### `DOOR_EFF_TYPE`

```c
enum DOOR_EFF_TYPE {
    DOOR_EFF_NORMAL = 0,
    DOOR_EFF_FADE = 1,
    DOOR_EFF_BLACK = 2,
    DOOR_EFF_MAX = 3
};
```

### `ACTION_TYPE`

Usado em `record + 0x4A` (`act_type`) e passado para `cActionButton::set`.

| Valor | Nome | Valor | Nome |
|---:|---|---:|---|
| 0 | `ACT_TALK` | 1 | `ACT_CHECK` |
| 2 | `ACT_JUMP_OUT` | 3 | `ACT_JUMP_IN` |
| 4 | `ACT_JUMP_DOWN` | 5 | `ACT_JUMP_OVER` |
| 6 | `ACT_PUSH` | 7 | `ACT_KICK` |
| 8 | `ACT_GO_UP` | 9 | `ACT_GET_DOWN` |
| 10 | `ACT_KNOCK_DOWN` | 11 | `ACT_STAND` |
| 12 | `ACT_JUMP_AT` | 13 | `ACT_LOOK` |
| 14 | `ACT_PEEP` | 15 | `ACT_SNIPING` |
| 16 | `ACT_OPEN` | 17 | `ACT_SWIM` |
| 18 | `ACT_JUMP_BACK` | 19 | `ACT_STOOP` |
| 20 | `ACT_OPERATION` | 21 | `ACT_RESCUE` |
| 22 | `ACT_RIDE_SHOULDER` | 23 | `ACT_SEARCH_ATTACK` |
| 24 | `ACT_SPRINT` | 25 | `ACT_CLIMB` |
| 26 | `ACT_JUMP` | 27 | `ACT_SLIDE_DOWN` |
| 28 | `ACT_CATCH` | 29 | `ACT_PULL_UP` |
| 30 | `ACT_WAIT_OTHER` | 31 | `ACT_SEPARATE_OTHER` |
| 32 | `ACT_HIDE_OTHER` | 33 | `ACT_FOLLOW_OTHER` |
| 34 | `ACT_HELP_OTHER` | 35 | `ACT_RIDE` |
| 36 | `ACT_GET_OFF` | 37 | `ACT_GUARD` |
| 38 | `ACT_HIDE` | 39 | `ACT_THROUGH` |
| 40 | `ACT_PICK_UP` | 41 | `ACT_QUICK_STICK` |
| 42 | `ACT_ROTATE` | 43 | `ACT_RESIST` |
| 44 | `ACT_SUPLEX` | 45 | `ACT_HELI_ORDER` |
| 46 | `ACT_SHOOTING` | 47 | `ACT_SAVE` |
| 48 | `ACT_NO_DISP` | 49 | `ACT_ACCELERATE` |
| 50 | `ACT_ACCELE` | 51 | `ACT_THROUGH_OTHER` |
| 52 | `ACT_SIT_DOWN` | 53 | `ACT_THROW_DOWN` |
| 54 | `ACT_ANSWER` | 55 | `ACT_SNEAK` |
| 56 | `ACT_SENPUU` | 57 | `ACT_BACKKICK` |
| 58 | `ACT_POISON_NEEDLE` | 59 | `ACT_EXECUTE` |
| 60 | `ACT_PALM_SHOCK` | 61 | `ACT_NERICHAGI` |
| 62 | `ACT_JUMP_MOVE` | 63 | `ACT_PUSH_SWITCH` |
| 64 | `ACT_GET_DOWN_1M` | 65 | `ACT_FIRE` |
| 66 | `ACT_OPERATION2` | 67 | `ACT_WIRE` |
| 68-80 | `ACT_padding_68..80` | 81 | `ACTION_TYPE_MAX` |

---

## 11. Fluxo runtime confirmado

### Inicialização

`SceAtInit(headerAEV, headerITA)`:

1. limpa o sistema `SceAtSys`;
2. valida `AEV` e versão `0x0104`;
3. grava ponteiro do header e ponteiro de dados (`header + 0x10`);
4. lê `num` em `+0x06`;
5. organiza entradas por `priority` (`record + 0x44`) no sistema OT;
6. se `ITA` existir, valida `ITA` versão `0x0105`, lê entradas de stride `0xB0`, marca `no` com `0x80` e mistura no mesmo sistema de check.

### Check por frame/interação

`SceAtCheck` chama `sceAtCheck_main` para player/modelos. O check principal:

1. percorre entradas ativas da lista OT;
2. filtra por `target_type`;
3. chama `sceAtHitCheck` usando `AREA_HIT_DATA`;
4. se hitou, seta bit em `HitFlg` usando `no`;
5. se `trg_type & 0x08`, cria/atualiza `ActionButton` com `act_type`, `priority`, `act_color` etc.;
6. se pode executar, indexa `sceAtFunc_tbl[id]` e chama o handler;
7. se handler retorna sucesso e `trg_type & 0x80`, desativa a entrada.

---

## 12. Regras para editor/repacker seguro

1. **Preservar bytes desconhecidos.** `pad2`, `pad32`, `pad`, ponteiros e payloads não usados pelo `id` original devem ser preservados em roundtrip.
2. **Não gerar ponteiros novos sem resolver endereço runtime.** Campos `pFunc`, `pParam`, `pParent`, `pExitFunc`, `pExitParam`, `pSat`, `pEat`, etc. são ponteiros MIPS/runtime. Valores inválidos causam crash.
3. **Recalcular apenas o header quando alterar contagem.** `num` deve bater com `len(entries)` e o tamanho final deve ser `0x10 + num * 0xA0`.
4. **`tag` não é dado autoral confiável.** É usado como tag/lista OT em runtime. Para criar nova entrada, use `0` e deixe o loader organizar, ou clone de entrada similar.
5. **`no` deve ser único dentro da faixa útil.** `no` alimenta bitfields de hit/exec; duplicar `no` causa colisão lógica.
6. **`id` deve ficar em `0..21`.** Valores fora disso não entram na tabela de handlers.
7. **Para porta**, clonar uma entrada `door` existente da mesma sala é mais seguro do que criar `pExitFunc`/`pExitParam` do zero.
8. **Para mensagens e saves**, os payloads `mes` e `save` são os mais seguros de editar, desde que IDs existam nos bancos do jogo.
9. **Para área**, `floor + height` define a faixa vertical; coordenadas X/Z seguem o espaço do jogo.
10. **Para ActionButton**, `trg_type`, `act_type`, `priority`, `act_color` e `target_type` trabalham juntos. Alterar só um deles pode fazer o prompt sumir ou executar automaticamente.

---

## 13. Layout irmão `.ITA` — item set data

O loader trata `.ITA` junto com `.AEV`, mas o arquivo não é `.AEV`. Ele usa:

```c
struct ITA_HEADER {
    char     id[4]; // "ITA\0"
    uint16_t ver;   // 0x0105
    uint16_t num;
    uint32_t pad1;
    uint32_t pad2;
};
```

Stride de item: **0xB0 bytes**.

```text
SCE_AT_ITEM
├─ SCE_AT_UNIT       0x60 bytes
└─ SCE_AT_DATA_ITEM  0x50 bytes
```

### `SCE_AT_DATA_ITEM`

Tamanho: **0x50 bytes**.

```c
struct SCE_AT_DATA_ITEM {
    tagVec   item_pos;         // +0x00
    tagVec   eff_offset;       // +0x10
    cModel*  pModel;           // +0x20
    uint16_t item_id;          // +0x24
    uint16_t item_flg;         // +0x26
    uint16_t item_num;         // +0x28
    uint16_t auto_item_flg;    // +0x2A
    uint8_t  eff_type;         // +0x2C
    uint8_t  eff_setno;        // +0x2D
    uint8_t  player_type;      // +0x2E
    uint8_t  pos_set;          // +0x2F
    uint8_t  ctrl_flag;        // +0x30
    uint8_t  disappear_timer;  // +0x31
    sint16_t save_no;          // +0x32
    float    radius;           // +0x34
    float    ang_x;            // +0x38
    float    ang_y;            // +0x3C
    float    open_ang;         // +0x40
    uint8_t  se_no;            // +0x44
    uint8_t  se_no2;           // +0x45
    uint8_t  padd01[2];        // +0x46
    uint8_t  pad_tail[8];      // +0x48, alinhamento até 0x50
};
```

---

## 14. Assinaturas/funções úteis no ELF debug

| Função/símbolo | Endereço | Relevância |
|---|---:|---|
| `SceAtInit__FP13SCE_AT_HEADERT0` | `0x002956B8` | valida `AEV`/`ITA`, versões e strides. |
| `sceAtSetOtStart__Fv` | `0x00295900` | início da lista OT de AEV. |
| `sceAtGetOtAddr__FP11SCE_AT_DATA` | `0x00295910` | iteração por `tag`. |
| `SceAtClearHitFlg__Fv` | `0x00295958` | limpa hit flags. |
| `SceAtSetHitFlg__FUi` | `0x00295988` | seta bit por `no`. |
| `SceAtClearExecFlg__Fv` | `0x002959C0` | limpa exec flags. |
| `SceAtSetExecFlg__FUi` | `0x002959F0` | seta bit por `no`. |
| `SceAtCheck__Fv` | `0x00295AD0` | loop principal. |
| `sceAtCheck_main__FP6cModelUi` | `0x00295D50` | filtra alvo, área, botão e execução. |
| `sceAtGetArea__FR13AREA_HIT_DATAP11SCE_AT_DATA` | `0x00296320` | copia/transforma `AREA_HIT_DATA`. |
| `sceAtHitCheck__FP11SCE_AT_DATAP6cModelP6tagVecT2` | `0x00296630` | hit test. |
| `sceAtFunc_tbl` | `0x0036A160` | tabela `id -> handler`. |
| `SceAtDataSet_exec__FUi9SCE_LEVELUiPvT3Uc` | `0x0029A1E8` | modifica campos exec/pFunc/pParam/trg. |
| `SceAtSetEnable__FUib` | `0x0029A348` | liga/desliga bit `be_flg & 1`. |
| `SceAtPtr__FUi` | `0x0029A070` | busca entrada pelo `no`. |
| `SceAtCreateExecAt__...` | `0x0029C1F0` | cria trigger exec runtime. |
| `SceAtCreateFieldAt__...` | `0x0029C410` | cria trigger field runtime. |
| `SceAtCreateItemAt__...` | `0x0029C638` | cria item at runtime. |

---

## 15. Modelo de parser mínimo

```python
import struct

AEV_HEADER = struct.Struct('<4sHHII')
AEV_RECORD_SIZE = 0xA0

with open('room.aev', 'rb') as f:
    data = f.read()

magic, ver, count, pad1, pad2 = AEV_HEADER.unpack_from(data, 0)
assert magic.rstrip(b'\0') == b'AEV'
assert ver == 0x0104
assert len(data) >= 0x10 + count * AEV_RECORD_SIZE

for i in range(count):
    off = 0x10 + i * AEV_RECORD_SIZE
    tag = struct.unpack_from('<I', data, off + 0x00)[0]
    area_type = data[off + 0x05]
    be_flg = data[off + 0x34]
    at_id = data[off + 0x35]
    at_no = data[off + 0x36]
    trg_type = data[off + 0x38]
    target_type = data[off + 0x39]
    priority = data[off + 0x44]
    act_type = data[off + 0x4A]
    payload = data[off + 0x60:off + 0xA0]
    print(i, hex(off), 'id=', at_id, 'no=', at_no, 'area=', area_type)
```

---

## 16. Checklist de validação para uma ferramenta

- [ ] Header tem magic `AEV\0`.
- [ ] Versão é `0x0104`.
- [ ] Tamanho é pelo menos `0x10 + num * 0xA0`.
- [ ] Cada `id` está em `0..21`.
- [ ] `AREA_HIT_DATA.type` está em `1..3` para triggers reais.
- [ ] `height` não é negativo, exceto se o original já usa isso.
- [ ] `no` não colide com outro `no` ativo, salvo comportamento proposital.
- [ ] `priority` está em faixa segura `0..15` para `SCE_PRIORITY` normal.
- [ ] Ponteiros são preservados se a ferramenta não resolver símbolos/relocações.
- [ ] Payload é interpretado de acordo com `id`, mas bytes não usados são preservados.

---

## 17. Observações finais

O `.AEV` não é apenas “event script”: ele é uma tabela de **áreas de interação**. O script/evento propriamente dito costuma ser chamado por handlers via `pFunc`, `pParam`, `task_level`, flags e IDs. Para modding estável, a estratégia mais segura é:

1. clonar uma entrada de mesmo `id` da mesma sala;
2. editar área, `no`, `act_type`, payload simples e flags;
3. preservar ponteiros e paddings;
4. só trocar callbacks quando o endereço e o contexto forem conhecidos no ELF/room.
