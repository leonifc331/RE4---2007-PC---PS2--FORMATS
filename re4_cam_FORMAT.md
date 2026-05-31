# Formato `.CAM` — Resident Evil 4 PS2

Documentação técnica do formato de câmera usado nos arquivos `.CAM` do Resident Evil 4 de PlayStation 2.

O formato armazena as câmeras de sala, as áreas de ativação, transições entre áreas e dados variáveis de trilha/curva. O jogo carrega o arquivo através do sistema `CameraControl`, relocaliza ponteiros internos e usa as tabelas fixas para alternar entre câmeras conforme a posição do jogador e o número da área atual.

---

## Características gerais

| Propriedade | Valor |
|---|---:|
| Endianness | Little-endian |
| Header | 16 bytes |
| Versão principal observada | `B404` |
| Tabela 1 | `_CUT_INFO`, stride `0x10` |
| Tabela 2 | `_CAMERA_DATA`, stride `0x30` no arquivo B404 |
| Tabela 3 | `_AREA_DATA`, stride `0x34` no arquivo B404 |
| Tabela 4 | `_LERP_DATA`, stride `0x10` |
| Ponteiros no arquivo | Offsets relativos ao início do `.CAM` |
| Ponteiros em runtime | Endereços absolutos após `calcAddr` |
| Principal loader | `calcAddr__13CameraControlPUc` |
| Leitura por sala | `RoomDataRead__13CameraControlPUc` |
| Leitura core/global | `CoreDataRead__13CameraControlPUc` |

---

## Funções relevantes do ELF

| Endereço | Símbolo | Função |
|---:|---|---|
| `0x00106858` | `cameraDataVersion__FPc` | Detecta a versão textual do arquivo |
| `0x001008D8` | `calcAddr__13CameraControlPUc` | Relocaliza offsets internos do `.CAM` |
| `0x00100AE8` | `RoomDataRead__13CameraControlPUc` | Carrega dados de câmera da sala |
| `0x00100B28` | `CoreDataRead__13CameraControlPUc` | Carrega dados de câmera globais |
| `0x001007C0` | `DataSearch__13CameraControlSc` | Procura uma entrada de área pelo número |
| `0x00100828` | `LerpDataSearch__13CameraControlScScScSc` | Procura transição entre duas áreas |
| `0x00101200` | `CutCall__13CameraControlSc` | Ativa uma câmera/cut pelo índice |
| `0x001012D0` | `switchCamera__13CameraControlP9_CUT_INFO` | Aplica uma entrada `_CUT_INFO` |
| `0x00101020` | `CameraSetCutData__FP6CAMERAP12_CAMERA_DATA` | Copia/aplica dados de câmera |
| `0x00101710` | `areaAttr__FP10_AREA_DATAUcUc` | Testa atributos de área |
| `0x00101750` | `areaHit__FP6tagVecP10_AREA_DATAf` | Teste geométrico de área |
| `0x00101B58` | `area_hit_pN__FP6tagVecP10_AREA_DATA` | Hit-test para área poligonal |
| `0x00101938` | `area_hit_p3__FP6tagVecP10_AREA_DATA` | Hit-test para área triangular |
| `0x00105090` | `Parametrize__FP12_CAMERA_DATAP13_CAM_B_SPLINE` | Prepara spline da câmera |
| `0x001054D0` | `BSpline__FP13_CAM_B_SPLINEP6CAMERAUi` | Avalia B-spline |
| `0x00105658` | `searchRail__FP13_CAM_B_SPLINEP12_CAMERA_DATAP6tagVecUi` | Procura ponto em trilha/rail |
| `0x00105A60` | `debugDrawRail__13CameraControlP12_CAMERA_DATA` | Desenho debug da trilha |
| `0x00100288` | `HermiteExport__13CameraControlP12_CAMERA_DATAPUc` | Export/debug de curva Hermite |

Rotinas de comportamento de câmera encontradas:

| Endereço | Símbolo |
|---:|---|
| `0x001031F8` | `r0_Wait__13CameraControl` |
| `0x00103200` | `r0_Debug__13CameraControl` |
| `0x001035C0` | `r0_Fix__13CameraControl` |
| `0x00103620` | `r0_Pan__13CameraControl` |
| `0x00103758` | `r0_Track__13CameraControl` |
| `0x00103890` | `r0_RailPan__13CameraControl` |
| `0x00103A08` | `r0_UpCut__13CameraControl` |
| `0x00103A10` | `r0_RailBehind__13CameraControl` |
| `0x00104898` | `r0_Free__13CameraControl` |

---

## Estruturas nomeadas no debug

O ELF possui nomes de estruturas de câmera. As principais são:

```c
struct _CAM_FILE_HEADER;
struct _CUT_INFO;
struct _CAMERA_DATA;
struct _AREA_DATA;
struct _LERP_DATA;
struct _CAM_B_SPLINE;
struct CAMERA_POINT;
struct QFPS_OFFSET;
```

As estruturas debug indicam o layout usado pelo código em runtime. A versão `B404` salva no disco usa uma forma compacta em alguns pontos, principalmente na entrada `_CAMERA_DATA`, que aparece no arquivo com stride `0x30`.

---

## Versões aceitas

A versão fica nos 4 primeiros bytes do arquivo.

| Bytes ASCII | Retorno de `cameraDataVersion` | Situação |
|---|---:|---|
| `B400` | `0` | Versão antiga |
| `B401` | `1` | Versão antiga |
| `B402` | `2` | Aceita pelo loader moderno |
| `B403` | `3` | Aceita pelo loader moderno |
| `B404` | `4` | Versão observada nos arquivos analisados |
| `EMPT` | `-1` | Marcador vazio/não utilizável |
| Outros | `-1` | Inválido |

