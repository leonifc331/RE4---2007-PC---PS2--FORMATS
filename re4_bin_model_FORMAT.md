# Resident Evil 4 PS2 — Formato `.BIN` de modelos 3D

Documentação técnica do formato de modelos 3D usado pelo Resident Evil 4 de PlayStation 2. O nome `.BIN` aparece no runtime como banco de modelos carregado por `cSmd::getBinPtr`, por objetos de sala, inimigos, armas, objetos de cenário e modelos auxiliares.

Esta documentação cobre o layout usado pelo engine para `cModelData`, incluindo o banco de modelos embutido em `.SMD`, as estruturas de partes, bounding box, grupos de polígonos, blocos primitivos e palettes de vértice/UV.

## Escopo

O formato de modelo pode aparecer de duas formas:

| Forma | Descrição |
|---|---|
| Banco `.BIN` dentro de `.SMD` | Usado em salas. O `.SMD` contém uma tabela de objetos `cSmdWork`, depois um banco de modelos indexado por offsets. |
| Arquivo `.BIN` standalone | Usado por modelos externos do jogo, por exemplo player, inimigos, armas, objetos e modelos auxiliares. Pode iniciar diretamente em `cModelData` ou usar um wrapper/banco específico do asset. |

No container de sala, a relação normal é:

```text
.DAT
 └── SMD
      ├── SMD header
      ├── cSmdWork[WorkCount]
      ├── BIN bank
      │    ├── offset table
      │    ├── cModelData #0
      │    ├── cModelData #1
      │    └── ...
      └── TPL/texture block
```

## Endianness

Todos os campos numéricos nos arquivos analisados são **little-endian**.

| Tipo | Endian |
|---|---|
| `uint8_t` | não aplicável |
| `uint16_t` | little-endian |
| `uint32_t` | little-endian |
| `float` | IEEE-754 little-endian |
| `int16_t` | little-endian |

## Alinhamento

| Estrutura | Alinhamento prático |
|---|---:|
| Header `.SMD` | `0x10` |
| `cSmdWork` | `0x40` |
| Tabela de offsets do BIN bank | `0x04` |
| `cModelData` | geralmente alinhado a `0x10` |
| `BIN_PART_ENTRY` | `0x10` |
| Bounding box | `0x20` |
| Palettes e blocos de primitivo | geralmente `0x10`/`0x20`, dependendo do bloco |

---

# 1. Banco de modelos dentro do `.SMD`

O `.SMD` de sala possui um header de `0x10` bytes. Após o header vem a tabela `cSmdWork`. O campo `BinBlockOffset` aponta para o banco de modelos.

## 1.1 Header `.SMD`

```c
typedef struct RE4_SMD_HEADER {
    uint8_t  Version;        // +0x00. Exemplo: 0x40.
    uint8_t  Reserved01;     // +0x01.
    uint16_t WorkCount;      // +0x02. Quantidade de cSmdWork.
    uint32_t BinBlockOffset; // +0x04. Offset relativo ao início do SMD.
    uint32_t TplBlockOffset; // +0x08. Offset relativo ao início do SMD.
    uint32_t Reserved0C;     // +0x0C.
} RE4_SMD_HEADER;
```

### Valores aceitos

| Campo | Tipo | Valores aceitos | Observações |
|---|---:|---|---|
| `Version` | `u8` | `0x00..0xFF` | Em salas modernas foi observado `0x40`. Versões antigas abaixo de `0x20` recebem tratamento especial de motion. |
| `Reserved01` | `u8` | preservar | Normalmente `0x00`. |
| `WorkCount` | `u16` | `0..256` recomendado | O runtime trabalha com tabelas fixas de objetos; editor deve validar limite prático. |
| `BinBlockOffset` | `u32` | offset válido | Deve apontar para o início do banco de modelos. |
| `TplBlockOffset` | `u32` | offset válido | Deve ser maior que `BinBlockOffset`. Usado como fim prático do banco BIN em sala. |
| `Reserved0C` | `u32` | preservar | Normalmente `0x00000000`. |

## 1.2 Estrutura `cSmdWork`

Cada entrada possui `0x40` bytes e instancia um modelo do banco BIN.

```c
typedef struct RE4_SMD_WORK {
    float Pos[4];       // +0x00. Posição do objeto.
    float Ang[4];       // +0x10. Rotação/orientação usada pelo runtime.
    float Scale[4];     // +0x20. Escala. Em salas, comum: 100.0,100.0,100.0,1.0.

    uint8_t BinNo;      // +0x30. Índice no BIN bank.
    uint8_t TplNo;      // +0x31. Índice de textura/TPL.
    uint8_t MotNo;      // +0x32. Índice de motion. 0xFF = sem motion.
    uint8_t ObjNo;      // +0x33. ID de objeto/work. 0xFF = não instanciar.

    uint8_t Reserved34[4]; // +0x34. Preservar.
    uint32_t Flag;         // +0x38. Flags de instância/render.
    uint32_t Reserved3C;   // +0x3C. Preservar.
} RE4_SMD_WORK;
```

### Valores aceitos em `cSmdWork`

| Campo | Tipo | Valores aceitos | Uso |
|---|---:|---|---|
| `Pos[0..2]` | `float` | qualquer float finito | Posição em coordenadas do jogo. |
| `Pos[3]` | `float` | preservar; comum `1.0` | Componente auxiliar/homogêneo. |
| `Ang[0..2]` | `float` | qualquer float finito | Rotação/orientação usada por `SmdSetParam`. |
| `Ang[3]` | `float` | preservar; comum `0.0`/`1.0` | Auxiliar. |
| `Scale[0..2]` | `float` | maior que `0.0` recomendado | Em sala, `100.0` é o padrão comum. |
| `Scale[3]` | `float` | preservar | Normalmente `1.0`. |
| `BinNo` | `u8` | `0..254`, `0xFF` especial | `0xFF` é corrigido para `0` pelo runtime em alguns fluxos. |
| `TplNo` | `u8` | `0..254`, `0xFF` especial | `0xFF` é corrigido para `0` pelo runtime em alguns fluxos. |
| `MotNo` | `u8` | `0..254`, `0xFF` | `0xFF` = sem motion. Em versões antigas, `0` pode virar `0xFF`. |
| `ObjNo` | `u8` | `0..254`, `0xFF` | `0xFF` = entrada ignorada/não instanciada. |
| `Flag` | `u32` | bitfield | Ver tabela abaixo. |
| `Reserved34`, `Reserved3C` | bytes/u32 | preservar | Não zerar em repack byte-safe. |

