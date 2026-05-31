# Resident Evil 4 PS2 — Formato `.EMI`

Documentação técnica do formato `.EMI` usado pelo sistema `EMINFO` de **Resident Evil 4** de PlayStation 2.

O `.EMI` armazena uma lista de marcadores posicionais associados ao sistema de inimigos. Cada marcador possui posição, direção e quatro campos de trabalho de 8 bits. No executável debug, o formato aparece como `EMINFO_DATA` contendo entradas `EMINFO_WK`.

## Sumário

- [1. Características gerais](#1-características-gerais)
- [2. Estrutura global](#2-estrutura-global)
- [3. Header `EMINFO_DATA`](#3-header-eminfo_data)
- [4. Entrada `EMINFO_WK`](#4-entrada-eminfo_wk)
- [5. Campos da entrada](#5-campos-da-entrada)
- [6. Valores aceitos por campo](#6-valores-aceitos-por-campo)
- [7. `WorkType`, `Work0`, `Work1` e `Work2`](#7-worktype-work0-work1-e-work2)
- [8. Coordenadas, direção e conversão para editores](#8-coordenadas-direção-e-conversão-para-editores)
- [9. Relação com `.ESL`](#9-relação-com-esl)
- [10. Funções e símbolos relevantes](#10-funções-e-símbolos-relevantes)
- [11. Amostra analisada: `r104_27.EMI`](#11-amostra-analisada-r104_27emi)
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
| Header em arquivo | `0x10` bytes |
| Estrutura principal | `EMINFO_DATA` |
| Estrutura de entrada | `EMINFO_WK` |
| Tamanho da entrada | `0x40` bytes |
| Máximo runtime | `256` entradas |
| Tamanho runtime completo | `0x4010` bytes |
| Tamanho compacto | `0x10 + Num * 0x40` |
| Sistema relacionado | `EMINFO` / `ToolEmInfo` / debug enemy info |
| Plataforma analisada | PlayStation 2 |

O formato não possui magic ASCII, versão ou tabela de offsets. A identificação é feita pelo contexto do arquivo e pela estrutura interna:

```text
uint32 Num
uint32 Id
uint32 Reserved0
uint32 Reserved1
EMINFO_WK Data[Num]
```

A estrutura debug do executável define `EMINFO_DATA` com `256` slots fixos em runtime:

```text
EMINFO_DATA = 0x4010 bytes
0x10 header + 256 * 0x40
```

Nos arquivos de sala, o formato pode aparecer em forma compacta, contendo apenas `Num` entradas, seguido ou não de padding de alinhamento.

---

## 2. Estrutura global

```text
Arquivo .EMI
├─ EMINFO_DATA header        0x10 bytes
│  ├─ Num                    uint32
│  ├─ Id                     uint32
│  ├─ Reserved0              uint32
│  └─ Reserved1              uint32
│
├─ EMINFO_WK[Num]
│  ├─ EMINFO_WK[0]           0x40 bytes
│  ├─ EMINFO_WK[1]           0x40 bytes
│  ├─ EMINFO_WK[2]           0x40 bytes
│  └─ ...
│
└─ Padding opcional          alinhamento externo
```

Validação básica:

```text
min_size = 0x10
entry_size = 0x40
max_entries = 256
compact_size = 0x10 + Num * 0x40
runtime_full_size = 0x4010
```

Um arquivo deve ser tratado como inválido quando:

```text
file_size < 0x10
Num > 256
file_size < 0x10 + Num * 0x40
```

Quando houver bytes após o final lógico, eles devem ser tratados como padding e preservados se o objetivo for roundtrip byte-safe.

---

## 3. Header `EMINFO_DATA`

O header possui `0x10` bytes.

| Offset | Tipo | Nome | Descrição |
|---:|---|---|---|
| `0x00` | `uint32` | `Num` | Quantidade de entradas válidas. |
| `0x04` | `uint32` | `Id` | Identificador do bloco. Normalmente `0`. |
| `0x08` | `uint32` | `Reserved0` | Campo reservado/padding. Normalmente `0`. |
| `0x0C` | `uint32` | `Reserved1` | Campo reservado/padding. Normalmente `0`. |

Definição debug equivalente:

```text
EMINFO_DATA:
    Num  uint32
    Id   uint32
    Data EMINFO_WK[256] em +0x10
```

A definição debug indica `Data` começando no bit offset `128`, ou seja, no offset `0x10`.

### `Num`

`Num` controla quantas entradas o jogo e as ferramentas de debug devem considerar.

| Valor | Interpretação |
|---:|---|
| `0` | Lista vazia. |
| `1..255` | Lista compacta comum. |
| `256` | Limite máximo runtime. |
| `>256` | Inválido para repack seguro. |

### `Id`

Campo de identificação do bloco. Na amostra analisada, o valor é `0`.

Recomendação:

```text
novo arquivo: Id = 0
edição de arquivo existente: preservar Id original
```

### `Reserved0` e `Reserved1`

Campos reservados entre o header lógico e a primeira entrada.

Recomendação:

```text
novo arquivo: Reserved0 = 0, Reserved1 = 0
edição de arquivo existente: preservar se não forem zero
```

---

## 4. Entrada `EMINFO_WK`

Cada entrada possui exatamente `0x40` bytes.

Estrutura:

```text
EMINFO_WK
├─ Pos         tagVec / vec4f    +0x00
├─ Dir         float             +0x10
├─ WorkType    uint8             +0x14
├─ Work0       uint8             +0x15
├─ Work1       uint8             +0x16
├─ Work2       uint8             +0x17
└─ Dummy       uint8[0x28]       +0x18
```

Tabela completa:

| Offset | Tamanho | Tipo | Nome | Descrição |
|---:|---:|---|---|---|
| `0x00` | `0x04` | `float` | `Pos.x` | Coordenada X. |
| `0x04` | `0x04` | `float` | `Pos.y` | Coordenada Y. |
| `0x08` | `0x04` | `float` | `Pos.z` | Coordenada Z. |
| `0x0C` | `0x04` | `float` | `Pos.w` | Componente W do `tagVec`; normalmente `1.0`. |
| `0x10` | `0x04` | `float` | `Dir` | Direção/ângulo horizontal em radianos. |
| `0x14` | `0x01` | `uint8` | `WorkType` | Tipo ou modo auxiliar da entrada. |
| `0x15` | `0x01` | `uint8` | `Work0` | Parâmetro auxiliar 0. |
| `0x16` | `0x01` | `uint8` | `Work1` | Parâmetro auxiliar 1. |
| `0x17` | `0x01` | `uint8` | `Work2` | Parâmetro auxiliar 2. |
| `0x18` | `0x28` | `uint8[40]` | `Dummy` | Bloco reservado/auxiliar. |

Definição debug original:

```text
EMINFO_WK:
    Pos       tagVec
    Dir       float
    WorkType  uint8
    Work0     uint8
    Work1     uint8
    Work2     uint8
    Dummy     char[40]
```

---

## 5. Campos da entrada

### `Pos`

`Pos` é um `tagVec` de 16 bytes:

```c
typedef struct tagVec {
    float x;
    float y;
    float z;
    float w;
} tagVec;
```

O componente `w` deve ser preservado. Nos arquivos analisados, ele aparece como `1.0`.

### `Dir`

`Dir` é um `float32` usado como direção. Os valores observados estão em radianos.

Exemplos observados:

| Hex | Float aproximado | Interpretação prática |
|---:|---:|---|
| `0x3F490FC3` | `0.785397` | Aproximadamente `45°`. |
| `0xC01D1465` | `-2.454370` | Aproximadamente `-140.62°`. |
| `0xBF6231ED` | `-0.883574` | Aproximadamente `-50.62°`. |
| `0x400A3ADC` | `2.159842` | Aproximadamente `123.75°`. |
| `0x40363655` | `2.847066` | Aproximadamente `163.13°`. |

### `WorkType`

Campo de classificação de 8 bits. O executável debug usa esse campo em `Draw_eminfo`.

Uso confirmado:

| Valor | Uso confirmado |
|---:|---|
| `0x02` | Entrada participa de linhas de ligação no debug draw de `EMINFO`. |
| `0x0F` | Valor observado na amostra `r104_27.EMI`. |

Outros valores cabem no tipo `uint8`, mas a semântica precisa ser validada por sala/função antes de uso.

### `Work0`

Parâmetro de 8 bits. Em `Draw_eminfo`, quando `WorkType == 0x02`, `Work0` influencia a cor da linha de ligação.

Valores observados na amostra:

```text
0x00
0x01
0x02
```

### `Work1`

Parâmetro de 8 bits. Na amostra analisada, aparece como `0x00` e `0x04`.

### `Work2`

Parâmetro de 8 bits. Na amostra analisada, aparece como `0x00` e `0x01`.

### `Dummy`

Bloco reservado de `0x28` bytes.

Na amostra analisada, todos os bytes de `Dummy` estão zerados. Para editores, este bloco deve ser preservado integralmente quando houver dados não nulos.

---

## 6. Valores aceitos por campo

| Campo | Tipo | Faixa binária | Valores seguros para edição | Observações |
|---|---|---:|---|---|
| `Num` | `uint32` | `0..0xFFFFFFFF` | `0..256` | Acima de `256` excede a área runtime. |
| `Id` | `uint32` | `0..0xFFFFFFFF` | preservar / `0` | Valor observado: `0`. |
| `Reserved0` | `uint32` | `0..0xFFFFFFFF` | preservar / `0` | Normalmente zero. |
| `Reserved1` | `uint32` | `0..0xFFFFFFFF` | preservar / `0` | Normalmente zero. |
| `Pos.x` | `float32` | float finito | coordenada válida do mapa | Evitar `NaN`/`Inf`. |
| `Pos.y` | `float32` | float finito | coordenada válida do mapa | Evitar `NaN`/`Inf`. |
| `Pos.z` | `float32` | float finito | coordenada válida do mapa | Evitar `NaN`/`Inf`. |
| `Pos.w` | `float32` | float finito | `1.0` | Preservar se o original usar outro valor. |
| `Dir` | `float32` | float finito | `-π..π` ou normalizado | O jogo aceita float, mas valores absurdos dificultam debug. |
| `WorkType` | `uint8` | `0..255` | preservar / `0x02` / `0x0F` | `0x02` tem uso confirmado em debug draw. |
| `Work0` | `uint8` | `0..255` | preservar / `0..2` | Afeta linha de debug quando `WorkType == 0x02`. |
| `Work1` | `uint8` | `0..255` | preservar / valores observados | Observado: `0x00`, `0x04`. |
| `Work2` | `uint8` | `0..255` | preservar / valores observados | Observado: `0x00`, `0x01`. |
| `Dummy` | `uint8[40]` | qualquer byte | preservar | Zerar apenas ao criar entrada nova. |

### Validação recomendada

```text
0 <= Num <= 256
file_size >= 0x10 + Num * 0x40
Pos.x/y/z/w finitos
Dir finito
Pos.w preferencialmente 1.0
Dummy preservado se não for zero
```

---

## 7. `WorkType`, `Work0`, `Work1` e `Work2`

Os campos `WorkType`, `Work0`, `Work1` e `Work2` formam um bloco auxiliar de quatro bytes:

```text
+0x14 WorkType
+0x15 Work0
+0x16 Work1
+0x17 Work2
```

### Tabela de valores conhecidos de `WorkType`

| Valor | Nome sugerido | Comportamento conhecido |
|---:|---|---|
| `0x00` | `NONE` | Entrada sem tipo especial confirmado. |
| `0x02` | `DEBUG_LINK` | O debug draw conecta esta entrada com outras entradas `WorkType == 0x02`. |
| `0x0F` | `ROOM_MARKER` | Valor observado no arquivo `r104_27.EMI`; manter em roundtrip. |
| `0x01`, `0x03..0x0E`, `0x10..0xFF` | `UNKNOWN` | Valor binariamente aceito; sem semântica confirmada. |

### `Work0` quando `WorkType == 0x02`

No debug draw, entradas com `WorkType == 0x02` são conectadas por linhas. O campo `Work0` participa da escolha de cor/estado visual.

| `Work0` | Comportamento observado no debug draw |
|---:|---|
| `0x00` | Linha padrão. |
| `0x01` | Linha com cor alternativa. |
| `0x02` | Linha padrão ou estado secundário. |
| Outros | Tratamento padrão. |

### Valores observados em `r104_27.EMI`

| Campo | Valores observados |
|---|---|
| `WorkType` | `0x0F` |
| `Work0` | `0x00`, `0x01`, `0x02` |
| `Work1` | `0x00`, `0x04` |
| `Work2` | `0x00`, `0x01` |
| `Dummy` | todos os bytes `0x00` |

### Recomendação prática

Para editar sem quebrar semântica desconhecida:

```text
preservar WorkType/Work0/Work1/Work2 quando apenas mover marcadores
copiar os quatro campos ao duplicar uma entrada
usar WorkType = 0x0F para novas entradas semelhantes às da amostra
zerar Dummy apenas em entradas novas criadas do zero
```

---

## 8. Coordenadas, direção e conversão para editores

O `.EMI` usa coordenadas internas do jogo em `float32`. Para visualização em Blender ou outra ferramenta 3D, uma conversão segura é:

```text
Blender X = Game X / scale
Blender Y = -Game Z / scale
Blender Z = Game Y / scale
```

Escalas úteis:

| Escala | Uso |
|---:|---|
| `1.0` | Debug bruto, sem conversão. |
| `100.0` | Visualização intermediária. |
| `1000.0` | Compatível com fluxo usado em vários formatos RE4/Blender. |

Conversão inversa:

```text
Game X = Blender X * scale
Game Y = Blender Z * scale
Game Z = -Blender Y * scale
```

`Dir` deve ser mantido como radiano. Para interface visual, pode ser mostrado também em graus:

```text
degrees = radians * 180.0 / π
radians = degrees * π / 180.0
```

Normalização recomendada:

```text
while Dir >  π: Dir -= 2π
while Dir < -π: Dir += 2π
```

---

## 9. Relação com `.ESL`

O `.ESL` é a tabela principal de enemy set. Ele define inimigos, flags, HP, sala, posição inicial compactada e outros dados de spawn.

O `.EMI` é uma estrutura separada:

| Formato | Função |
|---|---|
| `.ESL` | Lista de inimigos/spawn usada pelo sistema `EmSet`. |
| `.EMI` | Marcadores `EMINFO` com posição/direção e campos auxiliares de debug/ferramenta. |

O executável possui ponteiros separados para essas áreas. O `.ESL` é carregado por `readEmList`, enquanto o `.EMI` aparece associado ao ponteiro `pEmi` e às funções de debug `Draw_eminfo` / `ToolEmInfo`.

Na prática:

```text
editar .ESL altera spawns reais de inimigos
editar .EMI altera marcadores/dados auxiliares do sistema EMINFO
```

Para ferramentas completas de sala, o ideal é exibir `.ESL` e `.EMI` juntos, mas manter repack independente para cada arquivo.

---

## 10. Funções e símbolos relevantes

Símbolos relevantes encontrados no executável debug:

| Endereço | Símbolo | Uso |
|---:|---|---|
| `0x002952F0` | `Draw_eminfo__Fv` | Desenha marcadores `EMINFO` no debug draw. |
| `0x004C06E8` | `ToolEmInfo` | Instância global da ferramenta `cToolEmInfo`. |
| `0x002A8DD8` | `__11cToolEmInfo` | Construtor de `cToolEmInfo`. |
| `0x002A8DF8` | `CheckActiveEm__11cToolEmInfoP3cEmb` | Verifica inimigo ativo no tool/debug. |
| `0x002A8E80` | `CalcNextEmNo__11cToolEmInfo` | Calcula próximo inimigo selecionado. |
| `0x002A8F30` | `InfoDisp__11cToolEmInfo` | Exibe informações do inimigo/EM tool. |
| `0x002A91C0` | `EmDisp__11cToolEmInfoP3cEmUiii` | Exibe dados de inimigo no overlay. |

Estruturas debug confirmadas:

```text
EMINFO_WK:
    size 0x40
    Pos       +0x00 tagVec
    Dir       +0x10 float
    WorkType  +0x14 uint8
    Work0     +0x15 uint8
    Work1     +0x16 uint8
    Work2     +0x17 uint8
    Dummy     +0x18 uint8[40]

EMINFO_DATA:
    size 0x4010
    Num       +0x00 uint32
    Id        +0x04 uint32
    Data      +0x10 EMINFO_WK[256]
```

---

## 11. Amostra analisada: `r104_27.EMI`

Características do arquivo enviado:

| Propriedade | Valor |
|---|---:|
| Tamanho físico | `0x2A0` bytes |
| `Num` | `10` |
| `Id` | `0` |
| Tamanho lógico | `0x290` bytes |
| Padding final | `0x10` bytes |
| Byte de padding observado | `0xCD` |
| Entradas válidas | `10` |
| `Dummy` das entradas | zerado |

Tabela das entradas:

| Índice | Offset | X | Y | Z | W | Dir(rad) | WorkType | Work0 | Work1 | Work2 | Dummy |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| 0 | `0x0010` | 4154.338867 | -2089.955078 | -18651.566406 | 1.0 | -2.454370 | `0x0F` | `0x00` | `0x04` | `0x00` | zero |
| 1 | `0x0050` | -2429.620117 | 2440.549805 | -11605.425781 | 1.0 | 0.785397 | `0x0F` | `0x01` | `0x00` | `0x00` | zero |
| 2 | `0x0090` | 2678.139160 | 1229.551758 | -20465.597656 | 1.0 | 0.785397 | `0x0F` | `0x02` | `0x00` | `0x00` | zero |
| 3 | `0x00D0` | 1368.022461 | 1494.114258 | -19435.937500 | 1.0 | 0.785397 | `0x0F` | `0x02` | `0x00` | `0x00` | zero |
| 4 | `0x0110` | 63.309570 | 2235.088867 | -18365.210938 | 1.0 | 0.785397 | `0x0F` | `0x02` | `0x00` | `0x00` | zero |
| 5 | `0x0150` | -869.553955 | -1889.332520 | -12800.813477 | 1.0 | -0.883574 | `0x0F` | `0x00` | `0x04` | `0x01` | zero |
| 6 | `0x0190` | 3058.843750 | 1161.162109 | -20495.730469 | 1.0 | -0.883574 | `0x0F` | `0x01` | `0x00` | `0x01` | zero |
| 7 | `0x01D0` | -2421.362793 | 2440.551270 | -11764.953125 | 1.0 | 2.159842 | `0x0F` | `0x02` | `0x00` | `0x01` | zero |
| 8 | `0x0210` | -1254.741455 | 2435.664551 | -10712.789062 | 1.0 | 2.847066 | `0x0F` | `0x02` | `0x00` | `0x01` | zero |
| 9 | `0x0250` | 93.892578 | 2432.491211 | -11320.905273 | 1.0 | -2.454371 | `0x0F` | `0x02` | `0x00` | `0x01` | zero |

### Layout da amostra

```text
0x0000..0x000F  Header EMINFO_DATA
0x0010..0x028F  10 entradas EMINFO_WK
0x0290..0x029F  Padding 0xCD
```

---

## 12. Regras para editores e repackers

### Edição segura

Ao editar apenas posições e direção:

```text
manter Num
manter Id
manter Reserved0/Reserved1
manter WorkType/Work0/Work1/Work2
manter Dummy
manter padding final
reescrever apenas Pos e Dir
```

### Adicionar entrada

Para adicionar uma entrada:

```text
1. Num += 1
2. inserir EMINFO_WK de 0x40 bytes
3. preencher Pos.w = 1.0
4. preencher Dir em radianos
5. copiar WorkType/Work0/Work1/Work2 de entrada semelhante ou usar valores padrão
6. zerar Dummy em entrada nova
7. realinhar padding, se necessário
```

Limite:

```text
Num <= 256
```

### Remover entrada

Para remover uma entrada:

```text
1. remover exatamente 0x40 bytes
2. Num -= 1
3. atualizar padding final
```

### Padding final

O padding final não faz parte das entradas. Pode aparecer por alinhamento do container/arquivo.

Regras recomendadas:

| Situação | Ação |
|---|---|
| Roundtrip byte-safe | Preservar padding original. |
| Repack limpo | Alinhar para `0x10` com `0x00` ou preservar padrão do arquivo. |
| Arquivo com padding `0xCD` | Preservar `0xCD` se o objetivo for comparar com original. |

### Tamanho de saída

Dois modos são recomendados:

| Modo | Tamanho | Uso |
|---|---:|---|
| Compacto | `0x10 + Num * 0x40 + padding` | Arquivos de sala. |
| Runtime completo | `0x4010` | Dump interno/debug/ferramentas. |

Para modding comum, usar modo compacto.

---

## 13. Representação JSON/YAML recomendada

```yaml
emi:
  endian: little
  header:
    num: 10
    id: 0
    reserved0: 0
    reserved1: 0

  entries:
    - index: 0
      pos: { x: 4154.338867, y: -2089.955078, z: -18651.566406, w: 1.0 }
      dir_rad: -2.454370
      dir_deg: -140.624
      work:
        type: 0x0F
        work0: 0x00
        work1: 0x04
        work2: 0x00
      dummy_hex: "00000000000000000000000000000000000000000000000000000000000000000000000000000000"
```

Campos obrigatórios para repack:

```text
header.num
header.id
header.reserved0
header.reserved1
entries[].pos
entries[].dir_rad
entries[].work.type
entries[].work.work0
entries[].work.work1
entries[].work.work2
entries[].dummy_hex
```

---

## 14. Structs de referência

### C/C++

```c
#pragma pack(push, 1)

typedef struct tagVec {
    float x;
    float y;
    float z;
    float w;
} tagVec;

typedef struct EMINFO_WK {
    tagVec  Pos;        // +0x00
    float   Dir;        // +0x10
    uint8_t WorkType;   // +0x14
    uint8_t Work0;      // +0x15
    uint8_t Work1;      // +0x16
    uint8_t Work2;      // +0x17
    uint8_t Dummy[40];  // +0x18
} EMINFO_WK;            // sizeof = 0x40

typedef struct EMINFO_DATA_FILE {
    uint32_t Num;        // +0x00
    uint32_t Id;         // +0x04
    uint32_t Reserved0;  // +0x08
    uint32_t Reserved1;  // +0x0C
    EMINFO_WK Data[];    // +0x10, Num entries in compact files
} EMINFO_DATA_FILE;

typedef struct EMINFO_DATA_RUNTIME {
    uint32_t Num;        // +0x00
    uint32_t Id;         // +0x04
    uint32_t Reserved0;  // +0x08
    uint32_t Reserved1;  // +0x0C
    EMINFO_WK Data[256]; // +0x10
} EMINFO_DATA_RUNTIME;  // sizeof = 0x4010

#pragma pack(pop)
```

### C#

```csharp
using System;
using System.IO;
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct TagVec
{
    public float X;
    public float Y;
    public float Z;
    public float W;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public unsafe struct EmInfoWork
{
    public TagVec Pos;      // +0x00
    public float Dir;       // +0x10
    public byte WorkType;   // +0x14
    public byte Work0;      // +0x15
    public byte Work1;      // +0x16
    public byte Work2;      // +0x17
    public fixed byte Dummy[40]; // +0x18
}

public sealed class EmInfoData
{
    public uint Num;
    public uint Id;
    public uint Reserved0;
    public uint Reserved1;
    public EmInfoWork[] Entries = Array.Empty<EmInfoWork>();
}
```

---

## 15. Parser de referência em Python

```python
from __future__ import annotations

from dataclasses import dataclass
import math
import struct
from pathlib import Path


HEADER_SIZE = 0x10
ENTRY_SIZE = 0x40
MAX_ENTRIES = 256


@dataclass
class EmInfoEntry:
    index: int
    offset: int
    pos: tuple[float, float, float, float]
    dir_rad: float
    work_type: int
    work0: int
    work1: int
    work2: int
    dummy: bytes


@dataclass
class EmInfoFile:
    num: int
    file_id: int
    reserved0: int
    reserved1: int
    entries: list[EmInfoEntry]
    padding: bytes


def read_emi(path: str | Path) -> EmInfoFile:
    data = Path(path).read_bytes()

    if len(data) < HEADER_SIZE:
        raise ValueError("Arquivo EMI menor que 0x10 bytes.")

    num, file_id, reserved0, reserved1 = struct.unpack_from("<IIII", data, 0)

    if num > MAX_ENTRIES:
        raise ValueError(f"Num inválido: {num}. Máximo aceito: {MAX_ENTRIES}.")

    logical_size = HEADER_SIZE + num * ENTRY_SIZE

    if len(data) < logical_size:
        raise ValueError(
            f"Arquivo truncado: tamanho 0x{len(data):X}, necessário 0x{logical_size:X}."
        )

    entries: list[EmInfoEntry] = []

    for index in range(num):
        off = HEADER_SIZE + index * ENTRY_SIZE

        pos = struct.unpack_from("<4f", data, off + 0x00)
        dir_rad = struct.unpack_from("<f", data, off + 0x10)[0]
        work_type, work0, work1, work2 = struct.unpack_from("<BBBB", data, off + 0x14)
        dummy = data[off + 0x18 : off + 0x40]

        if not all(math.isfinite(v) for v in pos):
            raise ValueError(f"Entrada {index} possui Pos inválida.")

        if not math.isfinite(dir_rad):
            raise ValueError(f"Entrada {index} possui Dir inválido.")

        entries.append(
            EmInfoEntry(
                index=index,
                offset=off,
                pos=pos,
                dir_rad=dir_rad,
                work_type=work_type,
                work0=work0,
                work1=work1,
                work2=work2,
                dummy=dummy,
            )
        )

    padding = data[logical_size:]

    return EmInfoFile(
        num=num,
        file_id=file_id,
        reserved0=reserved0,
        reserved1=reserved1,
        entries=entries,
        padding=padding,
    )


def write_emi(path: str | Path, emi: EmInfoFile, preserve_padding: bool = True) -> None:
    if len(emi.entries) > MAX_ENTRIES:
        raise ValueError("EMI excede 256 entradas.")

    out = bytearray()
    out += struct.pack("<IIII", len(emi.entries), emi.file_id, emi.reserved0, emi.reserved1)

    for index, entry in enumerate(emi.entries):
        if len(entry.dummy) != 0x28:
            raise ValueError(f"Entrada {index} possui Dummy com tamanho inválido.")

        if not all(math.isfinite(v) for v in entry.pos):
            raise ValueError(f"Entrada {index} possui Pos inválida.")

        if not math.isfinite(entry.dir_rad):
            raise ValueError(f"Entrada {index} possui Dir inválido.")

        out += struct.pack("<4f", *entry.pos)
        out += struct.pack("<f", entry.dir_rad)
        out += struct.pack(
            "<BBBB",
            entry.work_type & 0xFF,
            entry.work0 & 0xFF,
            entry.work1 & 0xFF,
            entry.work2 & 0xFF,
        )
        out += entry.dummy

    if preserve_padding:
        out += emi.padding

    Path(path).write_bytes(out)


if __name__ == "__main__":
    emi = read_emi("r104_27.EMI")

    print(f"Num: {emi.num}")
    print(f"Id: {emi.file_id}")
    print(f"Padding: 0x{len(emi.padding):X} bytes")

    for e in emi.entries:
        x, y, z, w = e.pos
        print(
            f"{e.index:03d} "
            f"pos=({x:.3f}, {y:.3f}, {z:.3f}, {w:.1f}) "
            f"dir={e.dir_rad:.6f} "
            f"work=({e.work_type:02X},{e.work0:02X},{e.work1:02X},{e.work2:02X})"
        )
```

---

## 16. Checklist de roundtrip

Para validar um editor/repacker:

```text
1. abrir .EMI original
2. exportar sem modificar
3. comparar tamanho
4. comparar bytes
5. modificar apenas Pos/Dir de uma entrada
6. exportar
7. confirmar que apenas os floats alterados mudaram
8. adicionar entrada
9. confirmar Num atualizado
10. confirmar que cada entrada continua com stride 0x40
11. remover entrada
12. confirmar Num atualizado e ausência de desalinhamento
```

Comparação esperada para export sem alterações:

```text
original == exportado
```

Diferenças aceitáveis apenas quando o repacker for configurado para limpar padding.

---

## 17. Resumo de offsets

### Header

| Offset | Tipo | Nome |
|---:|---|---|
| `0x00` | `uint32` | `Num` |
| `0x04` | `uint32` | `Id` |
| `0x08` | `uint32` | `Reserved0` |
| `0x0C` | `uint32` | `Reserved1` |
| `0x10` | `EMINFO_WK[]` | `Data` |

### Entrada `EMINFO_WK`

| Offset relativo | Tipo | Nome |
|---:|---|---|
| `0x00` | `float` | `Pos.x` |
| `0x04` | `float` | `Pos.y` |
| `0x08` | `float` | `Pos.z` |
| `0x0C` | `float` | `Pos.w` |
| `0x10` | `float` | `Dir` |
| `0x14` | `uint8` | `WorkType` |
| `0x15` | `uint8` | `Work0` |
| `0x16` | `uint8` | `Work1` |
| `0x17` | `uint8` | `Work2` |
| `0x18` | `uint8[40]` | `Dummy` |
| `0x40` | — | próxima entrada |
