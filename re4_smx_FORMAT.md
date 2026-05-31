# Resident Evil 4 PS2 — Formato `.SMX`

Documentação técnica do formato `.SMX` usado pelo sistema de modelos/objetos de cenário do Resident Evil 4 de PlayStation 2.

O `.SMX` trabalha junto com o bloco `.SMD`/`cSmd`. Enquanto o `.SMD` define os objetos instanciados, modelos, animações, posição, rotação e escala, o `.SMX` aplica parâmetros extras por objeto: fila de renderização, culling, máscara de luz, flags internas, cores de material/especular, deslocamento de textura e um bloco opaco preservado pelo runtime.

## Características gerais

| Propriedade | Valor |
|---|---:|
| Endianness | Little-endian |
| Assinatura ASCII | Não possui |
| Header | `0x10` bytes |
| Tamanho da entrada | `0x90` bytes |
| Struct principal | `cSmxData` |
| Struct de entrada | `cSmxWork` |
| Fórmula mínima de tamanho | `0x10 + nData * 0x90` |
| Padding final | Opcional, geralmente `0x00` ou `0xCD` para alinhamento |

O primeiro byte do arquivo é a versão do formato, não uma assinatura. Nos arquivos analisados a versão é sempre `0x10`.

## Layout geral

```text
SMX
├─ cSmxData header       0x10 bytes
├─ cSmxWork[0]           0x90 bytes
├─ cSmxWork[1]           0x90 bytes
├─ ...
└─ padding opcional
```

Offset de uma entrada:

```text
entry_offset = 0x10 + index * 0x90
```

Tamanho mínimo esperado:

```text
expected_size = 0x10 + header.nData * 0x90
```

Alguns arquivos originais possuem bytes extras no final. Esse padding não é lido como entrada e deve ser preservado ou refeito com alinhamento seguro.

## `cSmxData`

Estrutura do header do arquivo.

```c
struct cSmxData
{
    uint8_t  Version;   // 0x00
    uint8_t  nData;     // 0x01
    uint8_t  Dummy02;   // 0x02
    uint8_t  Dummy03;   // 0x03
    uint32_t Dummy10;   // 0x04
    uint32_t Dummy20;   // 0x08
    uint32_t Dummy30;   // 0x0C
};
```

### Campos do header

| Offset | Campo | Tipo | Tamanho | Descrição |
|---:|---|---|---:|---|
| `0x00` | `Version` | `uint8` | `0x01` | Versão do bloco SMX. Valor original confirmado: `0x10`. |
| `0x01` | `nData` | `uint8` | `0x01` | Quantidade de entradas `cSmxWork`. |
| `0x02` | `Dummy02` | `uint8` | `0x01` | Reservado. Valor original observado: `0x00`. |
| `0x03` | `Dummy03` | `uint8` | `0x01` | Reservado. Valor original observado: `0x00`. |
| `0x04` | `Dummy10` | `uint32` | `0x04` | Reservado. Valor original observado: `0x00000000`. |
| `0x08` | `Dummy20` | `uint32` | `0x04` | Reservado. Valor original observado: `0x00000000`. |
| `0x0C` | `Dummy30` | `uint32` | `0x04` | Reservado. Valor original observado: `0x00000000`. |

### Valores aceitos no header

| Campo | Valores aceitos | Recomendação para edição |
|---|---|---|
| `Version` | `0x00..0xFF`, porém os arquivos originais usam `0x10` | Manter `0x10`. |
| `nData` | `0x00..0xFF` | Deve bater com a quantidade real de entradas. |
| `Dummy02` | `0x00..0xFF` | Manter `0x00`. |
| `Dummy03` | `0x00..0xFF` | Manter `0x00`. |
| `Dummy10` | `uint32` | Manter `0x00000000`. |
| `Dummy20` | `uint32` | Manter `0x00000000`. |
| `Dummy30` | `uint32` | Manter `0x00000000`. |

## `cSmxWork`

Estrutura de cada entrada SMX.