`calcAddr__13CameraControlPUc` só executa a relocalização completa quando a versão retornada é maior ou igual a `2`.

Para edição de arquivos originais do jogo, use `B404`.

---

## Layout global B404

```text
0x0000  _CAM_FILE_HEADER  0x10 bytes
0x0010  _CUT_INFO[]       nCutCamera * 0x10
...     _CAMERA_DATA[]    nCutCamera * 0x30
...     _AREA_DATA[]      nArea * 0x34
...     _LERP_DATA[]      nLerp * 0x10
...     Dados variáveis   vetores, curvas, roll, fovy, rails
```

Cálculo dos offsets fixos:

```c
cutInfoOffset = 0x10;
cameraOffset  = cutInfoOffset + nCutCamera * 0x10;
areaOffset    = cameraOffset  + nCutCamera * 0x30;
lerpOffset    = areaOffset    + nArea      * 0x34;
payloadOffset = lerpOffset    + nLerp      * 0x10;
```

Os ponteiros das estruturas continuam sendo a fonte mais confiável para localizar os dados variáveis. Em alguns arquivos, o início dos dados apontados por câmera pode coincidir com a região logo após a tabela de áreas. Portanto, ferramentas devem validar offsets por entrada, não apenas assumir que todo payload começa depois de `payloadOffset`.

---

# Header — `_CAM_FILE_HEADER`

Tamanho: `0x10`.

```c
struct CAM_FILE_HEADER
{
    char    Version[4];      // "B404"
    uint8_t nArea;           // quantidade de AREA_DATA; byte +0x04
    uint8_t nCutCamera;      // quantidade de CUT_INFO e CAMERA_DATA; byte +0x05
    uint8_t nLerp;           // quantidade de LERP_DATA; byte +0x06
    uint8_t reserved07;      // normalmente 0
    uint8_t reserved[8];     // normalmente zerado
};
```

Tabela de campos:

| Offset | Tipo | Nome usado nesta documentação | Descrição |
|---:|---|---|---|
| `0x00` | `char[4]` | `Version` | Versão textual: `B400` até `B404` |
| `0x04` | `uint8` | `nArea` | Quantidade de entradas `_AREA_DATA` |
| `0x05` | `uint8` | `nCutCamera` | Quantidade de entradas `_CUT_INFO` e `_CAMERA_DATA` |
| `0x06` | `uint8` | `nLerp` | Quantidade de entradas `_LERP_DATA` |
| `0x07` | `uint8` | `reserved07` | Reservado |
| `0x08` | `uint8[8]` | `reserved` | Reservado/alinhamento |

> Observação: os símbolos debug usam nomes históricos `nCdat`, `nAdat` e `nLdat`. Para ferramentas, os nomes `nArea`, `nCutCamera` e `nLerp` refletem melhor o uso real observado no loader.

---

## Valores aceitos do header

| Campo | Valores aceitos | Valores observados | Recomendação |
|---|---|---|---|
| `Version` | `B402`, `B403`, `B404` no loader moderno | `B404` | Usar `B404` |
| `nArea` | `0..255`, limitado pelo tamanho do arquivo | `7`, `15`, `16`, `18` | Recalcular ao editar áreas |
| `nCutCamera` | `0..255`, limitado pelo tamanho do arquivo | `9`, `15`, `16`, `19` | Recalcular ao editar cuts/câmeras |
| `nLerp` | `0..255`, limitado pelo tamanho do arquivo | `0`, `1` | Recalcular ao editar transições |
| `reserved07` | `0..255` | `0` | Manter `0` |
| `reserved` | bytes livres | tudo `0` | Manter zerado |

---

# CUT_INFO — tabela de cuts

Cada entrada tem `0x10` bytes.

A tabela `_CUT_INFO` vincula uma câmera a uma área. `switchCamera__13CameraControlP9_CUT_INFO` recebe uma entrada dessa tabela e aplica os dados referenciados.

```c
struct CUT_INFO_B404
{
    uint8_t  Attr;           // +0x00
    uint8_t  reserved[7];    // +0x01
    uint32_t pCdat;          // +0x08 offset para CAMERA_DATA
    uint32_t pAdat;          // +0x0C offset para AREA_DATA; pode ser 0 em casos especiais
};
```

Tabela de campos:

| Offset | Tipo | Nome | Descrição |
|---:|---|---|---|
| `0x00` | `uint8` | `Attr` | Atributos/modo do cut |
| `0x01` | `uint8[7]` | `reserved` | Área reservada; pode conter lixo/debug em alguns arquivos |
| `0x08` | `uint32` | `pCdat` | Offset relativo para `_CAMERA_DATA` |
| `0x0C` | `uint32` | `pAdat` | Offset relativo para `_AREA_DATA` |

Durante `calcAddr`, `pCdat` e `pAdat` são convertidos para endereços absolutos somando a base do arquivo carregado.

---

## Valores aceitos de `CUT_INFO.Attr`

`Attr` é um byte. O código preserva e usa combinações de bits, e os arquivos oficiais mostram um conjunto pequeno de valores.

| Valor | Bits | Situação | Uso recomendado |
|---:|---|---|---|
| `0x00` | nenhum | Válido como byte, não observado em cuts ativos | Evitar em cuts ativos |
| `0x03` | `0x01 | 0x02` | Observado com frequência | Seguro para câmera/cut padrão |
| `0x04` | `0x04` | Observado | Seguro quando já existir no arquivo |
| `0x23` | `0x20 | 0x03` | Observado | Preservar |
| `0x43` | `0x40 | 0x03` | Observado | Preservar |
| `0x63` | `0x40 | 0x20 | 0x03` | Observado | Preservar |
| `0x08` | `0x08` | O loader transforma em `0x28` na câmera | Usar apenas se o efeito for desejado |
| Outros | qualquer combinação `0x00..0xFF` | Aceito pelo tipo binário | Preservar, não inventar |

