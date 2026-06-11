# RE4 2007 PC — INIT Swap / Clone Room INIT Map
## Finding
The offset provided by the user, `0x4AFA20`, is confirmed as the **ST1 room INIT assignment routine** in the 2007 EXE. Unlike the UHD guide table, the 2007 EXE stores INIT assignments as x86 code: `C7 05 <runtime_table_address> <function_pointer>`.
The UHD method says to calculate a room entry by `RoomValue * 0x14 + 0x06` inside the UHD table. In this 2007 build, the equivalent runtime entry stride is **0x0C bytes**, with two dword function pointers per room. A room swap should normally copy both pointers.
## Stage setup bases

| Stage | Source | Setup file offset | Setup VA | Runtime table A | Runtime table B | Rooms detected | Direct safe rooms |
|---|---|---:|---:|---:|---:|---:|---:|
| ST0 | `st0.cpp` | `0x4AF8E0` | `0x008AF8E0` | `0x00A516D4` | `0x00A516D8` | 7 | 7 |
| ST1 | `st1.cpp` | `0x4AFA20` | `0x008AFA20` | `0x00A519FC` | `0x00A51A00` | 29 | 29 |
| ST2 | `st2.cpp` | `0x4AFD90` | `0x008AFD90` | `0x00A51B8C` | `0x00A51B90` | 42 | 42 |
| ST3 | `st3.cpp` | `0x4B0290` | `0x008B0290` | `0x00A51DB4` | `0x00A51DB8` | 38 | 38 |
| ST4 | `st4.cpp` | `0x4B0B70` | `0x008B0B70` | `0x00A52034` | `0x00A52038` | 14 | 14 |
| ST5 | `st5.cpp` | `0x4B0EF0` | `0x008B0EF0` | `0x00A5210C` | `0x00A52110` | 53 | 36 |
| ST6 | `st6.cpp` | `0x4B1400` | `0x008B1400` | `0x00A5240C` | `0x00A52410` | 32 | 30 |
| ST7 | `st7.cpp` | `0x4B17B0` | `0x008B17B0` | `0x00A5258C` | `0x00A52590` | 1 | 1 |

## Confirmed ST1 base

```text
ST1 setup routine file offset: 0x4AFA20
ST1 setup routine VA:          0x008AFA20
ST1 runtime table A base:      0x00A519FC
ST1 runtime table B base:      0x00A51A00
Entry formula in RAM:          base + RoomValue * 0x0C
Room r1XX uses RoomValue XX.
```

## How to clone INIT in 2007

For a normal direct row, copy **both** little-endian pointers from the source room to the target room:

1. Source `ptr_a_bytes_le` -> target `patch_a_file_off`
2. Source `ptr_b_bytes_le` -> target `patch_b_file_off`
3. Do not change file size. Use overwrite/paste-write, not insert.

Rows marked `NO_SHARED_OR_INCOMPLETE` are not safe for a one-room direct replacement, because the pointer is loaded through a shared register immediate and reused by several rooms. Those need a code-cave or a more careful rewrite.

## Example logic, matching UHD idea

Example: cloning `r307` over `r219` in this 2007 map:

- Source `r307`: ptr A `8B 75 40 00`, ptr B `3F B2 40 00`
- Target `r219`: overwrite file offset `0x4AFF8A` with source ptr A, and `0x4AFF94` with source ptr B.
- Current target bytes are A `85 85 40 00`, B `04 39 40 00`.

## ST1 direct table