```c
struct GXColor
{
    uint8_t r;
    uint8_t g;
    uint8_t b;
    uint8_t a;
};

struct cSmxWork
{
    uint8_t  ModelNo;        // 0x00
    uint8_t  Id;             // 0x01
    uint8_t  OtType;         // 0x02
    uint8_t  CullMode;       // 0x03
    uint32_t LitSelectMask;  // 0x04
    uint32_t Flag;           // 0x08
    GXColor  MaterialColor;  // 0x0C
    uint8_t  Free[116];      // 0x10
    GXColor  SpecularColor;  // 0x84
    float    TexU;           // 0x88
    float    TexV;           // 0x8C
};
```

Tamanho total: `0x90` bytes.

### Campos da entrada

| Offset | Campo | Tipo | Tamanho | Descrição |
|---:|---|---|---:|---|
| `0x00` | `ModelNo` | `uint8` | `0x01` | Chave de associação com o `WorkNo` do `.SMD`. O runtime procura a primeira entrada SMX cujo `ModelNo` seja igual ao `WorkNo` do objeto SMD. |
| `0x01` | `Id` | `uint8` | `0x01` | Identificador/modo interno aplicado ao objeto. Quando diferente de zero, o runtime marca o objeto com uma flag interna adicional. |
| `0x02` | `OtType` | `uint8` | `0x01` | Tipo/fila de renderização. Usa valores do enum `OT_TYPE`. |
| `0x03` | `CullMode` | `uint8` | `0x01` | Modo de culling. Usa valores compatíveis com `GXCullMode`. |
| `0x04` | `LitSelectMask` | `uint32` | `0x04` | Máscara de seleção de luz. Controla quais luzes/grupos de luz afetam o objeto. |
| `0x08` | `Flag` | `uint32` | `0x04` | Bitmask de flags SMX aplicada em `SmxSetFlag`. |
| `0x0C` | `MaterialColor` | `GXColor` | `0x04` | Cor principal do material, em ordem `R, G, B, A`. |
| `0x10` | `Free` | `uint8[116]` | `0x74` | Bloco opaco preservado pelo jogo. Parte dele é copiada para o objeto em runtime. Deve ser preservado byte a byte. |
| `0x84` | `SpecularColor` | `GXColor` | `0x04` | Cor especular, em ordem `R, G, B, A`. |
| `0x88` | `TexU` | `float32` | `0x04` | Deslocamento/ajuste de textura no eixo U. |
| `0x8C` | `TexV` | `float32` | `0x04` | Deslocamento/ajuste de textura no eixo V. |

## Associação com `.SMD`

O campo `ModelNo` é a chave que liga o `.SMX` ao objeto criado pelo `.SMD`.

Fluxo simplificado:

```text
SmdSetup()
└─ setObj()
   ├─ lê cSmdWork.WorkNo
   ├─ cria o objeto/modelo
   ├─ aplica posição, rotação, escala, BIN/TPL/MOT
   └─ smxInit(obj, WorkNo)
      └─ procura no SMX a primeira entrada onde cSmxWork.ModelNo == WorkNo
```

Consequências práticas:

- `ModelNo` precisa coincidir com o `WorkNo` do objeto no `.SMD`.
- Se não houver entrada SMX correspondente, o objeto usa os parâmetros padrão do modelo.
- Se houver mais de uma entrada com o mesmo `ModelNo`, apenas a primeira encontrada é aplicada.
- `ModelNo >= 0xFA` é tratado como inválido pelo runtime.
- No `.SMD`, `WorkNo = 0xFF` significa objeto não criado; `WorkNo = 0xFE` cria um objeto especial que não recebe SMX.

## Valores aceitos por campo

### `ModelNo`

| Valor | Significado | Uso recomendado |
|---:|---|---|
| `0x00..0xF9` | Índice válido de work/modelo. | Faixa segura. Deve corresponder ao `WorkNo` do `.SMD`. |
| `0xFA..0xFF` | Inválido para SMX. | Evitar. O runtime emite erro para `ModelNo >= 0xFA`. |

Valores observados nos arquivos analisados: `0x00..0x72`, `0xB4`, `0xC8..0xCC`, entre outros valores não contínuos.

### `Id`