### Bits observados de `Attr`

| Bit | Máscara | Comportamento conhecido |
|---:|---:|---|
| `0` | `0x01` | Parte dos modos `0x03`, `0x23`, `0x43`, `0x63` |
| `1` | `0x02` | Parte dos modos `0x03`, `0x23`, `0x43`, `0x63` |
| `2` | `0x04` | Modo/variante observado isolado |
| `3` | `0x08` | Em `_CAMERA_DATA`, `calcAddr` adiciona `0x20` quando esse bit está ativo |
| `5` | `0x20` | Flag secundária observada |
| `6` | `0x40` | Flag secundária observada |
| outros | `0x10`, `0x80` | Não observados nas amostras | Preservar se existirem |

---

# CAMERA_DATA — dados de câmera

No arquivo `B404`, cada entrada tem `0x30` bytes.

A estrutura debug `_CAMERA_DATA` descreve o layout runtime mais amplo, mas os arquivos analisados usam uma forma compacta com ponteiro principal no offset `+0x2C`. Esse ponteiro normalmente aponta para uma sequência de pontos `tagVec` de `0x10` ou vetores compactos de `0x0C`, dependendo do modo da câmera.

```c
struct CAMERA_DATA_B404
{
    uint8_t  Be_flag;        // +0x00
    int8_t   No;             // +0x01
    int8_t   Id;             // +0x02
    uint8_t  Attr;           // +0x03

    uint8_t  param04[0x1C];  // +0x04, dados dependentes do modo

    float    ValueA;         // +0x20, geralmente coordenada/parâmetro X ou target/campos
    float    ValueB;         // +0x24, geralmente coordenada/parâmetro Y/Z ou target/campos
    uint32_t nPoint;         // +0x28, quantidade de pontos/valores
    uint32_t pData;          // +0x2C, offset para dados variáveis
};
```

Tabela de campos:

| Offset | Tipo | Nome | Descrição |
|---:|---|---|---|
| `0x00` | `uint8` | `Be_flag` | Flag de ativação da entrada |
| `0x01` | `int8` | `No` | Número da câmera/área |
| `0x02` | `int8` | `Id` | Sufixo/variante da câmera |
| `0x03` | `uint8` | `Attr` | Atributos/modo da câmera |
| `0x04` | `float` / bytes | `Dir` / `param04` | Direção inicial, ângulo ou parâmetro do modo |
| `0x08` | `uint8[4]` | `compat` | Bytes de compatibilidade; em B404 geralmente começa com `01 FF` |
| `0x0C` | bytes | `mode_data0` | Dados dependentes do modo; pode conter valores não numéricos em arquivos debug |
| `0x10` | bytes | `mode_data1` | Dados dependentes do modo |
| `0x14` | bytes | `mode_data2` | Dados dependentes do modo |
| `0x18` | bytes | `mode_data3` | Dados dependentes do modo |
| `0x1C` | bytes | `mode_data4` | Dados dependentes do modo |
| `0x20` | `float` | `ValueA` | Parâmetro numérico da câmera |
| `0x24` | `float` | `ValueB` | Parâmetro numérico da câmera |
| `0x28` | `uint32` | `nPoint` | Quantidade de pontos lidos a partir de `pData` |
| `0x2C` | `uint32` | `pData` | Offset relativo para payload da câmera |

Durante `calcAddr`, `pData` é convertido para endereço absoluto.

---

## Relação com `_CAMERA_DATA` do debug

O símbolo debug apresenta estes nomes:

```c
struct _CAMERA_DATA
{
    uint8_t Be_flag;
    int8_t  No;
    int8_t  Id;
    uint8_t Attr;

    float   offset[3];
    void*   pFrame;

    union {
        uint8_t dummy[12];
        float   target_offset[3];
        float   floor_ratio;
    };

    int     nPoint;
    tagVec* pCampos;
    tagVec* pTarget;
    float*  pRoll;
    float*  pFovy;
};
```

A forma `B404` no disco compacta parte desses ponteiros/dados. Por isso, em ferramentas de edição, o mais seguro é tratar o trecho `+0x04..+0x27` como bloco de parâmetros dependente de `Attr`/modo, preservando bytes desconhecidos quando a ferramenta não tiver uma interpretação específica.

---

## Valores aceitos em `CAMERA_DATA`

| Campo | Tipo | Valores aceitos | Observado | Recomendação |
|---|---|---|---|---|
| `Be_flag` | `uint8` | `0..255` | `1` | Usar `1` para entrada ativa; `0` para desativada |
| `No` | `int8` | `-128..127` | `1..17`, `20` | Usar números positivos; deve casar com área/cut |
| `Id` | `int8` | `-128..127` | `0`, `1`, `2` | Usar como sufixo/variante |
| `Attr` | `uint8` | `0x00..0xFF` | `0x03`, `0x04`, `0x23`, `0x43`, `0x63` | Preservar valores existentes |
| `param04` | bytes | qualquer byte | diversos | Preservar quando não editado |
| `ValueA` | `float32` | qualquer float finito | coordenadas/valores de câmera | Editar como float |
| `ValueB` | `float32` | qualquer float finito | coordenadas/valores de câmera | Editar como float |
| `nPoint` | `uint32` | `0..0xFFFFFFFF`, limitado pelo payload | `4`, `5`, `6`, `8` | Manter coerente com tamanho de `pData` |
| `pData` | `uint32` | `0` ou offset válido | offsets válidos | Recalcular no repack |

---

## Interpretação prática de `CAMERA_DATA.Attr`

Os valores observados são os mesmos da tabela de cuts:

| Valor | Interpretação prática |
|---:|---|
| `0x03` | Câmera comum/fixa ou rail simples |
| `0x04` | Variante de câmera/cut com comportamento diferente |
| `0x23` | Câmera comum com flag `0x20` |
| `0x43` | Câmera comum com flag `0x40` |
| `0x63` | Câmera comum com flags `0x20` e `0x40` |

O mapeamento exato entre `Attr` e as rotinas `r0_Fix`, `r0_Pan`, `r0_Track`, `r0_RailPan`, `r0_RailBehind` e `r0_Free` depende de estado runtime e de parâmetros adicionais. Para edição segura, altere `Attr` apenas copiando valores já presentes no mesmo arquivo ou na mesma sala.

---

## Payload de câmera

`pData` normalmente aponta para um bloco de pontos. Nos arquivos analisados, a forma mais comum é:

```c
struct CAM_VEC3
{
    float x;
    float y;
    float z;
};
```

Tamanho: `0x0C`.

Quando `nPoint = 4`, o bloco pode ocupar `4 * 0x0C = 0x30` bytes.  
Quando `nPoint = 6`, pode ocupar `6 * 0x0C = 0x48` bytes.

Exemplo de sequência observada:

```text
nPoint = 4
pData  = 0x000006DC

0x06DC: float x0
0x06E0: float y0
0x06E4: float z0
0x06E8: float x1
0x06EC: float y1
0x06F0: float z1
...
```

Alguns modos podem usar `tagVec` alinhado a `0x10` bytes:

```c
struct tagVec
{
    float x;
    float y;
    float z;
    float w;
};
```

Por isso, um editor deve inferir o tamanho do bloco usando o próximo offset válido, o modo da câmera e o contexto da sala.

---

# AREA_DATA — áreas de ativação

No arquivo `B404`, cada entrada tem `0x34` bytes.

As áreas definem regiões onde uma câmera ou grupo de câmeras passa a valer. O código usa `areaHitCheck`, `areaHit`, `area_hit_p3` e `area_hit_pN` para testar a posição do jogador contra essas regiões.

```c
struct AREA_DATA_B404
{
    uint8_t  Be_flag;        // +0x00
    int8_t   No;             // +0x01
    int8_t   Suffix;         // +0x02
    uint8_t  Attr;           // +0x03

    float    Dir;            // +0x04
    float    Height;         // +0x08
    uint32_t mode0;          // +0x0C
    uint32_t pExtra;         // +0x10

    uint8_t  reserved[0x0C]; // +0x14..+0x1F

    uint32_t nVer;           // +0x20
    uint32_t pCampos;        // +0x24
    uint32_t pTarget;        // +0x28
    uint32_t pRoll;          // +0x2C
    uint32_t pFovy;          // +0x30
};
```

Tabela de campos:

| Offset | Tipo | Nome | Descrição |
|---:|---|---|---|
| `0x00` | `uint8` | `Be_flag` | Flag de ativação |
| `0x01` | `int8` | `No` | Número da área |
| `0x02` | `int8` | `Suffix` | Sufixo/variante da área |
| `0x03` | `uint8` | `Attr` | Atributos da área |
| `0x04` | `float` | `Dir` | Direção/rotação base |
| `0x08` | `float` | `Height` | Altura ou valor vertical da área |
| `0x0C` | `uint32` | `mode0` | Campo dependente do tipo |
| `0x10` | `uint32` | `pExtra` | Offset/ponteiro adicional; nem sempre é offset válido |
| `0x14` | bytes | `reserved0` | Dados dependentes do tipo |
| `0x18` | bytes | `reserved1` | Dados dependentes do tipo |
| `0x1C` | bytes | `reserved2` | Dados dependentes do tipo |
| `0x20` | `uint32` | `nVer` | Quantidade de vértices/pontos |
| `0x24` | `uint32` | `pCampos` / `pVer` | Offset para lista de pontos/vértices/campos |
| `0x28` | `uint32` | `pTarget` | Offset para lista de targets ou pontos auxiliares |
| `0x2C` | `uint32` | `pRoll` | Offset para lista de roll ou dados auxiliares |
| `0x30` | `uint32` | `pFovy` | Offset para lista de FOV ou dados auxiliares |

Durante `calcAddr`, os campos `+0x10`, `+0x24`, `+0x28`, `+0x2C` e `+0x30` são tratados como ponteiros e recebem a base do arquivo.

Atenção: nos arquivos oficiais, `+0x10` pode conter dados não tratados como offset válido por ferramentas externas. Em algumas entradas ele aponta para dados reais; em outras contém valores de runtime/debug ou dados dependentes do tipo. Não aplique validação rígida em `+0x10` sem considerar o `Suffix`.

---

## `_AREA_DATA` no debug

O símbolo debug indica a base:

```c
struct _AREA_DATA
{
    uint8_t Be_flag;
    int8_t  No;
    int8_t  Suffix;
    uint8_t Attr;

    float   Dir;
    uint8_t Type_char;
    uint8_t Type_addr;

    uint8_t dummy[22];

    union {
        struct {
            float   Height;
            float   Y;
            int     nVer;
            tagVec* pVer;
        };
    };
};
```

No arquivo `B404`, a área aparece estendida para `0x34` bytes porque o loader relocaliza vários ponteiros adicionais usados por modos de câmera e por `CameraQuasiFPS`.

---

## Valores aceitos em `AREA_DATA`

