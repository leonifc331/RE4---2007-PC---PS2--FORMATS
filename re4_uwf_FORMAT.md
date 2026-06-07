# Resident Evil 4 PS2 — Formato `.UWF`

Documentação técnica do formato `.UWF` usado por layouts de interface do Resident Evil 4 de PlayStation 2.

O formato é usado por arquivos `Core_*.UWF` e descreve unidades de HUD/menu em coordenadas 2D: quad, posição, cor, hierarquia, textura e controladores opcionais de animação. Esta versão foi validada com as amostras `Core_026.UWF` até `Core_037.UWF`.

---

## Resumo do formato

```text
0x0000  RE4_UWF_HEADER       0x10 bytes
0x0010  RE4_UWF_UNIT[]       UnitNum * 0xA0 bytes
...     RE4_UWF_CTRL blobs   payloads opcionais apontados pelas entradas
```

Características confirmadas:

- endian: little-endian;
- assinatura/versão textual: `"2.00"`;
- header fixo: `0x10` bytes;
- entrada fixa: `0xA0` bytes;
- ponteiros internos são offsets absolutos dentro do próprio `.UWF`;
- `0` em qualquer ponteiro significa canal ausente;
- payloads de controlador ficam depois da tabela de entradas;
- os payloads são alinhados normalmente em `0x10` bytes.

---

## Endianness

```text
uint8   = byte
uint16  = little-endian
uint32  = little-endian
float32 = IEEE754 little-endian
```

---

## Header — `RE4_UWF_HEADER`

Tamanho: `0x10` bytes.

```c
#pragma pack(push, 1)
typedef struct RE4_UWF_HEADER {
    char     Version[4];     // +0x00: "2.00"
    uint8_t  GroupNo;        // +0x04
    uint8_t  UnitNum;        // +0x05
    uint16_t Reserved06;     // +0x06
    uint8_t  Reserved08[8];  // +0x08
} RE4_UWF_HEADER;
#pragma pack(pop)
```

| Offset | Campo | Tipo | Tamanho | Descrição |
|---:|---|---|---:|---|
| `0x00` | `Version` | `char[4]` | `0x04` | Versão textual do arquivo. Nas amostras: `2.00`. |
| `0x04` | `GroupNo` | `uint8` | `0x01` | Grupo lógico do layout. Nas amostras enviadas ficou `0`. |
| `0x05` | `UnitNum` | `uint8` | `0x01` | Quantidade de entradas `RE4_UWF_UNIT`. |
| `0x06` | `Reserved06` | `uint16` | `0x02` | Reservado. Preservar. |
| `0x08` | `Reserved08` | `uint8[8]` | `0x08` | Reservado/padding. Nas amostras ficou zerado. |

### Valores aceitos do header

| Campo | Valores aceitos | Recomendação |
|---|---|---|
| `Version` | `"2.00"` confirmado | Para RE4 PS2, gerar `2.00`. |
| `GroupNo` | `0..255` | Preservar o original; para novo arquivo usar `0`. |
| `UnitNum` | `0..255` | Deve bater com a quantidade real de entradas. |
| `Reserved06` | `0..65535` | Preservar; para novo arquivo usar `0`. |
| `Reserved08` | bytes livres | Preservar; para novo arquivo usar `00 00 00 00 00 00 00 00`. |

---

## Entrada — `RE4_UWF_UNIT`

Tamanho: `0xA0` bytes.

A entrada representa uma unidade de interface. Ela pode ser um container, uma imagem/textura, um elemento de texto/ícone ou um elemento animado por controladores externos.