| Valor | Significado | Uso recomendado |
|---:|---|---|
| `0x00` | Modo normal/padrão. | Valor mais comum e seguro. |
| `0x01..0x0E` | Modo interno avançado. | Usar apenas ao preservar dados originais. |
| `0x0F` | Modo relacionado a mirror model. O runtime emite a mensagem `smxInit() : mirror model used.` | Evitar, exceto em objetos de espelho. |
| `0x10..0xFF` | Reservado/avançado. | Preservar se existir em arquivo original. |

Valores observados nos exemplos: `0x00` e `0x02`.

Quando `Id != 0`, o runtime marca o objeto com uma flag interna (`cObj` recebe `0x20`).

### `OtType`

`OtType` define em qual fila/camada de renderização o objeto entra. O enum confirmado no ELF é `OT_TYPE`.

| Valor | Nome | Descrição prática |
|---:|---|---|
| `0x00` | `OT_TYPE_TEX_RENDER0` | Renderização para textura, estágio 0. |
| `0x01` | `OT_TYPE_TEX_RENDER1` | Renderização para textura, estágio 1. |
| `0x02` | `OT_TYPE_SHADOW_SETUP` | Setup de sombra. |
| `0x03` | `OT_TYPE_SUBSCRN_FAR` | Subscreen/camada distante. |
| `0x04` | `OT_TYPE_SCROLL` | Camada/objeto com scroll. |
| `0x05` | `OT_TYPE_SUBSCRN` | Subscreen/camada intermediária. |
| `0x06` | `OT_TYPE_MODEL` | Modelo comum. |
| `0x07` | `OT_TYPE_SHADOW_DRAW` | Desenho de sombra. |
| `0x08` | `OT_TYPE_SUBSCRN_NEAR` | Subscreen/camada próxima. |
| `0x09` | `OT_TYPE_EFFECT` | Efeito. |
| `0x0A` | `OT_TYPE_WORLD` | Mundo/cenário. |
| `0x0B` | `OT_TYPE_EFFECT_VU1` | Efeito processado via VU1. |
| `0x0C` | `OT_TYPE_SCREEN` | Elemento de tela. |
| `0x0D` | `OT_TYPE_COCKPIT` | HUD/cockpit. |
| `0x0E` | `OT_TYPE_ID_MODEL` | Modelo com renderização por ID. |
| `0x0F` | `OT_TYPE_MESSAGE` | Mensagem/interface. |
| `0x10` | `OT_TYPE_AFTER_RENDER` | Pós-render. |
| `0x11` | `OT_TYPE_DEBUG` | Debug. |
| `0x12` | `OT_TYPE_MAX` | Sentinela, não usar como valor de objeto. |

Valores observados nos `.SMX` analisados:

| Valor | Ocorrências |
|---:|---:|
| `0x01` | `1` |
| `0x02` | `32` |
| `0x03` | `148` |
| `0x04` | `33` |
| `0x05` | `52` |

Para edição segura, preservar o valor original. Para novos objetos, usar valores já presentes no mesmo room.

### `CullMode`

O campo usa valores compatíveis com `_GXCullMode`.

| Valor | Nome | Efeito |
|---:|---|---|
| `0x00` | `GX_CULL_NONE` | Sem descarte por face. Renderiza frente e verso. |
| `0x01` | `GX_CULL_FRONT` | Descarta faces frontais. |
| `0x02` | `GX_CULL_BACK` | Descarta faces traseiras. |
| `0x03` | `GX_CULL_ALL` | Descarta tudo. Útil apenas para casos especiais/debug. |

Valores observados:

| Valor | Ocorrências |
|---:|---:|
| `0x00` | `199` |
| `0x02` | `67` |

Recomendação:

- `0x00`: usar para objetos que precisam aparecer dos dois lados.
- `0x02`: usar para modelos comuns fechados.
- `0x01` e `0x03`: preservar apenas se aparecerem em arquivos originais.

### `LitSelectMask`

Máscara de luz usada pelo objeto.

| Valor | Significado prático |
|---:|---|
| `0xFFFFFFFF` | Todas as luzes/grupos disponíveis. É o valor mais comum nos exemplos. |
| `0x00000000` | Nenhuma luz selecionada. Não observado nos exemplos; pode deixar o objeto sem contribuição de luz dependendo do renderer. |
| Qualquer `uint32` | Máscara customizada de seleção de luz. |

