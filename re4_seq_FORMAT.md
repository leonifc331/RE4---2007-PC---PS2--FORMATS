# Resident Evil 4 PS2 — Formato `.SEQ`

Documentação técnica do formato `.SEQ` usado por **Resident Evil 4** de PlayStation 2 para tabelas de sequência associadas a animações, câmera e eventos de som/efeito.

O `.SEQ` armazena uma curva compacta de reprodução: cada entrada liga um índice de sequência a um frame de animação em unidade fixa de `1/64` de frame e, opcionalmente, a um evento de sequência identificado por `type` e `no`.

## Sumário

- [1. Características gerais](#1-características-gerais)
- [2. Estrutura global](#2-estrutura-global)
- [3. Header](#3-header)
- [4. Entrada `SEQ_ENTRY`](#4-entrada-seq_entry)
- [5. Campos da entrada](#5-campos-da-entrada)
- [6. Valores aceitos por campo](#6-valores-aceitos-por-campo)
- [7. Interpretação de `frame64`](#7-interpretação-de-frame64)
- [8. Interpretação de `type` e `no`](#8-interpretação-de-type-e-no)
- [9. Fluxo runtime](#9-fluxo-runtime)
- [10. Funções e símbolos relevantes](#10-funções-e-símbolos-relevantes)
- [11. Amostras analisadas](#11-amostras-analisadas)
- [12. Regras para editores e repackers](#12-regras-para-editores-e-repackers)
- [13. Representação JSON/YAML recomendada](#13-representação-jsonyaml-recomendada)
- [14. Structs de referência](#14-structs-de-referência)
- [15. Parser de referência em Python](#15-parser-de-referência-em-python)
- [16. Checklist de roundtrip](#16-checklist-de-roundtrip)
- [17. Resumo de offsets](#17-resumo-de-offsets)

---

## 1. Características gerais

| Propriedade | Valor |
|---|---:|
| Assinatura | Não possui |
| Endianness | little-endian |
| Header em arquivo | `0x04` bytes |
| Estrutura principal | tabela compacta de sequência |
| Estrutura de entrada | `SEQ_ENTRY` |
| Tamanho da entrada | `0x04` bytes |
| Alinhamento observado | `0x20` bytes |
| Padding observado | `0xCD` |
| Unidade de frame | `1/64` de frame |
| Sistemas relacionados | `MotionSequenceCtrl`, `CameraSequenceCtrl`, `seqSeCtrl` |
| Plataforma analisada | PlayStation 2 |

O arquivo não possui magic ASCII nem versão. A identificação é feita pelo contexto do container de sala e pelo layout interno:

```text
uint32 EntryCount
SEQ_ENTRY Entries[EntryCount]
uint8 Padding[]
```

O padding final não é interpretado pelo runtime. Nos arquivos analisados, o preenchimento é feito com `0xCD` até múltiplo de `0x20` bytes.

> O executável também possui funções `SmfSeq_*` relacionadas a sequência MIDI/SMF. Essas rotinas pertencem ao sistema de áudio MIDI e não descrevem este `.SEQ` compacto usado por animação/eventos.

---

## 2. Estrutura global

```text
Arquivo .SEQ
├─ Header                         0x04 bytes
│  └─ EntryCount                  uint32
│
├─ SEQ_ENTRY[EntryCount]
│  ├─ Entry[0]                    0x04 bytes
│  ├─ Entry[1]                    0x04 bytes
│  ├─ Entry[2]                    0x04 bytes
│  └─ ...
│
└─ Padding opcional               até alinhamento 0x20
```

Tamanho mínimo válido:

```text
0x04 + EntryCount * 0x04
```

Tamanho alinhado recomendado para repack:

```text
align(0x04 + EntryCount * 0x04, 0x20)
```

---

## 3. Header

| Offset | Tamanho | Tipo | Nome | Descrição |
|---:|---:|---|---|---|
| `0x00` | `0x04` | `uint32` | `EntryCount` | Quantidade de entradas `SEQ_ENTRY`. |

### Struct

```c
typedef struct RE4_SEQ_HEADER {
    uint32_t EntryCount;
} RE4_SEQ_HEADER;
```

### Observações

- `EntryCount` é little-endian.
- O runtime usa a tabela como uma sequência indexada, não como uma lista de offsets.
- O valor deve bater exatamente com o número de entradas antes do padding.
- Para edição segura, não conte bytes `0xCD` como entradas.

---

## 4. Entrada `SEQ_ENTRY`

Cada entrada possui `0x04` bytes:

| Offset local | Tamanho | Tipo | Nome | Descrição |
|---:|---:|---|---|---|
| `0x00` | `0x01` | `uint8` | `Type` | Tipo/classe do evento de sequência. |
| `0x01` | `0x01` | `uint8` | `No` | Número do evento. `0` normalmente indica ausência de evento. |
| `0x02` | `0x02` | `uint16` | `Frame64` | Frame de animação em unidade fixa de `1/64`. |

### Struct

```c
typedef struct RE4_SEQ_ENTRY {
    uint8_t  Type;
    uint8_t  No;
    uint16_t Frame64;
} RE4_SEQ_ENTRY;
```

### Palavra compactada

Como a plataforma é little-endian, a entrada também pode ser vista como um `uint32`:

```text
bits  0..7   = Type
bits  8..15  = No
bits 16..31  = Frame64
```

Exemplo:

```text
00 2B E0 01
```

Interpretação:

```text
Type    = 0x00
No      = 0x2B
Frame64 = 0x01E0
Frame   = 0x01E0 / 64 = 7.5
```

---

## 5. Campos da entrada

### `Type`

Define a classe/modo do evento associado à entrada. O campo é lido como byte e interpretado de forma diferente dependendo do controlador que consome a sequência.

Nos arquivos analisados, todos os eventos usam `Type = 0x00`, que funciona como modo padrão/automático.

### `No`

Número do evento associado ao índice de sequência. Quando `No = 0`, a entrada apenas define mapeamento de frame e normalmente não dispara evento de som/efeito.

Quando `No != 0`, o valor é repassado aos controladores de sequência do personagem, câmera ou objeto. A semântica concreta depende do consumidor.

### `Frame64`

Frame de animação em ponto fixo, usando escala `64`:

```text
frame_float = Frame64 / 64.0
Frame64     = round(frame_float * 64.0)
```

O valor é usado por `MotionSequenceCtrl` para alimentar o frame atual de movimento, inclusive com interpolação entre duas entradas consecutivas.

---

## 6. Valores aceitos por campo

### `EntryCount`

| Valor | Uso |
|---:|---|
| `0` | Tabela vazia. Válido estruturalmente, mas normalmente inútil. |
| `1..65535` | Faixa segura por compatibilidade com contadores runtime de 16 bits. |
| `> 65535` | Não recomendado. Pode ultrapassar campos de trabalho usados pelo player. |

Validação obrigatória:

```text
0x04 + EntryCount * 0x04 <= file_size
```

### `Type`

| Valor | Interpretação prática |
|---:|---|
| `0x00` | Modo padrão. Valor usado em todas as amostras analisadas. |
| `0x01` | Classe alternativa 1. Usada por controladores que mascaram `Type & 0x03` ou `Type & 0x07`. |
| `0x02` | Classe alternativa 2. |
| `0x03` | Classe alternativa 3. |
| `0x04..0x07` | Classes/flags adicionais consumidas pelo player, pois `seqSeCtrl__7cPlayer` usa `Type & 0x07`. |
| `0x08..0xFF` | Aceito como byte, mas os bits altos tendem a ser ignorados pelos consumidores conhecidos. Para repack seguro, preservar. |

Regras por consumidor:

| Consumidor | Máscara aplicada | Faixa efetiva |
|---|---:|---:|
| `seqSeCtrl__8cSubChar` | `Type & 0x03` | `0..3` |
| `seqSeCtrl__7cPlayer` | `Type & 0x07` | `0..7` |

### `No`

| Valor | Interpretação prática |
|---:|---|
| `0x00` | Sem evento. A entrada só contribui para o mapeamento/interpolação de frame. |
| `0x01..0x15` | Faixa com tratamento específico no controlador do player. |
| `0x01..0x18` | Faixa com tratamento específico em `seqSeCtrl__8cSubChar`. |
| `0x01..0x63` | Faixa genérica usada por mapeamentos de som/efeito em controladores de personagem. |
| `0x64..0xFF` | Valor binariamente aceito, mas pode não ter handler útil para todos os consumidores. |

Valores observados nas amostras:

| `No` | Uso observado |
|---:|---|
| `0x00` | Sem evento / key de interpolação. |
| `0x03` | Evento de sequência. |
| `0x04` | Evento de sequência. |
| `0x2B` | Evento de sequência. |

### `Frame64`

| Valor | Interpretação |
|---:|---|
| `0x0000` | Frame `0.0`. |
| `0x0001..0xFFFF` | Frame `Frame64 / 64.0`. |

Regras recomendadas:

| Regra | Motivo |
|---|---|
| Manter valores não decrescentes | Evita regressão brusca de animação durante interpolação. |
| Primeiro valor geralmente `0x0000` | Início natural da curva. |
| Último valor deve representar o fim útil da animação | `MotionSequenceCtrl` usa a tabela para derivar o frame ativo. |
| Usar múltiplos coerentes com `1/64` | Preserva precisão de subframe. |

---

## 7. Interpretação de `frame64`

`Frame64` não é um offset nem uma contagem de bytes. Ele representa frame de animação em unidade fixa:

```text
Frame real = Frame64 / 64.0
```

Exemplos:

| `Frame64` | Frame real |
|---:|---:|
| `0x0000` | `0.000` |
| `0x0030` | `0.750` |
| `0x0040` | `1.000` |
| `0x0100` | `4.000` |
| `0x01C0` | `7.000` |
| `0x03C0` | `15.000` |

O controlador de movimento usa a posição atual da sequência para escolher a entrada atual. Quando a posição está entre duas entradas, o valor de frame é interpolado entre `Entry[i].Frame64` e `Entry[i + 1].Frame64`.

---

## 8. Interpretação de `type` e `no`

O par `Type/No` funciona como metadado de evento da key atual.

```text
Type = classe/modo do evento
No   = número do evento
```

A sequência pode ser usada apenas como curva de frame, deixando `No = 0` em todas as entradas. Quando `No` é diferente de zero, o valor é repassado ao controlador específico do ator.

### Comportamento prático

| Condição | Resultado |
|---|---|
| `No == 0` | Não dispara evento de sequência nos controladores conhecidos. |
| `No != 0` | O controlador do ator interpreta `No` e `Type`. |
| `Type == 0` | Modo padrão/automático. |
| `Type != 0` | Pode forçar classe/categoria de evento, dependendo do consumidor. |

### Consumidores conhecidos

| Consumidor | Uso |
|---|---|
| `seqSeCtrl__7cPlayer` | Interpreta eventos de sequência do player. Usa `Type & 0x07` e `No`. |
| `seqSeCtrl__8cSubChar` | Interpreta eventos de sequência de subpersonagens. Usa `Type & 0x03` e `No`. |
| `CameraSequenceCtrl__FP11MOTION_INFO` | Controla avanço/estado de sequência de câmera. |
| `MotionSequenceCtrl__FP11MOTION_INFO` | Resolve frame ativo a partir da tabela `.SEQ`. |

---

## 9. Fluxo runtime

Fluxo simplificado:

```text
arquivo .SEQ
    ↓
EntryCount + tabela SEQ_ENTRY
    ↓
ponteiro da tabela entra no MOTION_INFO
    ↓
MotionSequenceCtrl
    ↓
seleciona Entry[i] pela posição da sequência
    ↓
interpola Frame64
    ↓
atualiza frame de animação e metadados Type/No
    ↓
seqSeCtrl do ator pode disparar som/efeito/evento
```

Campos runtime relevantes observados em `MOTION_INFO`:

| Offset runtime | Uso |
|---:|---|
| `+0xBC` | Flags de controle da sequência. |
| `+0xC4` | Ponteiro para tabela `SEQ_ENTRY`. |
| `+0xC8` | Palavra compactada da entrada atual. |
| `+0xCA` | `Frame64` atual/interpolado. |
| `+0xCC` | Palavra anterior/backup da entrada. |
| `+0xD4` | Posição atual da sequência como `float`. |
| `+0xD8` | Velocidade da sequência. |
| `+0xDC` | Quantidade/duração usada pelo controlador. |

---

## 10. Funções e símbolos relevantes

| Endereço | Símbolo | Papel |
|---:|---|---|
| `0x0025C438` | `MotionSequenceCtrl__FP11MOTION_INFO` | Controlador principal da tabela `.SEQ` em animações. |
| `0x001B72E0` | `CameraSequenceCtrl__FP11MOTION_INFO` | Controlador de sequência de câmera. |
| `0x00273B98` | `seqSeCtrl__7cPlayer` | Interpreta `Type/No` para eventos do player. |
| `0x00168528` | `seqSeCtrl__8cSubChar` | Interpreta `Type/No` para eventos de subpersonagens. |
| `0x0025C438 + 0x190` aprox. | uso de `Frame64` | Conversão/interpolação com escala `64.0`. |

Notas de engenharia reversa:

- `MotionSequenceCtrl` lê entradas de `0x04` bytes.
- O campo em `Entry + 0x02` é lido como `uint16`.
- O valor é comparado e interpolado em escala fixa de `64.0`.
- `Type` e `No` são consumidos como bytes pelos controladores de sequência de som/efeito.

---

## 11. Amostras analisadas

### Resumo

| Arquivo | Tamanho | Entradas | Fim dos dados | Padding | Último Frame64 | Último frame | Monótono | Eventos não-zero |
|---|---|---|---|---|---|---|---|---|
| `r104_32.SEQ` | `0x60` | 21 | `0x58` | `0x8` | `0x03C0` | 15.000 | sim | #9: type=0x00, no=0x03, frame64=0x01B0 (6.750), #10: type=0x00, no=0x2B, frame64=0x01E0 (7.500), #19: type=0x00, no=0x04, frame64=0x0390 (14.250), #20: type=0x00, no=0x2B, frame64=0x03C0 (15.000) |
| `r104_33.SEQ` | `0x60` | 18 | `0x4C` | `0x14` | `0x03B8` | 14.875 | sim | #8: type=0x00, no=0x03, frame64=0x01C0 (7.000), #9: type=0x00, no=0x2B, frame64=0x01F8 (7.875), #16: type=0x00, no=0x04, frame64=0x0380 (14.000), #17: type=0x00, no=0x2B, frame64=0x03B8 (14.875) |
| `r104_34.SEQ` | `0x60` | 16 | `0x44` | `0x1C` | `0x03C0` | 15.000 | sim | #7: type=0x00, no=0x03, frame64=0x01C0 (7.000), #8: type=0x00, no=0x2B, frame64=0x0200 (8.000), #14: type=0x00, no=0x04, frame64=0x0380 (14.000), #15: type=0x00, no=0x2B, frame64=0x03C0 (15.000) |
| `r104_35.SEQ` | `0x40` | 14 | `0x3C` | `0x4` | `0x03A8` | 14.625 | sim | #6: type=0x00, no=0x03, frame64=0x01B0 (6.750), #7: type=0x00, no=0x2B, frame64=0x01F8 (7.875), #12: type=0x00, no=0x04, frame64=0x0360 (13.500), #13: type=0x00, no=0x2B, frame64=0x03A8 (14.625) |
| `r104_36.SEQ` | `0x40` | 13 | `0x38` | `0x8` | `0x03C0` | 15.000 | sim | #5: type=0x00, no=0x03, frame64=0x0190 (6.250), #6: type=0x00, no=0x2B, frame64=0x01E0 (7.500), #11: type=0x00, no=0x04, frame64=0x0370 (13.750), #12: type=0x00, no=0x2B, frame64=0x03C0 (15.000) |
| `r104_37.SEQ` | `0x40` | 11 | `0x30` | `0x10` | `0x0370` | 13.750 | sim | #4: type=0x00, no=0x03, frame64=0x0160 (5.500), #5: type=0x00, no=0x2B, frame64=0x01B8 (6.875), #9: type=0x00, no=0x04, frame64=0x0318 (12.375), #10: type=0x00, no=0x2B, frame64=0x0370 (13.750) |
| `r104_38.SEQ` | `0x40` | 11 | `0x30` | `0x10` | `0x03C0` | 15.000 | sim | #4: type=0x00, no=0x03, frame64=0x0180 (6.000), #5: type=0x00, no=0x2B, frame64=0x01E0 (7.500), #9: type=0x00, no=0x04, frame64=0x0360 (13.500), #10: type=0x00, no=0x2B, frame64=0x03C0 (15.000) |
| `r104_39.SEQ` | `0x40` | 10 | `0x2C` | `0x14` | `0x03A8` | 14.625 | sim | #3: type=0x00, no=0x03, frame64=0x0138 (4.875), #4: type=0x00, no=0x2B, frame64=0x01A0 (6.500), #8: type=0x00, no=0x04, frame64=0x0340 (13.000), #9: type=0x00, no=0x2B, frame64=0x03A8 (14.625) |


### Eventos não-zero

| Arquivo | Índice | Type | No hex | No dec | Frame64 | Frame |
|---|---|---|---|---|---|---|
| `r104_32.SEQ` | 9 | `0x00` | `0x03` | 3 | `0x01B0` | 6.750 |
| `r104_32.SEQ` | 10 | `0x00` | `0x2B` | 43 | `0x01E0` | 7.500 |
| `r104_32.SEQ` | 19 | `0x00` | `0x04` | 4 | `0x0390` | 14.250 |
| `r104_32.SEQ` | 20 | `0x00` | `0x2B` | 43 | `0x03C0` | 15.000 |
| `r104_33.SEQ` | 8 | `0x00` | `0x03` | 3 | `0x01C0` | 7.000 |
| `r104_33.SEQ` | 9 | `0x00` | `0x2B` | 43 | `0x01F8` | 7.875 |
| `r104_33.SEQ` | 16 | `0x00` | `0x04` | 4 | `0x0380` | 14.000 |
| `r104_33.SEQ` | 17 | `0x00` | `0x2B` | 43 | `0x03B8` | 14.875 |
| `r104_34.SEQ` | 7 | `0x00` | `0x03` | 3 | `0x01C0` | 7.000 |
| `r104_34.SEQ` | 8 | `0x00` | `0x2B` | 43 | `0x0200` | 8.000 |
| `r104_34.SEQ` | 14 | `0x00` | `0x04` | 4 | `0x0380` | 14.000 |
| `r104_34.SEQ` | 15 | `0x00` | `0x2B` | 43 | `0x03C0` | 15.000 |
| `r104_35.SEQ` | 6 | `0x00` | `0x03` | 3 | `0x01B0` | 6.750 |
| `r104_35.SEQ` | 7 | `0x00` | `0x2B` | 43 | `0x01F8` | 7.875 |
| `r104_35.SEQ` | 12 | `0x00` | `0x04` | 4 | `0x0360` | 13.500 |
| `r104_35.SEQ` | 13 | `0x00` | `0x2B` | 43 | `0x03A8` | 14.625 |
| `r104_36.SEQ` | 5 | `0x00` | `0x03` | 3 | `0x0190` | 6.250 |
| `r104_36.SEQ` | 6 | `0x00` | `0x2B` | 43 | `0x01E0` | 7.500 |
| `r104_36.SEQ` | 11 | `0x00` | `0x04` | 4 | `0x0370` | 13.750 |
| `r104_36.SEQ` | 12 | `0x00` | `0x2B` | 43 | `0x03C0` | 15.000 |
| `r104_37.SEQ` | 4 | `0x00` | `0x03` | 3 | `0x0160` | 5.500 |
| `r104_37.SEQ` | 5 | `0x00` | `0x2B` | 43 | `0x01B8` | 6.875 |
| `r104_37.SEQ` | 9 | `0x00` | `0x04` | 4 | `0x0318` | 12.375 |
| `r104_37.SEQ` | 10 | `0x00` | `0x2B` | 43 | `0x0370` | 13.750 |
| `r104_38.SEQ` | 4 | `0x00` | `0x03` | 3 | `0x0180` | 6.000 |
| `r104_38.SEQ` | 5 | `0x00` | `0x2B` | 43 | `0x01E0` | 7.500 |
| `r104_38.SEQ` | 9 | `0x00` | `0x04` | 4 | `0x0360` | 13.500 |
| `r104_38.SEQ` | 10 | `0x00` | `0x2B` | 43 | `0x03C0` | 15.000 |
| `r104_39.SEQ` | 3 | `0x00` | `0x03` | 3 | `0x0138` | 4.875 |
| `r104_39.SEQ` | 4 | `0x00` | `0x2B` | 43 | `0x01A0` | 6.500 |
| `r104_39.SEQ` | 8 | `0x00` | `0x04` | 4 | `0x0340` | 13.000 |
| `r104_39.SEQ` | 9 | `0x00` | `0x2B` | 43 | `0x03A8` | 14.625 |


### Padrão observado

Os arquivos `r104_32.SEQ` até `r104_39.SEQ` seguem o mesmo desenho:

- `EntryCount` entre `10` e `21`.
- Primeira entrada com `Frame64 = 0`.
- Frames crescentes.
- Quatro eventos não-zero por arquivo.
- `Type = 0x00` em todos os eventos.
- `No = 0x03`, `0x04` e `0x2B` aparecem como eventos relevantes.
- Padding final com `0xCD` até alinhamento `0x20`.

---

## 12. Regras para editores e repackers

### Validação de leitura

1. Ler `EntryCount` em `0x00`.
2. Calcular `data_end = 0x04 + EntryCount * 0x04`.
3. Rejeitar se `data_end > file_size`.
4. Ler apenas `EntryCount` entradas.
5. Tratar bytes após `data_end` como padding.
6. Validar se `Frame64` é não decrescente.

### Repack seguro

- Recriar header com `EntryCount` real.
- Escrever cada entrada como `<BBH`.
- Alinhar o tamanho final para `0x20`.
- Usar `0xCD` como padding para manter o padrão observado.
- Preservar `Type` e `No` desconhecidos quando apenas ajustar frames.
- Não gerar entradas fora da ordem de frame, salvo para testes controlados.

### Edição recomendada

| Operação | Recomendação |
|---|---|
| Alterar velocidade/tempo | Ajustar `Frame64` mantendo ordem crescente. |
| Remover evento | Zerar `No`, preservando `Frame64`. |
| Adicionar evento | Definir `No != 0` em uma key existente ou inserir nova key ordenada. |
| Criar sequência nova | Começar com `Entry[0].Frame64 = 0` e terminar no frame final da animação. |
| Preservar compatibilidade | Usar `Type = 0` quando a semântica do ator não for conhecida. |

---

## 13. Representação JSON/YAML recomendada

### YAML

```yaml
format: RE4_PS2_SEQ
entry_count: 4
entries:
  - index: 0
    type: 0x00
    no: 0x00
    frame64: 0x0000
    frame: 0.0
  - index: 1
    type: 0x00
    no: 0x03
    frame64: 0x01C0
    frame: 7.0
  - index: 2
    type: 0x00
    no: 0x2B
    frame64: 0x0200
    frame: 8.0
  - index: 3
    type: 0x00
    no: 0x00
    frame64: 0x03C0
    frame: 15.0
padding:
  align: 0x20
  value: 0xCD
```

### JSON

```json
{
  "format": "RE4_PS2_SEQ",
  "entry_count": 4,
  "entries": [
    { "index": 0, "type": 0, "no": 0,  "frame64": 0,   "frame": 0.0 },
    { "index": 1, "type": 0, "no": 3,  "frame64": 448, "frame": 7.0 },
    { "index": 2, "type": 0, "no": 43, "frame64": 512, "frame": 8.0 },
    { "index": 3, "type": 0, "no": 0,  "frame64": 960, "frame": 15.0 }
  ],
  "padding": { "align": 32, "value": 205 }
}
```

---

## 14. Structs de referência

### C/C++

```c
#include <stdint.h>

#pragma pack(push, 1)

typedef struct RE4_SEQ_ENTRY {
    uint8_t  Type;       // +0x00
    uint8_t  No;         // +0x01
    uint16_t Frame64;    // +0x02, little-endian, frame * 64
} RE4_SEQ_ENTRY;

typedef struct RE4_SEQ_FILE {
    uint32_t      EntryCount; // +0x00
    RE4_SEQ_ENTRY Entries[];  // +0x04
} RE4_SEQ_FILE;

#pragma pack(pop)
```

### C#

```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4SeqEntry
{
    public byte Type;
    public byte No;
    public ushort Frame64;

    public float Frame => Frame64 / 64.0f;
}

public sealed class Re4SeqFile
{
    public uint EntryCount;
    public Re4SeqEntry[] Entries = Array.Empty<Re4SeqEntry>();
}
```

### Escrita C#

```csharp
static void WriteSeq(string path, Re4SeqEntry[] entries)
{
    using var fs = File.Create(path);
    using var bw = new BinaryWriter(fs);

    bw.Write((uint)entries.Length);

    foreach (var e in entries)
    {
        bw.Write(e.Type);
        bw.Write(e.No);
        bw.Write(e.Frame64);
    }

    while ((fs.Position & 0x1F) != 0)
        bw.Write((byte)0xCD);
}
```

---

## 15. Parser de referência em Python

```python
from __future__ import annotations

import struct
from dataclasses import dataclass
from pathlib import Path


@dataclass
class SeqEntry:
    index: int
    type: int
    no: int
    frame64: int

    @property
    def frame(self) -> float:
        return self.frame64 / 64.0


@dataclass
class SeqFile:
    entry_count: int
    entries: list[SeqEntry]
    padding: bytes


def parse_seq(path: str | Path) -> SeqFile:
    data = Path(path).read_bytes()

    if len(data) < 4:
        raise ValueError("arquivo menor que o header .SEQ")

    entry_count = struct.unpack_from("<I", data, 0)[0]
    data_end = 4 + entry_count * 4

    if data_end > len(data):
        raise ValueError(
            f"EntryCount inválido: {entry_count} entradas exigem "
            f"0x{data_end:X} bytes, mas o arquivo tem 0x{len(data):X}"
        )

    entries: list[SeqEntry] = []
    last_frame64 = -1

    for i in range(entry_count):
        off = 4 + i * 4
        typ, no, frame64 = struct.unpack_from("<BBH", data, off)

        if frame64 < last_frame64:
            raise ValueError(
                f"Frame64 fora de ordem no índice {i}: "
                f"0x{frame64:04X} < 0x{last_frame64:04X}"
            )

        last_frame64 = frame64
        entries.append(SeqEntry(i, typ, no, frame64))

    padding = data[data_end:]
    return SeqFile(entry_count, entries, padding)


def write_seq(path: str | Path, entries: list[SeqEntry], align: int = 0x20) -> None:
    out = bytearray()
    out += struct.pack("<I", len(entries))

    last_frame64 = -1
    for e in entries:
        if not (0 <= e.type <= 0xFF):
            raise ValueError(f"type inválido no índice {e.index}")
        if not (0 <= e.no <= 0xFF):
            raise ValueError(f"no inválido no índice {e.index}")
        if not (0 <= e.frame64 <= 0xFFFF):
            raise ValueError(f"frame64 inválido no índice {e.index}")
        if e.frame64 < last_frame64:
            raise ValueError(f"frame64 fora de ordem no índice {e.index}")
        last_frame64 = e.frame64

        out += struct.pack("<BBH", e.type, e.no, e.frame64)

    while len(out) % align:
        out.append(0xCD)

    Path(path).write_bytes(out)


if __name__ == "__main__":
    seq = parse_seq("r104_32.SEQ")
    print(f"entries: {seq.entry_count}")
    for e in seq.entries:
        marker = "event" if e.no else "key"
        print(
            f"{e.index:03d} {marker:5s} "
            f"type=0x{e.type:02X} no=0x{e.no:02X} "
            f"frame64=0x{e.frame64:04X} frame={e.frame:.3f}"
        )
```

---

## 16. Checklist de roundtrip

Para validar um editor/repacker:

1. Abrir o `.SEQ` original.
2. Exportar sem alterações.
3. Comparar:
   - mesmo `EntryCount`;
   - mesmas entradas em `<BBH`;
   - mesmo alinhamento final;
   - padding final `0xCD` ou preservado.
4. Editar um `Frame64`.
5. Confirmar que apenas os 2 bytes do `Frame64` alterado mudaram.
6. Zerar um `No` e confirmar que o evento correspondente deixa de ser disparado.
7. Inserir nova entrada e confirmar que `EntryCount`, tamanho e padding foram recalculados.

---

## 17. Resumo de offsets

### Arquivo

| Offset | Campo | Tipo | Descrição |
|---:|---|---|---|
| `0x00` | `EntryCount` | `uint32` | Quantidade de entradas. |
| `0x04` | `Entries[0]` | `SEQ_ENTRY` | Primeira entrada. |
| `0x04 + i * 4` | `Entries[i]` | `SEQ_ENTRY` | Entrada `i`. |
| `0x04 + EntryCount * 4` | `Padding` | `uint8[]` | Padding opcional. |

### Entrada

| Offset local | Campo | Tipo | Descrição |
|---:|---|---|---|
| `0x00` | `Type` | `uint8` | Tipo/classe do evento. |
| `0x01` | `No` | `uint8` | Número do evento. |
| `0x02` | `Frame64` | `uint16` | Frame fixo `frame * 64`. |