```c
#pragma pack(push, 1)

typedef struct RE4_UWF_VEC4 {
    float x;
    float y;
    float z;
    float w;
} RE4_UWF_VEC4;

typedef struct RE4_UWF_COLOR {
    uint8_t r;
    uint8_t g;
    uint8_t b;
    uint8_t a;
} RE4_UWF_COLOR;

typedef struct RE4_UWF_UNIT {
    uint32_t be_flag;       // +0x00

    uint8_t  markNo;        // +0x04
    uint8_t  unitNo;        // +0x05
    uint8_t  levelNo;       // +0x06
    uint8_t  parentNo;      // +0x07, 0xFF = sem pai

    uint8_t  rowNo;         // +0x08
    uint8_t  type;          // +0x09
    uint8_t  id;            // +0x0A
    uint8_t  texId;         // +0x0B, 0xFF = sem textura

    uint8_t  pos_flag;      // +0x0C
    uint8_t  loop_flag;     // +0x0D
    uint8_t  size_flag;     // +0x0E
    uint8_t  rot_flag;      // +0x0F

    RE4_UWF_VEC4 pos0;      // +0x10
    RE4_UWF_VEC4 ver[4];    // +0x20..+0x5F

    float    size_w;        // +0x60
    float    size_h;        // +0x64

    RE4_UWF_COLOR color0;   // +0x68
    RE4_UWF_COLOR color1;   // +0x6C

    RE4_UWF_VEC4 rot0;      // +0x70

    uint32_t mode_flags;    // +0x80
    uint32_t unk84;         // +0x84

    uint32_t pCtrl0;        // +0x88
    uint32_t pCtrl1;        // +0x8C
    uint32_t pCtrl2;        // +0x90
    uint32_t pCtrl3;        // +0x94
    uint32_t pCtrl4;        // +0x98
    uint32_t pCtrl5;        // +0x9C
} RE4_UWF_UNIT;

#pragma pack(pop)
```

### Tabela de campos

| Offset | Campo | Tipo | Tamanho | Descrição |
|---:|---|---|---:|---|
| `0x00` | `be_flag` | `uint32` | `0x04` | Flag principal/estado da unidade. Valores observados: `0x0C`, `0x0D`, `0x2C`, `0x2D`. |
| `0x04` | `markNo` | `uint8` | `0x01` | Marcador/índice auxiliar. `0xFF` aparece como marcador especial. |
| `0x05` | `unitNo` | `uint8` | `0x01` | Número da unidade dentro do layout. Usado por referências de hierarquia. |
| `0x06` | `levelNo` | `uint8` | `0x01` | Nível hierárquico/camada lógica. Observado: `0..3`. |
| `0x07` | `parentNo` | `uint8` | `0x01` | Pai lógico da unidade. `0xFF` indica unidade raiz/sem pai. |
| `0x08` | `rowNo` | `uint8` | `0x01` | Linha/slot lógico dentro de grids/listas. |
| `0x09` | `type` | `uint8` | `0x01` | Tipo operacional. Observado: `0` e `1`. |
| `0x0A` | `id` | `uint8` | `0x01` | ID auxiliar. Nas amostras analisadas ficou `0`. |
| `0x0B` | `texId` | `uint8` | `0x01` | ID/slot de textura. `0xFF` indica ausência de textura. |
| `0x0C` | `pos_flag` | `uint8` | `0x01` | Flag/canal de posição. |
| `0x0D` | `loop_flag` | `uint8` | `0x01` | Flag de loop/modo de animação. |
| `0x0E` | `size_flag` | `uint8` | `0x01` | Flag de tamanho/escala; valores com `0x80` aparecem em elementos animados. |
| `0x0F` | `rot_flag` | `uint8` | `0x01` | Flag de rotação/canal extra. |
| `0x10` | `pos0` | `vec4` | `0x10` | Posição base da unidade. `w` normalmente `1.0`. |
| `0x20` | `ver[4]` | `vec4[4]` | `0x40` | Quatro vértices do quad em espaço 2D. `z` normalmente `0.0`, `w` normalmente `1.0`. |
| `0x60` | `size_w` | `float` | `0x04` | Largura base. |
| `0x64` | `size_h` | `float` | `0x04` | Altura base. |
| `0x68` | `color0` | `RGBA8` | `0x04` | Cor principal. Ordem confirmada: `R,G,B,A`. |
| `0x6C` | `color1` | `RGBA8` | `0x04` | Cor secundária/target. Pode ser zero quando não usada. |
| `0x70` | `rot0` | `vec4` | `0x10` | Rotação/base transform. `w` normalmente `1.0`. |
| `0x80` | `mode_flags` | `uint32` | `0x04` | Flags/parâmetros de modo. Não é ponteiro. |
| `0x84` | `unk84` | `uint32` | `0x04` | Campo auxiliar. Quase sempre `0`; observado `1` em uma entrada. |
| `0x88` | `pCtrl0` | `uint32` | `0x04` | Offset absoluto para controlador opcional. |
| `0x8C` | `pCtrl1` | `uint32` | `0x04` | Offset absoluto para controlador opcional. |
| `0x90` | `pCtrl2` | `uint32` | `0x04` | Offset absoluto para controlador opcional. |
| `0x94` | `pCtrl3` | `uint32` | `0x04` | Offset absoluto para controlador opcional. |
| `0x98` | `pCtrl4` | `uint32` | `0x04` | Offset absoluto para controlador opcional. É o slot mais usado nas amostras. |
| `0x9C` | `pCtrl5` | `uint32` | `0x04` | Offset absoluto para controlador opcional raro. |