| Campo | Tipo | Valores aceitos | Observado | Recomendação |
|---|---|---|---|---|
| `Be_flag` | `uint8` | `0..255` | `1` | Usar `1` para área ativa |
| `No` | `int8` | `-128..127` | `1..18`, `20` | Usar número positivo e único quando possível |
| `Suffix` | `int8` | `-128..127` | `0`, `6`, `8` | Usar valores observados |
| `Attr` | `uint8` | `0x00..0xFF` | `0x00` | Preservar; `0` é padrão |
| `Dir` | `float32` | qualquer float finito | `0`, ângulos em radiano | Editar como direção/ângulo |
| `Height` | `float32` | qualquer float finito | `1000.0` comum | Altura vertical da área |
| `mode0` | `uint32` | qualquer valor | geralmente `0` | Preservar |
| `pExtra` | `uint32` | `0`, offset válido ou dado dependente do tipo | misto | Não validar rigidamente |
| `nVer` | `uint32` | `0..0xFFFFFFFF`, limitado por payload | `0`, `1`, `2`, `24`, `25` | Recalcular com listas |
| `pCampos` | `uint32` | `0` ou offset válido | offsets válidos | Recalcular |
| `pTarget` | `uint32` | `0` ou offset válido | offsets válidos/zero | Recalcular |
| `pRoll` | `uint32` | `0` ou offset válido | offsets válidos/zero | Recalcular |
| `pFovy` | `uint32` | `0` ou offset válido | offsets válidos/zero | Recalcular |

---

## Valores conhecidos de `AREA_DATA.Suffix`

| Valor | Situação | Descrição prática |
|---:|---|---|
| `0x00` | Observado | Área base/default |
| `0x06` | Observado | Área com poucos pontos ou variante compacta |
| `0x08` | Observado | Área com listas longas, normalmente `nVer` alto |
| Outros | Aceitos pelo tipo | Não observados nas amostras; preservar se existirem |

---

## Interpretação de listas da área

Quando `nVer` é alto, as listas apontadas normalmente usam `tagVec` de `0x10` bytes:

```c
struct AREA_VEC
{
    float x;
    float y;
    float z;
    float w;
};
```

Exemplo comum para `nVer = 24`:

| Campo | Tamanho provável |
|---|---:|
| `pCampos` | `24 * 0x10 = 0x180` |
| `pTarget` | `24 * 0x10 = 0x180` |
| `pRoll` | `24 * 0x04 = 0x60` |
| `pFovy` | `24 * 0x04 = 0x60` |

Em várias áreas, `pRoll` contém `0.0` e `pFovy` contém valores como `45.0`, `50.0` ou `40.0`.

---

# LERP_DATA — transições

Cada entrada tem `0x10` bytes.

A tabela `_LERP_DATA` é usada por `LerpDataSearch__13CameraControlScScScSc` para localizar uma transição entre uma área de origem e uma área de destino.

```c
struct LERP_DATA
{
    uint8_t Be_flag;         // +0x00
    int8_t  SrcNo;           // +0x01
    int8_t  SrcSuffix;       // +0x02
    int8_t  DstNo;           // +0x03
    int8_t  DstSuffix;       // +0x04
    uint8_t Attr;            // +0x05
    uint16_t dummy;          // +0x06
    int32_t InterFrame;      // +0x08
    int32_t padding;         // +0x0C
};
```

Tabela de campos:

| Offset | Tipo | Nome | Descrição |
|---:|---|---|---|
| `0x00` | `uint8` | `Be_flag` | Flag de ativação |
| `0x01` | `int8` | `SrcNo` | Área de origem |
| `0x02` | `int8` | `SrcSuffix` | Sufixo da área de origem |
| `0x03` | `int8` | `DstNo` | Área de destino |
| `0x04` | `int8` | `DstSuffix` | Sufixo da área de destino |
| `0x05` | `uint8` | `Attr` | Atributo de transição |
| `0x06` | `uint16` | `dummy` | Reservado |
| `0x08` | `int32` | `InterFrame` | Duração da interpolação em frames |
| `0x0C` | `int32` | `padding` | Reservado |

---

## Valores aceitos em `LERP_DATA`

| Campo | Tipo | Valores aceitos | Recomendação |
|---|---|---|---|
| `Be_flag` | `uint8` | `0..255` | Usar `1` ativo, `0` inativo |
| `SrcNo` | `int8` | `-128..127` | Deve existir em `_AREA_DATA.No` |
| `SrcSuffix` | `int8` | `-128..127` | Deve casar com `_AREA_DATA.Suffix` |
| `DstNo` | `int8` | `-128..127` | Deve existir em `_AREA_DATA.No` |
| `DstSuffix` | `int8` | `-128..127` | Deve casar com `_AREA_DATA.Suffix` |
| `Attr` | `uint8` | `0x00..0xFF` | Preservar |
| `dummy` | `uint16` | `0..65535` | Usar `0` |
| `InterFrame` | `int32` | `0..2147483647` recomendado | Usar valor positivo |
| `padding` | `int32` | qualquer valor | Usar `0` |

---

# CAMERA_POINT

Estrutura runtime usada por interpolação/smooth:

```c
struct CAMERA_POINT
{
    tagVec Campos;   // posição da câmera
    tagVec Target;   // alvo/look-at
    float  Roll;     // rolagem
    float  Fovy;     // campo de visão vertical
};
```

Tamanho: `0x30`.

| Offset | Tipo | Nome |
|---:|---|---|
| `0x00` | `tagVec` | `Campos` |
| `0x10` | `tagVec` | `Target` |
| `0x20` | `float` | `Roll` |
| `0x24` | `float` | `Fovy` |
| `0x28` | `uint8[8]` | padding/alinhamento quando usado em buffers |

---

# CAM_B_SPLINE

Estrutura runtime usada para parametrização de rails/splines. Não aparece como bloco fixo direto no arquivo, mas é preenchida a partir de `_CAMERA_DATA`.