Valores observados nos arquivos analisados:

```text
0x0000E0FF
0x0000F0FF
0x11FEFFFF
0x11FFFFFF
0x1BFFFFFF
0xEFFFFFFF
0xFFFF7011
0xFFFF7111
0xFFFF7811
0xFFFF8000
0xFFFF8040
0xFFFF8200
0xFFFF82C0
0xFFFF8300
0xFFFF8400
0xFFFF9400
0xFFFFA4C0
0xFFFFC001
0xFFFFC01E
0xFFFFC03F
0xFFFFC050
0xFFFFC200
0xFFFFC4C0
0xFFFFC811
0xFFFFD400
0xFFFFF011
0xFFFFF031
0xFFFFF03F
0xFFFFF040
0xFFFFF051
0xFFFFF081
0xFFFFF090
0xFFFFF0C0
0xFFFFF200
0xFFFFF400
0xFFFFF801
0xFFFFF810
0xFFFFF811
0xFFFFF840
0xFFFFFA00
0xFFFFFA11
0xFFFFFC0E
0xFFFFFC1E
0xFFFFFC1F
0xFFFFFC3F
0xFFFFFEEB
0xFFFFFFEE
0xFFFFFFF7
0xFFFFFFFE
0xFFFFFFFF
```

Recomendação para editor:

- Exibir como hexadecimal de 8 dígitos.
- Permitir edição por bitmask.
- Preservar o valor original por padrão.
- Para criação de objeto novo, copiar o valor de um objeto visualmente parecido no mesmo room.

### `Flag`

`Flag` é uma bitmask de 32 bits. A função `SmxSetFlag` usa os bits baixos abaixo.

| Bit | Máscara | Ação observada em runtime | Uso recomendado |
|---:|---:|---|---|
| `0` | `0x00000001` | Seta flag interna `0x00000010` no objeto. | Preservar se existir. |
| `1` | `0x00000002` | Aparece no retorno de `SmxGetFlag`, mas não é escrito diretamente por `SmxSetFlag`. Relacionado ao estado do `cModelInfo`. | Preservar. |
| `2` | `0x00000004` | Seta flag interna `0x02000000`; quando ausente, essa flag é limpa. | Usar com cuidado. |
| `3` | `0x00000008` | Define byte interno do objeto como `0x80`. | Preservar se existir. |
| `4` | `0x00000010` | Seta flag interna `0x00008000`. | Preservar se existir. |
| `5` | `0x00000020` | Ativa bit baixo em um byte de estado interno do objeto. | Usado em vários objetos originais. |
| `6` | `0x00000040` | Define outro byte de estado interno como `1`. | Preservar se existir. |
| `7` | `0x00000080` | Define tipo/estado interno do objeto como `5`. | Raro; preservar se existir. |
| `8..31` | `0xFFFFFF00` | Não tratados por `SmxSetFlag` no trecho analisado. | Manter `0` salvo caso exista dado original confirmado. |

Combinações observadas:

| Valor | Bits ativos | Ocorrências |
|---:|---|---:|
| `0x00000000` | nenhum | `94` |
| `0x00000002` | `1` | `59` |
| `0x00000008` | `3` | `9` |
| `0x0000000A` | `1,3` | `49` |
| `0x00000020` | `5` | `18` |
| `0x00000022` | `1,5` | `3` |
| `0x00000028` | `3,5` | `17` |
| `0x0000002A` | `1,3,5` | `12` |
| `0x00000042` | `1,6` | `3` |
| `0x00000062` | `1,5,6` | `1` |
| `0x00000080` | `7` | `1` |

Para edição segura, usar apenas combinações já existentes no arquivo do mesmo room.

### `MaterialColor`

Formato: `GXColor`, quatro bytes em ordem direta:

```text
R G B A
```

| Componente | Tipo | Faixa |
|---|---|---|
| `r` | `uint8` | `0..255` |
| `g` | `uint8` | `0..255` |
| `b` | `uint8` | `0..255` |
| `a` | `uint8` | `0..255` |