---

## Coordenadas e quad

Os arquivos analisados usam espaço 2D de interface, com valores típicos como `-320..320` no eixo X e tamanhos como `640.0` para largura total. O padrão de vértices é um quad de quatro `vec4`:

```text
ver[0] = canto superior/esquerdo ou primeiro vértice lógico
ver[1] = canto superior/direito
ver[2] = canto inferior/direito
ver[3] = canto inferior/esquerdo
```

Campos usuais:

```text
z = 0.0
w = 1.0
```

Em vários elementos invisíveis, placeholders ou containers, os vértices podem estar zerados e `size_w/size_h` pode usar valores sentinela, como `-20000.0`.

---

## Cores

As cores são bytes em ordem `RGBA`.

```c
typedef struct RE4_UWF_COLOR {
    uint8_t r;
    uint8_t g;
    uint8_t b;
    uint8_t a;
} RE4_UWF_COLOR;
```

Exemplos observados:

| Bytes | Cor |
|---|---|
| `FF FF FF FF` | branco opaco |
| `00 00 00 FF` | preto opaco |
| `C3 C3 C3 FF` | cinza opaco |
| `00 00 00 00` | cor secundária vazia/não usada |
| `7D 7D 7D FF` | cinza médio opaco |

---

## Controladores/payloads

Os campos `pCtrl0..pCtrl5` apontam para payloads variáveis localizados depois da tabela `RE4_UWF_UNIT[]`.

| Slot | Offset na entrada | Qtde. referências nas amostras | Interpretação operacional |
|---|---:|---:|---|
| `pCtrl0` | `+0x88` | 53 | controlador/path de posição ou deslocamento |
| `pCtrl1` | `+0x8C` | 53 | controlador auxiliar de posição/spline |
| `pCtrl2` | `+0x90` | 78 | curva de tamanho/escala ou valor numérico |
| `pCtrl3` | `+0x94` | 74 | curva de cor/alpha/valor auxiliar |
| `pCtrl4` | `+0x98` | 105 | curva mais comum; usada para alpha/UV/estado em várias telas |
| `pCtrl5` | `+0x9C` | 3 | curva rara; usada como canal extra/rotação em poucas amostras |


### Regras dos ponteiros

| Regra | Descrição |
|---|---|
| `0x00000000` | Canal ausente. |
| `offset >= table_end` | Ponteiro válido para payload, desde que também seja `< file_size`. |
| `offset < table_end` | Inválido para payload em arquivo normal. Deve ser tratado como erro ou valor reservado. |
| alinhamento | As amostras usam payloads em offsets alinhados por `0x10`. |

### Estrutura operacional de controlador

Os controladores têm header de `0x10` bytes seguido de registros de `0x10` bytes.

```c
#pragma pack(push, 1)

typedef struct RE4_UWF_CTRL_HEADER {
    uint32_t count_or_flags; // +0x00
    uint32_t param0;         // +0x04
    uint32_t param1;         // +0x08
    uint32_t param2;         // +0x0C
} RE4_UWF_CTRL_HEADER;

typedef struct RE4_UWF_CTRL_KEY {
    float frame_or_t;        // +0x00
    float value0;            // +0x04
    float value1;            // +0x08
    float value2;            // +0x0C
} RE4_UWF_CTRL_KEY;

#pragma pack(pop)
```

