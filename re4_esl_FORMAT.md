# Resident Evil 4 PS2 — Formato `.ESL`

Documentação técnica do formato `.ESL` usado pelo sistema de enemy set do **Resident Evil 4** de PlayStation 2.

O `.ESL` armazena listas de inimigos por sala: quais inimigos existem, em qual sala devem ser criados, posição inicial, rotação inicial, HP, flags de comportamento e dados auxiliares usados pelo sistema `EmSet`.

## Sumário

- [1. Características gerais](#1-características-gerais)
- [2. Estrutura global](#2-estrutura-global)
- [3. Entrada `EM_LIST`](#3-entrada-em_list)
- [4. Campos da entrada](#4-campos-da-entrada)
- [5. Coordenadas, rotação e escala interna](#5-coordenadas-rotação-e-escala-interna)
- [6. Campo `room`](#6-campo-room)
- [7. Campo `be_flag`](#7-campo-be_flag)
- [8. Bancos de `.ESL`](#8-bancos-de-esl)
- [9. Seleção automática de banco por sala](#9-seleção-automática-de-banco-por-sala)
- [10. Fluxo de carregamento](#10-fluxo-de-carregamento)
- [11. Fluxo de criação de inimigos](#11-fluxo-de-criação-de-inimigos)
- [12. Funções relevantes no executável](#12-funções-relevantes-no-executável)
- [13. Regras para editores e repackers](#13-regras-para-editores-e-repackers)
- [14. Representação JSON/YAML recomendada](#14-representação-jsonyaml-recomendada)
- [15. Parser de referência em Python](#15-parser-de-referência-em-python)
- [16. Checklist de roundtrip](#16-checklist-de-roundtrip)
- [17. Resumo de offsets](#17-resumo-de-offsets)
- [18. Valores aceitos por campo](#18-valores-aceitos-por-campo)

---

## 1. Características gerais

| Propriedade | Valor |
|---|---:|
| Assinatura | Não possui |
| Header | Não possui |
| Endianness | little-endian |
| Tamanho da entrada | `0x20` bytes |
| Estrutura principal | `EM_LIST` |
| Slots máximos em runtime | `256` |
| Tamanho máximo carregado em runtime | `0x2000` bytes |
| Sistema runtime | `EmSet` / enemy set |
| Plataforma analisada | PlayStation 2 |

Diferente de formatos como `.AEV` e `.ITA`, o `.ESL` não possui header próprio com assinatura, versão ou contagem. O jogo trata o arquivo como uma tabela plana de estruturas `EM_LIST`.

A estrutura debug do executável define `EM_LIST` com tamanho fixo de `32` bytes:

```text
EM_LIST:T = s32
```

A área de trabalho usada pelo jogo comporta `256` entradas:

```text
256 * 0x20 = 0x2000 bytes
```

Em caso de falha de leitura, o jogo limpa `0x2000` bytes do buffer de enemy list, preservando a expectativa de tabela fixa.

---

## 2. Estrutura global

```text
Arquivo .ESL
└─ EM_LIST[n]
   ├─ EM_LIST[0]      0x20 bytes
   ├─ EM_LIST[1]      0x20 bytes
   ├─ EM_LIST[2]      0x20 bytes
   └─ ...
```

Formato recomendado para validação:

```text
entry_size = 0x20
max_entries = 256
max_size = 0x2000
```

Um arquivo pode ser considerado inválido quando:

```text
file_size == 0
file_size % 0x20 != 0
file_size > 0x2000
```

Para repack seguro, o tamanho mais compatível é `0x2000`, mantendo exatamente `256` slots. Entradas vazias devem ser preservadas ou zeradas conforme o estado original do arquivo.

---

## 3. Entrada `EM_LIST`

Cada entrada possui exatamente `0x20` bytes.

Estrutura equivalente:

```c
#pragma pack(push, 1)
typedef struct S16VEC3 {
    int16_t x;
    int16_t y;
    int16_t z;
} S16VEC3;

typedef struct EM_LIST {
    uint8_t  be_flag;      // +0x00
    uint8_t  id;           // +0x01
    uint8_t  type;         // +0x02
    uint8_t  set;          // +0x03
    uint32_t flag;         // +0x04
    uint16_t hp;           // +0x08
    uint8_t  emset_no;     // +0x0A
    uint8_t  Character;    // +0x0B
    S16VEC3  s_pos;        // +0x0C
    S16VEC3  s_ang;        // +0x12
    uint16_t room;         // +0x18
    int16_t  Guard_r;      // +0x1A
    uint32_t Dummy;        // +0x1C
} EM_LIST;
#pragma pack(pop)
```

Layout físico:

```text
Offset  00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
        BF ID TY ST FL FL FL FL HP HP ES CH PX PX PY PY

Offset  10 11 12 13 14 15 16 17 18 19 1A 1B 1C 1D 1E 1F
        PZ PZ AX AX AY AY AZ AZ RM RM GR GR DD DD DD DD
```

---

## 4. Campos da entrada

| Offset | Tamanho | Tipo | Campo | Descrição |
|---:|---:|---|---|---|
| `0x00` | 1 | `uint8` | `be_flag` | Flags de estado da entrada. Controla ativação, spawn e estado runtime. |
| `0x01` | 1 | `uint8` | `id` | ID/classe do inimigo criado pelo manager de `cEm`. Valor `0` normalmente indica slot sem inimigo. |
| `0x02` | 1 | `uint8` | `type` | Subtipo/variação do inimigo. Copiado para o objeto `cEm` durante o spawn. |
| `0x03` | 1 | `uint8` | `set` | Set/variação adicional. Também copiado para o objeto `cEm`. |
| `0x04` | 4 | `uint32` | `flag` | Flags específicas do inimigo. Copiadas para o estado runtime do `cEm`. |
| `0x08` | 2 | `uint16` | `hp` | HP inicial. Copiado para HP atual e HP máximo no objeto criado. |
| `0x0A` | 1 | `uint8` | `emset_no` | Número de set do enemy list. Usado como identificador auxiliar de agrupamento/controle. |
| `0x0B` | 1 | `uint8` | `Character` | Variante de personagem/modelo/comportamento. Copiado para o estado runtime. |
| `0x0C` | 6 | `S16VEC3` | `s_pos` | Posição inicial compactada em três `int16`. |
| `0x12` | 6 | `S16VEC3` | `s_ang` | Rotação inicial compactada em três `int16`. |
| `0x18` | 2 | `uint16` | `room` | Sala onde a entrada é válida. Usa estágio no byte alto e sala no byte baixo. |
| `0x1A` | 2 | `int16` | `Guard_r` | Raio/alcance de guarda. Convertido para escala float em runtime. |
| `0x1C` | 4 | `uint32` | `Dummy` | Campo reservado/auxiliar. Preservar em editores. |

### Campos copiados no spawn

Durante `EmSetFromList` e `EmSetFromList2_sub`, o jogo copia os principais campos para o objeto `cEm` recém-criado:

| Campo `.ESL` | Uso runtime observado |
|---|---|
| `id` | Parâmetro de criação do manager de inimigos. |
| `type` | Copiado para campo interno do `cEm`. |
| `set` | Copiado para campo interno do `cEm`. |
| `flag` | Copiado para flags runtime do inimigo. |
| `hp` | Copiado para HP atual e HP máximo. |
| `Character` | Copiado para campo interno de personagem/variante. |
| `s_pos` | Convertido para posição float do inimigo. |
| `s_ang` | Convertido para rotação float do inimigo. |
| `Guard_r` | Convertido para raio float. |
| índice da entrada | Gravado no `cEm` como índice de origem do enemy list. |

---

## 5. Coordenadas, rotação e escala interna

Os campos `s_pos`, `s_ang` e `Guard_r` são armazenados como inteiros compactados.

### `s_pos`

`S16VEC3`, tamanho `0x06` bytes:

| Offset local | Tipo | Campo |
|---:|---|---|
| `+0x00` | `int16` | `x` |
| `+0x02` | `int16` | `y` |
| `+0x04` | `int16` | `z` |

Conversão usada no runtime:

```c
world_x = s_pos.x * 10.0f;
world_y = s_pos.y * 10.0f;
world_z = s_pos.z * 10.0f;
```

Portanto, para converter posição do mundo para `.ESL`:

```c
s_pos.x = round(world_x / 10.0f);
s_pos.y = round(world_y / 10.0f);
s_pos.z = round(world_z / 10.0f);
```

### `s_ang`

`S16VEC3`, tamanho `0x06` bytes:

| Offset local | Tipo | Campo |
|---:|---|---|
| `+0x00` | `int16` | `x` |
| `+0x02` | `int16` | `y` |
| `+0x04` | `int16` | `z` |

Conversão usada no runtime:

```c
angle_rad = s_ang * 0.0001917476038f;
```

O fator corresponde a:

```text
PI / 16384
```

Fórmulas recomendadas:

```c
radians = raw_angle * (PI / 16384.0f);
raw_angle = round(radians * (16384.0f / PI));
```

Para graus:

```c
degrees = raw_angle * (180.0f / 16384.0f);
raw_angle = round(degrees * (16384.0f / 180.0f));
```

### `Guard_r`

Conversão usada no runtime:

```c
guard_radius = Guard_r * 1000.0f;
```

Para converter de volta:

```c
Guard_r = round(guard_radius / 1000.0f);
```

---

## 6. Campo `room`

O campo `room` é um `uint16` little-endian.

O jogo interpreta o valor como:

```c
stage = room >> 8;
room_no = room & 0xFF;
```

Como o arquivo é little-endian, os bytes aparecem fisicamente como:

```text
Offset +0x18 = room_no
Offset +0x19 = stage
```

Exemplo:

```text
room = 0x010C
stage = 0x01
room_no = 0x0C
```

Na comparação de spawn, a entrada só pode ser criada quando:

```c
entry.stage == current_stage
entry.room_no == current_room
```

Campos de sala fora da sala atual são ignorados durante `EmSetFromList` e `EmSetFromList2`.

---

## 7. Campo `be_flag`

`be_flag` é o principal campo de estado da entrada. O nome vem da estrutura debug `EM_LIST`.

| Bit | Máscara | Uso observado |
|---:|---:|---|
| 0 | `0x01` | Entrada habilitada. `EmSetFromList` ignora entradas sem esse bit. |
| 1 | `0x02` | Entrada já criada/spawnada na sala atual. O room init limpa esse bit. |
| 2 | `0x04` | Estado runtime alternado após criação. Preservar ao editar. |
| 3 | `0x08` | Estado runtime alternado após criação. Preservar ao editar. |
| 4..7 | `0xF0` | Não documentado pelo fluxo principal. Preservar. |

### Comportamento observado

`EmSetFromList` faz as seguintes checagens principais:

```c
if ((be_flag & 0x01) == 0)
    skip;

if ((be_flag & 0x02) != 0)
    skip;
```

Depois de criar o inimigo, o jogo marca a entrada como já criada:

```c
be_flag |= 0x02;
```

Durante `EmSetRoomInit`, o bit `0x02` é limpo para todos os `256` slots:

```c
be_flag &= ~0x02;
```

Para arquivos editáveis, recomenda-se tratar:

```text
be_flag & 0x01 != 0  -> entrada ativa
be_flag & 0x01 == 0  -> entrada inativa/vazia
```

Os demais bits devem ser preservados no roundtrip, exceto quando a ferramenta tiver uma opção explícita para resetar estado runtime.

---

## 8. Bancos de `.ESL`

O executável possui `19` bancos de enemy list. A função `getEmListNum` retorna `0x13`.

A tabela abaixo mostra o índice interno, o ID de arquivo usado pelo sistema de DVD e o nome debug associado ao banco.

| Índice | File ID | Nome debug |
|---:|---:|---|
| `0` | `0x0426` | `1st-1` |
| `1` | `0x0427` | `1st-2` |
| `2` | `0x0428` | `2st-1` |
| `3` | `0x0429` | `2st-2` |
| `4` | `0x042A` | `2st-3` |
| `5` | `0x042B` | `2st-4` |
| `6` | `0x042C` | `3st-1` |
| `7` | `0x042D` | `3st-2` |
| `8` | `0x042E` | `dummy` |
| `9` | `0x042F` | `biox` |
| `10` | `0x043B` | `ada` |
| `11` | `0x043C` | `etc` |
| `12` | `0x043D` | `etc2` |
| `13` | `0x043E` | `ps2-st1` |
| `14` | `0x043F` | `ps2-st2` |
| `15` | `0x0440` | `ps2-st3` |
| `16` | `0x0441` | `ps2-st4` |
| `17` | `0x0442` | `ps2-st5` |
| `18` | `0x0443` | `ps2-tgs` |

Esses nomes são labels de debug e não necessariamente correspondem ao nome físico do arquivo dentro do pacote de dados. O jogo acessa os recursos por File ID.

---

## 9. Seleção automática de banco por sala

A função `readEmList` chama uma rotina de seleção que recebe o identificador da sala atual e retorna o índice do banco `.ESL`.

O identificador de sala segue o mesmo formato do campo `room`:

```c
room_id = (stage << 8) | room_no;
```

### Pseudocódigo simplificado

```c
int checkEmListNo(uint16_t room_id) {
    uint8_t stage = room_id >> 8;

    switch (stage) {
    case 1:
        if (room_id == 0x0120) return 0;
        if (room_id == 0x010E) return flag_0x2000 ? 1 : -1;
        return (room_id < 0x010C) ? 0 : 1;

    case 2:
        if (room_id == 0x022B || room_id == 0x022C || room_id == 0x022D)
            return -1;

        if (room_id == 0x0200)
            return flag_0x00800000 ? 2 : 1;

        if (room_id < 0x0211) {
            if (special_castle_flag)
                return 4;

            if ((room_id >= 0x0200 && room_id < 0x0205) ||
                (room_id >= 0x0207 && room_id < 0x0209))
                return flag_0x00040000 ? 3 : 2;

            return 3;
        }

        if (room_id < 0x021A)
            return 4;

        return (room_id == 0x0222) ? 4 : 5;

    case 3:
        if (room_id == 0x0334) return 6;
        return (room_id < 0x0315) ? 6 : 7;

    case 4:
        if (room_id < 0x0403) return 11;
        if (room_id < 0x0405) return 12;
        return 10;

    case 5:
        if (room_id < 0x0504) return 13;
        if (room_id < 0x050C) return 14;
        if (room_id < 0x0511) return 15;
        if (room_id < 0x0518) return 16;
        if (room_id < 0x051F) return 17;
        return 18;

    case 6:
        return 18;

    case 7:
        return 9;

    default:
        return -1;
    }
}
```

Há caminhos condicionais por flags globais do jogo, especialmente para ramificações de progresso, modos especiais e variações de sala. Por isso, editores devem permitir selecionar manualmente o banco `.ESL` além de oferecer seleção automática por `stage/room`.

---

## 10. Fluxo de carregamento

Fluxo principal:

```text
StageSet
└─ readEmList(1)
   ├─ calcula o índice do banco pelo room atual
   ├─ compara com o banco já carregado
   ├─ resolve File ID por getEmListNo(index)
   ├─ carrega o recurso para o buffer Em_list
   └─ em caso de falha, limpa 0x2000 bytes do buffer
```

O buffer de destino é a tabela runtime de `EM_LIST`:

```text
Em_list[256]
```

Tamanho total:

```text
0x20 * 256 = 0x2000
```

O jogo guarda o índice do banco atualmente carregado em um campo runtime de controle (`em_list_no`). Quando o índice calculado muda, uma nova lista é carregada.

---

## 11. Fluxo de criação de inimigos

### `EmSetFromList`

Percorre todos os `256` slots da tabela carregada:

```text
for i in 0..255:
    entry = Em_list[i]

    if entry.be_flag bit 0 não está ligado:
        ignora

    if entry.be_flag bit 1 está ligado:
        ignora

    if entry.room não corresponde à sala atual:
        ignora

    if entrada já foi marcada como morta/persistida:
        ignora

    cria inimigo pelo entry.id
    copia type, set, flag, hp, Character
    converte posição, rotação e Guard_r
    grava o índice i no cEm
    marca a entrada como spawnada
```

### `EmSetFromList2`

Cria uma entrada específica por índice:

```c
cEm* EmSetFromList2(uint32_t list_index, uint32_t check_dead_flag);
```

Uso típico:

- spawn controlado por script;
- spawn tardio;
- spawn de inimigo específico sem percorrer a lista inteira.

### `EmSetEvent`

Cria inimigo a partir de um ponteiro direto para `EM_LIST`, sem depender do índice normal da tabela.

Uso típico:

- eventos;
- scripts;
- spawns especiais;
- inimigos criados fora do fluxo comum de sala.

---

## 12. Funções relevantes no executável

| Endereço | Função | Descrição |
|---:|---|---|
| `0x002A7530` | `readEmList__FUi` | Seleciona e carrega o banco `.ESL` atual. |
| `0x002A7140` | `checkEmListNo__FUs` | Calcula o índice do banco por sala/progresso. |
| `0x002A7390` | `getEmListNo__Fi` | Converte índice de banco em File ID. |
| `0x002A73A8` | `getEmListDbgName__Fi` | Retorna o nome debug do banco. |
| `0x002A73E8` | `getEmListNum__Fv` | Retorna `0x13`, total de bancos. |
| `0x001D87B8` | `EmSetFromList__Fv` | Percorre `Em_list[256]` e cria inimigos válidos para a sala atual. |
| `0x001D8B00` | `EmSetFromList2__FUiUi` | Wrapper para criação por índice de lista. |
| `0x001D8B58` | `EmSetFromList2_sub__FUiUi` | Implementação de criação por índice. |
| `0x001D8E68` | `EmSetEvent__FP7EM_LIST` | Cria inimigo por ponteiro direto de `EM_LIST`. |
| `0x001D90A0` | `GetEmPtrFromList__FUi` | Localiza `cEm` criado a partir de um índice `.ESL`. |
| `0x001D9130` | `GetListPtrFromEm__FP3cEm` | Retorna o `EM_LIST` de origem de um `cEm`. |
| `0x001D9160` | `GetEmIdFromList__FUi` | Retorna o campo `id` de uma entrada. |
| `0x001D9190` | `EmListSetAlive__FUib` | Liga/desliga o bit de ativo da entrada. |
| `0x001D91F0` | `EmListIsAlive__FUi` | Testa se a entrada está ativa. |
| `0x001D9218` | `EmSetDie__FP3cEm` | Marca persistência de morte/estado de inimigo. |
| `0x001D9378` | `EmSetRoomInit__Fv` | Limpa o bit de spawn runtime das entradas. |
| `0x001D93B8` | `EmListWaitDelete__Fv` | Rotina auxiliar de limpeza/espera de enemy list. |
| `0x002A0EE8` | `SceCountEmAlive__Fii` | Conta inimigos vivos. |
| `0x002A0FB8` | `SceCountEmAliveGanado__Fv` | Conta Ganados vivos. |
| `0x002A1070` | `SceDestroyEm__Fii` | Destrói inimigos por critérios de script. |

---

## 13. Regras para editores e repackers

### Preservar tamanho e alinhamento

O tamanho mais seguro para exportação é:

```text
0x2000 bytes
```

Isto preserva os `256` slots esperados pelo runtime.

### Não adicionar header

O `.ESL` não deve receber assinatura, versão, contagem ou padding extra no início.

Errado:

```text
ESL\0 + count + entries
```

Correto:

```text
EM_LIST[0] + EM_LIST[1] + ...
```

### Preservar slots vazios

Slots vazios normalmente possuem:

```text
be_flag = 0
id      = 0
```

Não compacte a tabela removendo entradas vazias, pois scripts e spawns por índice podem depender da posição do slot.

### Preservar campos desconhecidos

Preservar sempre:

```text
be_flag bits 0x04, 0x08, 0xF0
Dummy
flag
emset_no
Character
```

Mesmo quando a ferramenta não souber editar semanticamente esses campos.

### Validar limites de conversão

Campos `int16` devem permanecer na faixa:

```text
-32768..32767
```

Antes de exportar:

```text
s_pos = round(world_pos / 10.0)
s_ang = round(angle_rad * 16384 / PI)
Guard_r = round(guard_radius / 1000.0)
```

### Não alterar índice sem atualizar referências

O índice da entrada é usado em runtime para localizar o inimigo criado a partir do `.ESL`. Se uma ferramenta mover entradas, ela deve atualizar qualquer referência externa que use índice de enemy list.

Para edição segura:

```text
preferir editar in-place
não reordenar por padrão
não remover slots intermediários
```

---

## 14. Representação JSON/YAML recomendada

Exemplo em YAML:

```yaml
format: ESL
platform: ps2
endianness: little
entry_size: 0x20
max_entries: 256
bank:
  index: 0
  file_id: 0x0426
  debug_name: 1st-1
entries:
  - index: 0
    enabled: true
    raw:
      be_flag: 0x01
      id: 0x10
      type: 0x00
      set: 0x00
      flag: 0x00000000
      hp: 300
      emset_no: 0
      character: 0
      room: 0x0100
      guard_r: 2
      dummy: 0x00000000
    transform:
      position_raw: [0, 0, 0]
      position_world: [0.0, 0.0, 0.0]
      rotation_raw: [0, 8192, 0]
      rotation_degrees: [0.0, 90.0, 0.0]
      guard_radius_world: 2000.0
```

Campos recomendados para uma ferramenta visual:

| Campo visual | Campo binário |
|---|---|
| Ativo | `be_flag & 0x01` |
| Spawnado runtime | `be_flag & 0x02` |
| Enemy ID | `id` |
| Tipo | `type` |
| Set | `set` |
| Flags | `flag` |
| HP | `hp` |
| Sala | `room` |
| Stage | `room >> 8` |
| Room No | `room & 0xFF` |
| Posição | `s_pos * 10.0` |
| Rotação | `s_ang * 180.0 / 16384.0` |
| Guard Radius | `Guard_r * 1000.0` |

---

## 15. Parser de referência em Python

```python
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path
import math
import struct

ENTRY_SIZE = 0x20
MAX_ENTRIES = 256
MAX_SIZE = ENTRY_SIZE * MAX_ENTRIES
ANGLE_TO_RAD = math.pi / 16384.0


@dataclass
class S16Vec3:
    x: int
    y: int
    z: int

    def to_world_pos(self) -> tuple[float, float, float]:
        return (self.x * 10.0, self.y * 10.0, self.z * 10.0)

    def to_degrees(self) -> tuple[float, float, float]:
        factor = 180.0 / 16384.0
        return (self.x * factor, self.y * factor, self.z * factor)


@dataclass
class EmListEntry:
    index: int
    be_flag: int
    enemy_id: int
    enemy_type: int
    enemy_set: int
    flag: int
    hp: int
    emset_no: int
    character: int
    s_pos: S16Vec3
    s_ang: S16Vec3
    room: int
    guard_r: int
    dummy: int

    @property
    def enabled(self) -> bool:
        return bool(self.be_flag & 0x01)

    @property
    def spawned_runtime(self) -> bool:
        return bool(self.be_flag & 0x02)

    @property
    def stage(self) -> int:
        return (self.room >> 8) & 0xFF

    @property
    def room_no(self) -> int:
        return self.room & 0xFF

    @property
    def guard_radius_world(self) -> float:
        return self.guard_r * 1000.0


def parse_esl(data: bytes) -> list[EmListEntry]:
    if not data:
        raise ValueError("arquivo vazio")

    if len(data) % ENTRY_SIZE != 0:
        raise ValueError(f"tamanho inválido: 0x{len(data):X}; esperado múltiplo de 0x20")

    if len(data) > MAX_SIZE:
        raise ValueError(f"tamanho inválido: 0x{len(data):X}; máximo esperado 0x{MAX_SIZE:X}")

    entries: list[EmListEntry] = []

    for index in range(len(data) // ENTRY_SIZE):
        off = index * ENTRY_SIZE
        chunk = data[off:off + ENTRY_SIZE]

        (
            be_flag,
            enemy_id,
            enemy_type,
            enemy_set,
            flag,
            hp,
            emset_no,
            character,
            pos_x,
            pos_y,
            pos_z,
            ang_x,
            ang_y,
            ang_z,
            room,
            guard_r,
            dummy,
        ) = struct.unpack_from("<BBBBI HBB hhh hhh Hh I", chunk, 0)

        entries.append(EmListEntry(
            index=index,
            be_flag=be_flag,
            enemy_id=enemy_id,
            enemy_type=enemy_type,
            enemy_set=enemy_set,
            flag=flag,
            hp=hp,
            emset_no=emset_no,
            character=character,
            s_pos=S16Vec3(pos_x, pos_y, pos_z),
            s_ang=S16Vec3(ang_x, ang_y, ang_z),
            room=room,
            guard_r=guard_r,
            dummy=dummy,
        ))

    return entries


def pack_esl(entries: list[EmListEntry], pad_to_0x2000: bool = True) -> bytes:
    if len(entries) > MAX_ENTRIES:
        raise ValueError("máximo de 256 entradas excedido")

    out = bytearray()

    for e in entries:
        out += struct.pack(
            "<BBBBI HBB hhh hhh Hh I",
            e.be_flag & 0xFF,
            e.enemy_id & 0xFF,
            e.enemy_type & 0xFF,
            e.enemy_set & 0xFF,
            e.flag & 0xFFFFFFFF,
            e.hp & 0xFFFF,
            e.emset_no & 0xFF,
            e.character & 0xFF,
            int(e.s_pos.x),
            int(e.s_pos.y),
            int(e.s_pos.z),
            int(e.s_ang.x),
            int(e.s_ang.y),
            int(e.s_ang.z),
            e.room & 0xFFFF,
            int(e.guard_r),
            e.dummy & 0xFFFFFFFF,
        )

    if pad_to_0x2000:
        out += b"\x00" * (MAX_SIZE - len(out))

    return bytes(out)


def main(path: str) -> None:
    data = Path(path).read_bytes()
    entries = parse_esl(data)

    print(f"entries: {len(entries)}")
    print(f"size: 0x{len(data):X}")

    for e in entries:
        if not e.enabled and e.enemy_id == 0:
            continue

        print(
            f"[{e.index:03d}] "
            f"enabled={e.enabled} "
            f"id=0x{e.enemy_id:02X} "
            f"type=0x{e.enemy_type:02X} "
            f"set=0x{e.enemy_set:02X} "
            f"hp={e.hp} "
            f"room=0x{e.room:04X} "
            f"stage={e.stage:02X} room_no={e.room_no:02X} "
            f"pos={e.s_pos.to_world_pos()} "
            f"ang_deg={e.s_ang.to_degrees()} "
            f"guard={e.guard_radius_world}"
        )


if __name__ == "__main__":
    import sys

    if len(sys.argv) != 2:
        raise SystemExit("uso: python parse_esl.py arquivo.esl")

    main(sys.argv[1])
```

---

## 16. Checklist de roundtrip

Antes de salvar um `.ESL`, validar:

```text
[ ] O arquivo não possui header adicionado artificialmente.
[ ] O tamanho é múltiplo de 0x20.
[ ] O tamanho não passa de 0x2000.
[ ] A tabela possui no máximo 256 entradas.
[ ] Slots vazios foram preservados.
[ ] A ordem dos slots não foi alterada sem necessidade.
[ ] be_flag preserva bits desconhecidos.
[ ] Dummy foi preservado.
[ ] room usa formato stage no byte alto e room no byte baixo.
[ ] s_pos está dentro de int16.
[ ] s_ang está dentro de int16.
[ ] Guard_r está dentro de int16.
[ ] Exportação final foi testada com importação imediata.
```

---

## 17. Resumo de offsets

| Offset | Tamanho | Tipo | Campo | Conversão/observação |
|---:|---:|---|---|---|
| `0x00` | 1 | `uint8` | `be_flag` | Bit `0x01` ativo; bit `0x02` spawnado runtime. |
| `0x01` | 1 | `uint8` | `id` | ID/classe do inimigo. |
| `0x02` | 1 | `uint8` | `type` | Subtipo. |
| `0x03` | 1 | `uint8` | `set` | Variação/set. |
| `0x04` | 4 | `uint32` | `flag` | Flags do inimigo. |
| `0x08` | 2 | `uint16` | `hp` | HP inicial. |
| `0x0A` | 1 | `uint8` | `emset_no` | Agrupamento/identificador auxiliar. |
| `0x0B` | 1 | `uint8` | `Character` | Variante de personagem/modelo. |
| `0x0C` | 2 | `int16` | `s_pos.x` | `x * 10.0`. |
| `0x0E` | 2 | `int16` | `s_pos.y` | `y * 10.0`. |
| `0x10` | 2 | `int16` | `s_pos.z` | `z * 10.0`. |
| `0x12` | 2 | `int16` | `s_ang.x` | `x * PI / 16384`. |
| `0x14` | 2 | `int16` | `s_ang.y` | `y * PI / 16384`. |
| `0x16` | 2 | `int16` | `s_ang.z` | `z * PI / 16384`. |
| `0x18` | 2 | `uint16` | `room` | `(stage << 8) | room_no`. |
| `0x1A` | 2 | `int16` | `Guard_r` | `Guard_r * 1000.0`. |
| `0x1C` | 4 | `uint32` | `Dummy` | Preservar. |
---

## 18. Valores aceitos por campo

Esta seção consolida os valores que cada campo da estrutura `EM_LIST` aceita. Quando o ELF fornece enumeração/tabela global, os valores são listados nominalmente. Quando o campo é apenas um inteiro copiado para o objeto runtime, a documentação informa a faixa binária aceita e a regra segura de edição.

### 18.1 Tabela geral de domínio dos campos

| Campo | Tipo | Valores aceitos | Valor seguro/padrão | Observações de edição |
|---|---|---|---|---|
| `be_flag` | `uint8` | `0x00..0xFF`, tratado como bitfield | `0x00` vazio; `0x01` ativo | Bits `0x01`, `0x02`, `0x04` e `0x08` são usados pelo fluxo `EmSet`. Preservar bits altos. |
| `id` | `uint8` | `0x00..0x62` conforme `em_id_name`; `0x63..0xFF` sem nome debug global | `0x00` vazio; `0x10+` para inimigos comuns | `id == 0` é ignorado pelo spawn comum. Usar IDs sem nome apenas se já existirem em arquivo original. |
| `type` | `uint8` | `0x00..0xFF` | `0x00` | Subtipo interpretado pela implementação do `id`. Não existe enum global único no ELF para este campo. |
| `set` | `uint8` | `0x00..0xFF` | `0x00` | Variação/set copiado para `cEm::set`. O significado depende do `id`. |
| `flag` | `uint32` | `0x00000000..0xFFFFFFFF` | Preservar original; `0x00000000` para novas entradas simples | Copiado diretamente para `cEm::flag`. Tratar como bitfield específico do inimigo. |
| `hp` | `uint16` | `0..65535` | Valor observado/original; `1+` para spawn vivo | Copiado para `cEm::hp` e `cEm::hp_max`. `0` é tecnicamente aceito, mas pode gerar inimigo sem vida útil. |
| `emset_no` | `uint8` | `0x00..0xFF` | Preservar original; `0x00` para entrada nova | Copiado para `cEm::emset_no`. Usado como identificador auxiliar de grupo/set. |
| `Character` | `uint8` | `0x00..0xFF` | Preservar original; `0x00` para entrada nova | Copiado para `cEm::Character`. Sem enum global único no ELF; significado depende do inimigo. |
| `s_pos.x/y/z` | `int16` | `-32768..32767` | Coordenada original | Unidade de arquivo = mundo / `10.0`. Faixa em mundo: `-327680.0..327670.0`. |
| `s_ang.x/y/z` | `int16` | `-32768..32767` | `0` | Unidade angular = `PI / 16384`. A faixa cobre aproximadamente `-360°..+359.989°`. |
| `room` | `uint16` | `(stage << 8) | room_no` | Sala real da entrada | `stage` no byte alto; `room_no` no byte baixo. Stages usados pelo seletor: `0x01..0x07`. |
| `Guard_r` | `int16` | `-32768..32767` | `0` ou valor original | Runtime multiplica por `1000.0`. Valores negativos são aceitos pelo tipo, mas não são recomendados para raio físico. |
| `Dummy` | `uint32` | `0x00000000..0xFFFFFFFF` | Preservar original; `0x00000000` para entrada nova | Campo reservado/auxiliar. Não zerar em roundtrip quando a entrada original possui valor. |

### 18.2 `be_flag`

`be_flag` é um bitfield. O jogo testa e altera os quatro bits baixos durante criação, morte e reinicialização da sala.

| Bit | Máscara | Nome recomendado | Leitura runtime | Escrita recomendada em arquivo |
|---:|---:|---|---|---|
| 0 | `0x01` | `ACTIVE` | Entrada habilitada para spawn. | Ligar para entrada válida; desligar para desativar sem apagar dados. |
| 1 | `0x02` | `SPAWNED` | Entrada já criada na sala atual. | Normalmente salvar desligado; o jogo liga em runtime. |
| 2 | `0x04` | `PHASE_A` | Marcador alternado pelo fluxo de spawn. | Preservar se já existir; não usar como ativador principal. |
| 3 | `0x08` | `PHASE_B` | Marcador alternado pelo fluxo de spawn. | Preservar se já existir; não usar como ativador principal. |
| 4..7 | `0xF0` | reservado | Não há uso principal confirmado no fluxo `EmSet`. | Preservar sempre. |

Combinações comuns:

| Valor | Significado prático |
|---:|---|
| `0x00` | Slot inativo/vazio. |
| `0x01` | Entrada ativa, ainda não spawnada. Melhor valor para nova entrada simples. |
| `0x03` | Ativa + já spawnada. Estado runtime, geralmente não deve ser salvo manualmente. |
| `0x05` | Ativa com marcador `PHASE_A`. Preservar quando vier do arquivo original. |
| `0x09` | Ativa com marcador `PHASE_B`. Preservar quando vier do arquivo original. |
| `0x07` | Ativa + spawnada + `PHASE_A`. Pode aparecer após o jogo criar uma entrada `0x01`. |
| `0x0B` | Ativa + spawnada + `PHASE_B`. Pode aparecer após alternância runtime. |

Comportamento de spawn:

```c
// filtro principal
if ((be_flag & 0x01) == 0) skip;
if ((be_flag & 0x02) != 0) skip;

// depois do spawn
be_flag |= 0x02;

if (be_flag & 0x04) {
    be_flag |= 0x08;
    be_flag &= ~0x04;
} else if ((be_flag & 0x08) == 0) {
    be_flag |= 0x04;
}
```

Durante `EmSetRoomInit`, o jogo limpa apenas o bit `0x02` em todos os 256 slots:

```c
be_flag &= ~0x02;
```

### 18.3 `id` / tabela `em_id_name`

A tabela global `em_id_name` possui `99` entradas, de `0x00` a `0x62`. Para edição de `.ESL`, os valores mais seguros são os IDs já usados por arquivos originais da mesma sala/banco. IDs sem nome debug podem existir como placeholders, variações removidas ou classes não usadas diretamente.

| Hex | Dec | Nome debug | Grupo | Observação |
|---:|---:|---|---|---|
| `0x00` | `0` | `PL00:PLAYER` | Player/Sub-character | Não recomendado em ESL; `id == 0` é ignorado pelo spawn comum. |
| `0x01` | `1` | `PL01:PL01` | Player/Sub-character |  |
| `0x02` | `2` | `PL02:PL02` | Player/Sub-character |  |
| `0x03` | `3` | `PL03:ASHLEY` | Player/Sub-character |  |
| `0x04` | `4` | `PL04:LUIS` | Player/Sub-character |  |
| `0x05` | `5` | `PL05:ASHLEY2` | Player/Sub-character |  |
| `0x06` | `6` | `PL06:HUNK` | Player/Sub-character |  |
| `0x07` | `7` | `PL07:POLICE` | Player/Sub-character |  |
| `0x08` | `8` | `PL08:PL08` | Player/Sub-character |  |
| `0x09` | `9` | `PL09:PL09` | Player/Sub-character |  |
| `0x0A` | `10` | `PL0A:KLAUSER` | Player/Sub-character |  |
| `0x0B` | `11` | `PL0B:PL0B` | Player/Sub-character |  |
| `0x0C` | `12` | `PL0C:PL0C` | Player/Sub-character |  |
| `0x0D` | `13` | `PL0D:WESKER` | Player/Sub-character |  |
| `0x0E` | `14` | `PL0E:PL_JETSKI` | Player/Sub-character |  |
| `0x0F` | `15` | `PL0F:PL_BOAT` | Player/Sub-character | Criado por `createBack` no fluxo `EmSet`. |
| `0x10` | `16` | `EM10:GANADO10` | Enemy/creature |  |
| `0x11` | `17` | `EM11:GANADO11` | Enemy/creature |  |
| `0x12` | `18` | `EM12:GANADO12` | Enemy/creature |  |
| `0x13` | `19` | `EM13:GANADO13` | Enemy/creature |  |
| `0x14` | `20` | `EM14:GANADO14` | Enemy/creature |  |
| `0x15` | `21` | `EM15:GANADO15` | Enemy/creature |  |
| `0x16` | `22` | `EM16:GANADO16` | Enemy/creature |  |
| `0x17` | `23` | `EM17:GANADO17` | Enemy/creature |  |
| `0x18` | `24` | `EM18:GANADO18` | Enemy/creature |  |
| `0x19` | `25` | `EM19:GANADO19` | Enemy/creature |  |
| `0x1A` | `26` | `EM1A:GANADO1A` | Enemy/creature |  |
| `0x1B` | `27` | `EM1B:GANADO1B` | Enemy/creature |  |
| `0x1C` | `28` | `EM1C:GANADO1C` | Enemy/creature |  |
| `0x1D` | `29` | `EM1D:GANADO1D` | Enemy/creature |  |
| `0x1E` | `30` | `EM1E:GANADO1E` | Enemy/creature |  |
| `0x1F` | `31` | `EM1F:GANADO1F` | Enemy/creature |  |
| `0x20` | `32` | `EM20:GANADO20` | Enemy/creature |  |
| `0x21` | `33` | `EM21:DOG` | Enemy/creature |  |
| `0x22` | `34` | `EM22:EMDOG` | Enemy/creature |  |
| `0x23` | `35` | `EM23:CROW` | Enemy/creature |  |
| `0x24` | `36` | `EM24:SNAKE_S` | Enemy/creature |  |
| `0x25` | `37` | `EM25:PARASITE` | Enemy/creature | Criado por `createBack` no fluxo `EmSet`. |
| `0x26` | `38` | `EM26:COW` | Enemy/creature |  |
| `0x27` | `39` | `EM27:BLACKBASS` | Enemy/creature |  |
| `0x28` | `40` | `EM28:CHICKEN` | Enemy/creature |  |
| `0x29` | `41` | `EM29:BAT` | Enemy/creature |  |
| `0x2A` | `42` | `EM2a:TRAP` | Enemy/creature |  |
| `0x2B` | `43` | `EM2b:ELGIGANTE` | Enemy/creature |  |
| `0x2C` | `44` | `EM2c:INSECTBOSS` | Enemy/creature |  |
| `0x2D` | `45` | `EM2d:INSECTHUMAN` | Enemy/creature |  |
| `0x2E` | `46` | `EM2e:SPIDER_S` | Enemy/creature |  |
| `0x2F` | `47` | `EM2f:SALAMANDER` | Enemy/creature |  |
| `0x30` | `48` | `EM30:SADDLER` | Enemy/creature |  |
| `0x31` | `49` | `EM31:SADDLER_AFTER` | Enemy/creature |  |
| `0x32` | `50` | `EM32:U3` | Enemy/creature |  |
| `0x33` | `51` | `EM33:INSECTBOSS_EVENT` | Enemy/creature |  |
| `0x34` | `52` | `EM34:MAYOR` | Enemy/creature |  |
| `0x35` | `53` | `EM35:MAYOR_AFTER` | Enemy/creature |  |
| `0x36` | `54` | `EM36:REGENERATER` | Enemy/creature |  |
| `0x37` | `55` | `EM37:NO2` | Enemy/creature |  |
| `0x38` | `56` | `EM38:NO2_AFTER` | Enemy/creature |  |
| `0x39` | `57` | `EM39:NO3` | Enemy/creature |  |
| `0x3A` | `58` | `EM3a:SEEKER` | Enemy/creature |  |
| `0x3B` | `59` | `EM3b:TRUCK` | Enemy/creature |  |
| `0x3C` | `60` | `EM3c:ARMOR` | Enemy/creature |  |
| `0x3D` | `61` | `EM3d:HELICOPTER` | Enemy/creature |  |
| `0x3E` | `62` | `EM3e:MARK` | Enemy/creature |  |
| `0x3F` | `63` | `EM3f:SADDLER ADA` | Enemy/creature |  |
| `0x40` | `64` | `EM40:GANADO40` | Enemy/creature |  |
| `0x41` | `65` | `EM41:GANADO41` | Enemy/creature |  |
| `0x42` | `66` | `EM42:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x43` | `67` | `EM43:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x44` | `68` | `EM44:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x45` | `69` | `EM45:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x46` | `70` | `EM46:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x47` | `71` | `EM47:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x48` | `72` | `EM48:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x49` | `73` | `EM49:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x4A` | `74` | `EM4A:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x4B` | `75` | `EM4B:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x4C` | `76` | `EM4C:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x4D` | `77` | `EM4D:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x4E` | `78` | `EM4E:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x4F` | `79` | `EM4F:` | Enemy/creature | Nome debug vazio; manter apenas se observado em arquivo original. |
| `0x50` | `80` | `EM50:EMOBJ` | ETC/object |  |
| `0x51` | `81` | `EM51:EMDOOR` | ETC/object |  |
| `0x52` | `82` | `EM52:EMWEP` | ETC/object |  |
| `0x53` | `83` | `EM53:EMBOX` | ETC/object |  |
| `0x54` | `84` | `EM54:EMWALL` | ETC/object |  |
| `0x55` | `85` | `EM55:EMRACK` | ETC/object |  |
| `0x56` | `86` | `EM56:EMWINDOW` | ETC/object |  |
| `0x57` | `87` | `EM57:EMTORCH` | ETC/object |  |
| `0x58` | `88` | `EM58:EMBARREL` | ETC/object |  |
| `0x59` | `89` | `EM59:EMTREE` | ETC/object |  |
| `0x5A` | `90` | `EM5A:EMROCK` | ETC/object |  |
| `0x5B` | `91` | `EM5B:EMSWITCH` | ETC/object |  |
| `0x5C` | `92` | `EM5C:EMITEM` | ETC/object |  |
| `0x5D` | `93` | `EM5D:EMHIT` | ETC/object |  |
| `0x5E` | `94` | `EM5E:EMBARRED` | ETC/object |  |
| `0x5F` | `95` | `EM5F:EMMINE` | ETC/object |  |
| `0x60` | `96` | `EM60:EMSHIELD` | ETC/object |  |
| `0x61` | `97` | `EM61:EMBAR` | ETC/object |  |
| `0x62` | `98` | `EM??:` | ETC/object | Nome debug vazio; manter apenas se observado em arquivo original. |

Notas práticas:

- `0x10..0x20` são a faixa principal de Ganados listada por nome debug.
- `0x21..0x29` contém criaturas/animais e inimigos auxiliares.
- `0x2A..0x3F` contém chefes, inimigos especiais e entidades de evento.
- `0x50..0x62` contém entidades ETC/objeto usadas pelo sistema de inimigos/objetos.
- IDs `0x0F` e `0x25` seguem caminho de criação `createBack`; os demais usam `create` no fluxo normal.
- IDs `0x63..0xFF` cabem no campo, mas não possuem nome na tabela global analisada. Um editor deve bloquear por padrão ou exigir modo avançado.

### 18.4 `type`, `set` e `Character`

Esses três campos são `uint8` e não possuem enum global único associado à estrutura `EM_LIST`. O jogo apenas copia os valores para o objeto criado:

```c
cEm->type      = entry.type;
cEm->set       = entry.set;
cEm->Character = entry.Character;
```

Faixa aceita:

| Campo | Faixa | Uso recomendado |
|---|---:|---|
| `type` | `0x00..0xFF` | Manter o valor observado para o mesmo `id`; usar `0x00` em nova entrada simples. |
| `set` | `0x00..0xFF` | Manter o valor observado para o mesmo `id`; usar `0x00` em nova entrada simples. |
| `Character` | `0x00..0xFF` | Preservar; usar `0x00` quando não houver variação conhecida. |

Diretriz para ferramentas:

```text
id conhecido + type/set/Character observados em arquivo original -> permitir normalmente
id conhecido + type/set/Character novos                         -> permitir com aviso
id sem nome debug                                                -> exigir modo avançado
```

### 18.5 `flag`

`flag` é um `uint32` copiado diretamente para `cEm::flag`. O ELF não expõe uma enumeração global única para esse campo dentro da estrutura `EM_LIST`. O significado dos bits é específico da classe do inimigo/objeto.

Faixa aceita:

```text
0x00000000..0xFFFFFFFF
```

Regras de edição:

| Caso | Ação recomendada |
|---|---|
| Entrada existente | Preservar exatamente. |
| Nova entrada simples | Usar `0x00000000`. |
| Duplicar inimigo existente | Copiar `flag` da entrada original do mesmo `id`. |
| Alterar bits manualmente | Exigir modo avançado e mostrar valor hexadecimal. |

### 18.6 `hp`

`hp` é copiado para dois campos runtime:

```c
cEm->hp     = entry.hp;
cEm->hp_max = entry.hp;
```

| Valor | Efeito prático |
|---:|---|
| `0` | Aceito pelo tipo, mas não recomendado para inimigo vivo. Pode gerar entidade sem HP útil. |
| `1..65535` | Faixa normal para inimigo/objeto com vida. |

Para edição segura, manter o HP original do mesmo tipo de inimigo. Ao criar novas entradas, usar valores observados em `.ESL` original da mesma sala ou do mesmo banco.

### 18.7 `emset_no`

`emset_no` aceita qualquer `uint8`:

```text
0x00..0xFF
```

O campo é copiado para `cEm::emset_no`. A documentação debug confirma o nome, mas não há enum global associado. Tratar como agrupador/identificador auxiliar.

Regras:

- Preservar em roundtrip.
- Ao duplicar entrada, copiar o valor original.
- Para uma entrada nova simples, usar `0x00` até que uma relação de sala/script seja mapeada.

### 18.8 `s_pos`

Cada eixo aceita `int16`:

```text
raw:   -32768..32767
world: -327680.0..327670.0
step:  10.0 unidades de mundo
```

Conversão:

```c
world = raw * 10.0f;
raw   = round(world / 10.0f);
```

Tabela rápida:

| Mundo | Raw |
|---:|---:|
| `0.0` | `0` |
| `1000.0` | `100` |
| `-1000.0` | `-100` |
| `327670.0` | `32767` |
| `-327680.0` | `-32768` |

### 18.9 `s_ang`

Cada eixo aceita `int16`:

```text
raw: -32768..32767
rad: raw * PI / 16384
 deg: raw * 180 / 16384
```

Tabela rápida:

| Ângulo | Raw aproximado |
|---:|---:|
| `0°` | `0` |
| `45°` | `4096` |
| `90°` | `8192` |
| `180°` | `16384` |
| `-90°` | `-8192` |
| `-180°` | `-16384` |
| `360°` | `32768`, não representável como positivo em `int16`; usar `0` ou `-32768` conforme necessidade de equivalência. |

Para exportadores, normalizar preferencialmente para a faixa:

```text
-16384..16384  // -180°..180°
```

A faixa completa `int16` continua válida, mas pode representar giros equivalentes que dificultam comparação em diff.

### 18.10 `room`

`room` aceita qualquer `uint16`, mas o spawn comum só cria a entrada quando o valor combina com a sala atual:

```c
stage   = room >> 8;
room_no = room & 0xFF;
```

Faixa binária:

```text
0x0000..0xFFFF
```

Faixa prática do seletor de bancos analisado:

| Stage | Faixa prática / uso observado no seletor | Banco(s) `.ESL` |
|---:|---|---|
| `0x01` | `r100..r120`, com divisões internas | `1st-1`, `1st-2` |
| `0x02` | `r200..r22F`, com exceções e ramificações por flag | `2st-1`..`2st-4` |
| `0x03` | `r300..r334`, dividido em duas faixas principais | `3st-1`, `3st-2` |
| `0x04` | `r400..r4FF`, usado por Ada/ETC conforme faixa | `ada`, `etc`, `etc2` |
| `0x05` | `r500..r5FF`, bancos PS2 extras | `ps2-st1`..`ps2-st5`, `ps2-tgs` |
| `0x06` | Seleciona banco `ps2-tgs` | `ps2-tgs` |
| `0x07` | Seleciona banco `biox` | `biox` |

Valores fora dessas faixas cabem no campo, mas não são selecionados pelo fluxo normal de `checkEmListNo`.

Exemplos:

| Sala | `stage` | `room_no` | Valor `room` | Bytes no arquivo |
|---|---:|---:|---:|---|
| `r100` | `0x01` | `0x00` | `0x0100` | `00 01` |
| `r10C` | `0x01` | `0x0C` | `0x010C` | `0C 01` |
| `r21A` | `0x02` | `0x1A` | `0x021A` | `1A 02` |
| `r40E` | `0x04` | `0x0E` | `0x040E` | `0E 04` |

### 18.11 `Guard_r`

`Guard_r` é `int16` e é convertido para float no runtime:

```c
guard_radius = Guard_r * 1000.0f;
```

Faixa aceita:

```text
raw:   -32768..32767
world: -32768000.0..32767000.0
```

Uso recomendado:

| Valor | Interpretação prática |
|---:|---|
| `0` | Sem raio adicional ou raio nulo. |
| `1` | `1000.0` unidades de mundo. |
| `2` | `2000.0` unidades de mundo. |
| `10` | `10000.0` unidades de mundo. |
| `< 0` | Tecnicamente aceito, mas não recomendado. |

### 18.12 `Dummy`

`Dummy` aceita qualquer `uint32`:

```text
0x00000000..0xFFFFFFFF
```

Como o campo aparece na estrutura debug mas não possui semântica global confirmada no fluxo principal de spawn, a regra de editor é:

```text
entrada existente -> preservar
entrada nova      -> 0x00000000
```

### 18.13 Perfil de validação recomendado para editores

Para evitar arquivos que carregam mas quebram gameplay, usar duas camadas de validação.

#### Validação estrita

```text
be_flag:    aceitar 0x00 ou valores preservados do original; novas entradas usam 0x01
id:         0x10..0x62, exceto IDs sem nome debug salvo modo avançado
hp:         1..65535 para entrada ativa
room:       stage 0x01..0x07
s_pos:      int16 válido
s_ang:      int16 válido
Guard_r:    0..32767
Dummy:      preservar ou 0
```

#### Validação avançada

```text
be_flag:    0x00..0xFF
id:         0x00..0xFF, com aviso para 0x63..0xFF
hp:         0..65535
room:       0x0000..0xFFFF
Guard_r:    -32768..32767
flag:       0x00000000..0xFFFFFFFF
Dummy:      0x00000000..0xFFFFFFFF
```