| Room | RoomValue | Ptr A LE | Ptr B LE | Patch A | Patch B | Safe |
|---|---:|---:|---:|---:|---:|---|
| `r100` | `0x00` | `35 99 40 00` | `98 B3 40 00` | `0x4AFA26` | `0x4AFA30` | YES |
| `r101` | `0x01` | `E1 D3 40 00` | `96 4C 40 00` | `0x4AFA3A` | `0x4AFA44` | YES |
| `r102` | `0x02` | `85 3F 40 00` | `6A C3 40 00` | `0x4AFA4E` | `0x4AFA58` | YES |
| `r103` | `0x03` | `AC 54 40 00` | `CC 52 40 00` | `0x4AFA62` | `0x4AFA6C` | YES |
| `r104` | `0x04` | `C5 31 40 00` | `8D 2D 40 00` | `0x4AFA76` | `0x4AFA80` | YES |
| `r105` | `0x05` | `1F 5A 40 00` | `8C D3 40 00` | `0x4AFA8A` | `0x4AFA94` | YES |
| `r106` | `0x06` | `E2 AF 40 00` | `A8 49 40 00` | `0x4AFA9E` | `0x4AFAA8` | YES |
| `r107` | `0x07` | `F2 95 40 00` | `D3 23 40 00` | `0x4AFAB2` | `0x4AFABC` | YES |
| `r108` | `0x08` | `1F 41 40 00` | `7D 65 40 00` | `0x4AFAC6` | `0x4AFAD0` | YES |
| `r109` | `0x09` | `9B 3D 40 00` | `38 9B 40 00` | `0x4AFADA` | `0x4AFAE4` | YES |
| `r10A` | `0x0A` | `0C 77 40 00` | `B3 52 40 00` | `0x4AFAEE` | `0x4AFAF8` | YES |
| `r10B` | `0x0B` | `28 6F 40 00` | `85 3A 40 00` | `0x4AFB02` | `0x4AFB0C` | YES |
| `r10C` | `0x0C` | `C6 BC 40 00` | `7C A2 40 00` | `0x4AFB16` | `0x4AFB20` | YES |
| `r10D` | `0x0D` | `DF 21 40 00` | `83 19 40 00` | `0x4AFB2A` | `0x4AFB34` | YES |
| `r10E` | `0x0E` | `6A 14 40 00` | `75 AE 40 00` | `0x4AFB3E` | `0x4AFB48` | YES |
| `r10F` | `0x0F` | `4D 36 40 00` | `29 19 40 00` | `0x4AFB52` | `0x4AFB5C` | YES |
| `r111` | `0x11` | `76 C6 40 00` | `8B 25 40 00` | `0x4AFB66` | `0x4AFB70` | YES |
| `r112` | `0x12` | `90 75 40 00` | `44 B2 40 00` | `0x4AFB7A` | `0x4AFB84` | YES |
| `r113` | `0x13` | `90 C0 40 00` | `1C B2 40 00` | `0x4AFB8E` | `0x4AFB98` | YES |
| `r117` | `0x17` | `22 52 40 00` | `77 66 40 00` | `0x4AFBA2` | `0x4AFBAC` | YES |
| `r118` | `0x18` | `CB 7B 40 00` | `1F 64 40 00` | `0x4AFBB6` | `0x4AFBC0` | YES |
| `r119` | `0x19` | `C5 40 40 00` | `63 1B 40 00` | `0x4AFBCA` | `0x4AFBD4` | YES |
| `r11A` | `0x1A` | `A5 4C 40 00` | `AC C2 40 00` | `0x4AFBDE` | `0x4AFBE8` | YES |
| `r11B` | `0x1B` | `BB 36 40 00` | `E5 2F 40 00` | `0x4AFBF2` | `0x4AFBFC` | YES |
| `r11C` | `0x1C` | `AF 79 40 00` | `64 C9 40 00` | `0x4AFC06` | `0x4AFC10` | YES |
| `r11D` | `0x1D` | `7D BF 40 00` | `4F 43 40 00` | `0x4AFC1A` | `0x4AFC24` | YES |
| `r11E` | `0x1E` | `8C 15 40 00` | `F8 3A 40 00` | `0x4AFC2E` | `0x4AFC38` | YES |
| `r11F` | `0x1F` | `6D 39 40 00` | `AE 39 40 00` | `0x4AFC42` | `0x4AFC4C` | YES |
| `r120` | `0x20` | `A7 54 40 00` | `C7 52 40 00` | `0x4AFC56` | `0x4AFC60` | YES |

## Important limitations

- INIT swap clones hardcoded room behavior, not dependencies. The UHD guide notes that dependencies inside UDAS and related room files still need to match/copied, and that some behavior is independent of room INIT.
- The complete room list may still be incomplete for rooms that are not assigned directly in these routines or are assigned elsewhere dynamically.
- In the 2007 EXE, ST5/ST6 contain shared/default pointer stores. These are explicitly marked in the CSV.