Para `pCtrl2`, `pCtrl3`, `pCtrl4` e `pCtrl5`, o primeiro `uint32` normalmente é a quantidade de keys. O tamanho lógico comum é:

```text
size = 0x10 + key_count * 0x10
```

Para `pCtrl0` e `pCtrl1`, o header pode carregar flags compactadas nos bytes do primeiro e do segundo `uint32`. Por isso, um editor seguro deve preservar o payload completo ou calcular o tamanho pelo próximo offset/pelo fim da área de payload.

### Recomendação para editor

Para roundtrip byte-safe:

1. ler todos os offsets não nulos de `pCtrl0..pCtrl5`;
2. ordenar os offsets;
3. considerar o tamanho de cada payload como `next_offset - current_offset`;
4. para o último payload, usar `file_size - current_offset`;
5. preservar payloads desconhecidos sem reserializar;
6. só reserializar payloads editados de canais conhecidos.

---

## Valores aceitos e observados

| Campo | Intervalo observado | Qtd. valores | Valores observados nas amostras |
|---|---:|---:|---|
| `be_flag` | `12`..`45` | 4 | `0xC`, `0xD`, `0x2C`, `0x2D` |
| `levelNo` | `0`..`3` | 4 | `0`, `1`, `2`, `3` |
| `type` | `0`..`1` | 2 | `0`, `1` |
| `id` | `0`..`0` | 1 | `0` |
| `parentNo` | `0`..`255` | 48 | `0`, `1`, `2`, `3`, `4`, `5`, `6`, `7`, `9`, `11`, `12`, `13`, `14`, `16`, `21`, `22`, `23`, `24`, `25`, `27`, `28`, `29`, `32`, `34`, `40`, `42`, `44`, `46`, `48`, `51`, `52`, `54`, `55`, `60`, `61`, `68`, `74`, `79`, `83`, `88`, `93`, `102`, `110`, `112`, `119`, `120`, `126`, `255` |
| `texId` | `3`..`255` | 66 | `3`, `7`, `10`, `11`, `13`, `14`, `15`, `16`, `18`, `20`, `21`, `23`, `25`, `29`, `30`, `32`, `33`, `35`, `37`, `39`, `40`, `41`, `42`, `43`, `47`, `48`, `49`, `50`, `52`, `53`, `57`, `58`, `62`, `64`, `73`, `74`, `80`, `81`, `82`, `85`, `97`, `111`, `112`, `113`, `114`, `119`, `131`, `132`, `133`, `134`, `135`, `136`, `137`, `138`, `144`, `145`, `146`, `147`, `148`, `186`, `187`, `228`, `234`, `240`, `242`, `255` |
| `pos_flag` | `0`..`4` | 5 | `0`, `1`, `2`, `3`, `4` |
| `loop_flag` | `0`..`14` | 11 | `0`, `1`, `3`, `4`, `5`, `6`, `7`, `8`, `9`, `10`, `14` |
| `size_flag` | `0`..`160` | 6 | `0`, `16`, `32`, `128`, `144`, `160` |
| `rot_flag` | `0`..`2` | 3 | `0`, `1`, `2` |
| `mode_flags` | `0`..`2432696321` | 8 | `0x0`, `0x1`, `0x2`, `0x30000`, `0x12000000`, `0x17000000`, `0x17000001`, `0x91000001` |
| `unk84` | `0`..`1` | 2 | `0`, `1` |


### Interpretação prática dos principais campos

| Campo | Valor | Uso prático |
|---|---:|---|
| `parentNo` | `0xFF` | unidade raiz/sem pai |
| `parentNo` | `0..254` | referência ao `unitNo` do pai lógico |
| `texId` | `0xFF` | sem textura; normalmente container, grupo ou elemento lógico |
| `texId` | `0..254` | índice/slot de textura usado pelo sistema de UI |
| `type` | `0` | unidade comum/texturizada ou elemento de layout |
| `type` | `1` | unidade de grupo/container ou elemento especial sem textura direta |
| `size_flag` | `0x80` | elemento com canal especial/animado observado em várias entradas |
| `mode_flags` | `0` | sem modo extra |
| `mode_flags` | `1` | modo extra ativo; comum em filhos animados/texturizados |

