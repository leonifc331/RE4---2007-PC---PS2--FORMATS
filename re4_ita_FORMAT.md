# Resident Evil 4 PS2 — Formato `.ITA`

Documentação técnica do formato `.ITA` usado pelo sistema `SCE_AT` do **Resident Evil 4** de PlayStation 2.

O `.ITA` armazena entradas de item do cenário: pickups, drops, itens vinculados a objetos/inimigos, efeitos de brilho/drop, flags de coleta e integração com persistência de sala/save.

## Sumário

- [1. Características gerais](#1-características-gerais)
- [2. Estrutura global](#2-estrutura-global)
- [3. Header `SCE_AT_HEADER`](#3-header-sce_at_header)
- [4. Entrada `SCE_AT_ITEM`](#4-entrada-sce_at_item)
- [5. Bloco comum `SCE_AT_UNIT`](#5-bloco-comum-sce_at_unit)
- [6. Área de interação `AREA_HIT_DATA`](#6-área-de-interação-area_hit_data)
- [7. Payload `SCE_AT_DATA_ITEM`](#7-payload-sce_at_data_item)
- [8. Tipos e enums relevantes](#8-tipos-e-enums-relevantes)
- [9. Flags de comportamento](#9-flags-de-comportamento)
- [10. Itens, quantidade, flags e persistência](#10-itens-quantidade-flags-e-persistência)
- [11. Fluxo de execução no jogo](#11-fluxo-de-execução-no-jogo)
- [12. Funções relevantes no executável](#12-funções-relevantes-no-executável)
- [13. Regras para editores e repackers](#13-regras-para-editores-e-repackers)
- [14. Representação JSON/YAML recomendada](#14-representação-jsonyaml-recomendada)
- [15. Parser de referência em Python](#15-parser-de-referência-em-python)
- [16. Checklist de roundtrip](#16-checklist-de-roundtrip)
- [17. Resumo de offsets](#17-resumo-de-offsets)

---

## 1. Características gerais

| Propriedade | Valor |
|---|---:|
| Assinatura | `ITA\0` |
| Versão | `0x0105` |
| Endianness | little-endian |
| Header | `0x10` bytes |
| Tamanho da entrada | `0xB0` bytes |
| Estrutura principal | `SCE_AT_ITEM` |
| Payload específico | `SCE_AT_DATA_ITEM` |
| Sistema runtime | `SCE_AT` / Atari |
| Plataforma analisada | PlayStation 2 |

O formato é irmão do `.AEV`, mas com payload especializado para itens. A principal diferença estrutural é o tamanho da entrada: `.AEV` usa stride `0xA0`, enquanto `.ITA` usa stride `0xB0`.

Durante o carregamento, o jogo valida:

```text
id      == "ITA"
version == 0x0105
entry   == header + 0x10
stride  == 0xB0
```

Além disso, o campo `no` de cada entrada `.ITA` recebe `+0x80` em runtime. O valor salvo no arquivo deve permanecer na faixa original do arquivo, normalmente `0x00..0x7F`.

---

## 2. Estrutura global

```text
Arquivo .ITA
├─ SCE_AT_HEADER                    0x10 bytes
└─ SCE_AT_ITEM[num]                 num * 0xB0 bytes
   ├─ SCE_AT_UNIT                   0x60 bytes
   │  ├─ AREA_HIT_DATA              0x30 bytes
   │  └─ campos comuns SCE_AT       0x30 bytes
   └─ SCE_AT_DATA_ITEM              0x50 bytes
```

Fórmula de tamanho:

```text
file_size = 0x10 + header.num * 0xB0
```

Um arquivo deve ser considerado inválido quando:

```text
file_size < 0x10
magic != "ITA"
version != 0x0105
file_size != 0x10 + num * 0xB0
```

---

## 3. Header `SCE_AT_HEADER`

Tamanho: `0x10` bytes.

| Offset | Tamanho | Tipo | Campo | Valor/uso |
|---:|---:|---|---|---|
| `0x00` | 4 | `char[4]` | `id` | Assinatura. Esperado: `ITA\0`. |
| `0x04` | 2 | `uint16` | `version` | Versão. Esperado: `0x0105`. |
| `0x06` | 2 | `uint16` | `num` | Quantidade de entradas `SCE_AT_ITEM`. |
| `0x08` | 4 | `uint32` | `pad1` | Reservado. Preservar. |
| `0x0C` | 4 | `uint32` | `pad2` | Reservado. Preservar. |

Estrutura equivalente:

```c
#pragma pack(push, 1)
typedef struct SCE_AT_HEADER {
    char     id[4];      // "ITA\0"
    uint16_t version;    // 0x0105
    uint16_t num;        // quantidade de entradas
    uint32_t pad1;       // reservado
    uint32_t pad2;       // reservado
} SCE_AT_HEADER;
#pragma pack(pop)
```

---

## 4. Entrada `SCE_AT_ITEM`

Cada entrada possui exatamente `0xB0` bytes.

```c
#pragma pack(push, 1)
typedef struct SCE_AT_ITEM {
    SCE_AT_UNIT      unit;   // +0x00, tamanho 0x60
    SCE_AT_DATA_ITEM item;   // +0x60, tamanho 0x50
} SCE_AT_ITEM;
#pragma pack(pop)
```

| Offset local | Tamanho | Bloco | Descrição |
|---:|---:|---|---|
| `0x00` | `0x60` | `SCE_AT_UNIT` | Campos comuns do sistema `SCE_AT`: área, tipo, número, prioridade, trigger, dependências e filtros. |
| `0x60` | `0x50` | `SCE_AT_DATA_ITEM` | Dados específicos do item: posição, item ID, quantidade, efeito, flags, raio e sons. |

---

## 5. Bloco comum `SCE_AT_UNIT`

`SCE_AT_UNIT` é a base compartilhada por vários tipos de entrada do sistema `SCE_AT`. Em arquivos `.ITA`, o campo `id` normalmente é `3`, equivalente a `SCEAT_ID_ITEM`.

Tamanho: `0x60` bytes.

```c
#pragma pack(push, 1)
typedef struct SCE_AT_UNIT {
    uint32_t      tag;              // +0x00
    AREA_HIT_DATA area;             // +0x04, 0x30 bytes

    uint8_t       be_flg;           // +0x34
    uint8_t       id;               // +0x35
    uint8_t       no;               // +0x36
    uint8_t       hit_type;         // +0x37
    uint8_t       trg_type;         // +0x38
    uint8_t       target_type;      // +0x39
    uint8_t       task_level;       // +0x3A
    uint8_t       trg_type_bak;     // +0x3B

    void*         pParam;           // +0x3C, ponteiro runtime
    void*         pFunc;            // +0x40, ponteiro runtime

    uint8_t       priority;         // +0x44
    uint8_t       kind;             // +0x45
    uint8_t       waiting_type;     // +0x46
    uint8_t       waiting_no;       // +0x47
    int8_t        hit_dir_ang;      // +0x48
    int8_t        hit_open_ang;     // +0x49
    uint8_t       act_type;         // +0x4A
    uint8_t       waiting_etc_id;   // +0x4B

    void*         pParent;          // +0x4C, ponteiro runtime
    int16_t       parts_no;         // +0x50
    uint16_t      country;          // +0x52
    uint8_t       act_color;        // +0x53
    uint8_t       pad2[8];          // +0x54
    uint32_t      pad32;            // +0x5C
} SCE_AT_UNIT;
#pragma pack(pop)
```

### Campos do `SCE_AT_UNIT`

| Offset | Tamanho | Tipo | Campo | Descrição |
|---:|---:|---|---|---|
| `0x00` | 4 | `uint32` | `tag` | Controle/lista interna. Preservar em roundtrip. |
| `0x04` | `0x30` | `AREA_HIT_DATA` | `area` | Área de interação/colisão do item. |
| `0x34` | 1 | `uint8` | `be_flg` | Flag base de habilitação/estado. Itens criados em runtime usam `7`. |
| `0x35` | 1 | `uint8` | `id` | Tipo da entrada. Para `.ITA`: `3` (`SCEAT_ID_ITEM`). |
| `0x36` | 1 | `uint8` | `no` | Número local da entrada. O loader soma `0x80` em runtime. |
| `0x37` | 1 | `uint8` | `hit_type` | Flags de hit/interação. Itens criados em runtime usam o bit `0x01`. |
| `0x38` | 1 | `uint8` | `trg_type` | Tipo de trigger. Itens comuns usam `8`. |
| `0x39` | 1 | `uint8` | `target_type` | Tipo de alvo. Itens criados em runtime usam `1`. |
| `0x3A` | 1 | `uint8` | `task_level` | Nível de tarefa/processamento. |
| `0x3B` | 1 | `uint8` | `trg_type_bak` | Backup do tipo de trigger. |
| `0x3C` | 4 | pointer | `pParam` | Ponteiro runtime. Não deve receber endereço fixo no arquivo. |
| `0x40` | 4 | pointer | `pFunc` | Ponteiro runtime para handler. O handler funcional de item é `sceAtFunc_item`. |
| `0x44` | 1 | `uint8` | `priority` | Prioridade de processamento. Itens criados em runtime usam `8`. |
| `0x45` | 1 | `uint8` | `kind` | Subtipo/agrupamento. Preservar quando desconhecido. |
| `0x46` | 1 | `uint8` | `waiting_type` | Tipo de dependência antes da ativação. |
| `0x47` | 1 | `uint8` | `waiting_no` | Índice associado a `waiting_type`. |
| `0x48` | 1 | `int8` | `hit_dir_ang` | Ângulo/direção de hit. |
| `0x49` | 1 | `int8` | `hit_open_ang` | Abertura angular de hit/interação. |
| `0x4A` | 1 | `uint8` | `act_type` | Tipo de ação. Itens comuns usam `0x28`. |
| `0x4B` | 1 | `uint8` | `waiting_etc_id` | Sub-índice de dependência/objeto. |
| `0x4C` | 4 | pointer | `pParent` | Ponteiro runtime para parent/modelo/objeto. |
| `0x50` | 2 | `int16` | `parts_no` | Parte do modelo/objeto parent associado. |
| `0x52` | 2 | `uint16` | `country` | Filtro de campanha/personagem/estado. |
| `0x53` | 1 | `uint8` | `act_color` | Cor/variante visual da ação/interação. |
| `0x54` | 8 | `uint8[8]` | `pad2` | Reservado. Preservar. |
| `0x5C` | 4 | `uint32` | `pad32` | Reservado. Preservar. |

---

## 6. Área de interação `AREA_HIT_DATA`

`AREA_HIT_DATA` começa em `SCE_AT_ITEM + 0x04` e ocupa `0x30` bytes.

```c
#pragma pack(push, 1)
typedef struct AREA_HIT_DATA {
    uint8_t  be_flag;      // +0x00
    uint8_t  type;         // +0x01
    uint8_t  pad[2];       // +0x02
    uint8_t  data[0x2C];   // +0x04, payload geométrico
} AREA_HIT_DATA;
#pragma pack(pop)
```

Offsets absolutos dentro da entrada:

```text
entry + 0x04 = AREA_HIT_DATA
entry + 0x05 = area.type
entry + 0x08 = início do payload geométrico
```

### Tipos de área encontrados no sistema

| Nome | Uso |
|---|---|
| `xz4` | Área quadrilateral no plano XZ. Usada em regiões retangulares/quadradas. |
| `cylinder` | Área cilíndrica/circular ao redor de uma posição. Adequada para pickups simples. |
| `eye_trigger` | Área dependente de direção/visão. Usada por triggers direcionais. |

### Área automática

O runtime possui uma rotina para gerar área de item a partir da posição e do raio:

```c
SceAtItemAutoArea(AREA_HIT_DATA& area, const tagVec& item_pos, float radius)
```

Regra de edição:

```text
ctrl_flag & 0x02 == 0  -> a área pode ser recalculada automaticamente
ctrl_flag & 0x02 != 0  -> preservar AREA_HIT_DATA manual/autoral
```

Para um item simples, normalmente basta ajustar `item_pos` e `radius` e deixar a área ser regenerada pelo jogo. Para item com área customizada, preserve o bloco `AREA_HIT_DATA` e mantenha o bit `0x02` em `ctrl_flag`.

---

## 7. Payload `SCE_AT_DATA_ITEM`

Bloco específico do `.ITA`.

Tamanho: `0x50` bytes.  
Offset dentro da entrada: `0x60`.

```c
#pragma pack(push, 1)
typedef struct SCE_AT_DATA_ITEM {
    tagVec    item_pos;           // +0x00, entrada +0x60
    tagVec    eff_offset;         // +0x10, entrada +0x70
    void*     pModel;             // +0x20, entrada +0x80

    uint16_t  item_id;            // +0x24, entrada +0x84
    uint16_t  item_flg;           // +0x26, entrada +0x86
    uint16_t  item_num;           // +0x28, entrada +0x88
    uint16_t  auto_item_flg;      // +0x2A, entrada +0x8A

    uint8_t   eff_type;           // +0x2C, entrada +0x8C
    uint8_t   eff_setno;          // +0x2D, entrada +0x8D
    uint8_t   player_type;        // +0x2E, entrada +0x8E
    uint8_t   pos_set;            // +0x2F, entrada +0x8F
    uint8_t   ctrl_flag;          // +0x30, entrada +0x90
    uint8_t   disappear_timer;    // +0x31, entrada +0x91
    int16_t   save_no;            // +0x32, entrada +0x92

    float     radius;             // +0x34, entrada +0x94
    float     ang_x;              // +0x38, entrada +0x98
    float     ang_y;              // +0x3C, entrada +0x9C
    float     open_ang;           // +0x40, entrada +0xA0

    uint8_t   se_no;              // +0x44, entrada +0xA4
    uint8_t   se_no2;             // +0x45, entrada +0xA5
    uint8_t   padd01[2];          // +0x46, entrada +0xA6
    uint8_t   tail_pad[8];        // +0x48, entrada +0xA8
} SCE_AT_DATA_ITEM;
#pragma pack(pop)
```

### Campos do payload

| Offset entrada | Offset payload | Tam. | Tipo | Campo | Descrição |
|---:|---:|---:|---|---|---|
| `0x60` | `0x00` | 16 | `tagVec` | `item_pos` | Posição do item no mundo (`x`, `y`, `z`, `w`). |
| `0x70` | `0x10` | 16 | `tagVec` | `eff_offset` | Offset aplicado ao efeito visual do item. |
| `0x80` | `0x20` | 4 | pointer | `pModel` | Ponteiro runtime para modelo. Preservar/zerar, nunca inventar endereço. |
| `0x84` | `0x24` | 2 | `uint16` | `item_id` | ID do item. Usado por inventário, modelo, mensagem e efeito. |
| `0x86` | `0x26` | 2 | `uint16` | `item_flg` | Flag de coleta/persistência. |
| `0x88` | `0x28` | 2 | `uint16` | `item_num` | Quantidade do item. |
| `0x8A` | `0x2A` | 2 | `uint16` | `auto_item_flg` | Flag auxiliar para item automático/save. Preservar se desconhecido. |
| `0x8C` | `0x2C` | 1 | `uint8` | `eff_type` | Tipo de efeito visual. Ver `SCEAT_ITEM_EFF_TYPE`. |
| `0x8D` | `0x2D` | 1 | `uint8` | `eff_setno` | Slot/número do efeito associado. |
| `0x8E` | `0x2E` | 1 | `uint8` | `player_type` | Filtro de personagem/player. |
| `0x8F` | `0x2F` | 1 | `uint8` | `pos_set` | Flags de estado/modelo temporário. |
| `0x90` | `0x30` | 1 | `uint8` | `ctrl_flag` | Flags principais de comportamento do item. |
| `0x91` | `0x31` | 1 | `uint8` | `disappear_timer` | Timer/contador de desaparecimento. |
| `0x92` | `0x32` | 2 | `int16` | `save_no` | Índice em `ITEM_SAVE_WORK`. `-1` indica ausência de slot explícito. |
| `0x94` | `0x34` | 4 | `float` | `radius` | Raio de interação usado na área automática. |
| `0x98` | `0x38` | 4 | `float` | `ang_x` | Parâmetro angular X. |
| `0x9C` | `0x3C` | 4 | `float` | `ang_y` | Parâmetro angular Y. |
| `0xA0` | `0x40` | 4 | `float` | `open_ang` | Parâmetro angular/abertura extra. |
| `0xA4` | `0x44` | 1 | `uint8` | `se_no` | Sound effect principal. |
| `0xA5` | `0x45` | 1 | `uint8` | `se_no2` | Sound effect secundário. |
| `0xA6` | `0x46` | 2 | `uint8[2]` | `padd01` | Reservado. Preservar. |
| `0xA8` | `0x48` | 8 | `uint8[8]` | `tail_pad` | Padding final. Preservar. |

### `tagVec`

Representação prática:

```c
typedef struct tagVec {
    float x;
    float y;
    float z;
    float w;
} tagVec;
```

Para posições, preserve `w` do arquivo original. Em muitos casos `w == 1.0f`, mas não é seguro forçar esse valor em todos os arquivos.

---

## 8. Tipos e enums relevantes

### `SCEAT_ID`

```c
typedef enum SCEAT_ID {
    SCEAT_ID_NORMAL       = 0,
    SCEAT_ID_DOOR         = 1,
    SCEAT_ID_EXEC         = 2,
    SCEAT_ID_ITEM         = 3,
    SCEAT_ID_FLG          = 4,
    SCEAT_ID_MES          = 5,
    SCEAT_ID_PLANTER      = 6,
    SCEAT_ID_JUMP         = 7,
    SCEAT_ID_SAVE         = 8,
    SCEAT_ID_SHD_DISP     = 9,
    SCEAT_ID_DAMAGE       = 10,
    SCEAT_ID_SCR_AT       = 11,
    SCEAT_ID_CAM_CTRL     = 12,
    SCEAT_ID_FIELD_INFO   = 13,
    SCEAT_ID_STOOP        = 14,
    SCEAT_ID_SKEY         = 15,
    SCEAT_ID_LADDER       = 16,
    SCEAT_ID_USE          = 17,
    SCEAT_ID_HIDE         = 18,
    SCEAT_ID_POS_JUMP     = 19,
    SCEAT_ID_ITEM_PARENT  = 20,
    SCEAT_ID_ADA_WIRE     = 21,
    SCEAT_ID_MAX          = 22
} SCEAT_ID;
```

Para `.ITA`, o valor esperado em `SCE_AT_UNIT.id` é:

```text
SCEAT_ID_ITEM = 3
```

### `SCEAT_ITEM_EFF_TYPE`

```c
typedef enum SCEAT_ITEM_EFF_TYPE {
    ITEM_EFF_TYPE_NONE        = 0,
    ITEM_EFF_TYPE_KIRA        = 1,
    ITEM_EFF_TYPE_EM_DROP_W   = 2,
    ITEM_EFF_TYPE_EM_DROP_B   = 3,
    ITEM_EFF_TYPE_EM_DROP_G   = 4,
    ITEM_EFF_TYPE_EM_DROP_R   = 5,
    ITEM_EFF_TYPE_AUTO        = 6,
    ITEM_EFF_TYPE_EM_DROP_BIG = 7,
    ITEM_EFF_TYPE_EM_DROP_KEY = 8,
    ITEM_EFF_TYPE_SANDGLASS   = 9,
    ITEM_EFF_TYPE_MAX         = 10
} SCEAT_ITEM_EFF_TYPE;
```

| Valor | Nome | Uso |
|---:|---|---|
| `0` | `ITEM_EFF_TYPE_NONE` | Sem efeito especial. |
| `1` | `ITEM_EFF_TYPE_KIRA` | Brilho normal de item. |
| `2` | `ITEM_EFF_TYPE_EM_DROP_W` | Drop de inimigo, variante W. |
| `3` | `ITEM_EFF_TYPE_EM_DROP_B` | Drop de inimigo, variante B. |
| `4` | `ITEM_EFF_TYPE_EM_DROP_G` | Drop de inimigo, variante G. |
| `5` | `ITEM_EFF_TYPE_EM_DROP_R` | Drop de inimigo, variante R. |
| `6` | `ITEM_EFF_TYPE_AUTO` | Efeito escolhido pelo runtime com base no `item_id`. |
| `7` | `ITEM_EFF_TYPE_EM_DROP_BIG` | Drop grande. |
| `8` | `ITEM_EFF_TYPE_EM_DROP_KEY` | Drop de chave/item importante. |
| `9` | `ITEM_EFF_TYPE_SANDGLASS` | Efeito de ampulheta/tempo. |

Quando `eff_type == ITEM_EFF_TYPE_AUTO`, o runtime usa uma checagem equivalente a:

```c
sceAtCheckItemEffectCol(item_id)
```

O retorno define o efeito efetivo usado para o item.

### `ITEM_CHAR_TYPE`

```c
typedef enum ITEM_CHAR_TYPE {
    ITEM_CHAR_LEON   = 0,
    ITEM_CHAR_ASHLEY = 1
} ITEM_CHAR_TYPE;
```

No `.ITA`, `player_type` atua como filtro de personagem/estado. Para itens comuns, `0` é o valor mais seguro. Valores não-zero devem ser preservados em salas especiais ou conteúdo de personagem específico.

---

## 9. Flags de comportamento

### `ctrl_flag`

Offset absoluto: `entry + 0x90`.

| Bit | Máscara | Uso observado |
|---:|---:|---|
| `0` | `0x01` | Gate/estado de controle. Usado em caminhos dependentes de objeto/inimigo. Preservar em itens existentes. |
| `1` | `0x02` | Mantém área manual. Sem esse bit, a área pode ser recalculada por `SceAtItemAutoArea`. |
| `4` | `0x10` | Item associado a modelo/parent ou item derrubável/atingível. Ativa caminhos de `SceAtSetShootDownItem`. |
| `6` | `0x40` | Força criação/associação de objeto/modelo de item via `setItemObj` / `SceAtSetItemModel`. |
| `7` | `0x80` | Modo sistêmico/especial. Afeta trigger/action durante configuração. |

Configuração típica para item simples:

```text
id          = 3
trg_type    = 8
act_type    = 0x28
priority    = 8
ctrl_flag   = 0x00 ou valor clonado de item similar
item_pos    = posição do item
radius      = raio de interação
```

Item com área manual:

```text
ctrl_flag |= 0x02
preservar AREA_HIT_DATA
```

Item vinculado a objeto/inimigo:

```text
preservar waiting_type
preservar waiting_no
preservar waiting_etc_id
preservar bits 0x10/0x40 quando presentes
```

### `pos_set`

Offset absoluto: `entry + 0x8F`.

`pos_set` controla estado de modelo/posição, principalmente em fluxos de zoom, carregamento temporário e restauração de modelo.

| Bit | Máscara | Uso observado |
|---:|---:|---|
| `1` | `0x02` | Modelo carregado dinamicamente para zoom/exame. Liberado por `releaseModel`. |
| `2` | `0x04` | Estado auxiliar de modelo/posição. Preservar. |
| `3` | `0x08` | Modelo existente movido/salvo temporariamente para zoom e depois restaurado. |

Regra segura:

```text
Não criar combinações novas de pos_set sem referência.
Para item novo simples, usar 0x00 ou clonar de item equivalente.
```

### `waiting_type`, `waiting_no`, `waiting_etc_id`

Offsets:

```text
entry + 0x46 = waiting_type
entry + 0x47 = waiting_no
entry + 0x4B = waiting_etc_id
```

| `waiting_type` | Comportamento |
|---:|---|
| `0` | Item independente, sem dependência especial. |
| `1` | Dependência de inimigo/lista de inimigos. `waiting_no` referencia o inimigo. |
| `2` | Dependência de objeto quebrável/objeto de sala. Usa `waiting_no` e `waiting_etc_id`. |

Aplicação prática:

| Caso | Valor comum |
|---|---|
| Pickup direto no mapa | `waiting_type = 0` |
| Drop de inimigo | `waiting_type = 1` |
| Item em caixa/barril/objeto quebrável | `waiting_type = 2` |

---

## 10. Itens, quantidade, flags e persistência

### `item_id`

Offset: `entry + 0x84`.

`item_id` é o identificador principal do item. Ele alimenta rotinas de inventário, mensagem, modelo e efeito:

```c
itemInfo(item_id)
ItemGetBinTplAddr(item_id & 0xFF)
SceAtCheckSystemItemSet(...)
sceAtCheckItemEffectCol(item_id)
```

Um ID inválido pode resultar em:

- item invisível;
- item sem modelo;
- fallback para coleta sem modelo;
- mensagem incorreta;
- falha ao buscar BIN/TPL;
- travamento por acesso inválido em tabela interna.

### `item_num`

Offset: `entry + 0x88`.

| Tipo de item | Uso de `item_num` |
|---|---|
| Munição | Quantidade de balas. |
| Dinheiro | Valor/quantidade. |
| Ervas/consumíveis simples | Normalmente `1`. |
| Chaves/tesouros | Normalmente `1`. |

### `item_flg`

Offset: `entry + 0x86`.

`item_flg` controla persistência de coleta. Funções associadas:

```c
sceAtItemFlgOn(SCE_AT_DATA_ITEM*)
sceAtItemFlgCk(SCE_AT_DATA_ITEM*)
sceAtItemFindFlgOn(SCE_AT_DATA_ITEM*)
sceAtItemFindFlgCk(SCE_AT_DATA_ITEM*)
```

Fluxo geral:

```text
1. O runtime verifica se item_flg já está marcado.
2. Se estiver marcado, a entrada pode ser escondida/desabilitada.
3. Ao coletar, o jogo marca item_flg.
4. A entrada SCE_AT é desabilitada.
```

Regra de modding:

```text
Não reutilizar o mesmo item_flg em pickups independentes.
Flags duplicadas fazem itens diferentes compartilharem o mesmo estado de coleta.
```

### `ITEM_INFO`

```c
typedef struct ITEM_INFO {
    uint16_t id;       // ITEM_ID
    uint8_t  type;
    uint8_t  defNum;
    uint16_t maxNum;
} ITEM_INFO;
```

`item_id` define mais que o visual: também influencia tipo, quantidade padrão e quantidade máxima.

### Save de itens

Rotinas associadas:

```c
SceAtInitSaveItem()
SceAtSetSaveItem()
sceAtPullItemSaveWork()
sceAtCheckSaveItem(item_id, save_no)
SceAtCheckSaveItemId(...)
```

Estrutura identificada:

```c
#pragma pack(push, 1)
typedef struct ITEM_SAVE_WORK {
    uint8_t  item_type;    // +0x00
    uint8_t  item_at;      // +0x01
    uint8_t  item_eff;     // +0x02
    uint8_t  padd0;        // +0x03
    uint16_t room_no;      // +0x04
    uint16_t item_id;      // +0x06
    uint16_t item_num;     // +0x08
    int16_t  item_pos_x;   // +0x0A
    int16_t  item_pos_y;   // +0x0C
    int16_t  item_pos_z;   // +0x0E
} ITEM_SAVE_WORK;
#pragma pack(pop)
```

Campo relacionado no `.ITA`:

```text
entry + 0x92 = save_no
```

| Valor | Significado |
|---:|---|
| `-1` / `0xFFFF` | Sem slot explícito de save item. |
| `>= 0` | Índice em estrutura de persistência de item. |

Para item novo comum, o valor mais seguro é:

```text
save_no = -1
```

---

## 11. Fluxo de execução no jogo

### Carregamento

Fluxo principal:

```text
1. Validar assinatura "ITA".
2. Validar versão 0x0105.
3. Ler quantidade em header + 0x06.
4. Primeira entrada em header + 0x10.
5. Para cada entrada:
   - acessar SCE_AT_ITEM com stride 0xB0;
   - somar 0x80 ao campo no;
   - registrar entrada no sistema SCE_AT.
```

### Preparação do item

Rotinas envolvidas:

```c
sceAtSetItem(SCE_AT_ITEM*)
SceAtItemAutoArea(AREA_HIT_DATA&, const tagVec&, float)
sceAtItemEffSet(SCE_AT_ITEM*, cModel*)
SceAtSetItemModel(SCE_AT_ITEM*, cObj*)
SceAtSetShootDownItem(SCE_AT_ITEM*, ...)
```

Fluxo observado:

```text
1. Verificar flag de coleta.
2. Verificar filtro de player/personagem.
3. Resolver dependências waiting_type/waiting_no.
4. Aplicar conversões de item por lógica de sistema.
5. Recalcular área automática, se permitido.
6. Associar/carregar modelo, quando necessário.
7. Criar efeito visual conforme eff_type.
```

### Interação/coleta

Handler principal:

```c
sceAtFunc_item(SCE_AT_DATA* at, cModel* model)
```

Fluxo simplificado:

```text
1. Limpar estado de input/key.
2. Chamar itemZoom(SCE_AT_ITEM*).
3. Se itemZoom retornar 1:
      executar sceAtGetItem(SCE_AT_ITEM*)
   Caso contrário:
      executar sceAtGetItem_NoModel(SCE_AT_ITEM*)
4. Ao coletar:
      marcar item_flg
      desabilitar entrada SCE_AT
      remover/liberar modelo temporário quando necessário
      remover efeito visual do item
```

### Zoom e modelo

Função:

```c
itemZoom(SCE_AT_ITEM*)
```

Comportamento:

```text
- Usa modelo existente quando disponível.
- Caso contrário, tenta carregar BIN/TPL pelo item_id.
- Se o modelo carregar, ajusta bits em pos_set e segue pelo fluxo com zoom.
- Se não houver modelo, usa o fluxo de coleta sem modelo.
```

IDs com tratamento especial em caminhos de zoom/modelo:

```text
item_id == 0x95
item_id == 0x97
```

---

## 12. Funções relevantes no executável

| Endereço | Função | Finalidade |
|---:|---|---|
| `0x002956B8` | `SceAtInit` | Inicializa `.AEV`/`.ITA`, valida assinatura/versão e registra entradas. |
| `0x002970C0` | `itemZoom` | Prepara modelo/zoom do item antes da coleta. |
| `0x002972F8` | `releaseModel` | Libera ou restaura modelo temporário. |
| `0x002973C0` | `sceAtGetItem` | Fluxo de coleta com modelo/zoom. |
| `0x00297F60` | `sceAtGetItem_NoModel` | Fluxo de coleta sem modelo. |
| `0x00298838` | `sceAtFunc_item` | Handler principal de interação/coleta do item. |
| `0x0029BDC8` | `sceAtItemFlgOn` | Marca flag de item como coletado/encontrado. |
| `0x0029BEA0` | `sceAtItemFlgCk` | Verifica se flag de item já foi marcada. |
| `0x0029C638` | `SceAtCreateItemAt` | Cria item Atari dinamicamente. |
| `0x0029C9C0` | `SceAtReserveItemAt` | Reserva/cria item associado a inimigo/drop. |
| `0x0029CBE8` | `SceAtCancelItemAt` | Cancela item reservado. |
| `0x0029D1E0` | `SceAtSetSaveItem` | Atualiza lista de itens salvos. |
| `0x0029D398` | `SceAtInitSaveItem` | Inicializa estrutura de save de itens. |
| `0x0029D500` | `sceAtSetItemModelParent` | Define parent do modelo de item. |
| `0x0029D600` | `SceAtSetItemModel(uint, cObj*)` | Associa modelo ao item por número. |
| `0x0029D660` | `SceAtSetItemModel(SCE_AT_ITEM*, cObj*)` | Associa modelo diretamente à entrada `.ITA`. |
| `0x0029D748` | `SceAtSetShootDownItem` | Configura item derrubável/atingível/parented. |
| `0x0029D8B8` | `SceAtItemHitCheck` | Checa colisão/hit de item. |
| `0x0029DC70` | `SceAtCheckSystemItemSet` | Ajusta/substitui item conforme lógica de sistema. |
| `0x0029DF78` | `sceAtSetItem` | Configuração/manutenção principal de item. |
| `0x0029E620` | `SceAtItemAutoArea` | Gera área automática a partir de posição e raio. |
| `0x0029E6A0` | `SceAtDeleteItem` | Remove item. |
| `0x0029E6E8` | `sceAtItemEffDelete` | Deleta efeito visual do item. |
| `0x0029E760` | `sceAtItemEffSet` | Cria efeito visual do item. |
| `0x0029EFE0` | `sceAtItemDisappearEffSet` | Cria efeito de desaparecimento. |

---

## 13. Regras para editores e repackers

### Abrir

1. Ler `0x10` bytes de header.
2. Validar assinatura `ITA`.
3. Validar versão `0x0105`.
4. Ler `num` em `0x06`.
5. Validar tamanho `0x10 + num * 0xB0`.
6. Parsear cada entrada com stride fixo `0xB0`.

### Salvar

1. Manter little-endian.
2. Manter header com `0x10` bytes.
3. Atualizar `num` ao adicionar/remover entradas.
4. Recalcular tamanho final.
5. Preservar padding e campos desconhecidos.
6. Não aplicar `no += 0x80` no arquivo.
7. Não gravar ponteiros runtime inventados.
8. Preservar `AREA_HIT_DATA` quando `ctrl_flag & 0x02` estiver ativo.

### Campos que devem ser preservados

```text
tag
pParam
pFunc
pParent
pad2
pad32
pModel
padd01
tail_pad
pos_set desconhecido
ctrl_flag bits desconhecidos
waiting_* em itens dependentes
country/player_type em salas especiais
```

### Adicionar item simples

Fluxo recomendado:

```text
1. Clonar uma entrada simples da mesma sala ou de sala semelhante.
2. Alterar:
   - no
   - item_pos
   - item_id
   - item_num
   - item_flg
   - radius
   - eff_type / eff_setno, se necessário
3. Manter:
   - id = 3
   - priority
   - hit_type
   - trg_type
   - target_type
   - act_type
   - country/player_type quando aplicável
4. Zerar ponteiros runtime somente quando o padrão do arquivo original também usar zero.
5. Testar no jogo.
```

### Adicionar drop de inimigo ou objeto quebrável

Não criar do zero. Clonar uma entrada com o mesmo padrão de dependência:

```text
waiting_type
waiting_no
waiting_etc_id
ctrl_flag
pos_set
parts_no
pParent/pModel, quando preservados no arquivo
```

Depois alterar apenas campos de item, efeito, quantidade, flag e posição quando necessário.

### Remover item

Opções comuns:

```text
A. Remover a entrada e decrementar num.
B. Manter a entrada e desabilitar por estado/flag.
C. Mover o item para fora da área jogável, preservando a estrutura.
```

A opção C costuma ser a mais segura para testes em salas com lógica dependente.

---

## 14. Representação JSON/YAML recomendada

Representação editável com preservação de bytes desconhecidos:

```yaml
magic: ITA
version: 0x0105
entries:
  - index: 0
    unit:
      tag: 0x00000000
      area:
        raw_hex: ""
        type: 1
      be_flg: 7
      id: 3
      no: 0
      hit_type: 1
      trg_type: 8
      target_type: 1
      task_level: 0
      priority: 8
      kind: 0
      waiting_type: 0
      waiting_no: 0
      hit_dir_ang: 0
      hit_open_ang: 0
      act_type: 0x28
      waiting_etc_id: 0
      parts_no: -1
      country: 0
      act_color: 0
      reserved_hex: ""
    item:
      item_pos: [0.0, 0.0, 0.0, 1.0]
      eff_offset: [0.0, 0.0, 0.0, 0.0]
      item_id: 0x0000
      item_flg: 0x0000
      item_num: 1
      auto_item_flg: 0x0000
      eff_type: ITEM_EFF_TYPE_AUTO
      eff_setno: 0
      player_type: 0
      pos_set: 0x00
      ctrl_flag: 0x00
      disappear_timer: 0
      save_no: -1
      radius: 100.0
      ang_x: 0.0
      ang_y: 0.0
      open_ang: 0.0
      se_no: 0
      se_no2: 0
      padding_hex: ""
```

Recomendação: guardar `raw_hex`, `reserved_hex` ou blocos equivalentes para qualquer byte ainda não editado semanticamente. Isso permite roundtrip fiel mesmo antes de todos os campos serem nomeados.

---

## 15. Parser de referência em Python

```python
from __future__ import annotations

import struct
from dataclasses import dataclass
from pathlib import Path

HEADER_SIZE = 0x10
ITA_VERSION = 0x0105
ITA_ENTRY_SIZE = 0xB0


@dataclass
class ItaHeader:
    magic: bytes
    version: int
    count: int
    pad1: int
    pad2: int


@dataclass
class ItaItem:
    index: int
    offset: int
    raw: bytes

    tag: int
    area_type: int
    be_flg: int
    sce_id: int
    no: int
    hit_type: int
    trg_type: int
    target_type: int
    task_level: int
    trg_type_bak: int
    priority: int
    kind: int
    waiting_type: int
    waiting_no: int
    hit_dir_ang: int
    hit_open_ang: int
    act_type: int
    waiting_etc_id: int
    parts_no: int
    country: int
    act_color: int

    item_pos: tuple[float, float, float, float]
    eff_offset: tuple[float, float, float, float]
    p_model: int
    item_id: int
    item_flg: int
    item_num: int
    auto_item_flg: int
    eff_type: int
    eff_setno: int
    player_type: int
    pos_set: int
    ctrl_flag: int
    disappear_timer: int
    save_no: int
    radius: float
    ang_x: float
    ang_y: float
    open_ang: float
    se_no: int
    se_no2: int


def read_header(data: bytes) -> ItaHeader:
    if len(data) < HEADER_SIZE:
        raise ValueError("Arquivo muito pequeno para header ITA")

    magic, version, count, pad1, pad2 = struct.unpack_from("<4sHHII", data, 0)

    if magic[:3] != b"ITA":
        raise ValueError(f"Assinatura inválida: {magic!r}; esperado ITA")

    if version != ITA_VERSION:
        raise ValueError(f"Versão inválida: 0x{version:04X}; esperado 0x{ITA_VERSION:04X}")

    expected_size = HEADER_SIZE + count * ITA_ENTRY_SIZE
    if len(data) != expected_size:
        raise ValueError(
            f"Tamanho inválido: arquivo tem 0x{len(data):X}, "
            f"esperado 0x{expected_size:X} para {count} entradas"
        )

    return ItaHeader(magic, version, count, pad1, pad2)


def parse_entry(data: bytes, index: int, offset: int) -> ItaItem:
    raw = data[offset:offset + ITA_ENTRY_SIZE]
    if len(raw) != ITA_ENTRY_SIZE:
        raise ValueError(f"Entrada {index} incompleta")

    tag = struct.unpack_from("<I", data, offset + 0x00)[0]
    area_type = struct.unpack_from("<B", data, offset + 0x05)[0]

    be_flg, sce_id, no, hit_type = struct.unpack_from("<BBBB", data, offset + 0x34)
    trg_type, target_type, task_level, trg_type_bak = struct.unpack_from("<BBBB", data, offset + 0x38)

    priority = struct.unpack_from("<B", data, offset + 0x44)[0]
    kind = struct.unpack_from("<B", data, offset + 0x45)[0]
    waiting_type, waiting_no = struct.unpack_from("<BB", data, offset + 0x46)
    hit_dir_ang, hit_open_ang = struct.unpack_from("<bb", data, offset + 0x48)
    act_type = struct.unpack_from("<B", data, offset + 0x4A)[0]
    waiting_etc_id = struct.unpack_from("<B", data, offset + 0x4B)[0]
    parts_no = struct.unpack_from("<h", data, offset + 0x50)[0]
    country = struct.unpack_from("<H", data, offset + 0x52)[0]
    act_color = struct.unpack_from("<B", data, offset + 0x53)[0]

    item_pos = struct.unpack_from("<4f", data, offset + 0x60)
    eff_offset = struct.unpack_from("<4f", data, offset + 0x70)
    p_model = struct.unpack_from("<I", data, offset + 0x80)[0]

    item_id, item_flg, item_num, auto_item_flg = struct.unpack_from("<HHHH", data, offset + 0x84)
    eff_type, eff_setno, player_type, pos_set = struct.unpack_from("<BBBB", data, offset + 0x8C)
    ctrl_flag, disappear_timer = struct.unpack_from("<BB", data, offset + 0x90)
    save_no = struct.unpack_from("<h", data, offset + 0x92)[0]
    radius, ang_x, ang_y, open_ang = struct.unpack_from("<4f", data, offset + 0x94)
    se_no, se_no2 = struct.unpack_from("<BB", data, offset + 0xA4)

    return ItaItem(
        index=index,
        offset=offset,
        raw=raw,
        tag=tag,
        area_type=area_type,
        be_flg=be_flg,
        sce_id=sce_id,
        no=no,
        hit_type=hit_type,
        trg_type=trg_type,
        target_type=target_type,
        task_level=task_level,
        trg_type_bak=trg_type_bak,
        priority=priority,
        kind=kind,
        waiting_type=waiting_type,
        waiting_no=waiting_no,
        hit_dir_ang=hit_dir_ang,
        hit_open_ang=hit_open_ang,
        act_type=act_type,
        waiting_etc_id=waiting_etc_id,
        parts_no=parts_no,
        country=country,
        act_color=act_color,
        item_pos=item_pos,
        eff_offset=eff_offset,
        p_model=p_model,
        item_id=item_id,
        item_flg=item_flg,
        item_num=item_num,
        auto_item_flg=auto_item_flg,
        eff_type=eff_type,
        eff_setno=eff_setno,
        player_type=player_type,
        pos_set=pos_set,
        ctrl_flag=ctrl_flag,
        disappear_timer=disappear_timer,
        save_no=save_no,
        radius=radius,
        ang_x=ang_x,
        ang_y=ang_y,
        open_ang=open_ang,
        se_no=se_no,
        se_no2=se_no2,
    )


def parse_ita(path: str | Path) -> tuple[ItaHeader, list[ItaItem]]:
    data = Path(path).read_bytes()
    header = read_header(data)
    items: list[ItaItem] = []

    for index in range(header.count):
        offset = HEADER_SIZE + index * ITA_ENTRY_SIZE
        items.append(parse_entry(data, index, offset))

    return header, items


if __name__ == "__main__":
    import sys

    header, items = parse_ita(sys.argv[1])
    print(f"ITA version=0x{header.version:04X} count={header.count}")

    for item in items:
        print(
            f"[{item.index:03}] off=0x{item.offset:06X} "
            f"no=0x{item.no:02X}->runtime=0x{(item.no + 0x80) & 0xFF:02X} "
            f"id={item.sce_id} item_id=0x{item.item_id:04X} num={item.item_num} "
            f"flg=0x{item.item_flg:04X} eff={item.eff_type} "
            f"pos=({item.item_pos[0]:.2f}, {item.item_pos[1]:.2f}, {item.item_pos[2]:.2f}) "
            f"radius={item.radius:.2f} ctrl=0x{item.ctrl_flag:02X}"
        )
```

---

## 16. Checklist de roundtrip

```text
[ ] Header mantém "ITA\0".
[ ] version mantém 0x0105.
[ ] count corresponde ao número real de entradas.
[ ] Tamanho final = 0x10 + count * 0xB0.
[ ] Cada entrada possui exatamente 0xB0 bytes.
[ ] Floats continuam little-endian.
[ ] uint16/uint32 continuam little-endian.
[ ] no é salvo sem aplicar +0x80 manualmente.
[ ] pModel, pParam, pFunc e pParent não recebem ponteiros inventados.
[ ] Padding e bytes reservados são preservados.
[ ] item_flg não foi duplicado acidentalmente.
[ ] item_id existe nas tabelas internas do jogo.
[ ] eff_type está no intervalo 0..9.
[ ] save_no usa -1 quando não há integração explícita com save item.
[ ] Área manual mantém ctrl_flag bit 0x02.
[ ] Itens dependentes preservam waiting_type, waiting_no e waiting_etc_id.
```

Campos de maior risco:

| Campo | Risco |
|---|---|
| `item_id` | ID inválido pode quebrar modelo, inventário, mensagem ou efeito. |
| `item_flg` | Duplicação faz itens compartilharem estado de coleta. |
| `no` | Salvar já com `+0x80` pode causar conflito na faixa runtime. |
| `ctrl_flag` | Afeta área, modelo, trigger, parent e comportamento de drop. |
| `waiting_type/waiting_no` | Pode quebrar drops de inimigos/objetos. |
| `pModel/pParent/pFunc/pParam` | Ponteiros runtime não devem ser inventados em arquivo estático. |
| `save_no` | Slot inválido pode quebrar persistência. |
| `AREA_HIT_DATA` | Área incorreta pode impedir coleta ou ativar item longe demais. |
| `player_type/country` | Pode ocultar item dependendo do personagem/campanha. |

---

## 17. Resumo de offsets

```text
0x0000 SCE_AT_HEADER
       00 char[4]  id       = "ITA\0"
       04 u16      version  = 0x0105
       06 u16      num
       08 u32      pad1
       0C u32      pad2

0x0010 SCE_AT_ITEM[num]
       tamanho por entrada = 0xB0

Entrada SCE_AT_ITEM:
0x00 SCE_AT_UNIT
     00 u32              tag
     04 AREA_HIT_DATA    area[0x30]
     34 u8               be_flg
     35 u8               id = 3
     36 u8               no, loader soma +0x80
     37 u8               hit_type
     38 u8               trg_type
     39 u8               target_type
     3A u8               task_level
     3B u8               trg_type_bak
     3C ptr              pParam
     40 ptr              pFunc
     44 u8               priority
     45 u8               kind
     46 u8               waiting_type
     47 u8               waiting_no
     48 s8               hit_dir_ang
     49 s8               hit_open_ang
     4A u8               act_type
     4B u8               waiting_etc_id
     4C ptr              pParent
     50 s16              parts_no
     52 u16              country
     53 u8               act_color
     54 u8[8]            pad2
     5C u32              pad32

0x60 SCE_AT_DATA_ITEM
     60 tagVec           item_pos
     70 tagVec           eff_offset
     80 ptr              pModel
     84 u16              item_id
     86 u16              item_flg
     88 u16              item_num
     8A u16              auto_item_flg
     8C u8               eff_type
     8D u8               eff_setno
     8E u8               player_type
     8F u8               pos_set
     90 u8               ctrl_flag
     91 u8               disappear_timer
     92 s16              save_no
     94 float            radius
     98 float            ang_x
     9C float            ang_y
     A0 float            open_ang
     A4 u8               se_no
     A5 u8               se_no2
     A6 u8[2]            padd01
     A8 u8[8]            padding final
```

---

## Modelo mental para edição

Uma entrada `.ITA` é a soma de vários sistemas:

```text
área de interação
+ tipo de ação
+ filtro de personagem/cenário
+ ID do item
+ quantidade
+ flag de persistência
+ efeito visual
+ modelo BIN/TPL opcional
+ vínculo opcional com inimigo/objeto
+ slot opcional de save
```

A forma mais segura de modificar é clonar uma entrada parecida e alterar poucos campos por vez.

Campos normalmente seguros para edição controlada:

```text
item_pos
item_id
item_num
item_flg, quando uma flag livre for conhecida
radius
eff_type
eff_setno
se_no
se_no2
```

Campos que exigem cautela:

```text
ctrl_flag
pos_set
waiting_type
waiting_no
waiting_etc_id
country
player_type
save_no
AREA_HIT_DATA manual
ponteiros runtime
```
