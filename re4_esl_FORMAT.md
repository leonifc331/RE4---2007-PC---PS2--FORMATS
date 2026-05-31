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