`be_flag` deve ser tratado como bitfield. Nas amostras, os valores aparecem sempre na família `0x0C/0x0D/0x2C/0x2D`; o bit `0x20` diferencia um subtipo de unidade, e o bit `0x01` diferencia duas variantes próximas. Para repack seguro, preservar o valor original.

---

## Tabela das amostras analisadas

| Arquivo | Tamanho | Version | GroupNo | UnitNum | Fim da tabela | Payload extra | Ptr +88 | Ptr +8C | Ptr +90 | Ptr +94 | Ptr +98 | Ptr +9C |
|---|---:|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `Core_026.UWF` | `0x800` | `2.00` | `0` | `12` | `0x790` | `0x70` | 0 | 0 | 0 | 0 | 2 | 0 |
| `Core_027.UWF` | `0x27C0` | `2.00` | `0` | `54` | `0x21D0` | `0x5F0` | 4 | 4 | 1 | 3 | 15 | 1 |
| `Core_028.UWF` | `0x7C80` | `2.00` | `0` | `142` | `0x58D0` | `0x23B0` | 14 | 14 | 43 | 56 | 25 | 2 |
| `Core_029.UWF` | `0x8A0` | `2.00` | `0` | `13` | `0x830` | `0x70` | 0 | 0 | 0 | 0 | 2 | 0 |
| `Core_030.UWF` | `0x160` | `2.00` | `0` | `2` | `0x150` | `0x10` | 0 | 0 | 0 | 0 | 0 | 0 |
| `Core_031.UWF` | `0x6040` | `2.00` | `0` | `86` | `0x35D0` | `0x2A70` | 31 | 31 | 30 | 10 | 53 | 0 |
| `Core_033.UWF` | `0x2C0` | `2.00` | `0` | `4` | `0x290` | `0x30` | 0 | 0 | 0 | 0 | 1 | 0 |
| `Core_034.UWF` | `0x7E0` | `2.00` | `0` | `12` | `0x790` | `0x50` | 0 | 0 | 0 | 0 | 1 | 0 |
| `Core_035.UWF` | `0x9E0` | `2.00` | `0` | `9` | `0x5B0` | `0x430` | 4 | 4 | 4 | 5 | 2 | 0 |
| `Core_036.UWF` | `0xEE0` | `2.00` | `0` | `22` | `0xDD0` | `0x110` | 0 | 0 | 0 | 0 | 4 | 0 |
| `Core_037.UWF` | `0x5C0` | `2.00` | `0` | `9` | `0x5B0` | `0x10` | 0 | 0 | 0 | 0 | 0 | 0 |


---

## Layout visual simplificado

```text
RE4_UWF_HEADER
├── Version = "2.00"
├── UnitNum
└── RE4_UWF_UNIT[UnitNum]
    ├── hierarquia: markNo/unitNo/levelNo/parentNo
    ├── identificação: rowNo/type/id/texId
    ├── flags: pos_flag/loop_flag/size_flag/rot_flag/be_flag/mode_flags
    ├── transform 2D: pos0/ver[4]/size/rot0
    ├── cores: color0/color1
    └── controladores opcionais: pCtrl0..pCtrl5
```

---

## Struct C#

```csharp
using System;
using System.IO;
using System.Numerics;
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct UwfHeader
{
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 4)]
    public byte[] Version;      // "2.00"
    public byte GroupNo;
    public byte UnitNum;
    public ushort Reserved06;
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 8)]
    public byte[] Reserved08;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct UwfColor
{
    public byte R, G, B, A;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct UwfVec4
{
    public float X, Y, Z, W;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct UwfUnit
{
    public uint BeFlag;

    public byte MarkNo;
    public byte UnitNo;
    public byte LevelNo;
    public byte ParentNo;

    public byte RowNo;
    public byte Type;
    public byte Id;
    public byte TexId;

    public byte PosFlag;
    public byte LoopFlag;
    public byte SizeFlag;
    public byte RotFlag;

    public UwfVec4 Pos0;

    public UwfVec4 Ver0;
    public UwfVec4 Ver1;
    public UwfVec4 Ver2;
    public UwfVec4 Ver3;

    public float SizeW;
    public float SizeH;

    public UwfColor Color0;
    public UwfColor Color1;

    public UwfVec4 Rot0;

    public uint ModeFlags;
    public uint Unk84;

    public uint PCtrl0;
    public uint PCtrl1;
    public uint PCtrl2;
    public uint PCtrl3;
    public uint PCtrl4;
    public uint PCtrl5;
}
```