### Bits conhecidos de `Flag`

| Bit | Máscara | Comportamento conhecido |
|---:|---:|---|
| 0 | `0x00000001` | Ativa flag de objeto associada ao bit `0x10` em `SmxSetFlag`. |
| 2 | `0x00000004` | Ativa flag interna `0x02000000`; se ausente, o runtime limpa esse estado. |
| 3 | `0x00000008` | Ativa estado byte em `cObj + 0x11F` com valor `0x80`. Muito comum em objetos de sala. |
| 4 | `0x00000010` | Usa banco comum/external para BIN/TPL (`pSmdComn`) e marca o objeto com flag `0x08000000`. |
| 6 | `0x00000040` | Usa banco comum/external para MOT. |
| demais | variável | Preservar. |

## 1.3 Banco BIN dentro do `.SMD`

O banco começa em `SMD + BinBlockOffset`.

```c
typedef struct RE4_BIN_BANK {
    uint32_t ModelOffset[]; // offsets relativos ao início do BIN bank
    // cModelData model_0;
    // cModelData model_1;
    // ...
} RE4_BIN_BANK;
```

A quantidade de slots pode ser obtida pelo primeiro offset:

```text
TableCount = FirstModelOffset / 4
```

Offsets `0x00000000` no final da tabela podem ser padding. Os offsets válidos devem apontar para chunks `cModelData`.

### Valores aceitos na tabela de offsets

| Campo | Valores aceitos | Regra |
|---|---|---|
| `ModelOffset[i]` | `0` ou offset válido | `0` no final pode ser padding; no meio deve ser evitado. |
| Primeiro offset | múltiplo de `4` | Define o tamanho da tabela. |
| Offsets válidos | crescente | Devem ficar dentro do banco BIN. |
| Último modelo | termina em `TplBlockOffset - BinBlockOffset` | No `.SMD`, o início do TPL é o fim prático do banco BIN. |

---

# 2. Chunk de modelo `cModelData`

Cada modelo é um chunk independente iniciado por `cModelData`.

## 2.1 Header `cModelData`

```c
typedef struct RE4_BIN_MODEL_DATA {
    uint16_t HeaderSize;       // +0x00. Normalmente 0x0030.
    uint16_t Format;           // +0x02. Normalmente 0x003A.
    uint32_t PartsOffset;      // +0x04. Offset para BIN_PART_ENTRY[].

    uint8_t  VertexFrac;       // +0x08. Escala fixa: 1.0 / (1 << VertexFrac).
    uint8_t  PartCount;        // +0x09. Quantidade de partes/bones locais.
    uint16_t PolyGroupCount;   // +0x0A. Quantidade de grupos de render/polígonos.

    uint32_t PolyOffset;       // +0x0C. Offset para BIN_POLY_GROUP[PolyGroupCount].
    uint32_t VertexBlockOffset;// +0x10. Offset para bloco/palette associado ao modelo.
    uint32_t VertexBlockInfo;  // +0x14. Informação auxiliar do bloco acima.

    uint32_t JointMagic;       // +0x18. Identificador/flags de joint. Ex.: 0x20010801.
    uint32_t JointOffset;      // +0x1C. Offset para dados de joint, quando aplicável.
    uint32_t ShapeOffset;      // +0x20. Offset opcional para shape/morph. 0 = ausente.
    uint32_t RenderParam;      // +0x24. Parâmetro preservado usado pelo runtime/render.

    uint32_t BoundingBoxOffset;// +0x28. Offset para BIN_BOUNDING_BOX.
    uint32_t ExtraOffset;      // +0x2C. Offset auxiliar/reservado. Preservar.
} RE4_BIN_MODEL_DATA;
```

> `VertexBlockOffset`, `VertexBlockInfo`, `RenderParam` e `ExtraOffset` variam conforme o tipo do modelo. Em modelos de cenário analisados, alguns campos aparecem como `0xCDCDCDCD` ou valores de controle. Ferramentas devem preservar esses campos quando não forem recalculados.

## 2.2 Valores aceitos em `cModelData`

| Campo | Tipo | Valores aceitos | Regra para editor/repacker |
|---|---:|---|---|
| `HeaderSize` | `u16` | `0x0030` observado | Manter `0x0030` para modelos comuns. |
| `Format` | `u16` | `0x003A` observado | Manter `0x003A` para compatibilidade. |
| `PartsOffset` | `u32` | offset válido ou `0` | Se `PartCount > 0`, deve apontar para tabela de partes. |
| `VertexFrac` | `u8` | `0..15` recomendado | Valores observados em sala: `2,4,5,6,7,9`. |
| `PartCount` | `u8` | `0..255` | Quantidade de `BIN_PART_ENTRY`. Em cenário comum: `1`. |
| `PolyGroupCount` | `u16` | `0..65535` | Quantidade de grupos em `PolyOffset`. Validar contra tamanho do chunk. |
| `PolyOffset` | `u32` | offset válido | Deve apontar para grupos de polígonos/render. |
| `VertexBlockOffset` | `u32` | offset válido, `0`, ou sentinel preservado | Não zerar se aparecer `0xCDCDCDCD`. |
| `VertexBlockInfo` | `u32` | valor de runtime/sentinel | Preservar se o editor não recalcular palettes. |
| `JointMagic` | `u32` | `0x20010801`, `0x20030818`, outros preservados | `0x20030818` aciona leitura de dados de joint no runtime. |
| `JointOffset` | `u32` | offset válido ou `0` | Usado se o modelo tiver dados de joint. |
| `ShapeOffset` | `u32` | offset válido ou `0` | Se diferente de zero, o runtime marca shape/morph como presente. |
| `RenderParam` | `u32` | preservar | Observado: `0xA0000000`, `0xE0000000`. |
| `BoundingBoxOffset` | `u32` | offset válido | Deve apontar para `BIN_BOUNDING_BOX`. |
| `ExtraOffset` | `u32` | preservar | Uso variável. |

## 2.3 Escala de vértice

O campo `VertexFrac` controla a escala dos vértices fixos de 16 bits:

```text
VertexScale = 1.0 / (1 << VertexFrac)
WorldVertex = RawInt16Vertex * VertexScale
```