```c
struct CAM_B_SPLINE
{
    int   order;
    float cand_t;
    int   history_i;
    int   p_num;

    float c_alpha[26];
    float c_beta[26];
    float c_gamma[26];

    float t_alpha[26];
    float t_beta[26];
    float t_gamma[26];

    float r_alpha[26];
    float f_alpha[26];

    float B[26];
};
```

Tamanho runtime: `0x3B8`.

| Campo | Descrição |
|---|---|
| `order` | Ordem da spline |
| `cand_t` | Parâmetro candidato atual |
| `history_i` | Índice anterior/cache |
| `p_num` | Quantidade de pontos |
| `c_*` | Coeficientes de posição da câmera |
| `t_*` | Coeficientes de alvo |
| `r_alpha` | Coeficientes de roll |
| `f_alpha` | Coeficientes de FOV |
| `B` | Base/pesos da spline |

---

# Relação entre tabelas

## CUT_INFO → CAMERA_DATA / AREA_DATA

Cada cut aponta para uma câmera e uma área:

```text
CUT_INFO.pCdat -> CAMERA_DATA
CUT_INFO.pAdat -> AREA_DATA
```

Essa relação permite que várias cuts compartilhem a mesma área ou a mesma câmera.

---

## CAMERA_DATA → payload

```text
CAMERA_DATA.pData -> lista de pontos/curvas
CAMERA_DATA.nPoint -> quantidade de pontos/valores
```

O tipo exato do payload depende do `Attr` e dos parâmetros do bloco `+0x04..+0x27`.

---

## AREA_DATA → listas de pontos/roll/fovy

```text
AREA_DATA.pCampos -> lista de pontos/campos/vértices
AREA_DATA.pTarget -> lista de targets/pontos auxiliares
AREA_DATA.pRoll   -> lista de floats de roll
AREA_DATA.pFovy   -> lista de floats de FOV
```

Para áreas poligonais, `pCampos` ou `pVer` representa os vértices da área usados por `areaHit`.

---

# Campos com valores aceitos

## Resumo por campo

| Estrutura | Campo | Tipo | Valores seguros |
|---|---|---|---|
| `HEADER` | `Version` | `char[4]` | `B404` |
| `HEADER` | `nArea` | `uint8` | `0..255`, validado por tamanho |
| `HEADER` | `nCutCamera` | `uint8` | `0..255`, validado por tamanho |
| `HEADER` | `nLerp` | `uint8` | `0..255`, validado por tamanho |
| `CUT_INFO` | `Attr` | `uint8` | `0x03`, `0x04`, `0x23`, `0x43`, `0x63` |
| `CUT_INFO` | `pCdat` | `uint32` | offset válido para `_CAMERA_DATA` |
| `CUT_INFO` | `pAdat` | `uint32` | `0` ou offset válido para `_AREA_DATA` |
| `CAMERA_DATA` | `Be_flag` | `uint8` | `1` ativo, `0` inativo |
| `CAMERA_DATA` | `No` | `int8` | `1..127` recomendado |
| `CAMERA_DATA` | `Id` | `int8` | `0..127` recomendado |
| `CAMERA_DATA` | `Attr` | `uint8` | `0x03`, `0x04`, `0x23`, `0x43`, `0x63` |
| `CAMERA_DATA` | `nPoint` | `uint32` | `0..64` recomendado para ferramenta |
| `CAMERA_DATA` | `pData` | `uint32` | `0` ou offset válido |
| `AREA_DATA` | `Be_flag` | `uint8` | `1` ativo, `0` inativo |
| `AREA_DATA` | `No` | `int8` | `1..127` recomendado |
| `AREA_DATA` | `Suffix` | `int8` | `0`, `6`, `8` |
| `AREA_DATA` | `Attr` | `uint8` | `0x00` observado |
| `AREA_DATA` | `nVer` | `uint32` | `0`, `1`, `2`, `24`, `25` observados |
| `AREA_DATA` | ponteiros | `uint32` | `0` ou offset válido |
| `LERP_DATA` | `SrcNo/DstNo` | `int8` | deve existir na tabela de áreas |
| `LERP_DATA` | `InterFrame` | `int32` | `0` ou positivo |

---

# Offsets relocalizados pelo loader

`calcAddr__13CameraControlPUc` converte offsets relativos em ponteiros absolutos.

| Estrutura | Offset do campo | Campo |
|---|---:|---|
| `_CUT_INFO` | `+0x08` | `pCdat` |
| `_CUT_INFO` | `+0x0C` | `pAdat` |
| `_CAMERA_DATA` | `+0x2C` | `pData` |
| `_AREA_DATA` | `+0x10` | `pExtra` |
| `_AREA_DATA` | `+0x24` | `pCampos` / `pVer` |
| `_AREA_DATA` | `+0x28` | `pTarget` |
| `_AREA_DATA` | `+0x2C` | `pRoll` |
| `_AREA_DATA` | `+0x30` | `pFovy` |

Para repack, esses campos devem voltar a ser offsets relativos ao início do arquivo.

---

# Arquivos analisados

| Arquivo | Tamanho | Versão | `nArea` | `nCutCamera` | `nLerp` | `_CUT_INFO` | `_CAMERA_DATA` | `_AREA_DATA` | `_LERP_DATA` |
|---|---:|---|---:|---:|---:|---:|---:|---:|---:|
| `r100_000.cam` | `0x35E0` | `B404` | `15` | `15` | `1` | `0x0010` | `0x0100` | `0x03D0` | `0x06DC` |
| `r10a_000.CAM` | `0x1CA0` | `B404` | `7` | `9` | `0` | `0x0010` | `0x00A0` | `0x0250` | `0x03BC` |
| `r108_000.CAM` | `0x1B40` | `B404` | `16` | `16` | `0` | `0x0010` | `0x0110` | `0x0410` | `0x0750` |
| `r105_000.CAM` | `0x1EE0` | `B404` | `18` | `19` | `0` | `0x0010` | `0x0140` | `0x04D0` | `0x0878` |