---

## Parser Python de referência

```python
import struct
from dataclasses import dataclass
from pathlib import Path

UNIT_SIZE = 0xA0
HEADER_SIZE = 0x10

@dataclass
class UwfHeader:
    version: str
    group_no: int
    unit_count: int
    reserved06: int
    reserved08: bytes

@dataclass
class UwfUnit:
    index: int
    be_flag: int
    mark_no: int
    unit_no: int
    level_no: int
    parent_no: int
    row_no: int
    type: int
    id: int
    tex_id: int
    pos_flag: int
    loop_flag: int
    size_flag: int
    rot_flag: int
    pos0: tuple
    vertices: list
    size_w: float
    size_h: float
    color0: tuple
    color1: tuple
    rot0: tuple
    mode_flags: int
    unk84: int
    ctrls: list


def read_vec4(data, off):
    return struct.unpack_from('<4f', data, off)


def parse_uwf(path):
    data = Path(path).read_bytes()
    if len(data) < HEADER_SIZE:
        raise ValueError('arquivo menor que o header UWF')

    version_raw = data[0:4]
    version = version_raw.decode('ascii', errors='replace')
    group_no = data[0x04]
    unit_count = data[0x05]
    reserved06 = struct.unpack_from('<H', data, 0x06)[0]
    reserved08 = data[0x08:0x10]

    table_end = HEADER_SIZE + unit_count * UNIT_SIZE
    if table_end > len(data):
        raise ValueError('UnitNum excede o tamanho do arquivo')

    header = UwfHeader(version, group_no, unit_count, reserved06, reserved08)
    units = []

    for i in range(unit_count):
        off = HEADER_SIZE + i * UNIT_SIZE
        be_flag = struct.unpack_from('<I', data, off + 0x00)[0]

        pos0 = read_vec4(data, off + 0x10)
        vertices = [read_vec4(data, off + 0x20 + j * 0x10) for j in range(4)]
        size_w, size_h = struct.unpack_from('<2f', data, off + 0x60)
        color0 = tuple(data[off + 0x68:off + 0x6C])
        color1 = tuple(data[off + 0x6C:off + 0x70])
        rot0 = read_vec4(data, off + 0x70)
        mode_flags = struct.unpack_from('<I', data, off + 0x80)[0]
        unk84 = struct.unpack_from('<I', data, off + 0x84)[0]
        ctrls = [struct.unpack_from('<I', data, off + 0x88 + c * 4)[0] for c in range(6)]

        units.append(UwfUnit(
            index=i,
            be_flag=be_flag,
            mark_no=data[off + 0x04],
            unit_no=data[off + 0x05],
            level_no=data[off + 0x06],
            parent_no=data[off + 0x07],
            row_no=data[off + 0x08],
            type=data[off + 0x09],
            id=data[off + 0x0A],
            tex_id=data[off + 0x0B],
            pos_flag=data[off + 0x0C],
            loop_flag=data[off + 0x0D],
            size_flag=data[off + 0x0E],
            rot_flag=data[off + 0x0F],
            pos0=pos0,
            vertices=vertices,
            size_w=size_w,
            size_h=size_h,
            color0=color0,
            color1=color1,
            rot0=rot0,
            mode_flags=mode_flags,
            unk84=unk84,
            ctrls=ctrls,
        ))

    return data, header, units


def validate_uwf(path):
    data, header, units = parse_uwf(path)
    table_end = HEADER_SIZE + header.unit_count * UNIT_SIZE
    errors = []

    if header.version != '2.00':
        errors.append(f'Version inesperada: {header.version!r}')

    for u in units:
        for slot, ptr in enumerate(u.ctrls):
            if ptr == 0:
                continue
            if not (table_end <= ptr < len(data)):
                errors.append(f'unit {u.index} pCtrl{slot} fora do payload: 0x{ptr:X}')
            if ptr % 0x10 != 0:
                errors.append(f'unit {u.index} pCtrl{slot} não alinhado em 0x10: 0x{ptr:X}')

    return errors


if __name__ == '__main__':
    import sys
    for filename in sys.argv[1:]:
        data, header, units = parse_uwf(filename)
        print(filename)
        print('  version:', header.version)
        print('  group:', header.group_no)
        print('  units:', header.unit_count)
        print('  size:', len(data))
        for err in validate_uwf(filename):
            print('  ERROR:', err)
```

