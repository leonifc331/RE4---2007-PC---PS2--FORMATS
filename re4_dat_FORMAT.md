# Resident Evil 4 PS2 — Formato `.DAT`

Documentação técnica do container `.DAT` usado por **Resident Evil 4** de PlayStation 2 para agrupar os arquivos de dados de uma sala ou pacote lógico em um único bloco carregável.

O `.DAT` não é um formato de dados único. Ele é um **container indexado por tags de três caracteres**, onde cada entrada aponta para um subarquivo interno: câmera, luzes, modelos, colisão, triggers, itens, efeitos, animações e outros blocos específicos do jogo.

## Sumário

- [1. Características gerais](#1-características-gerais)
- [2. Estrutura global](#2-estrutura-global)
- [3. Header](#3-header)
- [4. Tabela de offsets](#4-tabela-de-offsets)
- [5. Tabela de tags](#5-tabela-de-tags)
- [6. Blocos de dados](#6-blocos-de-dados)
- [7. Algoritmo `GetDataExt`](#7-algoritmo-getdataext)
- [8. Valores aceitos por campo](#8-valores-aceitos-por-campo)
- [9. Tags conhecidas](#9-tags-conhecidas)
- [10. Amostra `r402.dat`](#10-amostra-r402dat)
- [11. Regras para extração](#11-regras-para-extração)
- [12. Regras para repack](#12-regras-para-repack)
- [13. Manifesto JSON/YAML recomendado](#13-manifesto-jsonyaml-recomendado)
- [14. Structs de referência](#14-structs-de-referência)
- [15. Parser/extractor de referência em Python](#15-parserextractor-de-referência-em-python)
- [16. Repacker de referência em Python](#16-repacker-de-referência-em-python)
- [17. Funções e símbolos relevantes](#17-funções-e-símbolos-relevantes)
- [18. Checklist de roundtrip](#18-checklist-de-roundtrip)
- [19. Resumo de offsets](#19-resumo-de-offsets)

---

## 1. Características gerais

| Propriedade | Valor |
|---|---:|
| Assinatura | Não possui |
| Endianness | little-endian |
| Header fixo | `0x10` bytes |
| Tabela de offsets | `FileCount * 0x04` bytes |
| Tabela de tags | `FileCount * 0x04` bytes |
| Tag interna | 3 caracteres ASCII + `0x00` |
| Tamanho de entrada | Não armazenado diretamente |
| Cálculo de tamanho | `NextOffset - CurrentOffset` |
| Alinhamento observado | `0x20` bytes |
| Padding do container | `0x00` |
| Duplicatas de tag | Permitidas |
| Seleção de duplicata | Por ordinal zero-based |
| Compressão no container | Não possui |
| Plataforma analisada | PlayStation 2 |

O container possui três regiões principais:

```text
Arquivo .DAT
├─ Header fixo                    0x10 bytes
├─ Tabela de offsets              FileCount * 0x04
├─ Tabela de tags                 FileCount * 0x04
├─ Padding até alinhamento        geralmente 0x20
└─ Blocos internos                dados reais dos subarquivos
```

O runtime não usa nomes completos de arquivo dentro do `.DAT`. A busca é feita por tag interna, por exemplo `CAM`, `LIT`, `SMD`, `SMX`, `AEV`, `ITA` e `FCV`.

---

## 2. Estrutura global

```text
0x00000000  DAT_HEADER
0x00000010  uint32 DataOffset[FileCount]
...         char   DataTag[FileCount][4]
...         Padding até align 0x20
...         DataBlock[0]
...         DataBlock[1]
...         DataBlock[2]
...         ...
```

O tamanho mínimo da região de índice é:

```text
IndexSize = 0x10 + FileCount * 0x04 + FileCount * 0x04
```

O primeiro bloco normalmente começa em:

```text
FirstDataOffset = align(IndexSize, 0x20)
```

No arquivo `r402.dat`:

```text
FileCount       = 0x25
IndexSize       = 0x138
FirstDataOffset = 0x140
```

---

## 3. Header

| Offset | Tamanho | Tipo | Nome | Descrição |
|---:|---:|---|---|---|
| `0x00` | `0x04` | `uint32` | `FileCount` | Quantidade de blocos internos. |
| `0x04` | `0x04` | `uint32` | `Reserved0` | Reservado. Observado como `0`. |
| `0x08` | `0x04` | `uint32` | `Reserved1` | Reservado. Observado como `0`. |
| `0x0C` | `0x04` | `uint32` | `Reserved2` | Reservado. Observado como `0`. |

### Struct

```c
typedef struct RE4_DAT_HEADER {
    uint32_t FileCount;
    uint32_t Reserved0;
    uint32_t Reserved1;
    uint32_t Reserved2;
} RE4_DAT_HEADER;
```

### Observações

- O arquivo não possui magic ASCII.
- `FileCount` controla o tamanho das duas tabelas seguintes.
- Os campos reservados não são usados por `GetDataExt`, mas devem ser preservados ou escritos como `0`.
- Um `FileCount` inválido faz o runtime ler fora do índice e pode causar crash.

---

## 4. Tabela de offsets

A tabela de offsets começa imediatamente após o header fixo:

```text
OffsetTable = Base + 0x10
```

Cada item possui `0x04` bytes:

| Offset local | Tamanho | Tipo | Nome | Descrição |
|---:|---:|---|---|---|
| `0x00` | `0x04` | `uint32` | `DataOffset` | Offset absoluto do bloco, relativo ao início do `.DAT`. |

### Struct lógica

```c
typedef struct RE4_DAT_OFFSET_ENTRY {
    uint32_t DataOffset;
} RE4_DAT_OFFSET_ENTRY;
```

### Regras

- Os offsets são relativos ao início do `.DAT`, não ao fim do header.
- Devem apontar para dentro do arquivo.
- Devem ser crescentes.
- O primeiro offset deve ser maior ou igual a `align(0x10 + FileCount * 8, 0x20)`.
- O alinhamento seguro observado é `0x20`.
- O tamanho de um bloco é calculado pelo próximo offset.
- Para o último bloco, o tamanho é calculado por `FileSize - LastOffset`.

---

## 5. Tabela de tags

A tabela de tags começa depois da tabela de offsets:

```text
TagTable = Base + 0x10 + FileCount * 0x04
```

Cada tag possui `0x04` bytes:

| Offset local | Tamanho | Tipo | Nome | Descrição |
|---:|---:|---|---|---|
| `0x00` | `0x03` | `char[3]` | `Ext` | Identificador ASCII usado pela busca runtime. |
| `0x03` | `0x01` | `uint8` | `Terminator` | Normalmente `0x00`. |

### Struct

```c
typedef struct RE4_DAT_TAG_ENTRY {
    char Ext[3];
    uint8_t Terminator;
} RE4_DAT_TAG_ENTRY;
```

### Observações

- O runtime compara apenas os três primeiros bytes da tag.
- O quarto byte deve ser mantido como `0x00` para compatibilidade e legibilidade.
- Tags duplicadas são válidas.
- A seleção de uma tag duplicada é feita pelo ordinal da ocorrência.

Exemplo:

```text
LIT ordinal 0 -> primeira tabela de luz
LIT ordinal 1 -> segunda tabela de luz
FCV ordinal 0 -> primeira curva/animação FCV
FCV ordinal 1 -> segunda curva/animação FCV
...
```

---

## 6. Blocos de dados

Cada bloco é um subarquivo completo. O `.DAT` apenas armazena o bloco e permite encontrá-lo pela tag.

```text
DataBlock[i].Offset = DataOffset[i]
DataBlock[i].Size   = DataOffset[i + 1] - DataOffset[i]
```

Para o último bloco:

```text
DataBlock[last].Size = FileSize - DataOffset[last]
```

O container não armazena checksum, compressão nem tabela de tamanhos.

### Padding

O padding entre o fim da tabela de tags e o primeiro bloco usa `0x00`. Os blocos analisados começam alinhados a `0x20`.

Ao fazer repack, recomenda-se:

```text
1. Recriar o header.
2. Recriar a tabela de offsets.
3. Recriar a tabela de tags.
4. Preencher com 0x00 até align 0x20.
5. Escrever cada bloco.
6. Preencher com 0x00 até align 0x20 antes do próximo bloco.
```

---

## 7. Algoritmo `GetDataExt`

O símbolo `GetDataExt__FPUiPcUi` resolve um bloco interno do `.DAT` a partir da tag de três caracteres e do ordinal.

Assinatura lógica:

```c
uint8_t* GetDataExt(uint32_t* pDat, const char* ext, uint32_t ordinal);
```

Comportamento:

```c
uint8_t* RE4_GetDataExt(uint8_t* base, const char ext[3], uint32_t ordinal) {
    if (base == NULL || ext == NULL) {
        return NULL;
    }

    uint32_t count = *(uint32_t*)(base + 0x00);
    uint32_t* offsets = (uint32_t*)(base + 0x10);
    uint8_t* tags = base + 0x10 + count * 0x04;

    uint32_t match_count = 0;

    for (uint32_t i = 0; i < count; i++) {
        uint8_t* tag = tags + i * 0x04;

        if (tag[0] == ext[0] && tag[1] == ext[1] && tag[2] == ext[2]) {
            if (match_count == ordinal) {
                return base + offsets[i];
            }
            match_count++;
        }
    }

    return NULL;
}
```

Pontos importantes:

- O ordinal é **zero-based**.
- Apenas os três primeiros caracteres da tag são comparados.
- O quarto byte da tag não participa da comparação.
- A função retorna ponteiro direto para o bloco interno.
- A função não retorna o tamanho do bloco.

---

## 8. Valores aceitos por campo

### `FileCount`

| Valor | Interpretação | Uso seguro |
|---:|---|---|
| `0` | Container vazio | Evitar em arquivos de sala. |
| `1..255` | Faixa segura para editor/repacker | Recomendado. |
| `>255` | Tecnicamente possível como `uint32`, mas arriscado | Evitar sem validação completa. |

Regras:

- Deve bater com a quantidade de offsets.
- Deve bater com a quantidade de tags.
- Deve permitir que `0x10 + FileCount * 8` caiba antes do primeiro bloco.

### `Reserved0`, `Reserved1`, `Reserved2`

| Valor | Interpretação | Uso seguro |
|---:|---|---|
| `0` | Valor observado e recomendado | Sim |
| Diferente de `0` | Não usado por `GetDataExt`; possível lixo/preservação | Preservar se existir em arquivo original |

Para arquivos novos, escreva `0`.

### `DataOffset`

| Valor | Interpretação | Uso seguro |
|---:|---|---|
| `< FirstDataOffset` | Offset inválido ou aponta para tabela | Não |
| `>= FirstDataOffset` e `< FileSize` | Offset válido | Sim |
| `FileSize` | Só seria aceitável para bloco vazio no final | Evitar |
| `> FileSize` | Inválido | Não |

Regras obrigatórias:

```text
DataOffset[i] < DataOffset[i + 1]
DataOffset[i] % 0x20 == 0    ; recomendado/observado
```

### `Ext[3]`

| Valor | Interpretação | Uso seguro |
|---|---|---|
| `A-Z`, `0-9`, `_` | Identificador normal | Sim |
| Letras minúsculas | Tecnicamente comparável, mas não observado | Evitar |
| Bytes `0x00` dentro dos três primeiros caracteres | Tag truncada/inválida | Não |
| Caracteres não ASCII | Inválido para ferramentas | Não |

### `Terminator`

| Valor | Interpretação | Uso seguro |
|---:|---|---|
| `0x00` | Terminador/padding normal | Sim |
| Diferente de `0x00` | Ignorado pela comparação runtime | Evitar; preservar se existir |

### Tamanho do bloco

O tamanho não é armazenado em campo próprio.

| Caso | Cálculo |
|---|---|
| Bloco intermediário | `DataOffset[i + 1] - DataOffset[i]` |
| Último bloco | `FileSize - DataOffset[i]` |

Para repack, mantenha blocos alinhados a `0x20` quando possível.

---

## 9. Tags conhecidas

A tabela abaixo lista tags observadas no container `r402.dat` e seu sistema relacionado. Algumas tags descrevem subformatos já conhecidos; outras são apenas identificadores internos do pacote.

| Tag | Sistema relacionado | Observação |
|---|---|---|
| `CAM` | Câmeras de sala | Subformato `.CAM`. |
| `SAT` | Dados grandes de sala/colisão/atributos | Estrutura própria; não é o mesmo layout de `.AEV`. |
| `LIT` | Iluminação | Pode aparecer mais de uma vez. |
| `SMD` | Modelo/mesh | Bloco grande de geometria. |
| `SMX` | Instâncias de modelo | Relaciona modelos, draw flags e iluminação. |
| `SHD` | Shadow data | Dados de sombra/projeção. |
| `EFF` | Efeitos | Pacote de efeitos. |
| `TEX` | Tabela auxiliar de textura | Bloco pequeno em `r402.dat`. |
| `ITM` | Dados de item/modelos de item | Bloco grande. |
| `ETM` | Modelos/objetos auxiliares de sala | Relacionado ao sistema `EtXX_init`. |
| `ETS` | Set/lista de objetos auxiliares | Usado junto de `ETM`. |
| `EAR` | Bloco de área/efeito auxiliar | Subformato próprio. |
| `SAR` | Bloco auxiliar de área/som | Subformato próprio. |
| `EAT` | Tabela grande de área/ação | Estrutura própria; inicia com padrão similar a `SAT`. |
| `AEV` | Triggers/eventos de área | Subformato `.AEV`. |
| `RTP` | Route/path data | Header interno observado como `PTR2`. |
| `MDT` | Message/data table | Tabela de mensagens/dados. |
| `CNS` | Dados auxiliares de controle | Bloco pequeno. |
| `STB` | Stage table/bloco auxiliar | Subformato próprio. |
| `FSE` | Sequência/efeito | Possui magic `FSE`. |
| `ESE` | Sequência/efeito | Possui magic `ESE`. |
| `OSD` | Objeto/som/dados auxiliares | Subformato próprio. |
| `BLK` | Block data | Possui magic `BLK`. |
| `DRA` | Draw/área auxiliar | Possui magic `DRA`. |
| `ITA` | Triggers de item | Subformato `.ITA`. |
| `DSE` | Dados auxiliares | Bloco pequeno. |
| `FCV` | Curvas/animações | Pode aparecer várias vezes. |

Do ponto de vista do container, qualquer tag de três bytes pode ser armazenada. Do ponto de vista do jogo, cada sistema chama `GetDataExt` com uma tag específica e espera que o bloco retornado tenha o layout correspondente.

---

## 10. Amostra `r402.dat`

Propriedades da amostra:

| Campo | Valor |
|---|---:|
| Tamanho do arquivo | `0x358FE0` bytes |
| `FileCount` | `0x25` / `37` |
| Tamanho bruto do índice | `0x138` |
| Primeiro bloco | `0x140` |
| Alinhamento do primeiro bloco | `0x20` |

| Index | Tag | Ordinal | Offset | Size | End | Primeiros 16 bytes |
|---:|---|---:|---:|---:|---:|---|
| 0 | `CAM` | 0 | `0x00000140` | `0x1340` | `0x00001480` | `42 34 30 34 0C 0C 00 00 00 00 00 00 00 00 00 00` |
| 1 | `SAT` | 0 | `0x00001480` | `0x19FA0` | `0x0001B420` | `80 01 23 00 08 00 00 00 20 00 73 05 A5 02 1F 0C` |
| 2 | `LIT` | 0 | `0x0001B420` | `0x1420` | `0x0001C840` | `3C 00 31 25 F4 00 00 00 00 00 00 00 00 00 00 00` |
| 3 | `LIT` | 1 | `0x0001C840` | `0x160` | `0x0001C9A0` | `01 00 31 02 08 00 00 00 0F 0F 0F 80 02 00 00 00` |
| 4 | `SMD` | 0 | `0x0001C9A0` | `0x2067C0` | `0x00223160` | `40 00 65 00 50 19 00 00 10 83 19 00 00 00 00 00` |
| 5 | `SMX` | 0 | `0x00223160` | `0x3AA0` | `0x00226C00` | `10 68 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| 6 | `SHD` | 0 | `0x00226C00` | `0x940` | `0x00227540` | `40 00 00 01 00 00 00 60 00 00 06 E0 00 00 09 40` |
| 7 | `EFF` | 0 | `0x00227540` | `0x2CC0` | `0x0022A200` | `0B 00 00 00 00 00 00 00 00 00 00 00 30 00 00 00` |
| 8 | `TEX` | 0 | `0x0022A200` | `0x40` | `0x0022A240` | `00 00 00 03 00 00 00 20 00 00 00 00 00 00 00 00` |
| 9 | `ITM` | 0 | `0x0022A240` | `0x31DA0` | `0x0025BFE0` | `03 00 00 00 20 00 00 00 A0 00 00 00 00 78 01 00` |
| 10 | `ETM` | 0 | `0x0025BFE0` | `0x47380` | `0x002A3360` | `24 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| 11 | `ETS` | 0 | `0x002A3360` | `0x5A0` | `0x002A3900` | `16 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| 12 | `EAR` | 0 | `0x002A3900` | `0x3A0` | `0x002A3CA0` | `06 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| 13 | `SAR` | 0 | `0x002A3CA0` | `0x100` | `0x002A3DA0` | `01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| 14 | `EAT` | 0 | `0x002A3DA0` | `0x95B40` | `0x003398E0` | `80 01 23 00 08 00 00 00 20 00 7A 19 97 15 3E 4D` |
| 15 | `AEV` | 0 | `0x003398E0` | `0x1100` | `0x0033A9E0` | `41 45 56 00 04 01 1B 00 00 00 00 00 00 00 00 00` |
| 16 | `RTP` | 0 | `0x0033A9E0` | `0x3460` | `0x0033DE40` | `50 54 52 32 00 00 61 00 D0 00 C1 24 20 00 00 00` |
| 17 | `MDT` | 0 | `0x0033DE40` | `0x28C0` | `0x00340700` | `06 00 00 00 1C 00 00 00 5C 03 00 00 5C 0A 00 00` |
| 18 | `CNS` | 0 | `0x00340700` | `0x60` | `0x00340760` | `10 00 00 00 E3 05 00 00 64 00 00 00 C8 00 00 00` |
| 19 | `STB` | 0 | `0x00340760` | `0xD60` | `0x003414C0` | `00 00 00 00 00 00 00 00 C9 CC 4C 3D FE FF BF 3F` |
| 20 | `FSE` | 0 | `0x003414C0` | `0x2C0` | `0x00341780` | `46 53 45 00 03 01 05 00 00 00 00 00 00 00 00 00` |
| 21 | `ESE` | 0 | `0x00341780` | `0x20` | `0x003417A0` | `45 53 45 00 00 01 00 00 00 00 00 00 00 00 00 00` |
| 22 | `OSD` | 0 | `0x003417A0` | `0x580` | `0x00341D20` | `00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| 23 | `BLK` | 0 | `0x00341D20` | `0x20` | `0x00341D40` | `42 4C 4B 00 00 01 00 00 00 00 00 00 14 00 00 00` |
| 24 | `DRA` | 0 | `0x00341D40` | `0x20` | `0x00341D60` | `44 52 41 00 00 01 00 00 00 00 00 00 10 00 00 00` |
| 25 | `ITA` | 0 | `0x00341D60` | `0x16C0` | `0x00343420` | `49 54 41 00 05 01 21 00 00 00 00 00 00 00 00 00` |
| 26 | `DSE` | 0 | `0x00343420` | `0x20` | `0x00343440` | `00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| 27 | `FCV` | 0 | `0x00343440` | `0x10E0` | `0x00344520` | `13 00 17 01 00 04 00 02 50 02 50 02 60 02 50 02` |
| 28 | `FCV` | 1 | `0x00344520` | `0x11A0` | `0x003456C0` | `16 00 17 01 00 04 00 02 50 02 50 02 60 02 50 02` |
| 29 | `FCV` | 2 | `0x003456C0` | `0x1A40` | `0x00347100` | `28 00 17 01 00 04 00 02 50 02 50 02 60 02 50 02` |
| 30 | `FCV` | 3 | `0x00347100` | `0x2520` | `0x00349620` | `37 00 17 01 00 04 00 02 50 02 50 02 50 02 50 02` |
| 31 | `FCV` | 4 | `0x00349620` | `0x3CA0` | `0x0034D2C0` | `46 00 16 01 00 04 00 02 00 02 50 02 50 02 50 02` |
| 32 | `FCV` | 5 | `0x0034D2C0` | `0x2760` | `0x0034FA20` | `32 00 16 01 00 04 00 02 50 02 50 02 50 02 50 02` |
| 33 | `FCV` | 6 | `0x0034FA20` | `0x26E0` | `0x00352100` | `2D 00 16 01 00 04 00 02 50 02 50 02 50 02 50 02` |
| 34 | `FCV` | 7 | `0x00352100` | `0x32C0` | `0x003553C0` | `37 00 1F 01 00 04 00 02 50 02 50 02 50 02 50 02` |
| 35 | `FCV` | 8 | `0x003553C0` | `0x36A0` | `0x00358A60` | `2D 00 1E 01 00 04 00 02 50 02 50 02 50 02 50 02` |
| 36 | `FCV` | 9 | `0x00358A60` | `0x580` | `0x00358FE0` | `2C 01 05 04 06 04 06 02 A6 02 66 04 06 00 01 02` |

Observações da amostra:

- O header usa `FileCount = 0x25`.
- As três palavras reservadas em `0x04`, `0x08` e `0x0C` são `0`.
- O índice bruto termina em `0x138`.
- O primeiro bloco começa em `0x140`.
- O padding entre `0x138` e `0x13F` é `0x00`.
- Todos os offsets observados estão alinhados a `0x20`.
- `LIT` aparece duas vezes.
- `FCV` aparece dez vezes.

---

## 11. Regras para extração

Ao extrair, não use apenas a tag como nome final, pois tags duplicadas sobrescrevem arquivos.

Formato recomendado:

```text
000_CAM_00.CAM
001_SAT_00.SAT
002_LIT_00.LIT
003_LIT_01.LIT
...
027_FCV_00.FCV
028_FCV_01.FCV
```

Regras:

1. Ler `FileCount`.
2. Ler `FileCount` offsets a partir de `0x10`.
3. Ler `FileCount` tags a partir de `0x10 + FileCount * 4`.
4. Calcular cada tamanho pelo próximo offset.
5. Extrair bytes exatamente como estão.
6. Gerar manifesto com ordem, tag, ordinal, offset original e tamanho original.

---

## 12. Regras para repack

O `.DAT` é sensível à ordem das entradas. A ordem do índice deve ser preservada quando possível.

### Repack byte-safe

Use para substituir blocos sem reorganizar a ordem lógica:

```text
1. Ler manifesto original.
2. Substituir bytes dos blocos editados.
3. Recriar offsets em ordem.
4. Recriar tags em ordem.
5. Alinhar cada bloco a 0x20.
6. Atualizar FileCount.
```

### Repack estrutural

Use para adicionar/remover blocos:

```text
1. Montar lista final de entradas.
2. Ordenar na ordem desejada pelo runtime.
3. Recalcular FileCount.
4. Escrever header fixo de 0x10 bytes.
5. Reservar DataOffset[FileCount].
6. Escrever Tag[FileCount][4].
7. Alinhar para 0x20.
8. Escrever blocos e recalcular offsets.
9. Voltar e preencher a tabela de offsets.
```

### Recomendações

- Preserve tags duplicadas com ordinal.
- Não remova blocos que o room script espera encontrar.
- Não mude `CAM`, `LIT`, `SMD`, `SMX`, `AEV`, `ITA` e `FCV` sem atualizar também sistemas dependentes.
- Para troca simples de subarquivo, mantenha a tag e a posição da entrada.
- Para arquivos novos, escreva `Reserved0..2 = 0`.
- Use padding `0x00` entre blocos.

---

## 13. Manifesto JSON/YAML recomendado

### JSON

```json
{
  "format": "RE4_PS2_DAT",
  "endianness": "little",
  "alignment": 32,
  "reserved": [0, 0, 0],
  "entries": [
    {
      "index": 0,
      "tag": "CAM",
      "ordinal": 0,
      "file": "000_CAM_00.CAM",
      "original_offset": 320,
      "original_size": 4928
    },
    {
      "index": 2,
      "tag": "LIT",
      "ordinal": 0,
      "file": "002_LIT_00.LIT",
      "original_offset": 111648,
      "original_size": 5152
    },
    {
      "index": 3,
      "tag": "LIT",
      "ordinal": 1,
      "file": "003_LIT_01.LIT",
      "original_offset": 116800,
      "original_size": 352
    }
  ]
}
```

### YAML

```yaml
format: RE4_PS2_DAT
endianness: little
alignment: 32
reserved: [0, 0, 0]
entries:
  - index: 0
    tag: CAM
    ordinal: 0
    file: 000_CAM_00.CAM
    original_offset: 0x00000140
    original_size: 0x1340

  - index: 2
    tag: LIT
    ordinal: 0
    file: 002_LIT_00.LIT
    original_offset: 0x0001B420
    original_size: 0x1420

  - index: 3
    tag: LIT
    ordinal: 1
    file: 003_LIT_01.LIT
    original_offset: 0x0001C840
    original_size: 0x160
```

---

## 14. Structs de referência

### C/C++

```c
#pragma pack(push, 1)

typedef struct RE4_DAT_HEADER {
    uint32_t file_count;
    uint32_t reserved0;
    uint32_t reserved1;
    uint32_t reserved2;
} RE4_DAT_HEADER;

typedef struct RE4_DAT_TAG {
    char tag[3];
    uint8_t terminator;
} RE4_DAT_TAG;

#pragma pack(pop)
```

Representação em memória para ferramenta:

```c
typedef struct RE4_DAT_ENTRY {
    uint32_t index;
    char tag[4];
    uint32_t ordinal;
    uint32_t offset;
    uint32_t size;
    uint8_t* data;
} RE4_DAT_ENTRY;
```

### C#

```csharp
public sealed class Re4DatFile
{
    public uint FileCount { get; set; }
    public uint Reserved0 { get; set; }
    public uint Reserved1 { get; set; }
    public uint Reserved2 { get; set; }
    public List<Re4DatEntry> Entries { get; } = new();
}

public sealed class Re4DatEntry
{
    public int Index { get; set; }
    public string Tag { get; set; } = "";
    public int Ordinal { get; set; }
    public uint Offset { get; set; }
    public uint Size { get; set; }
    public byte[] Data { get; set; } = Array.Empty<byte>();
}
```

Leitura básica:

```csharp
static uint ReadU32(BinaryReader br)
{
    return br.ReadUInt32(); // little-endian em .NET em plataformas comuns
}
```

---

## 15. Parser/extractor de referência em Python

```python
from __future__ import annotations

import json
import struct
from collections import defaultdict
from pathlib import Path

ALIGNMENT = 0x20


def align(value: int, alignment: int = ALIGNMENT) -> int:
    return (value + alignment - 1) & ~(alignment - 1)


def parse_dat(data: bytes) -> dict:
    if len(data) < 0x10:
        raise ValueError("arquivo pequeno demais para ser .DAT")

    file_count, reserved0, reserved1, reserved2 = struct.unpack_from("<4I", data, 0)

    index_size = 0x10 + file_count * 8
    if index_size > len(data):
        raise ValueError("FileCount inválido: índice passa do fim do arquivo")

    offsets = list(struct.unpack_from(f"<{file_count}I", data, 0x10))
    tag_base = 0x10 + file_count * 4

    tags = []
    for i in range(file_count):
        raw = data[tag_base + i * 4: tag_base + i * 4 + 4]
        tag = raw[:3].decode("ascii", errors="replace")
        tags.append(tag)

    first_data = align(index_size)
    if offsets and offsets[0] < first_data:
        raise ValueError("primeiro offset aponta para dentro do índice")

    entries = []
    seen = defaultdict(int)

    for i, (offset, tag) in enumerate(zip(offsets, tags)):
        if offset >= len(data):
            raise ValueError(f"offset fora do arquivo na entrada {i}: 0x{offset:X}")

        end = offsets[i + 1] if i + 1 < file_count else len(data)
        if end < offset or end > len(data):
            raise ValueError(f"tamanho inválido na entrada {i}")

        ordinal = seen[tag]
        seen[tag] += 1

        entries.append({
            "index": i,
            "tag": tag,
            "ordinal": ordinal,
            "offset": offset,
            "size": end - offset,
            "data": data[offset:end],
        })

    return {
        "file_count": file_count,
        "reserved": [reserved0, reserved1, reserved2],
        "entries": entries,
    }


def extract_dat(dat_path: str | Path, out_dir: str | Path) -> None:
    dat_path = Path(dat_path)
    out_dir = Path(out_dir)
    out_dir.mkdir(parents=True, exist_ok=True)

    parsed = parse_dat(dat_path.read_bytes())
    manifest = {
        "format": "RE4_PS2_DAT",
        "endianness": "little",
        "alignment": ALIGNMENT,
        "reserved": parsed["reserved"],
        "entries": [],
    }

    for e in parsed["entries"]:
        name = f"{e['index']:03d}_{e['tag']}_{e['ordinal']:02d}.{e['tag']}"
        (out_dir / name).write_bytes(e["data"])
        manifest["entries"].append({
            "index": e["index"],
            "tag": e["tag"],
            "ordinal": e["ordinal"],
            "file": name,
            "original_offset": e["offset"],
            "original_size": e["size"],
        })

    (out_dir / "manifest.json").write_text(
        json.dumps(manifest, indent=2),
        encoding="utf-8"
    )
```

---

## 16. Repacker de referência em Python

```python
from __future__ import annotations

import json
import struct
from pathlib import Path

ALIGNMENT = 0x20


def align(value: int, alignment: int = ALIGNMENT) -> int:
    return (value + alignment - 1) & ~(alignment - 1)


def pad_to(buf: bytearray, alignment: int = ALIGNMENT, fill: int = 0x00) -> None:
    target = align(len(buf), alignment)
    if target > len(buf):
        buf.extend(bytes([fill]) * (target - len(buf)))


def build_dat(folder: str | Path, output_path: str | Path) -> None:
    folder = Path(folder)
    output_path = Path(output_path)

    manifest = json.loads((folder / "manifest.json").read_text(encoding="utf-8"))
    entries = sorted(manifest["entries"], key=lambda e: e["index"])
    reserved = manifest.get("reserved", [0, 0, 0])

    file_count = len(entries)
    index_size = 0x10 + file_count * 8
    data_start = align(index_size)

    chunks = []
    for e in entries:
        tag = e["tag"]
        if len(tag) != 3:
            raise ValueError(f"tag inválida: {tag!r}")
        chunks.append((tag, (folder / e["file"]).read_bytes()))

    buf = bytearray()
    buf += struct.pack("<4I", file_count, reserved[0], reserved[1], reserved[2])

    offset_table_pos = len(buf)
    buf += b" " * (file_count * 4)

    for tag, _chunk in chunks:
        buf += tag.encode("ascii") + b" "

    if len(buf) > data_start:
        raise ValueError("índice maior que data_start calculado")

    buf += b" " * (data_start - len(buf))

    offsets = []
    for _tag, chunk in chunks:
        pad_to(buf, ALIGNMENT, 0x00)
        offsets.append(len(buf))
        buf += chunk

    pad_to(buf, ALIGNMENT, 0x00)

    for i, off in enumerate(offsets):
        struct.pack_into("<I", buf, offset_table_pos + i * 4, off)

    output_path.write_bytes(buf)
```

---

## 17. Funções e símbolos relevantes

| Endereço | Símbolo | Uso |
|---:|---|---|
| `0x002912F8` | `GetDataExt__FPUiPcUi` | Busca bloco interno por tag e ordinal. |
| `0x00291470` | `GetOPDataHeaderSize__Fv` | Retorna `0`; não há header adicional de OP data. |
| `0x00291478` | `ReadOPData__FUiPUi` | Leitura de pacote/dados de sala para memória. |
| `0x001C07C8` | `setData__9cDataCtrlPCc` | Registra/carrega dados pelo controlador de dados. |
| `0x001C0680` | `init__9cDataCtrl` | Inicialização do controlador de dados. |
| `0x001C06C8` | `initDataUnit__9cDataCtrl` | Inicialização das unidades de dados. |
| `0x001BF6C0` | `setLoadToMram__9cDataUnit` | Agenda leitura para memória principal. |
| `0x001BFAB0` | `setLoadToAram__9cDataUnit` | Agenda leitura para ARAM. |
| `0x0028FE38` | `InitModule__FP10MODULE_DAT` | Inicialização de módulo de dados. |
| `0x0028FFD0` | `readEmData__FP10MODULE_DATUiPvUi` | Leitura de dados de módulo de inimigo. |
| `0x00290500` | `setEmModule__FP10MODULE_DATUi` | Configuração de módulo de inimigo. |
| `0x004C0018` | `RoomData` | Estrutura global relacionada a dados de sala. |

### Observação sobre `GetDataExt`

O código acessa:

```text
Count        = *(base + 0x00)
Offset[i]    = *(base + 0x10 + i * 4)
Tag[i]       =  (base + 0x10 + Count * 4 + i * 4)
ReturnValue  = base + Offset[i]
```

Isso confirma:

- header fixo de `0x10` bytes;
- tabela de offsets em `+0x10`;
- tabela de tags logo após a tabela de offsets;
- offsets relativos ao início do `.DAT`;
- busca por tag de três caracteres;
- suporte nativo a múltiplas ocorrências da mesma tag.

---

## 18. Checklist de roundtrip

Para validar uma ferramenta de edição/repack:

```text
[ ] Ler FileCount corretamente.
[ ] Preservar Reserved0, Reserved1 e Reserved2.
[ ] Ler exatamente FileCount offsets.
[ ] Ler exatamente FileCount tags.
[ ] Preservar tags duplicadas com ordinal.
[ ] Calcular tamanho por próximo offset.
[ ] Extrair cada bloco sem alterar bytes.
[ ] Recriar o header em little-endian.
[ ] Recalcular offsets após qualquer alteração de tamanho.
[ ] Alinhar índice e blocos a 0x20.
[ ] Usar padding 0x00 no container.
[ ] Garantir offsets crescentes.
[ ] Garantir que nenhum offset aponta para dentro do índice.
[ ] Testar GetDataExt lógico para todas as tags.
[ ] Comparar extração do DAT original e do DAT repackado.
```

Teste mínimo:

```text
original.dat -> extract -> repack sem editar -> extract novamente
```

O resultado deve preservar:

```text
FileCount
ordem das entradas
tags
ordinais
dados internos de cada bloco
```

Diferenças aceitáveis em repack estrutural:

```text
offsets recalculados
padding entre blocos
posição física dos blocos
```

---

## 19. Resumo de offsets

| Região | Offset | Tamanho |
|---|---:|---:|
| Header fixo | `0x00` | `0x10` |
| `FileCount` | `0x00` | `0x04` |
| `Reserved0` | `0x04` | `0x04` |
| `Reserved1` | `0x08` | `0x04` |
| `Reserved2` | `0x0C` | `0x04` |
| Tabela de offsets | `0x10` | `FileCount * 0x04` |
| Tabela de tags | `0x10 + FileCount * 0x04` | `FileCount * 0x04` |
| Fim bruto do índice | `0x10 + FileCount * 0x08` | — |
| Primeiro bloco recomendado | `align(0x10 + FileCount * 0x08, 0x20)` | — |

### Fórmulas

```c
uint32_t OffsetTable = 0x10;
uint32_t TagTable    = 0x10 + FileCount * 4;
uint32_t RawIndexEnd = 0x10 + FileCount * 8;
uint32_t DataStart   = align(RawIndexEnd, 0x20);
```

---

## 20. Resumo técnico

O `.DAT` de Resident Evil 4 PS2 é um container simples, sem assinatura e sem tabela de tamanhos. O índice é formado por uma contagem, uma tabela de offsets absolutos e uma tabela paralela de tags de três caracteres. O runtime usa `GetDataExt` para localizar o bloco desejado por tag e ordinal.

A serialização correta depende de três pontos:

1. preservar a ordem das entradas;
2. recalcular offsets sempre que qualquer bloco mudar de tamanho;
3. manter o alinhamento físico de `0x20` bytes.

Para ferramentas de modding, o fluxo mais seguro é extrair cada bloco com manifesto, editar subformatos específicos separadamente e reconstruir o `.DAT` a partir da ordem original.