Comportamento em runtime:

- A cor é copiada para `cModelInfo.MaterialColor`.
- Se `R/G/B` forem todos `0`, o runtime substitui a cor por branco (`0xFF,0xFF,0xFF`) para evitar material completamente preto por padrão.
- O byte `A` é usado como parâmetro separado pelo runtime e a alpha efetiva da cor principal é normalizada para `0xFF`.

Exemplos em arquivo:

| Bytes | Interpretação |
|---|---|
| `FF FF FF 00` | Branco, parâmetro alpha/extra `0x00`. |
| `46 46 46 03` | Cinza escuro, parâmetro `0x03`. |
| `F5 F5 F5 03` | Quase branco, parâmetro `0x03`. |

Recomendação:

- Editar como RGBA de 8 bits.
- Evitar `00 00 00 xx` caso a intenção seja preto real, porque o runtime pode substituir por branco.
- Para escurecer, usar valores baixos não zerados, por exemplo `01 01 01 00` ou valores compatíveis com o material original.

### `Free[116]`

Bloco opaco de `0x74` bytes.

| Faixa | Tamanho | Tratamento |
|---:|---:|---|
| `0x10..0x83` | `0x74` | Copiado/preservado como dados internos do objeto. |

Nos arquivos analisados, a maior parte das entradas usa `0x00` em todo esse bloco. Algumas entradas possuem dados não nulos em subfaixas específicas. Como a estrutura interna não possui nomes no debug, esse campo deve ser tratado como raw bytes.

Regras para ferramentas:

- Preservar byte a byte no roundtrip.
- Não limpar automaticamente.
- Permitir edição hexadecimal avançada.
- Para novas entradas, iniciar com `0x00` e copiar de um objeto semelhante quando necessário.

### `SpecularColor`

Formato: `GXColor`, quatro bytes em ordem direta:

```text
R G B A
```

| Componente | Tipo | Faixa |
|---|---|---|
| `r` | `uint8` | `0..255` |
| `g` | `uint8` | `0..255` |
| `b` | `uint8` | `0..255` |
| `a` | `uint8` | `0..255` |

Valores observados:

| Bytes | Ocorrências |
|---|---:|
| `00 00 00 00` | `264` |
| `0A 0A 0A 00` | `1` |
| `91 91 91 00` | `1` |

Comportamento em runtime:

- A cor é copiada para `cModelInfo.SpecularColor`.
- O canal alpha especular pode ser normalizado internamente conforme os canais da cor.
- `00 00 00 00` equivale a sem especular customizado.

### `TexU` e `TexV`

Valores `float32` little-endian usados como ajuste/deslocamento de textura.

| Campo | Tipo | Faixa binária | Recomendação |
|---|---|---|---|
| `TexU` | `float32` | Qualquer float finito | Usar valores pequenos. Evitar `NaN`/`Inf`. |
| `TexV` | `float32` | Qualquer float finito | Usar valores pequenos. Evitar `NaN`/`Inf`. |

Comportamento em runtime:

- Os valores são copiados para o bloco de animação/offset de textura do `cModelInfo`.
- Se `TexU > 2.0`, o runtime substitui `TexU` por `0.0`.
- Se `TexV > 2.0`, o runtime substitui `TexV` por `0.0`.
- Se `TexU` ou `TexV` for diferente de zero, o runtime ativa um bit de estado relacionado a ajuste de textura.

Valores observados nos arquivos analisados são muito pequenos:

```text
TexU = -0.0012149999, TexV =  0.0
TexU = -0.0012119958, TexV =  0.0000009996
TexU = -0.0009400007, TexV =  0.0
TexU = -0.0002770014, TexV = -0.0000329999
```

Recomendação:

- Manter `0.0` quando o objeto não usa ajuste de textura.
- Para ajustes manuais, trabalhar com valores pequenos, normalmente entre `-0.01` e `0.01`.
- Não usar valores maiores que `2.0`, pois serão descartados pelo runtime.

## Tabela de funções relevantes