---

## Valores observados nos arquivos analisados

| Campo | Valores observados |
|---|---|
| `Version` | `B404` |
| `CUT_INFO.Attr` | `0x03`, `0x04`, `0x23`, `0x43`, `0x63` |
| `CAMERA_DATA.Be_flag` | `0x01` |
| `CAMERA_DATA.No` | `1..17`, `20` |
| `CAMERA_DATA.Id` | `0`, `1`, `2` |
| `CAMERA_DATA.Attr` | `0x03`, `0x04`, `0x23`, `0x43`, `0x63` |
| `CAMERA_DATA.nPoint` | `4`, `5`, `6`, `8` |
| `AREA_DATA.Be_flag` | `0x01` |
| `AREA_DATA.No` | `1..18`, `20` |
| `AREA_DATA.Suffix` | `0`, `6`, `8` |
| `AREA_DATA.Attr` | `0x00` |
| `AREA_DATA.nVer` | `0`, `1`, `2`, `24`, `25` |
| `LERP_DATA` | `nLerp = 0` na maioria; `nLerp = 1` em `r100_000.cam` |

---

# Regras para editor/repacker

## Validação mínima

Um `.CAM` válido deve cumprir:

1. Tamanho mínimo de `0x10`.
2. `Version` igual a `B402`, `B403` ou `B404`.
3. Tabelas fixas cabem dentro do arquivo.
4. Cada `CUT_INFO.pCdat` aponta para uma entrada `_CAMERA_DATA`.
5. Cada `CUT_INFO.pAdat` aponta para uma entrada `_AREA_DATA` ou é `0`.
6. Cada `CAMERA_DATA.pData` é `0` ou offset válido.
7. Cada ponteiro de `AREA_DATA` é `0`, offset válido ou campo dependente do tipo quando documentado.
8. `nPoint` e `nVer` não podem exceder o payload disponível.

---

## Repack seguro

Ao salvar:

1. Escrever o header com `B404`.
2. Escrever `_CUT_INFO[]`.
3. Escrever `_CAMERA_DATA[]`.
4. Escrever `_AREA_DATA[]`.
5. Escrever `_LERP_DATA[]`.
6. Escrever payloads em ordem.
7. Recalcular todos os offsets relativos.
8. Manter alinhamento de 4 bytes.
9. Preservar bytes desconhecidos em `reserved`, `param04` e campos dependentes do modo.
10. Não converter offsets para endereços absolutos no arquivo salvo.

---

## Campos que podem ser editados com segurança

| Campo | Segurança | Observação |
|---|---|---|
| `CAMERA_DATA.ValueA` | Alta | Float simples |
| `CAMERA_DATA.ValueB` | Alta | Float simples |
| `CAMERA_DATA.pData` payload | Média | Requer atualizar `nPoint` |
| `AREA_DATA.Height` | Alta | Altura/valor vertical |
| `AREA_DATA.Dir` | Média | Direção/ângulo |
| `AREA_DATA.pCampos/pVer` | Média | Requer atualizar `nVer` |
| `AREA_DATA.pFovy` | Média | Lista de floats |
| `AREA_DATA.pRoll` | Média | Lista de floats |
| `Attr` | Baixa | Melhor copiar valores existentes |
| `reserved/param04` | Baixa | Preservar |

---

# YAML recomendado para edição

Exemplo de representação editável:

```yaml
version: B404

cuts:
  - index: 0
    attr: 0x03
    camera_index: 0
    area_index: 0

cameras:
  - index: 0
    be_flag: 1
    no: 1
    id: 0
    attr: 0x03
    param04_hex: "0000000001ff00000000000000000000000000000000000000000000"
    value_a: 8300.0
    value_b: -300.0
    points:
      - [12369.96, -300.0, -16497.55]
      - [11298.41, -300.0, -3987.67]
      - [40796.04, -300.0, 4087.89]
      - [47747.90, -300.0, -6149.38]

areas:
  - index: 0
    be_flag: 1
    no: 1
    suffix: 8
    attr: 0
    dir: 0.0
    height: 1000.0
    n_ver: 1
    campos:
      - [0.0, 0.0, 0.0, 1.0]
    target: []
    roll: []
    fovy: []

lerp:
  - enabled: false
```

Para repack byte-perfect, guardar também os campos desconhecidos em hexadecimal.

---

# Parser Python de referência

