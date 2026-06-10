# Resident Evil 4 PC 2007 — documentação do formato `.snd` v0.3
Esta versão integra os `rxxx_prog.out.lst` e os `rxxxf_prog.out.lst`, fechando a nomeação dos dois bancos internos encontrados nos `.snd` de sala. A análise foi feita nos arquivos enviados `r100` a `r120`, no `BIO4.exe` PC 2007 e nas listas `.lst` correspondentes.
## Resultado principal
Cada `.snd` de sala válido contém dois bancos de áudio independentes:
| Banco | Chunks | Lista correspondente | Função observada |
|---:|---|---|---|
| 5 | `FFF4` + `FFF2` + `FFF3` | `rXXxf_prog.out.lst` | Footsteps/superfícies/foley de terreno. |
| 6 | `FFF4` + `FFF2` + `FFF3` | `rXXX_prog.out.lst` | SFX normais da sala: portas, ambiente, objetos, inimigos, loops etc. |
O arquivo `r120.snd` é especial/vazio nos samples: possui somente o cabeçalho mágico e sentinel, sem bancos internos.
## Cadeia de resolução de nome
A ligação entre `.lst`, `Prog`, `Smpl`, `Vagi` e dados raw fica assim:
```text
rXXX[f]_prog.out.lst
  Programa N / Prog# N
    Linha de som local I
      key = 0x0C + I
        Head.Prog[N].Keymap
          smpl_index
            Head.Smpl[smpl_index].vagi_index
              Head.Vagi[vagi_index]
                offset/tamanho/sample_rate dentro do chunk raw FFF3 do mesmo banco
```
A regra `key = 0x0C + índice local no LST` vale para os dois bancos. Em praticamente todos os keymaps analisados, `key_min == key_max`, então cada entrada aciona um único som.
## Layout global do arquivo
```c
struct SndFile
{
    uint8_t magic[0x20]; // CA B6 BE 20 repetido 8 vezes
    SndChunkDesc chunks[]; // entradas de 0x20 até kind_or_count == 0xFFFFFFFF
    // payloads apontados por SndChunkDesc.offset
};

struct SndChunkDesc
{
    uint32_t kind_or_count; // normalmente quantidade no FFF4/FFF2/FFF3
    uint32_t size;          // tamanho do payload em bytes
    uint32_t type;          // 0xFFF4, 0xFFF2, 0xFFF3
    uint32_t offset;        // offset absoluto do payload no .snd
    uint32_t bank_id;       // 5 ou 6
    uint32_t room_id;       // id de sala/stage
    uint32_t reserved0;
    uint32_t reserved1;
};
```
Tipos de chunk confirmados:

| Type | Nome operacional | Descrição |
|---:|---|---|
| `0x0000FFF4` | `SndParam` | Tabela de parâmetros/eventos usada pelo engine de SND. Entrada de `0x1C` bytes. |
| `0x0000FFF2` | `Head` | Container interno com seções `Prog`, `Smpl` e `Vagi`. |
| `0x0000FFF3` | `Raw/VAGi data` | Dados de áudio ADPCM/VAG raw referenciados pela tabela `Vagi`. |
## Chunk `FFF4` — tabela de parâmetros
A tabela `FFF4` é composta por entradas fixas de `0x1C` bytes. Ela existe nos bancos 5 e 6 e é preservável em roundtrip sem recalcular o áudio.
```c
struct SndParamEntry
{
    uint16_t sound_id;
    uint16_t program_id;

    uint8_t  id2;
    int8_t   priority;
    int8_t   pan;
    int8_t   volume;

    uint8_t  aux_a;
    int8_t   id1;
    int8_t   vol_flag;
    int8_t   pitch_l;

    int8_t   pitch_h;
    uint8_t  enc_vol;
    uint8_t  grob;
    uint8_t  srd_type;

    uint8_t  span;
    uint8_t  svol;
    uint32_t flags;
    uint32_t unk18;
};
```
Observações sobre `FFF4`:

- O banco 6 costuma ter `program_id` compatível com os programas nomeados no `rXXX_prog.out.lst`.
- O banco 5 mostra muito `program_id = 0` e `10` na tabela `FFF4`, enquanto o `Head.Prog` do mesmo banco usa os programas `0..N`. Portanto, para nomear áudio, o caminho confiável é `Head.Prog -> Smpl -> Vagi`, não a tabela `FFF4` isolada.
- Campos `pan`, `volume`, `priority`, `pitch_l` e `pitch_h` foram tratados como signed 8-bit nos samples, porque aparecem valores `0xFF` com comportamento típico de sentinela/valor neutro.
## Chunk `FFF2` — `Head`
```c
struct SndHead
{
    char     magic[4];      // "Head"
    uint32_t header_size;   // 0x20 nos samples
    uint32_t version;
    uint32_t head_size;
    uint32_t raw_size;      // tamanho esperado do chunk FFF3 do mesmo banco
    uint32_t prog_offset;   // relativo ao início de Head
    uint32_t smpl_offset;   // relativo ao início de Head
    uint32_t vagi_offset;   // relativo ao início de Head
};
```
As três subseções internas têm o mesmo cabeçalho básico:
```c
struct SndSectionHeader
{
    char     magic[4];      // "Prog", "Smpl" ou "Vagi"
    uint32_t size;          // tamanho total da seção, incluindo este cabeçalho
    uint32_t max_index;     // count - 1
    uint32_t sentinel;      // normalmente 0xFFFFFFFF
};
```
## Seção `Prog`
`Prog` começa com o cabeçalho de seção, seguido por uma tabela de offsets relativos ao início da própria seção `Prog`. Cada offset aponta para um blob de programa.
```c
struct SndProgSection
{
    SndSectionHeader header;
    uint32_t program_offsets[header.max_index + 1];
    // blobs de programa
};

struct SndProgramBlob
{
    uint8_t keymap_count;
    uint8_t default_flag;   // 0xFF nos samples
    uint8_t default_volume; // 0x7F nos samples
    uint8_t default_pan;    // 0x40 nos samples
    uint16_t unk04;
    uint8_t unk06;          // 0xFF nos samples
    uint8_t unk07;          // 0xFF nos samples
    SndProgramKeymapEntry keymaps[keymap_count];
};

struct SndProgramKeymapEntry
{
    uint8_t  unk00;
    uint8_t  flags;
    uint8_t  key_min;
    uint8_t  key_max;
    uint16_t pitch_center;  // 0x00C8 comum
    uint16_t pitch_base;    // 0x00C8 comum
    uint8_t  volume;        // 0x7F comum
    uint8_t  pan;           // 0x40 comum
    uint16_t unk0A;
    uint16_t smpl_index;
};
```
## Seção `Smpl`
```c
struct SndSmplEntry
{
    uint16_t adsr1;
    uint16_t adsr2;
    uint8_t  unk04;
    uint8_t  key;
    uint8_t  volume;
    uint8_t  pan;
    uint16_t unk08;
    uint16_t vagi_index;
};
```
Entradas completamente `0xFF` aparecem como slots vazios/padding lógico, especialmente no banco 5. Não devem ser usadas para extrair áudio.
## Seção `Vagi`
```c
struct SndVagiEntry
{
    uint32_t data_offset;   // relativo ao início do chunk FFF3 do mesmo banco
    uint32_t data_size;
    uint32_t loop_or_flag;  // bate com flag 1 em vários .lst de loop
    uint32_t sample_rate;   // 8000, 16000, 22050 etc. conforme o banco/som
};
```
O offset absoluto do áudio é:

```text
raw_abs_offset = chunk_FFF3.offset + Vagi.data_offset
```
## Interpretação dos `.lst`
Formato observado:
```text
//nome_do_programa ( Prog# N )
    0,N,
    "arquivo.ogg",flag,
    ...
    0,-1,
```
Regras práticas:

- `Prog# N` corresponde ao índice `N` em `Head.Prog`.
- A primeira linha numérica `0,N,` declara o programa.
- Cada linha `"nome.ogg",flag,` é um som local.
- O índice local da primeira linha de som é `0`.
- A key usada no `Prog` é `0x0C + índice_local`.
- `flag = 1` no LST coincide com `Vagi.loop_or_flag = 1` nos casos de loop observados.
## Estatística dos arquivos analisados
- Salas com bancos nomeados: `16`.
- Linhas de keymap banco 5 / foot: `858`; nomeadas não vazias: `858`; vazias: `0`; fora da lista: `0`.
- Linhas de keymap banco 6 / stage SFX: `357`; nomeadas não vazias: `356`; vazias: `1`; fora da lista: `0`.
- Banco 5 total `Vagi`: `613`; Banco 6 total `Vagi`: `357`.
- Entradas com `loop_or_flag != 0`: banco 5 `0`, banco 6 `15`.