Exemplos:

| `VertexFrac` | Escala |
|---:|---:|
| `2` | `1 / 4` |
| `4` | `1 / 16` |
| `5` | `1 / 32` |
| `6` | `1 / 64` |
| `7` | `1 / 128` |
| `9` | `1 / 512` |

---

# 3. Partes / hierarquia local

A tabela de partes é apontada por `PartsOffset` e possui `PartCount` entradas de `0x10` bytes.

```c
typedef struct RE4_BIN_PART_ENTRY {
    uint8_t PartId;       // +0x00. ID/índice da parte.
    uint8_t ParentId;     // +0x01. 0xFF = root/sem pai.
    uint8_t Reserved02;   // +0x02. Preservar.
    uint8_t Reserved03;   // +0x03. Preservar.
    float   OffsetX;      // +0x04. Offset local X.
    float   OffsetY;      // +0x08. Offset local Y.
    float   OffsetZ;      // +0x0C. Offset local Z.
} RE4_BIN_PART_ENTRY;
```

## Valores aceitos

| Campo | Valores aceitos | Observações |
|---|---|---|
| `PartId` | `0..254` | Deve ser único dentro do modelo sempre que possível. |
| `ParentId` | `0..254`, `0xFF` | `0xFF` indica root. Se não for root, deve apontar para uma parte existente. |
| `Reserved02/03` | preservar | Evitar alterar. |
| `OffsetX/Y/Z` | float finito | Offset local da parte. |

## Regras de hierarquia

- `ParentId == 0xFF`: parte ligada ao root do modelo.
- `ParentId != 0xFF`: parte filha de outra entrada.
- A hierarquia é montada em `setPartsParent__6cModel`.
- A posição local é copiada para a estrutura de partes do runtime durante `setPartsOffset__6cModelP10cModelData`.

---

# 4. Bounding box

O bounding box é apontado por `BoundingBoxOffset`.

```c
typedef struct RE4_BIN_BOUNDING_BOX {
    float MinX;          // +0x00
    float MinY;          // +0x04
    float MinZ;          // +0x08
    uint32_t Reserved0C; // +0x0C. Preservar.
    float MaxX;          // +0x10
    float MaxY;          // +0x14
    float MaxZ;          // +0x18
    uint32_t Reserved1C; // +0x1C. Preservar.
} RE4_BIN_BOUNDING_BOX;
```

## Valores aceitos

| Campo | Valores aceitos | Observações |
|---|---|---|
| `MinX/Y/Z` | float finito | Mínimos locais do modelo. |
| `MaxX/Y/Z` | float finito | Máximos locais do modelo. |
| `Reserved0C/1C` | preservar | Pode conter sentinel ou dado auxiliar. Não usar como componente `W`. |

## Regra de validação

```text
MinX <= MaxX
MinY <= MaxY
MinZ <= MaxZ
```

Se o editor recalcular geometria, o bounding box deve ser atualizado.

---

# 5. Grupos de polígonos / material

O campo `PolyOffset` aponta para uma sequência de grupos. A quantidade é `PolyGroupCount`.

```c
typedef struct RE4_BIN_POLY_GROUP {
    uint8_t  Attr0;          // +0x00. Flags/atributo.
    uint8_t  TexNo;          // +0x01. Índice de textura/TPL.
    uint8_t  Attr2;          // +0x02.
    uint8_t  Attr3;          // +0x03.
    uint8_t  Attr4;          // +0x04.
    uint8_t  Attr5;          // +0x05.
    uint8_t  Attr6;          // +0x06.
    uint8_t  Attr7;          // +0x07.
    uint8_t  Attr8;          // +0x08.
    uint8_t  Attr9;          // +0x09.
    uint8_t  AttrA;          // +0x0A.
    uint8_t  AttrB;          // +0x0B.
    uint32_t PrimBlockCount; // +0x0C. Quantidade de blocos primitivos.
    // RE4_BIN_PRIM_BLOCK PrimBlocks[PrimBlockCount]
} RE4_BIN_POLY_GROUP;
```

## Valores aceitos

| Campo | Valores aceitos | Observações |
|---|---|---|
| `TexNo` | `0..255` | Índice de textura do TPL associado ao modelo/instância. |
| `Attr0..AttrB` | `0x00..0xFF` | Flags de render/material. Preservar quando não houver tabela mapeada. |
| `PrimBlockCount` | `0..N` | Deve caber dentro do chunk do modelo. Valores muito altos indicam offset errado. |

## Observação sobre `PrimBlockCount`

O valor em `+0x0C` é tratado como contador de blocos primitivos no fluxo de render. Ele não deve ser confundido com offset em bytes.

---

# 6. Blocos primitivos

Cada bloco primitivo começa com um comando de 1 byte. O runtime mascara o comando com `0xF8` para escolher a rotina de montagem.

```c
typedef struct RE4_BIN_PRIM_HEADER {
    uint8_t Cmd;       // +0x00. Tipo + flags.
    uint8_t SubAttr;   // +0x01. Atributo secundário. Preservar.
    // payload variável
} RE4_BIN_PRIM_HEADER;
```

## Tipos aceitos de primitivo

| Valor mascarado `Cmd & 0xF8` | Nome | Rotina |
|---:|---|---|
| `0x80` | Quad | `insert_shape_quad` |
| `0x90` | Triangle | `insert_shape_triangle` |
| `0x98` | Triangle Strip | `insert_shape_tristrip` |
| outros | Tratamento de fallback | Em fluxos analisados cai para lógica equivalente a strip ou deve ser preservado. |

## Bits baixos de `Cmd`

| Bits | Máscara | Uso recomendado |
|---:|---:|---|
| `0..2` | `0x07` | Flags locais do primitivo. Preservar. |
| `3..7` | `0xF8` | Tipo primitivo. |

## Payload de triangle strip

```c
typedef struct RE4_BIN_TRISTRIP_BLOCK {
    uint8_t  Cmd;         // +0x00. Cmd & 0xF8 == 0x98.
    uint8_t  SubAttr;     // +0x01.
    uint16_t VertexCount; // +0x02. Quantidade de registros de índice.
    // RE4_BIN_VERTEX_REF Refs[VertexCount]
} RE4_BIN_TRISTRIP_BLOCK;
```

## Registro de vértice usado pelo primitivo