| Função | Endereço | Papel |
|---|---:|---|
| `SmdInit(cSmd*, cSmxData*, cSmxData*)` | `0x002A5330` | Inicializa ponteiros globais de SMD/SMX e aloca work interno. |
| `SmdSetup(int)` | `0x002A5528` | Valida SMD e chama a criação de objetos. |
| `setObj(int)` | `0x002A5588` | Itera os works do `.SMD`, cria objetos e aplica SMX. |
| `SmdSetParam(cObjScr*, cSmdWork*)` | `0x002A5778` | Inicializa modelo, textura, motion, transform e parâmetros base. |
| `SmxSetFlag(cObj*, uint32)` | `0x002A5AE8` | Aplica os bits do campo `Flag`. |
| `SmxGetFlag(cObj*)` | `0x002A5B98` | Reconstrói/consulta a bitmask SMX a partir do estado runtime. |
| `smxInit(cObjScr*, int)` | `0x002A5C20` | Procura entrada SMX por `ModelNo`. |
| `smxInit(cObjScr*, cSmxWork*)` | `0x002A5C88` | Aplica uma entrada `cSmxWork` ao objeto. |
| `getWorkPtr(cSmd*, int)` | `0x002A6150` | Acessa work do SMD. |
| `getBinPtr(cSmd*, int)` | `0x002A6190` | Resolve ponteiro de BIN/modelo. |
| `getTplPtr(cSmd*, int)` | `0x002A61B0` | Resolve ponteiro de TPL/textura. |
| `getMotPtr(cSmd*, int)` | `0x002A61D0` | Resolve ponteiro de motion. |
| `getWorkNum(cSmd*)` | `0x002A61F0` | Retorna quantidade total de works do SMD. |

## Arquivos analisados

| Arquivo | Tamanho | `Version` | `nData` | Tamanho mínimo calculado | Padding final |
|---|---:|---:|---:|---:|---:|
| `r100_005.SMX` | `13984` | `0x10` | `97` | `13984` | `0` |
| `r106_005.SMX` | `13568` | `0x10` | `94` | `13552` | `16` bytes `0xCD` |
| `r10d_005.SMX` | `2752` | `0x10` | `19` | `2752` | `0` |
| `r120_006.SMX` | `8096` | `0x10` | `56` | `8080` | `16` bytes `0xCD` |

## Validação recomendada

Uma ferramenta de edição/repack deve validar:

1. O arquivo tem pelo menos `0x10` bytes.
2. `Version == 0x10` para arquivos originais conhecidos.
3. `file_size >= 0x10 + nData * 0x90`.
4. Se houver bytes extras, tratá-los como padding.
5. Cada `ModelNo` está em `0x00..0xF9`.
6. `OtType != OT_TYPE_MAX`.
7. `CullMode <= 3`.
8. `TexU` e `TexV` são floats finitos.
9. `Free[116]` é preservado exatamente.
10. Ordem das entradas é mantida, principalmente quando houver `ModelNo` duplicado.

## YAML recomendado para edição

```yaml
version: 0x10
reserved:
  dummy02: 0x00
  dummy03: 0x00
  dummy10: 0x00000000
  dummy20: 0x00000000
  dummy30: 0x00000000
padding: "cdcdcdcdcdcdcdcdcdcdcdcdcdcdcdcd"
entries:
  - index: 0
    model_no: 0x00
    id: 0x00
    ot_type: 0x05
    ot_type_name: OT_TYPE_SUBSCRN
    cull_mode: 0x00
    cull_mode_name: GX_CULL_NONE
    lit_select_mask: 0x1BFFFFFF
    flag: 0x00000020
    material_color:
      r: 0x46
      g: 0x46
      b: 0x46
      a: 0x03
    free: "00000000000000000000000000000000..."
    specular_color:
      r: 0x00
      g: 0x00
      b: 0x00
      a: 0x00
    tex_u: 0.0
    tex_v: 0.0
```

Regras do YAML:

- `free` deve conter exatamente `116` bytes quando convertido de hexadecimal.
- `padding` deve ser preservado quando existir.
- Valores hexadecimais devem ser exportados com largura fixa.
- O repack deve reconstruir as entradas em ordem.

## Parser Python de referência