### Resumo por sala

| Sala | Banco 5: progs/keymaps/smpl/vagi | Banco 6: progs/keymaps/smpl/vagi |
|---|---:|---:|
| `r100` | 5/55/66/39 | 2/16/16/16 |
| `r101` | 6/66/66/52 | 5/46/52/46 |
| `r102` | 4/44/44/26 | 3/19/19/19 |
| `r103` | 4/44/44/32 | 2/24/24/24 |
| `r104` | 6/66/66/51 | 2/17/17/17 |
| `r105` | 6/66/66/46 | 4/33/33/33 |
| `r106` | 6/66/66/43 | 2/23/23/23 |
| `r107` | 6/66/66/50 | 3/24/24/24 |
| `r108` | 5/55/55/39 | 4/15/15/15 |
| `r109` | 3/33/33/26 | 3/8/8/8 |
| `r111` | 6/66/66/52 | 4/28/30/28 |
| `r112` | 4/44/44/26 | 4/23/23/23 |
| `r113` | 5/55/55/39 | 3/29/32/29 |
| `r117` | 3/33/33/21 | 4/23/23/23 |
| `r118` | 6/66/66/45 | 5/19/21/19 |
| `r119` | 3/33/33/26 | 4/10/10/10 |

## Procedimento seguro para extrair e nomear
1. Ler o header mágico de `0x20` bytes.
2. Ler `SndChunkDesc` até `kind_or_count == 0xFFFFFFFF`.
3. Para cada banco, localizar `FFF2` e `FFF3`.
4. Em `FFF2`, ler `Head`, depois `Prog`, `Smpl` e `Vagi`.
5. Escolher a lista: banco 5 usa `rXXxf_prog.out.lst`; banco 6 usa `rXXX_prog.out.lst`.
6. Para cada `Prog# N`, mapear `key_min - 0x0C` para a linha do `.lst`.
7. Resolver `smpl_index`, depois `vagi_index`.
8. Extrair bytes em `FFF3.offset + data_offset` com tamanho `data_size`.
9. Nomear como o `.ogg` do LST, mas manter metadados de ADPCM/VAG porque o raw do `.snd` não é OGG.
## Roundtrip/repack recomendado
- Para edição segura inicial, preserve integralmente `FFF4` e os blobs `Prog`, alterando apenas `Vagi`, `Smpl` e o raw `FFF3` quando substituir áudio por dados compatíveis.
- Se o tamanho do raw mudar, atualizar `Vagi.data_offset`, `Vagi.data_size`, `Head.raw_size`, o tamanho do chunk `FFF3` e offsets dos chunks posteriores.
- Para adicionar/remover sons, também será necessário reconstruir `Prog` e possivelmente a tabela `FFF4`. Essa parte ainda deve ser tratada como fase separada.
## Confirmação por `BIO4.exe` PC 2007
O executável contém referências textuais a `ps2snd.cpp`, aos padrões `r%3x_prog.out.lst` / `r%03xf_prog.out.lst` e às pastas `ogg\st%d\` / `ogg\foot\`. Isso confirma que o PC 2007 mantém a arquitetura herdada do pipeline PS2, mas usa listas e nomes `.ogg` para resolver/organizar sons no lado PC.
## Estado dos campos
| Campo/área | Estado |
|---|---|
| Header global, chunks, bancos 5/6 | Confirmado nos samples. |
| `Head`, `Prog`, `Smpl`, `Vagi` | Confirmado nos samples. |
| Nomeação banco 5 via `rXXxf_prog.out.lst` | Confirmada. |
| Nomeação banco 6 via `rXXX_prog.out.lst` | Confirmada. |
| `Vagi.loop_or_flag` vs flag do LST | Confirmado para os loops observados; manter como flag/loop até testar mais casos. |
| Semântica fina de todos os campos de `FFF4` | Parcial/inferida. Preservar no repack. |