Os primitivos referenciam dados por índices em palettes internas.

```c
typedef struct RE4_BIN_VERTEX_REF {
    uint16_t ShapeVertexIndex; // +0x00. Índice na palette de shape/posição A.
    uint16_t BaseVertexIndex;  // +0x02. Índice na palette base/posição B.
    uint16_t AuxIndex;         // +0x04. Índice auxiliar; preservar se não for remapeado.
    uint16_t UVIndex;          // +0x06. Índice na palette UV.
} RE4_BIN_VERTEX_REF;
```

## Valores aceitos

| Campo | Valores aceitos | Regra |
|---|---|---|
| `VertexCount` | `0..65535` | Para strip, mínimo prático `3`. Para quad/tri, quantidade depende do tipo. |
| `ShapeVertexIndex` | índice válido | Deve apontar para `RE4_BIN_VPALETTE`. |
| `BaseVertexIndex` | índice válido | Deve apontar para segunda palette/posição base. |
| `AuxIndex` | índice válido ou valor preservado | Pode referenciar normal/cor/peso dependendo do fluxo. |
| `UVIndex` | índice válido | Deve apontar para `RE4_BIN_UPALETTE`. |

---

# 7. Palettes de vértice e UV

Os dados geométricos são armazenados em palettes e acessados por índices dos blocos primitivos.

## 7.1 Palette de vértice

```c
typedef struct RE4_BIN_VPALETTE {
    int16_t X;               // +0x00
    int16_t Y;               // +0x02
    int16_t Z;               // +0x04
    int16_t MatrixOrWeightId;// +0x06. Índice local de matriz/peso.
} RE4_BIN_VPALETTE;
```

### Conversão para coordenada local

```text
FloatX = X * (1.0 / (1 << VertexFrac))
FloatY = Y * (1.0 / (1 << VertexFrac))
FloatZ = Z * (1.0 / (1 << VertexFrac))
```

## 7.2 Palette de UV

```c
typedef struct RE4_BIN_UPALETTE {
    int16_t U; // +0x00
    int16_t V; // +0x02
} RE4_BIN_UPALETTE;
```

### Conversão prática de UV

O runtime aplica uma fração fixa interna (`texcrd_frac`). Para ferramenta de modding, há duas abordagens seguras:

| Modo | Regra |
|---|---|
| Preserve/raw | Exportar `U` e `V` exatamente como lidos. Recomendado para roundtrip. |
| Editável | Expor como `float`, mas salvar de volta em `int16` usando o mesmo fator do arquivo original. |

Evite normalizar UV automaticamente para `0..1` sem validar o fator do modelo, pois muitos modelos usam escala fixa própria.

---

# 8. Dados de joint / skinning

O campo `JointMagic` define se o modelo possui dados especiais de joint.

| `JointMagic` | Significado prático |
|---:|---|
| `0x20010801` | Valor comum em modelos estáticos/de sala analisados. |
| `0x20030818` | Reconhecido por `setJointInfo__6cModelP10cModelData` como layout com joint info. |
| outros | Preservar. |

Quando `JointMagic == 0x20030818`, o runtime usa `JointOffset` para localizar dados adicionais de joint. Para modelos estáticos, `JointOffset` geralmente é `0`.

## Regra para editor

- Para modelos de cenário sem skinning, preserve `JointMagic` e `JointOffset`.
- Para modelos skinned, não remover nem compactar o bloco apontado por `JointOffset` sem atualizar as referências.
- `MatrixOrWeightId` em `RE4_BIN_VPALETTE` participa da seleção de matriz local no render.

---

# 9. Tabela de valores aceitos por campo

## 9.1 Header de modelo

| Campo | Domínio seguro | Domínio avançado | Valor observado |
|---|---|---|---|
| `HeaderSize` | `0x0030` | outro apenas com validação | `0x0030` |
| `Format` | `0x003A` | outro apenas com validação | `0x003A` |
| `VertexFrac` | `0..12` | `0..31` | `2,4,5,6,7,9` |
| `PartCount` | `1..64` | `0..255` | `1` em modelos de sala analisados |
| `PolyGroupCount` | `1..64` | `0..65535` | `1,2,4` em amostras de sala |
| `JointMagic` | preservar | `0x20010801`, `0x20030818`, outros | `0x20010801` |
| `RenderParam` | preservar | qualquer `u32` | `0xA0000000`, `0xE0000000` |

## 9.2 Partes

| Campo | Domínio seguro | Observação |
|---|---|---|
| `PartId` | `0..PartCount-1` | Evitar duplicados. |
| `ParentId` | `0..PartCount-1`, `0xFF` | `0xFF` = root. |
| `OffsetX/Y/Z` | float finito | Offset local. |

## 9.3 Polígonos

| Campo | Domínio seguro | Observação |
|---|---|---|
| `TexNo` | `0..TextureCount-1` | Se fora do TPL, pode renderizar com textura errada ou crashar. |
| `Cmd & 0xF8` | `0x80`, `0x90`, `0x98` | Tipos primários conhecidos. |
| `Cmd & 0x07` | preservar | Flags locais. |
| `VertexCount` | `3..4096` recomendado | Validar contra tamanho do chunk. |

## 9.4 Vertex/UV palette

| Campo | Domínio | Observação |
|---|---|---|
| `X/Y/Z` | `-32768..32767` | `int16`. Usa `VertexFrac`. |
| `MatrixOrWeightId` | `-32768..32767` | Interpretado como índice/slot de matriz/peso. |
| `U/V` | `-32768..32767` | `int16`. Preservar escala original. |

---

# 10. Estruturas C completas