```python
from __future__ import annotations

import math
import struct
from dataclasses import dataclass
from pathlib import Path

HEADER_SIZE = 0x10
ENTRY_SIZE = 0x90

OT_TYPE = {
    0x00: "OT_TYPE_TEX_RENDER0",
    0x01: "OT_TYPE_TEX_RENDER1",
    0x02: "OT_TYPE_SHADOW_SETUP",
    0x03: "OT_TYPE_SUBSCRN_FAR",
    0x04: "OT_TYPE_SCROLL",
    0x05: "OT_TYPE_SUBSCRN",
    0x06: "OT_TYPE_MODEL",
    0x07: "OT_TYPE_SHADOW_DRAW",
    0x08: "OT_TYPE_SUBSCRN_NEAR",
    0x09: "OT_TYPE_EFFECT",
    0x0A: "OT_TYPE_WORLD",
    0x0B: "OT_TYPE_EFFECT_VU1",
    0x0C: "OT_TYPE_SCREEN",
    0x0D: "OT_TYPE_COCKPIT",
    0x0E: "OT_TYPE_ID_MODEL",
    0x0F: "OT_TYPE_MESSAGE",
    0x10: "OT_TYPE_AFTER_RENDER",
    0x11: "OT_TYPE_DEBUG",
    0x12: "OT_TYPE_MAX",
}

CULL_MODE = {
    0x00: "GX_CULL_NONE",
    0x01: "GX_CULL_FRONT",
    0x02: "GX_CULL_BACK",
    0x03: "GX_CULL_ALL",
}

@dataclass
class GXColor:
    r: int
    g: int
    b: int
    a: int

    @classmethod
    def read(cls, data: bytes, offset: int) -> "GXColor":
        return cls(*struct.unpack_from("<BBBB", data, offset))

@dataclass
class SmxEntry:
    index: int
    offset: int
    model_no: int
    id: int
    ot_type: int
    cull_mode: int
    lit_select_mask: int
    flag: int
    material_color: GXColor
    free: bytes
    specular_color: GXColor
    tex_u: float
    tex_v: float

@dataclass
class SmxFile:
    version: int
    n_data: int
    dummy02: int
    dummy03: int
    dummy10: int
    dummy20: int
    dummy30: int
    entries: list[SmxEntry]
    padding: bytes


def parse_smx(path: str | Path) -> SmxFile:
    data = Path(path).read_bytes()

    if len(data) < HEADER_SIZE:
        raise ValueError("arquivo menor que o header SMX")

    version, n_data, dummy02, dummy03, dummy10, dummy20, dummy30 = struct.unpack_from(
        "<BBBBIII", data, 0
    )

    expected_size = HEADER_SIZE + n_data * ENTRY_SIZE
    if len(data) < expected_size:
        raise ValueError(
            f"arquivo truncado: tamanho={len(data)}, esperado mínimo={expected_size}"
        )

    entries: list[SmxEntry] = []

    for index in range(n_data):
        off = HEADER_SIZE + index * ENTRY_SIZE

        model_no, smx_id, ot_type, cull_mode = struct.unpack_from("<BBBB", data, off)
        lit_select_mask, flag = struct.unpack_from("<II", data, off + 0x04)
        material_color = GXColor.read(data, off + 0x0C)
        free = data[off + 0x10 : off + 0x84]
        specular_color = GXColor.read(data, off + 0x84)
        tex_u, tex_v = struct.unpack_from("<ff", data, off + 0x88)

        if model_no >= 0xFA:
            raise ValueError(f"entrada {index}: ModelNo inválido 0x{model_no:02X}")

        if ot_type == 0x12:
            raise ValueError(f"entrada {index}: OT_TYPE_MAX não deve ser usado em objeto")

        if ot_type not in OT_TYPE:
            raise ValueError(f"entrada {index}: OtType desconhecido 0x{ot_type:02X}")

        if cull_mode not in CULL_MODE:
            raise ValueError(f"entrada {index}: CullMode desconhecido 0x{cull_mode:02X}")

        if not math.isfinite(tex_u) or not math.isfinite(tex_v):
            raise ValueError(f"entrada {index}: TexU/TexV inválido")

        entries.append(
            SmxEntry(
                index=index,
                offset=off,
                model_no=model_no,
                id=smx_id,
                ot_type=ot_type,
                cull_mode=cull_mode,
                lit_select_mask=lit_select_mask,
                flag=flag,
                material_color=material_color,
                free=free,
                specular_color=specular_color,
                tex_u=tex_u,
                tex_v=tex_v,
            )
        )

    padding = data[expected_size:]

    return SmxFile(
        version=version,
        n_data=n_data,
        dummy02=dummy02,
        dummy03=dummy03,
        dummy10=dummy10,
        dummy20=dummy20,
        dummy30=dummy30,
        entries=entries,
        padding=padding,
    )


def print_summary(path: str | Path) -> None:
    smx = parse_smx(path)
    print(f"Version: 0x{smx.version:02X}")
    print(f"Entries: {smx.n_data}")
    print(f"Padding: {len(smx.padding)} bytes")

    for e in smx.entries:
        print(
            f"[{e.index:03}] "
            f"off=0x{e.offset:06X} "
            f"model_no=0x{e.model_no:02X} "
            f"id=0x{e.id:02X} "
            f"ot={OT_TYPE[e.ot_type]} "
            f"cull={CULL_MODE[e.cull_mode]} "
            f"lit=0x{e.lit_select_mask:08X} "
            f"flag=0x{e.flag:08X} "
            f"mat=({e.material_color.r:02X},{e.material_color.g:02X},{e.material_color.b:02X},{e.material_color.a:02X}) "
            f"spec=({e.specular_color.r:02X},{e.specular_color.g:02X},{e.specular_color.b:02X},{e.specular_color.a:02X}) "
            f"tex=({e.tex_u:.8f},{e.tex_v:.8f})"
        )


if __name__ == "__main__":
    import sys

    for filename in sys.argv[1:]:
        print("=", filename)
        print_summary(filename)
```