```python
from pathlib import Path
import struct

def u8(data, off):
    return data[off]

def i8(data, off):
    return struct.unpack_from("<b", data, off)[0]

def u16(data, off):
    return struct.unpack_from("<H", data, off)[0]

def u32(data, off):
    return struct.unpack_from("<I", data, off)[0]

def i32(data, off):
    return struct.unpack_from("<i", data, off)[0]

def f32(data, off):
    return struct.unpack_from("<f", data, off)[0]

def is_valid_offset(data, off):
    return 0 <= off < len(data)

def parse_cam(path):
    data = Path(path).read_bytes()

    if len(data) < 0x10:
        raise ValueError("arquivo pequeno demais")

    version = data[0:4].decode("ascii", "replace")
    if version not in {"B402", "B403", "B404"}:
        raise ValueError(f"versao CAM nao suportada: {version}")

    n_area = u8(data, 0x04)
    n_cut_camera = u8(data, 0x05)
    n_lerp = u8(data, 0x06)

    cut_off = 0x10
    cam_off = cut_off + n_cut_camera * 0x10
    area_off = cam_off + n_cut_camera * 0x30
    lerp_off = area_off + n_area * 0x34

    fixed_end = lerp_off + n_lerp * 0x10
    if fixed_end > len(data):
        raise ValueError("tabelas fixas excedem o tamanho do arquivo")

    cuts = []
    for i in range(n_cut_camera):
        off = cut_off + i * 0x10
        cuts.append({
            "index": i,
            "offset": off,
            "attr": u8(data, off + 0x00),
            "reserved": data[off + 0x01:off + 0x08].hex(),
            "p_cdat": u32(data, off + 0x08),
            "p_adat": u32(data, off + 0x0C),
        })

    cameras = []
    for i in range(n_cut_camera):
        off = cam_off + i * 0x30
        p_data = u32(data, off + 0x2C)
        n_point = u32(data, off + 0x28)

        cameras.append({
            "index": i,
            "offset": off,
            "be_flag": u8(data, off + 0x00),
            "no": i8(data, off + 0x01),
            "id": i8(data, off + 0x02),
            "attr": u8(data, off + 0x03),
            "param04": data[off + 0x04:off + 0x20].hex(),
            "value_a": f32(data, off + 0x20),
            "value_b": f32(data, off + 0x24),
            "n_point": n_point,
            "p_data": p_data,
            "p_data_valid": is_valid_offset(data, p_data),
        })

    areas = []
    for i in range(n_area):
        off = area_off + i * 0x34
        ptrs = {
            "p_extra": u32(data, off + 0x10),
            "p_campos": u32(data, off + 0x24),
            "p_target": u32(data, off + 0x28),
            "p_roll": u32(data, off + 0x2C),
            "p_fovy": u32(data, off + 0x30),
        }

        areas.append({
            "index": i,
            "offset": off,
            "be_flag": u8(data, off + 0x00),
            "no": i8(data, off + 0x01),
            "suffix": i8(data, off + 0x02),
            "attr": u8(data, off + 0x03),
            "dir": f32(data, off + 0x04),
            "height": f32(data, off + 0x08),
            "mode0": u32(data, off + 0x0C),
            "reserved": data[off + 0x14:off + 0x20].hex(),
            "n_ver": u32(data, off + 0x20),
            "ptrs": ptrs,
            "ptrs_valid": {
                name: (value == 0 or is_valid_offset(data, value))
                for name, value in ptrs.items()
            },
        })

    lerps = []
    for i in range(n_lerp):
        off = lerp_off + i * 0x10
        lerps.append({
            "index": i,
            "offset": off,
            "be_flag": u8(data, off + 0x00),
            "src_no": i8(data, off + 0x01),
            "src_suffix": i8(data, off + 0x02),
            "dst_no": i8(data, off + 0x03),
            "dst_suffix": i8(data, off + 0x04),
            "attr": u8(data, off + 0x05),
            "dummy": u16(data, off + 0x06),
            "inter_frame": i32(data, off + 0x08),
            "padding": i32(data, off + 0x0C),
        })

    return {
        "version": version,
        "size": len(data),
        "n_area": n_area,
        "n_cut_camera": n_cut_camera,
        "n_lerp": n_lerp,
        "offsets": {
            "cut_info": cut_off,
            "camera_data": cam_off,
            "area_data": area_off,
            "lerp_data": lerp_off,
            "fixed_end": fixed_end,
        },
        "cuts": cuts,
        "cameras": cameras,
        "areas": areas,
        "lerp": lerps,
    }

if __name__ == "__main__":
    import sys
    parsed = parse_cam(sys.argv[1])
    print("version:", parsed["version"])
    print("size:", hex(parsed["size"]))
    print("areas:", parsed["n_area"])
    print("cuts/cameras:", parsed["n_cut_camera"])
    print("lerp:", parsed["n_lerp"])
    print("offsets:", {k: hex(v) for k, v in parsed["offsets"].items()})
```

---

# Checklist de roundtrip

Para validar uma ferramenta de edição/repack:

- [ ] Ler `Version`, `nArea`, `nCutCamera`, `nLerp`.
- [ ] Calcular corretamente offsets das tabelas fixas.
- [ ] Ler todos os `CUT_INFO`.
- [ ] Resolver `pCdat` para câmera.
- [ ] Resolver `pAdat` para área.
- [ ] Ler todos os `CAMERA_DATA`.
- [ ] Preservar `param04`.
- [ ] Ler e salvar payloads de câmera.
- [ ] Ler todos os `AREA_DATA`.
- [ ] Preservar `pExtra` quando não interpretado.
- [ ] Ler e salvar listas `pCampos`, `pTarget`, `pRoll`, `pFovy`.
- [ ] Ler `LERP_DATA` quando `nLerp > 0`.
- [ ] Recalcular todos os offsets no repack.
- [ ] Manter alinhamento de 4 bytes.
- [ ] Comparar arquivo original e reempacotado sem alterações.
- [ ] Testar no jogo e em sala com troca real de câmera.

---

# Notas de implementação

- O arquivo salvo deve conter offsets relativos, não endereços absolutos.
- `calcAddr` modifica o buffer em memória. Não use dumps de RAM como arquivo final sem desfazer a relocalização.
- Os campos reservados podem conter texto ou bytes de debug em builds específicos. Isso não significa que sejam strings funcionais do formato.
- `Attr` deve ser tratado como bitfield, não como enum fechado.
- `AREA_DATA.pExtra` deve ser preservado byte a byte quando não houver interpretação específica.
- Para adicionar câmeras, atualize `nCutCamera`, crie uma entrada `_CUT_INFO`, crie uma entrada `_CAMERA_DATA` e recalcule todos os offsets posteriores.
- Para adicionar áreas, atualize `nArea`, crie uma entrada `_AREA_DATA`, adicione os payloads necessários e recalcule os offsets posteriores.
- Para alterar trajetórias, prefira editar os pontos em `pData`, `pCampos` e `pTarget`, mantendo `Attr` e blocos dependentes do modo.