```c
#include <stdint.h>

typedef struct RE4_SMD_HEADER {
    uint8_t  Version;
    uint8_t  Reserved01;
    uint16_t WorkCount;
    uint32_t BinBlockOffset;
    uint32_t TplBlockOffset;
    uint32_t Reserved0C;
} RE4_SMD_HEADER;

typedef struct RE4_SMD_WORK {
    float Pos[4];
    float Ang[4];
    float Scale[4];
    uint8_t BinNo;
    uint8_t TplNo;
    uint8_t MotNo;
    uint8_t ObjNo;
    uint8_t Reserved34[4];
    uint32_t Flag;
    uint32_t Reserved3C;
} RE4_SMD_WORK;

typedef struct RE4_BIN_MODEL_DATA {
    uint16_t HeaderSize;
    uint16_t Format;
    uint32_t PartsOffset;
    uint8_t  VertexFrac;
    uint8_t  PartCount;
    uint16_t PolyGroupCount;
    uint32_t PolyOffset;
    uint32_t VertexBlockOffset;
    uint32_t VertexBlockInfo;
    uint32_t JointMagic;
    uint32_t JointOffset;
    uint32_t ShapeOffset;
    uint32_t RenderParam;
    uint32_t BoundingBoxOffset;
    uint32_t ExtraOffset;
} RE4_BIN_MODEL_DATA;

typedef struct RE4_BIN_PART_ENTRY {
    uint8_t PartId;
    uint8_t ParentId;
    uint8_t Reserved02;
    uint8_t Reserved03;
    float OffsetX;
    float OffsetY;
    float OffsetZ;
} RE4_BIN_PART_ENTRY;

typedef struct RE4_BIN_BOUNDING_BOX {
    float MinX;
    float MinY;
    float MinZ;
    uint32_t Reserved0C;
    float MaxX;
    float MaxY;
    float MaxZ;
    uint32_t Reserved1C;
} RE4_BIN_BOUNDING_BOX;

typedef struct RE4_BIN_POLY_GROUP {
    uint8_t  Attr0;
    uint8_t  TexNo;
    uint8_t  Attr2;
    uint8_t  Attr3;
    uint8_t  Attr4;
    uint8_t  Attr5;
    uint8_t  Attr6;
    uint8_t  Attr7;
    uint8_t  Attr8;
    uint8_t  Attr9;
    uint8_t  AttrA;
    uint8_t  AttrB;
    uint32_t PrimBlockCount;
} RE4_BIN_POLY_GROUP;

typedef struct RE4_BIN_PRIM_HEADER {
    uint8_t Cmd;
    uint8_t SubAttr;
} RE4_BIN_PRIM_HEADER;

typedef struct RE4_BIN_TRISTRIP_BLOCK {
    uint8_t  Cmd;
    uint8_t  SubAttr;
    uint16_t VertexCount;
} RE4_BIN_TRISTRIP_BLOCK;

typedef struct RE4_BIN_VERTEX_REF {
    uint16_t ShapeVertexIndex;
    uint16_t BaseVertexIndex;
    uint16_t AuxIndex;
    uint16_t UVIndex;
} RE4_BIN_VERTEX_REF;

typedef struct RE4_BIN_VPALETTE {
    int16_t X;
    int16_t Y;
    int16_t Z;
    int16_t MatrixOrWeightId;
} RE4_BIN_VPALETTE;

typedef struct RE4_BIN_UPALETTE {
    int16_t U;
    int16_t V;
} RE4_BIN_UPALETTE;
```

---

# 11. Estruturas C#

```csharp
using System;
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4SmdHeader
{
    public byte Version;
    public byte Reserved01;
    public ushort WorkCount;
    public uint BinBlockOffset;
    public uint TplBlockOffset;
    public uint Reserved0C;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public unsafe struct Re4SmdWork
{
    public fixed float Pos[4];
    public fixed float Ang[4];
    public fixed float Scale[4];
    public byte BinNo;
    public byte TplNo;
    public byte MotNo;
    public byte ObjNo;
    public fixed byte Reserved34[4];
    public uint Flag;
    public uint Reserved3C;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4BinModelData
{
    public ushort HeaderSize;
    public ushort Format;
    public uint PartsOffset;
    public byte VertexFrac;
    public byte PartCount;
    public ushort PolyGroupCount;
    public uint PolyOffset;
    public uint VertexBlockOffset;
    public uint VertexBlockInfo;
    public uint JointMagic;
    public uint JointOffset;
    public uint ShapeOffset;
    public uint RenderParam;
    public uint BoundingBoxOffset;
    public uint ExtraOffset;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4BinPartEntry
{
    public byte PartId;
    public byte ParentId;
    public byte Reserved02;
    public byte Reserved03;
    public float OffsetX;
    public float OffsetY;
    public float OffsetZ;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4BinBoundingBox
{
    public float MinX;
    public float MinY;
    public float MinZ;
    public uint Reserved0C;
    public float MaxX;
    public float MaxY;
    public float MaxZ;
    public uint Reserved1C;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4BinPolyGroup
{
    public byte Attr0;
    public byte TexNo;
    public byte Attr2;
    public byte Attr3;
    public byte Attr4;
    public byte Attr5;
    public byte Attr6;
    public byte Attr7;
    public byte Attr8;
    public byte Attr9;
    public byte AttrA;
    public byte AttrB;
    public uint PrimBlockCount;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4BinVertexRef
{
    public ushort ShapeVertexIndex;
    public ushort BaseVertexIndex;
    public ushort AuxIndex;
    public ushort UVIndex;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4BinVPalette
{
    public short X;
    public short Y;
    public short Z;
    public short MatrixOrWeightId;
}

[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Re4BinUPalette
{
    public short U;
    public short V;
}
```

---

# 12. Parser Python de referência

O parser abaixo identifica um `.SMD` com banco BIN e lista os modelos `cModelData`. Ele também consegue ler um chunk standalone iniciado diretamente em `cModelData`.