---

## Repacker seguro

Para repack completo:

1. escrever o header `0x10`;
2. escrever `UnitNum` entradas de `0xA0`;
3. iniciar payload em `0x10 + UnitNum * 0xA0`;
4. alinhar cada payload em `0x10`;
5. atualizar `pCtrl0..pCtrl5` com offsets absolutos;
6. manter `0` para canais ausentes;
7. preservar `be_flag`, `mode_flags`, `unk84` e campos desconhecidos quando a ferramenta não tiver semântica do canal;
8. validar que todo ponteiro aponta para fora da tabela e dentro do arquivo.

### Regras de edição recomendadas

| Operação | Seguro | Observação |
|---|---|---|
| mover UI/quad | sim | editar `pos0` e/ou `ver[4]` |
| alterar tamanho estático | sim | editar `size_w`, `size_h` e, se necessário, recalcular `ver[4]` |
| trocar textura | sim | editar `texId`, mantendo compatibilidade com o banco de textura carregado |
| trocar cor | sim | editar `color0`/`color1` |
| alterar hierarquia | sim, com cuidado | `parentNo` deve apontar para um `unitNo` existente ou `0xFF` |
| adicionar unidade | sim | incrementar `UnitNum` e repackar tabela/payloads |
| remover unidade | sim, com cuidado | atualizar referências de `parentNo` e payloads associados |
| editar curvas | parcial | canais `pCtrl2..pCtrl5` são mais simples; `pCtrl0/pCtrl1` devem ser preservados até mapear todos os flags |

---

## YAML recomendado para ferramenta

```yaml
version: "2.00"
group_no: 0
units:
  - index: 0
    be_flag: 0x0000000D
    mark_no: 0
    unit_no: 0
    level_no: 0
    parent_no: 255
    row_no: 2
    type: 1
    id: 0
    tex_id: 255
    flags:
      pos: 0
      loop: 0
      size: 0
      rot: 1
    pos0: [217.0, -130.0, 0.0, 1.0]
    vertices:
      - [165.0, -73.75, 0.0, 1.0]
      - [269.0, -73.75, 0.0, 1.0]
      - [269.0, -186.25, 0.0, 1.0]
      - [165.0, -186.25, 0.0, 1.0]
    size: [104.0, 112.5]
    color0: [255, 255, 255, 255]
    color1: [0, 0, 0, 0]
    rot0: [0.0, 0.0, 0.0, 1.0]
    mode_flags: 0
    unk84: 0
    controllers:
      pCtrl0: null
      pCtrl1: null
      pCtrl2: null
      pCtrl3: null
      pCtrl4: null
      pCtrl5: null
```

---

## Checklist de validação

- `Version == "2.00"`.
- `file_size >= 0x10 + UnitNum * 0xA0`.
- Cada `pCtrl` é `0` ou `table_end <= pCtrl < file_size`.
- Cada `pCtrl` não nulo deve estar alinhado em `0x10`.
- `parentNo == 0xFF` ou aponta para um `unitNo` existente.
- `texId == 0xFF` ou é compatível com o banco de textura do contexto.
- `vec4.w` de `pos0`, `ver[]` e `rot0` deve ser preservado; normalmente `1.0`.
- `color.a` deve ficar em `0..255`.
- Em roundtrip sem edição, o arquivo exportado deve ficar byte idêntico.