## Regras de repack seguro

Para reconstruir um `.SMX` sem quebrar compatibilidade:

1. Escrever o header em little-endian.
2. Atualizar `nData` com a quantidade real de entradas.
3. Escrever cada entrada com exatamente `0x90` bytes.
4. Preservar `Free[116]` exatamente.
5. Preservar ou recriar padding final de forma consistente.
6. Não ordenar automaticamente por `ModelNo`; manter a ordem original.
7. Não criar entradas com `ModelNo >= 0xFA`.
8. Não usar `OT_TYPE_MAX` (`0x12`) como `OtType` de objeto.
9. Não usar `CullMode > 3`.
10. Evitar floats `NaN`, `Inf` ou valores de textura maiores que `2.0`.

## Checklist para editor

- [ ] Mostrar lista de entradas com `index`, `ModelNo`, `Id`, `OtType`, `CullMode`, `Flag` e `LitSelectMask`.
- [ ] Exibir `OtType` com nome do enum.
- [ ] Exibir `CullMode` com nome do enum.
- [ ] Editor de bitmask para `Flag`.
- [ ] Editor hexadecimal para `LitSelectMask`.
- [ ] Editor RGBA para `MaterialColor` e `SpecularColor`.
- [ ] Editor float para `TexU` e `TexV` com validação de `NaN`/`Inf`.
- [ ] Aba avançada para `Free[116]` em hexadecimal.
- [ ] Preservar padding final.
- [ ] Validar tamanho mínimo antes de salvar.
- [ ] Avisar quando houver `ModelNo` duplicado.
- [ ] Avisar quando `Id == 0x0F` por envolver mirror model.

## Resumo prático

O `.SMX` é uma tabela simples de modificadores de objeto. O arquivo não tem assinatura, usa header fixo de `0x10` bytes e entradas fixas de `0x90` bytes. O campo mais importante para associação é `ModelNo`, que deve bater com o `WorkNo` do `.SMD`. Os campos mais importantes para edição visual são `OtType`, `CullMode`, `LitSelectMask`, `Flag`, `MaterialColor`, `SpecularColor`, `TexU` e `TexV`.

Para edição segura, a regra principal é preservar tudo que não for alterado explicitamente, especialmente `Free[116]`, ordem das entradas, flags e padding final.