```python
import struct
from dataclasses import dataclass
from pathlib import Path


def u8(data, off):
    return data[off]


def u16(data, off):
    return struct.unpack_from('<H', data, off)[0]


def u32(data, off):
    return struct.unpack_from('<I', data, off)[0]


def f32(data, off):
    return struct.unpack_from('<f', data, off)[0]


def valid_range(data, off, size):
    return 0 <= off <= len(data) - size


@dataclass
class BinModelHeader:
    offset: int
    size: int
    header_size: int
    fmt: int
    parts_offset: int
    vertex_frac: int
    part_count: int
    poly_group_count: int
    poly_offset: int
    vertex_block_offset: int
    vertex_block_info: int
    joint_magic: int
    joint_offset: int
    shape_offset: int
    render_param: int
    bbox_offset: int
    extra_offset: int


def parse_model_header(data, off, size=0):
    if not valid_range(data, off, 0x30):
        raise ValueError(f'cModelData fora do arquivo: 0x{off:X}')

    return BinModelHeader(
        offset=off,
        size=size,
        header_size=u16(data, off + 0x00),
        fmt=u16(data, off + 0x02),
        parts_offset=u32(data, off + 0x04),
        vertex_frac=u8(data, off + 0x08),
        part_count=u8(data, off + 0x09),
        poly_group_count=u16(data, off + 0x0A),
        poly_offset=u32(data, off + 0x0C),
        vertex_block_offset=u32(data, off + 0x10),
        vertex_block_info=u32(data, off + 0x14),
        joint_magic=u32(data, off + 0x18),
        joint_offset=u32(data, off + 0x1C),
        shape_offset=u32(data, off + 0x20),
        render_param=u32(data, off + 0x24),
        bbox_offset=u32(data, off + 0x28),
        extra_offset=u32(data, off + 0x2C),
    )


def parse_smd_bin_bank(smd):
    version = u8(smd, 0x00)
    work_count = u16(smd, 0x02)
    bin_off = u32(smd, 0x04)
    tpl_off = u32(smd, 0x08)

    if not valid_range(smd, 0x10, work_count * 0x40):
        raise ValueError('Tabela cSmdWork fora do arquivo')

    if not (0 <= bin_off < tpl_off <= len(smd)):
        raise ValueError('Offsets BinBlock/TplBlock inválidos')

    bank = smd[bin_off:tpl_off]
    first_model_off = u32(bank, 0)
    table_count = first_model_off // 4

    offsets = [u32(bank, i * 4) for i in range(table_count)]
    valid_offsets = [(i, off) for i, off in enumerate(offsets) if off != 0]

    models = []
    for n, (model_id, rel) in enumerate(valid_offsets):
        next_rel = tpl_off - bin_off
        for _, candidate in valid_offsets[n + 1:]:
            if candidate > rel:
                next_rel = candidate
                break
        size = next_rel - rel
        models.append(parse_model_header(bank, rel, size))

    return {
        'version': version,
        'work_count': work_count,
        'bin_offset': bin_off,
        'tpl_offset': tpl_off,
        'table_count': table_count,
        'models': models,
    }


def parse_standalone_bin(data):
    # Modelo standalone que começa diretamente em cModelData.
    h = parse_model_header(data, 0, len(data))
    if h.header_size != 0x30 or h.fmt != 0x3A:
        raise ValueError('Arquivo não parece iniciar em cModelData comum')
    return h


if __name__ == '__main__':
    path = Path('r402_smd.bin')
    data = path.read_bytes()

    # Se for um SMD completo:
    info = parse_smd_bin_bank(data)
    print('version:', hex(info['version']))
    print('work_count:', info['work_count'])
    print('models:', len(info['models']))
    for i, model in enumerate(info['models']):
        print(i, model)
```

---

# 13. Extração do banco BIN a partir de `.DAT`

Para salas, primeiro extraia o subarquivo `SMD` do `.DAT`. A tag é `SMD`. Depois use `BinBlockOffset` e `TplBlockOffset`.

```python
import struct
from pathlib import Path


def extract_tag_from_dat(dat, wanted_tag, ordinal=0):
    count = struct.unpack_from('<I', dat, 0)[0]
    offset_table = 0x10
    tag_table = offset_table + count * 4

    offsets = [struct.unpack_from('<I', dat, offset_table + i * 4)[0] for i in range(count)]
    tags = [dat[tag_table + i * 4:tag_table + i * 4 + 3].decode('ascii', 'replace') for i in range(count)]

    hit = 0
    for i, tag in enumerate(tags):
        if tag == wanted_tag:
            if hit == ordinal:
                start = offsets[i]
                end = offsets[i + 1] if i + 1 < count else len(dat)
                return dat[start:end]
            hit += 1
    raise KeyError(f'tag não encontrada: {wanted_tag}[{ordinal}]')


def extract_smd_bin_bank(dat_path):
    dat = Path(dat_path).read_bytes()
    smd = extract_tag_from_dat(dat, 'SMD', 0)

    bin_off = struct.unpack_from('<I', smd, 0x04)[0]
    tpl_off = struct.unpack_from('<I', smd, 0x08)[0]

    return smd[bin_off:tpl_off]
```

---

# 14. Exemplo observado em `r402.dat`

No arquivo `r402.dat`, o subarquivo `SMD` possui:

| Campo | Valor |
|---|---:|
| `Version` | `0x40` |
| `WorkCount` | `101` |
| `BinBlockOffset` | `0x00001950` |
| `TplBlockOffset` | `0x00198310` |
| Fim da tabela `cSmdWork` | `0x00001950` |
| Tamanho do banco BIN | `0x001969C0` |
| Primeiro offset do banco | `0x000001A0` |
| Slots na tabela de offsets | `104` |
| Modelos válidos | `101` |

## 14.1 Amostra de headers `cModelData`

| Model ID | Offset relativo | Tamanho | `HeaderSize` | `Format` | `VertexFrac` | `PartCount` | `PolyGroupCount` | `PartsOffset` | `PolyOffset` | `BoundingBoxOffset` | `JointMagic` |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `0` | `0x000001A0` | `0x003780` | `0x30` | `0x3A` | `2` | `1` | `1` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `1` | `0x00003920` | `0x000630` | `0x30` | `0x3A` | `2` | `1` | `1` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `2` | `0x00003F50` | `0x000630` | `0x30` | `0x3A` | `4` | `1` | `1` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `3` | `0x00004580` | `0x001340` | `0x30` | `0x3A` | `5` | `1` | `1` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `4` | `0x000058C0` | `0x002AA0` | `0x30` | `0x3A` | `9` | `1` | `1` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `5` | `0x00008360` | `0x000FB0` | `0x30` | `0x3A` | `6` | `1` | `1` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `19` | `0x00011520` | `0x007DF0` | `0x30` | `0x3A` | `6` | `1` | `4` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `20` | `0x00019310` | `0x004450` | `0x30` | `0x3A` | `6` | `1` | `4` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `50` | `0x000B1B40` | `0x004A00` | `0x30` | `0x3A` | `6` | `1` | `2` | `0x50` | `0x60` | `0x30` | `0x20010801` |
| `100` | `0x00194E40` | `0x001B80` | `0x30` | `0x3A` | `7` | `1` | `2` | `0x50` | `0x60` | `0x30` | `0x20010801` |

## 14.2 Amostra de `cSmdWork`

