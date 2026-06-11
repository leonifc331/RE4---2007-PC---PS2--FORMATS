# RE4 2007 - RoomWarp / Player Coordinates Map ST1-ST5

Mapa dos calls ao thunk `0x00401285`, que encaminha para a rotina usada aqui como `RoomWarp_SetDestination`. O alvo anterior `New Game Player Coordinates` é um desses calls.

Assinatura observada no decompile:

```c
thunk_FUN_0086f2d0(uint16 room, Vec4 *position, Vec4 *rotation, byte mode, int fadeOrKind);
```

## Resumo por stage

| Stage | Quantidade | Rooms hardcoded | Observação |
|---|---:|---|---|
| ST1 | 1 | r100 |  |
| ST2 | 2 | r22B |  |
| ST3 | 6 | r30A, r310, r316, r332, r333, r334 |  |
| ST4 | 0 | - | sem hardcoded direto encontrado |
| ST5 | 1 | r51E |  |
| dynamic | 2 | dynamic | handlers genéricos de AEV/room jump |

## Hardcoded warps ST1-ST5

| Room | Function VA | Call VA | Room offset | X/Y/Z | Coord offsets | Rot Y | Nearby strings |
|---|---:|---:|---:|---|---|---:|---|
| r100 | `0x00766790` | `0x00766982` | `0x366946` | -109450.0, -515.0, 820.0 | 0x36694E / 0x366956 / 0x36695E | 0.0 | 0x766838 -> event/evd/r120s99.evd; 0x7668b0 -> event/evd/r120s00.evd; 0x7668d0 -> event/evd/r120s01.evd; 0x766bdc -> evt_r120s00_func; 0x766bf5 -> evt_r120s01_func; 0x766c09 -> evt_r120s99_func |
| r22B | `0x007AD4B0` | `0x007AD704` | `0x3AD6FE` | 0.0, 0.0, 0.0 | 0x5DB85C / 0x5DB860 / 0x5DB864 | 1.3200000524520874 |  |
| r22B | `0x007AE060` | `0x007AE2C9` | `0x3AE2C3` | 0.0, 0.0, 0.0 | 0x5DB85C / 0x5DB860 / 0x5DB864 | 1.3200000524520874 |  |
| r316 | `0x007D7500` | `0x007D75D9` | `0x3D7595` | 12717.0, 2000.0, 7351.0 | 0x3D759D / 0x3D75A5 / 0x3D75AD | 1.465999960899353 | 0x7d749a -> event/evd/r30as10.evd; 0x7d7563 -> event/evd/r30as00.evd; 0x7d769c -> evt_r30as00_func; 0x7d76b5 -> evt_r30as98_func; 0x7d7767 -> event/evd/r30as10.evd |
| r310 | `0x007D83D0` | `0x007D84D1` | `0x3D84A5` | -6200.0, 0.0, -26100.0 | 0x3D84AD / 0x3D84B1 / 0x3D84B9 | 0.0 | 0x7d8448 -> event/evd/r30bs00.evd |
| r334 | `0x007DE620` | `0x007DE790` | `0x3DE754` | 22780.0, -40523.0, -25364.0 | 0x3DE75C / 0x3DE764 / 0x3DE76C | 1.559999942779541 |  |
| r30A | `0x007EA6B0` | `0x007EA7D2` | `0x3EA796` | 0.0, 0.0, 0.0 | 0x3EA79E / 0x3EA7A6 / 0x3EA7AE | 0.0 | 0x7ea5d1 -> event/evd/r316s00.evd; 0x7ea8a1 -> evt_r316s00_func; 0x7ea8df -> event/evd/r316s00.evd |
| r332 | `0x0080D860` | `0x0080D971` | `0x40D92D` | -42000.0, 15800.0, 46900.0 | 0x40D935 / 0x40D93D / 0x40D945 | 0.0 | 0x80d607 -> event/evd/r330s00.evd; 0x80d738 -> evt_r330s00_func; 0x80d8c8 -> event/evd/r331s00.evd; 0x80da4b -> event/evd/r331s10.evd |
| r333 | `0x0080D9D0` | `0x0080DB15` | `0x40DAE1` | -646166.0, -12718.0, -588290.0 | 0x40DAE9 / 0x40DAF1 / 0x40DAF9 | -1.7799999713897705 | 0x80d738 -> evt_r330s00_func; 0x80d8c8 -> event/evd/r331s00.evd; 0x80da4b -> event/evd/r331s10.evd; 0x80dcc9 -> evt_r331s00_func; 0x80dce2 -> evt_r331s10_func |
| r51E | `0x00859AA0` | `0x00859C02` | `0x459BC6` | -18576.0, 23311.0, 41248.0 | 0x459BCE / 0x459BD6 / 0x459BDE | -0.9300000071525574 | 0x859b74 -> event/evd/r51ds00.evd; 0x859e6c -> event/evd/r329s00.evd |

## Handlers dinâmicos

Esses dois não possuem destino fixo no EXE; eles recebem room/posição/rotação de uma estrutura em runtime. Eles podem enviar para rooms ST1-ST5 dependendo dos dados do evento/AEV.

| Function VA | Call VA | Origem dos dados | Descrição |
|---:|---:|---|---|
| `0x007BAA40` | `0x007BADCE` | `[EDI+0x64] room, [EDI+0x44] position, [EDI+0x54] rotation` | Dynamic warp. Destination room/position/rotation are read from event/parameter struct at EDI: room=[EDI+0x64], pos=[EDI+0x44], rot=[EDI+0x54]. |
| `0x008746A0` | `0x008749E9` | `[EDI+0x64] room, [EDI+0x44] position, [EDI+0x54] rotation` | Dynamic warp. Destination room/position/rotation are read from event/parameter struct at EDI: room=[EDI+0x64], pos=[EDI+0x44], rot=[EDI+0x54]. |

## Notas de patch

- Em entries com `inline stack constants`, os offsets X/Y/Z apontam diretamente para o `float32` little-endian editável no HxD.
- Em `r22B`, posição/rotação vêm de `DAT_009DB85C`, que pode ser compartilhado. Tratar como risco médio.
- ST4 não apresentou call hardcoded direto para `RoomWarp_SetDestination` nessa varredura. Transições de ST4 provavelmente usam AEV/door normal ou handler dinâmico.
- Os handlers dinâmicos não devem ser patchados para trocar um único destino; edite os dados AEV/room ou localize o caller específico primeiro.