| Work | `BinNo` | `TplNo` | `MotNo` | `ObjNo` | `Flag` | Escala |
|---:|---:|---:|---:|---:|---:|---|
| `0` | `0` | `0` | `0xFF` | `0` | `0x00000008` | `100,100,100` |
| `1` | `19` | `0` | `0xFF` | `2` | `0x00000008` | `100,100,100` |
| `2` | `20` | `0` | `0xFF` | `3` | `0x00000008` | `100,100,100` |
| `3` | `21` | `0` | `0xFF` | `4` | `0x00000008` | `100,100,100` |
| `4` | `22` | `0` | `0xFF` | `5` | `0x00000008` | `100,100,100` |

Distribuição observada em `r402.dat`:

| Campo | Observação |
|---|---|
| `BinNo` | `0..100`, 101 valores únicos. |
| `TplNo` | Todos `0`. |
| `MotNo` | Todos `0xFF`. |
| `Flag` | Predominante `0x00000008`; também `0x00000009` e `0x00000000`. |

---

# 15. Fluxo runtime resumido

## 15.1 Carregamento por SMD

```text
SmdInit
 ├── recebe ponteiro para SMD
 ├── recebe ponteiro para SMX
 └── prepara tabelas de trabalho

SmdSetup
 ├── percorre cSmdWork
 ├── lê BinNo/TplNo/MotNo/ObjNo
 ├── resolve modelo com SmdGetBinPtr
 ├── resolve textura com SmdGetTplPtr
 ├── instancia cObj/cModel
 └── aplica SmdSetParam
```

## 15.2 Carregamento do modelo

```text
modelInit(cModel, ...)
 ├── init(cModelInfo, cModelData, TEXPalette)
 ├── setPartsOffset(cModelData)
 ├── setPartsParent()
 ├── setJointInfo(cModelData)
 ├── getBoundingBox(cModelData)
 └── render usa insert_shape / insert_normal
```

## 15.3 Renderização de shape

```text
insert_shape
 ├── percorre grupos de polígonos
 ├── lê Cmd de cada bloco primitivo
 ├── Cmd & 0xF8 == 0x80 -> quad
 ├── Cmd & 0xF8 == 0x90 -> triangle
 └── Cmd & 0xF8 == 0x98 -> triangle strip
```

---

# 16. Funções relevantes no ELF debug

| Função | Endereço | Uso |
|---|---:|---|
| `modelInit__6cModelUiUi` | `0x00255370` | Inicialização geral de `cModel`. |
| `init__10cModelInfoUiUi` | `0x00257230` | Interpreta `cModelData` e ponteiros internos. |
| `setPartsOffset__6cModelP10cModelData` | `0x00255588` | Lê tabela de partes. |
| `setPartsParent__6cModel` | `0x00255720` | Monta hierarquia local de partes. |
| `setJointInfo__6cModelP10cModelData` | `0x00257688` | Lê dados de joint quando presentes. |
| `getBoundingBox__FP10cModelDataP12cBoundingBox` | `0x00257AB0` | Lê bounding box apontado por `BoundingBoxOffset`. |
| `insert_shape__FP6cModelP10cModelInfofUiRC6tagVecT4` | `0x002869C8` | Despacha blocos primitivos. |
| `insert_shape_quad__FP6cModel...` | `0x00286D70` | Monta quads. |
| `insert_shape_triangle__FP6cModel...` | `0x00287F30` | Monta triângulos. |
| `insert_shape_tristrip__FP6cModel...` | `0x00288CF8` | Monta triangle strips. |
| `insert_normal__FP6cModelP10cModelInfofUiRC6tagVecT4` | `0x002892C0` | Processa normals. |
| `SmdSetParam__FP7cObjScrP8cSmdWork` | `0x002A5778` | Aplica posição/rotação/escala e índices da entrada SMD. |
| `SmdGetBinPtr__4cSmdi` / `SmdGetBinPtr__Fi` | `0x002A6190` / variação global | Resolve `BinNo` para ponteiro de modelo. |
| `SmdGetTplPtr__4cSmdi` / `SmdGetTplPtr__Fi` | `0x002A61B0` / `0x002A5F60` | Resolve `TplNo` para textura. |
| `SmdGetMotPtr__4cSmdi` | `0x002A61D0` | Resolve `MotNo` para motion. |
| `SetObjSmd__FUiUiRC6tagVecT2Ucb` | `0x002A64C8` | Criação de objeto baseada em SMD. |
| `ShapeSet__FP10cModelInfosUiUi` | `0x002A6A98` | Controle de shape. |
| `SetShape__FP10cModelInfofUi` | `0x002A6D00` | Atualização de shape/morph. |

---

# 17. Regras para editor/repacker

## 17.1 Roundtrip seguro

Um repacker seguro deve:

1. Preservar todos os campos desconhecidos.
2. Preservar offsets relativos ou recalculá-los de forma consistente.
3. Recalcular a tabela de offsets do banco BIN.
4. Recalcular `BoundingBoxOffset` se mover o bloco de bounding box.
5. Recalcular `PartsOffset` se mover a tabela de partes.
6. Recalcular `PolyOffset` se mover grupos de polígonos.
7. Manter `HeaderSize = 0x30` e `Format = 0x3A` para modelos comuns.
8. Validar que todos os índices de primitivo apontam para palettes válidas.
9. Validar `TexNo` contra a quantidade de texturas no TPL usado pela instância.
10. Manter alinhamento igual ou superior ao original.

## 17.2 Edição de geometria

Ao editar vértices:

- Converta `float` para `int16` usando `VertexFrac`.
- Se o valor estourar `int16`, escolha uma destas opções:
  - aumentar `VertexFrac` e reescalar todo o modelo;
  - dividir o modelo em partes;
  - manter o modelo em escala original e alterar a escala da instância `cSmdWork`.
- Atualize bounding box.
- Preserve `MatrixOrWeightId` se não houver editor de skinning.

## 17.3 Edição de UV

Ao editar UV:

- Preserve a escala fixa original.
- Não force clamp em `0..1`, pois UVs de jogo podem usar tiling.
- Atualize apenas `RE4_BIN_UPALETTE` se a topologia não mudar.
- Se remapear índices, atualize todos os `UVIndex` dos primitivos.

## 17.4 Edição de topologia

Ao criar/remover faces:

- Prefira exportar como triangle strip (`0x98`) para seguir o formato comum do jogo.
- Atualize `VertexCount` de cada bloco.
- Atualize `PrimBlockCount` do grupo.
- Garanta que o parser do jogo nunca leia além do chunk do modelo.
- Mantenha `Cmd & 0xF8` em `0x80`, `0x90` ou `0x98`.

## 17.5 Edição de partes

Ao adicionar partes:

- Aumente `PartCount`.
- Recrie `BIN_PART_ENTRY[]`.
- Atualize `PartsOffset`.
- Atualize `ParentId` para manter hierarquia válida.
- Se houver skinning, atualize também dados de joint/matrix.

---

# 18. Checklist de validação

Antes de salvar:

- [ ] O arquivo é little-endian.
- [ ] `HeaderSize == 0x30`.
- [ ] `Format == 0x3A`.
- [ ] Todos os offsets apontam para dentro do chunk.
- [ ] `PartsOffset + PartCount * 0x10` cabe no chunk.
- [ ] `BoundingBoxOffset + 0x20` cabe no chunk.
- [ ] `PolyOffset` cabe no chunk.
- [ ] `PolyGroupCount` é coerente com os grupos existentes.
- [ ] `PrimBlockCount` não faz o leitor passar do fim do modelo.
- [ ] `Cmd & 0xF8` é conhecido ou preservado.
- [ ] Todos os índices de vértice/UV são válidos.
- [ ] `TexNo` é válido para o TPL associado.
- [ ] `Min <= Max` no bounding box.
- [ ] Offsets do BIN bank foram recalculados se algum modelo mudou de tamanho.
- [ ] Padding e alinhamento foram preservados.

---

# 19. Layout recomendado para YAML/JSON editável

```yaml
smd:
  version: 0x40
  works:
    - id: 0
      pos: [0.0, 0.0, 0.0, 1.0]
      ang: [0.0, 0.0, 0.0, 0.0]
      scale: [100.0, 100.0, 100.0, 1.0]
      bin_no: 0
      tpl_no: 0
      mot_no: 255
      obj_no: 0
      flag: 0x00000008

bin_bank:
  models:
    - id: 0
      header:
        header_size: 0x30
        format: 0x3A
        vertex_frac: 2
        part_count: 1
        poly_group_count: 1
        joint_magic: 0x20010801
        render_param: 0xE0000000
      parts:
        - part_id: 0
          parent_id: 255
          offset: [0.0, 0.0, 0.0]
      bbox:
        min: [-1.0, -1.0, -1.0]
        max: [1.0, 1.0, 1.0]
      poly_groups:
        - tex_no: 0
          attrs: [0x04, 0x00, 0xFF, 0x01, 0xFF, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0xFF]
          primitives:
            - type: tristrip
              cmd: 0x98
              sub_attr: 0x00
              refs:
                - [0, 0, 0, 0]
                - [1, 1, 0, 1]
                - [2, 2, 0, 2]
```

---

# 20. Observações práticas para Blender/Noesis

## Eixos

Para visualização em Blender, um mapeamento prático costuma ser:

```text
Blender X = Game X
Blender Y = -Game Z
Blender Z = Game Y
```

A escala final depende do pipeline da ferramenta. Para modelos de sala, é comum trabalhar com divisão visual por `100.0` ou `1000.0`, mantendo os valores binários originais no export.

## Exportação recomendada

- Importar vertices como float aplicando `VertexFrac`.
- Guardar `VertexFrac` original em custom property.
- Guardar índices originais de palettes sempre que possível.
- Exportar com modo preserve-layout quando só mover vértices.
- Usar repack estrutural apenas quando alterar quantidade de vertices, UVs, partes ou primitivos.

---

# 21. Pontos que devem ser preservados

Os campos abaixo têm uso de runtime variável e devem ser preservados até que a ferramenta implemente suporte completo:

| Campo | Motivo |
|---|---|
| `VertexBlockOffset` | Pode apontar para palettes/blocos internos dependendo do modelo. |
| `VertexBlockInfo` | Informação auxiliar ou sentinel. |
| `RenderParam` | Usado por render/model info. |
| `ExtraOffset` | Uso dependente do asset. |
| `Attr0..AttrB` | Flags de render/material. |
| `SubAttr` dos primitivos | Pode alterar renderização ou interpretação do bloco. |
| `AuxIndex` em `RE4_BIN_VERTEX_REF` | Pode referenciar normal, cor, peso ou dado auxiliar. |
| `Reserved0C/1C` do bounding box | Não são `W` garantido; preservar. |

---

# 22. Assinaturas úteis para detecção

Um chunk `cModelData` comum de sala costuma iniciar com:

```text
30 00 3A 00 ?? ?? ?? ?? VV PP GG GG ...
```

Onde:

| Bytes | Campo |
|---|---|
| `30 00` | `HeaderSize = 0x0030` |
| `3A 00` | `Format = 0x003A` |
| `VV` | `VertexFrac` |
| `PP` | `PartCount` |
| `GG GG` | `PolyGroupCount` |

Exemplo observado:

```text
30 00 3A 00 50 00 00 00 02 01 01 00 60 00 00 00
CD CD CD CD CD CD CD CD 01 08 01 20 00 00 00 00
00 00 00 00 00 00 00 E0 30 00 00 00 00 00 00 00
```

---

# 23. Resumo das estruturas

| Estrutura | Tamanho | Função |
|---|---:|---|
| `RE4_SMD_HEADER` | `0x10` | Header do `.SMD` que contém o banco BIN. |
| `RE4_SMD_WORK` | `0x40` | Instância de objeto que referencia `BinNo`, `TplNo` e `MotNo`. |
| `RE4_BIN_BANK` | variável | Tabela de offsets + chunks `cModelData`. |
| `RE4_BIN_MODEL_DATA` | `0x30` | Header principal do modelo. |
| `RE4_BIN_PART_ENTRY` | `0x10` | Parte/bone local. |
| `RE4_BIN_BOUNDING_BOX` | `0x20` | Bounding box local. |
| `RE4_BIN_POLY_GROUP` | `0x10` + payload | Grupo de render/material. |
| `RE4_BIN_PRIM_HEADER` | `0x02` | Cabeçalho de primitivo. |
| `RE4_BIN_VERTEX_REF` | `0x08` | Índices para palettes. |
| `RE4_BIN_VPALETTE` | `0x08` | Vértice fixo `int16`. |
| `RE4_BIN_UPALETTE` | `0x04` | UV fixo `int16`. |
```
