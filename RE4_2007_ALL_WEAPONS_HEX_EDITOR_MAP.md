# RE4 2007 PC - All Weapons Hex Editor Map

Escopo: mapa prático para HxD/hex editor, com offsets de hit, bytes originais, fórmulas de patch, init/modelo e tabela de recursos das armas conhecidas no `BIO4.exe`.

## Regra principal

- `File offset` é o offset para usar direto no HxD.
- Não inserir/remover bytes; apenas sobrescrever os bytes existentes.
- `XX` é o hit desejado, 1 byte em hex.
- Patches em grupos compartilhados afetam todas as armas do grupo. Para alterar uma arma individual, precisa code cave com filtro de `CurrentWeaponKind`.

## Hit values observados

| Hit | Count | Interpretação |
|---:|---:|---|
| `0x13` | 37 | Grenade/explosion hit observed |
| `0x12` | 30 | Fire/explosion environmental hit observed |
| `0x10` | 7 | Knife slash hit observed |
| `current_weapon` | 6 | CurrentWeaponKind / weapon id actual em runtime |
| `0x2E` | 4 | PRL412/Laser hit observed |
| `0xC` | 4 | Observed fixed hit 0x0C / rocket-like candidate |
| `0xD` | 2 | Observed fixed hit 0x0D |
| `0xA` | 2 | Observed fixed hit 0x0A |
| `0x3` | 2 | Knife/close hit observed |
| `0xF` | 1 | Observed fixed hit 0x0F |
| `0x1B` | 1 | Observed fixed hit 0x1B |
| `0x1F` | 1 | Wep17/VP70 move candidate fixed hit |
| `0x17` | 1 | Flash/stun candidate observed |
| `0x2D` | 1 | Special close/weapon hit observed |
| `0x14` | 1 | Observed fixed hit 0x14 |

Valores úteis para teste: `0x13` explosão/granada, `0x12` fogo/explosão ambiental observado, `0x17` flash/stun candidato, `0x2E` PRL/Laser, `0x10` knife slash.

## Grupos de patch por família

| Grupo | Armas afetadas | Offset do hit | Original | Patch genérico | Observação |
|---|---|---:|---|---|---|
| `HANDGUN_SHARED` | `wep01,wep02,wep04,wep05,wep06,wep15,wep38,wep43,wep44,wep54` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 <XX> 00 00 00 90 90 90` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `MACHINE_SHARED_A` | `wep11,wep12,wep27,wep29,wep39,wep49` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA <XX> 00 00 00 90 90 90` | Primeiro callsite de machine/TMP/Tompson. |
| `MACHINE_SHARED_B` | `wep11,wep12,wep27,wep29,wep39,wep49` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 <XX> 00 00 00 90 90 90` | Segundo callsite de machine/TMP/Tompson. |
| `RIFLE_SHARED` | `wep09,wep10,wep40,wep47` | `0x002EE77B` | `66 0F B6 88 96 5F 00 00` | `B9 <XX> 00 00 00 90 90 90` | Rifle/sniper usa CurrentWeaponKind via ECX. |
| `SHOTGUN_SHARED` | `wep07,wep08,wep33,wep48` | `0x002F19A3` | `66 0F B6 82 96 5F 00 00` | `B8 <XX> 00 00 00 90 90 90` | Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX. |
| `THROWABLE_DYNAMIC_CANDIDATE` | `wep19,wep30,wep41,wep42,wep45` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` | `B9 <XX> 00 00 00 90 90 90` | Candidato para família de granadas/projéteis; validar em jogo porque algumas granadas usam ETM/área e não hit direto. |
| `KNIFE_SWEEP_1` | `wep16,wep26` | `0x002D7866` | `6A 10` | `6A <XX>` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. |
| `KNIFE_SWEEP_2` | `wep16,wep26` | `0x002D78F0` | `6A 10` | `6A <XX>` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. |
| `KNIFE_SWEEP_3` | `wep16,wep26` | `0x002D7942` | `6A 10` | `6A <XX>` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. |
| `KNIFE_SWEEP_4` | `wep16,wep26` | `0x002D79BD` | `6A 10` | `6A <XX>` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. |
| `KNIFE_SWEEP_5` | `wep16,wep26` | `0x002D79FC` | `6A 10` | `6A <XX>` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. |
| `KNIFE_SWEEP_6` | `wep16,wep26` | `0x002D7A33` | `6A 10` | `6A <XX>` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. |
| `KNIFE_SWEEP_7` | `wep16,wep26` | `0x002D7A67` | `6A 10` | `6A <XX>` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. |
| `KNIFE_SPECIAL_HIT` | `wep16/wep26 candidate` | `0x002DB8A2` | `6A 03` | `6A <XX>` | Callsite extra com hit 0x03 próximo ao fluxo de knife/close hit. |
| `WEP17_VP70_CANDIDATE` | `wep17` | `0x002A2ABB` | `B8 1F 00 00 00` | `B8 <XX> 00 00 00` | Candidato fixo do Wep17/VP70: move EAX,0x1F antes do push. |
| `LASER_PRL_1` | `wep51` | `0x00295477` | `6A 2E` | `6A <XX>` | PRL/Laser hit. 0x2E é o hit observado do PRL-like. |
| `LASER_PRL_2` | `wep51` | `0x0029551D` | `6A 2E` | `6A <XX>` | PRL/Laser hit. 0x2E é o hit observado do PRL-like. |
| `LASER_PRL_3` | `wep51` | `0x002955C9` | `6A 2E` | `6A <XX>` | PRL/Laser hit. 0x2E é o hit observado do PRL-like. |
| `LASER_PRL_4` | `wep51` | `0x002957AB` | `6A 2E` | `6A <XX>` | PRL/Laser hit. 0x2E é o hit observado do PRL-like. |

## Lista compacta por arma

| WEP | Hex | Família/objeto | Init/model file off | Modelos PMD | Hit offset(s) | Patch |
|---|---:|---|---:|---|---|---|
| `wep00` | `0x00` | Hand | `` | `` | `` | `` |
| `wep01` | `0x01` | Fn57 | `0x002919E0` | `wep0100.pmd; wep0101.pmd` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep02` | `0x02` | Mauser | `0x002A0780` | `wep0200.pmd; wep0201.pmd` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep03` | `0x03` | Unknown | `0x00297AA0` | `wep0300.pmd; wep0301.pmd` | `` | `` |
| `wep04` | `0x04` | Xd9 | `0x002FAFA0` | `wep0400.pmd; wep0401.pmd` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep05` | `0x05` | Civilian | `0x00290FD0` | `wep0500.pmd` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep06` | `0x06` | Government | `` | `` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep07` | `0x07` | Shotgun | `0x002FC790` | `wep0700.pmd` | `0x002F19A3` | `B8 <XX> 00 00 00 90 90 90` |
| `wep08` | `0x08` | Striker | `0x002FD180` | `wep0800.pmd` | `0x002F19A3` | `B8 <XX> 00 00 00 90 90 90` |
| `wep09` | `0x09` | Sniper | `0x002A1150` | `wep0900.pmd; wep0901.pmd; wep0902.pmd` | `0x002EE77B` | `B9 <XX> 00 00 00 90 90 90` |
| `wep10` | `0x0A` | HkSniper | `0x00294760` | `wep1000.pmd; wep1002.pmd` | `0x002EE77B` | `B9 <XX> 00 00 00 90 90 90` |
| `wep11` | `0x0B` | Machinegun | `0x00296BF0` | `wep1100.pmd; wep1101.pmd; wep1102.pmd; wep1103.pmd` | `0x002E0688 | 0x002E0826` | `BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90` |
| `wep12` | `0x0C` | Tompson | `0x002A4170` | `wep1200.pmd` | `0x002E0688 | 0x002E0826` | `BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90` |
| `wep13` | `0x0D` | Rocket-family | `0x0029FC60` | `wep1300.pmd` | `` | `` |
| `wep14` | `0x0E` | Mine | `0x00298B80` | `wep1400.pmd; wep1401.pmd` | `` | `` |
| `wep15` | `0x0F` | Magnum | `0x002FF020` | `wep1500.pmd` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep16` | `0x10` | Knife | `0x002FF540` | `wep1600.pmd` | `0x002D7866 | 0x002D78F0 | 0x002D7942 | 0x002D79BD | 0x002D79FC | 0x00…` | `6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <…` |
| `wep17` | `0x11` | Vp70 | `0x002A6960` | `wep1700.pmd` | `0x002A2ABB` | `B8 <XX> 00 00 00` |
| `wep18` | `0x12` | DAT/SND only or resource-only entry | `` | `` | `` | `` |
| `wep19` | `0x13` | HandGre | `0x002A3410` | `wep1900.pmd` | `0x0030086E` | `B9 <XX> 00 00 00 90 90 90` |
| `wep20` | `0x14` | DAT/SND only or resource-only entry | `` | `` | `` | `` |
| `wep21` | `0x15` | DAT/SND only or resource-only entry | `` | `` | `` | `` |
| `wep22` | `0x16` | Grenade/incendiary candidate model | `0x002A3410 | 0x00304570 | 0x004906…` | `wep2200.pmd` | `` | `` |
| `wep23` | `0x17` | Grenade/flash candidate model | `0x001E4390 | 0x002A3410 | 0x003045…` | `wep2300.pmd` | `` | `` |
| `wep24` | `0x18` | DAT/SND only or resource-only entry | `` | `` | `` | `` |
| `wep25` | `0x19` | DAT/SND only or resource-only entry | `` | `` | `` | `` |
| `wep26` | `0x1A` | Knife | `0x00302070` | `wep2600.pmd` | `0x002D7866 | 0x002D78F0 | 0x002D7942 | 0x002D79BD | 0x002D79FC | 0x00…` | `6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <…` |
| `wep27` | `0x1B` | Machinegun | `0x00302B50` | `wep2700.pmd` | `0x002E0688 | 0x002E0826` | `BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90` |
| `wep28` | `0x1C` | Bow | `0x003038D0` | `wep2800.pmd` | `` | `` |
| `wep29` | `0x1D` | Machinegun | `0x00296BF0` | `wep2900.pmd` | `0x002E0688 | 0x002E0826` | `BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90` |
| `wep30` | `0x1E` | HandGre | `` | `` | `0x0030086E` | `B9 <XX> 00 00 00 90 90 90` |
| `wep31` | `0x1F` | DAT/SND only or resource-only entry | `` | `` | `` | `` |
| `wep32` | `0x20` | DAT/SND only or resource-only entry | `` | `` | `` | `` |
| `wep33` | `0x21` | Shotgun | `0x003051C0` | `wep3300.pmd` | `0x002F19A3` | `B8 <XX> 00 00 00 90 90 90` |
| `wep34` | `0x22` | Hand | `` | `` | `` | `` |
| `wep35` | `0x23` | Hand | `` | `` | `` | `` |
| `wep36` | `0x24` | Hand | `` | `` | `` | `` |
| `wep37` | `0x25` | Hand | `` | `` | `` | `` |
| `wep38` | `0x26` | Ruger | `0x00306FC0` | `wep3800.pmd` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep39` | `0x27` | Machinegun | `0x00307B40` | `wep3900.pmd` | `0x002E0688 | 0x002E0826` | `BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90` |
| `wep40` | `0x28` | HkSniper | `0x00308470` | `wep4000.pmd; wep4001.pmd` | `0x002EE77B` | `B9 <XX> 00 00 00 90 90 90` |
| `wep41` | `0x29` | HandGre | `` | `` | `0x0030086E` | `B9 <XX> 00 00 00 90 90 90` |
| `wep42` | `0x2A` | HandGre | `` | `` | `0x0030086E` | `B9 <XX> 00 00 00 90 90 90` |
| `wep43` | `0x2B` | Ruger | `` | `` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep44` | `0x2C` | Government | `` | `` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep45` | `0x2D` | HandGre | `` | `` | `0x0030086E` | `B9 <XX> 00 00 00 90 90 90` |
| `wep46` | `0x2E` | Rocket-family | `` | `` | `` | `` |
| `wep47` | `0x2F` | HkSniper | `` | `` | `0x002EE77B` | `B9 <XX> 00 00 00 90 90 90` |
| `wep48` | `0x30` | Shotgun | `0x0030C260` | `wep4800.pmd` | `0x002F19A3` | `B8 <XX> 00 00 00 90 90 90` |
| `wep49` | `0x31` | Tompson | `0x002A4170` | `wep4900.pmd` | `0x002E0688 | 0x002E0826` | `BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90` |
| `wep50` | `0x32` | AdaBow | `0x0030D8D0` | `wep5000.pmd` | `` | `` |
| `wep51` | `0x33` | Laser | `` | `` | `0x00295477 | 0x0029551D | 0x002955C9 | 0x002957AB` | `6A <XX> | 6A <XX> | 6A <XX> | 6A <XX>` |
| `wep52` | `0x34` | DAT/SND only or resource-only entry | `0x002961B0` | `wep5200.pmd` | `` | `` |
| `wep54` | `0x36` | Xd9 | `` | `` | `0x002D4DAE` | `B9 <XX> 00 00 00 90 90 90` |
| `wep55` | `0x37` | DAT/SND only or resource-only entry | `` | `` | `` | `` |

## Detalhe por arma

### wep00 / 0x00 - Hand

```txt
REL:        wep00.rel
DAT:        em\wep00.dat
SND:        
REL init:   Wep00_init__FP7cPlayer @ 0x000001A0
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x00622F98 / string off: 0x005CAD74
DAT ptr file off: 0x00622F94 / string off: 0x005CAD84
SND ptr file off:  / string off: 
```

### wep01 / 0x01 - Fn57

```txt
REL:        wep01.rel
DAT:        em\wep01.dat
SND:        em\wep01.snd
REL init:   Wep01_init__FP7cPlayer @ 0x00002068
x86 init:   VA 0x006919E0 / file off 0x002919E0
PMD:        wep0100.pmd; wep0101.pmd
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00622FA4 / string off: 0x005CAD44
DAT ptr file off: 0x00622FA0 / string off: 0x005CAD54
SND ptr file off: 0x00622F9C / string off: 0x005CAD64
```

### wep02 / 0x02 - Mauser

```txt
REL:        wep02.rel
DAT:        em\wep02.dat
SND:        em\wep02.snd
REL init:   Wep02_init__FP7cPlayer @ 0x00002FE8
x86 init:   VA 0x006A0780 / file off 0x002A0780
PMD:        wep0200.pmd; wep0201.pmd
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623070 / string off: 0x005CAD14
DAT ptr file off: 0x00622FAC / string off: 0x005CAD24
SND ptr file off: 0x00622FA8 / string off: 0x005CAD34
```

### wep03 / 0x03 - Unknown

```txt
REL:        wep03.rel
DAT:        em\wep03.dat
SND:        em\wep03.snd
REL init:   Wep03_init__FP7cPlayer @ 0x000001A0
x86 init:   VA 0x00697AA0 / file off 0x00297AA0
PMD:        wep0300.pmd; wep0301.pmd
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x006234C8 / string off: 0x005C96B4
DAT ptr file off: 0x00622FB8 / string off: 0x005CACF4
SND ptr file off: 0x00622FB4 / string off: 0x005CAD04
```

### wep04 / 0x04 - Xd9

```txt
REL:        wep04.rel
DAT:        em\wep04.dat
SND:        em\wep04.snd
REL init:   Wep04_init__FP7cPlayer @ 0x000015D8
x86 init:   VA 0x006FAFA0 / file off 0x002FAFA0
PMD:        wep0400.pmd; wep0401.pmd
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00622FC8 / string off: 0x005CACC4
DAT ptr file off: 0x00622FC4 / string off: 0x005CACD4
SND ptr file off: 0x00622FC0 / string off: 0x005CACE4
```

### wep05 / 0x05 - Civilian

```txt
REL:        wep05.rel
DAT:        em\wep05.dat
SND:        em\wep05.snd
REL init:   Wep05_init__FP7cPlayer @ 0x00001E30
x86 init:   VA 0x00690FD0 / file off 0x00290FD0
PMD:        wep0500.pmd
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00622FD4 / string off: 0x005CAC94
DAT ptr file off: 0x00622FD0 / string off: 0x005CACA4
SND ptr file off: 0x00622FCC / string off: 0x005CACB4
```

### wep06 / 0x06 - Government

```txt
REL:        wep06.rel
DAT:        em\wep06.dat
SND:        em\wep06.snd
REL init:   Wep06_init__FP7cPlayer @ 0x000015D8
x86 init:   VA  / file off 
PMD:        
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00622FE0 / string off: 0x005CAC64
DAT ptr file off: 0x00622FDC / string off: 0x005CAC74
SND ptr file off: 0x00622FD8 / string off: 0x005CAC84
```

### wep07 / 0x07 - Shotgun

```txt
REL:        wep07.rel
DAT:        em\wep07.dat
SND:        em\wep07.snd
REL init:   Wep07_init__FP7cPlayer @ 0x00001980
x86 init:   VA 0x006FC790 / file off 0x002FC790
PMD:        wep0700.pmd
Hit group:  SHOTGUN_SHARED
Hit off:    0x002F19A3
Original:   current_weapon
Patch:      B8 <XX> 00 00 00 90 90 90
Notas:      Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `SHOTGUN_SHARED` | `0x002F19A3` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00622FEC / string off: 0x005CAC34
DAT ptr file off: 0x00622FE8 / string off: 0x005CAC44
SND ptr file off: 0x00622FE4 / string off: 0x005CAC54
```

### wep08 / 0x08 - Striker

```txt
REL:        wep08.rel
DAT:        em\wep08.dat
SND:        em\wep08.snd
REL init:   Wep08_init__FP7cPlayer @ 0x00001980
x86 init:   VA 0x006FD180 / file off 0x002FD180
PMD:        wep0800.pmd
Hit group:  SHOTGUN_SHARED
Hit off:    0x002F19A3
Original:   current_weapon
Patch:      B8 <XX> 00 00 00 90 90 90
Notas:      Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `SHOTGUN_SHARED` | `0x002F19A3` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00622FF8 / string off: 0x005CAC04
DAT ptr file off: 0x00622FF4 / string off: 0x005CAC14
SND ptr file off: 0x00622FF0 / string off: 0x005CAC24
```

### wep09 / 0x09 - Sniper

```txt
REL:        wep09.rel
DAT:        em\wep09.dat
SND:        em\wep09.snd
REL init:   Wep09_init__FP7cPlayer @ 0x00002080
x86 init:   VA 0x006A1150 / file off 0x002A1150
PMD:        wep0900.pmd; wep0901.pmd; wep0902.pmd
Hit group:  RIFLE_SHARED
Hit off:    0x002EE77B
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Rifle/sniper usa CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `RIFLE_SHARED` | `0x002EE77B` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006230A0 / string off: 0x005CABD4
DAT ptr file off: 0x00623000 / string off: 0x005CABE4
SND ptr file off: 0x00622FFC / string off: 0x005CABF4
```

### wep10 / 0x0A - HkSniper

```txt
REL:        wep10.rel
DAT:        em\wep10.dat
SND:        em\wep10.snd
REL init:   Wep10_init__FP7cPlayer @ 0x00001EA0
x86 init:   VA 0x00694760 / file off 0x00294760
PMD:        wep1000.pmd; wep1002.pmd
Hit group:  RIFLE_SHARED
Hit off:    0x002EE77B
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Rifle/sniper usa CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `RIFLE_SHARED` | `0x002EE77B` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006230D0 / string off: 0x005CABA4
DAT ptr file off: 0x0062300C / string off: 0x005CABB4
SND ptr file off: 0x00623008 / string off: 0x005CABC4
```

### wep11 / 0x0B - Machinegun

```txt
REL:        wep11.rel
DAT:        em\wep11.dat
SND:        em\wep11.snd
REL init:   Wep11_init__FP7cPlayer @ 0x00002138
x86 init:   VA 0x00696BF0 / file off 0x00296BF0
PMD:        wep1100.pmd; wep1101.pmd; wep1102.pmd; wep1103.pmd
Hit group:  MACHINE_SHARED_A | MACHINE_SHARED_B
Hit off:    0x002E0688 | 0x002E0826
Original:   current_weapon | current_weapon
Patch:      BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90
Notas:      Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `MACHINE_SHARED_A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA 13 00 00 00 90 90 90` | `BA 17 00 00 00 90 90 90` | `BA 2E 00 00 00 90 90 90` |
| `MACHINE_SHARED_B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623088 / string off: 0x005CAB74
DAT ptr file off: 0x00623018 / string off: 0x005CAB84
SND ptr file off: 0x00623014 / string off: 0x005CAB94
```

### wep12 / 0x0C - Tompson

```txt
REL:        wep12.rel
DAT:        em\wep12.dat
SND:        em\wep12.snd
REL init:   Wep12_init__FP7cPlayer @ 0x00002120
x86 init:   VA 0x006A4170 / file off 0x002A4170
PMD:        wep1200.pmd
Hit group:  MACHINE_SHARED_A | MACHINE_SHARED_B
Hit off:    0x002E0688 | 0x002E0826
Original:   current_weapon | current_weapon
Patch:      BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90
Notas:      Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `MACHINE_SHARED_A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA 13 00 00 00 90 90 90` | `BA 17 00 00 00 90 90 90` | `BA 2E 00 00 00 90 90 90` |
| `MACHINE_SHARED_B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006230E8 / string off: 0x005CAB44
DAT ptr file off: 0x00623024 / string off: 0x005CAB54
SND ptr file off: 0x00623020 / string off: 0x005CAB64
```

### wep13 / 0x0D - Rocket-family

```txt
REL:        wep13.rel
DAT:        em\wep13.dat
SND:        em\wep13.snd
REL init:   Wep13_init__FP7cPlayer @ 0x00001958
x86 init:   VA 0x0069FC60 / file off 0x0029FC60
PMD:        wep1300.pmd
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x006230B8 / string off: 0x005CAB14
DAT ptr file off: 0x00623030 / string off: 0x005CAB24
SND ptr file off: 0x0062302C / string off: 0x005CAB34
```

### wep14 / 0x0E - Mine

```txt
REL:        wep14.rel
DAT:        em\wep14.dat
SND:        em\wep14.snd
REL init:   Wep14_init__FP7cPlayer @ 0x00001090
x86 init:   VA 0x00698B80 / file off 0x00298B80
PMD:        wep1400.pmd; wep1401.pmd
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x006230AC / string off: 0x005CAAE4
DAT ptr file off: 0x0062303C / string off: 0x005CAAF4
SND ptr file off: 0x00623038 / string off: 0x005CAB04
```

### wep15 / 0x0F - Magnum

```txt
REL:        wep15.rel
DAT:        em\wep15.dat
SND:        em\wep15.snd
REL init:   Wep15_init__FP7cPlayer @ 0x000015D8
x86 init:   VA 0x006FF020 / file off 0x002FF020
PMD:        wep1500.pmd
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x0062304C / string off: 0x005CAAB4
DAT ptr file off: 0x00623048 / string off: 0x005CAAC4
SND ptr file off: 0x00623044 / string off: 0x005CAAD4
```

### wep16 / 0x10 - Knife

```txt
REL:        wep16.rel
DAT:        em\wep16.dat
SND:        em\wep16.snd
REL init:   Wep16_init__FP7cPlayer @ 0x000019B0
x86 init:   VA 0x006FF540 / file off 0x002FF540
PMD:        wep1600.pmd
Hit group:  KNIFE_SWEEP_1 | KNIFE_SWEEP_2 | KNIFE_SWEEP_3 | KNIFE_SWEEP_4 | KNIFE_SWEEP_5 | KNIFE_SWEEP_6 | KNIFE_SWEEP_7 | KNIFE_SPECIAL_HIT
Hit off:    0x002D7866 | 0x002D78F0 | 0x002D7942 | 0x002D79BD | 0x002D79FC | 0x002D7A33 | 0x002D7A67 | 0x002DB8A2
Original:   0x10 | 0x10 | 0x10 | 0x10 | 0x10 | 0x10 | 0x10 | 0x03
Patch:      6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX>
Notas:      Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Callsite extra com hit 0x03 próximo ao fluxo de knife/close hit.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `KNIFE_SWEEP_1` | `0x002D7866` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_2` | `0x002D78F0` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_3` | `0x002D7942` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_4` | `0x002D79BD` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_5` | `0x002D79FC` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_6` | `0x002D7A33` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_7` | `0x002D7A67` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SPECIAL_HIT` | `0x002DB8A2` | `6A 03` | `6A 13` | `6A 17` | `6A 2E` |

Resource table offsets:
```txt
REL ptr file off: 0x00623058 / string off: 0x005CAA84
DAT ptr file off: 0x00623054 / string off: 0x005CAA94
SND ptr file off: 0x00623050 / string off: 0x005CAAA4
```

### wep17 / 0x11 - Vp70

```txt
REL:        wep17.rel
DAT:        em\wep17.dat
SND:        em\wep17.snd
REL init:   Wep17_init__FP7cPlayer @ 0x00000BD0
x86 init:   VA 0x006A6960 / file off 0x002A6960
PMD:        wep1700.pmd
Hit group:  WEP17_VP70_CANDIDATE
Hit off:    0x002A2ABB
Original:   0x1F
Patch:      B8 <XX> 00 00 00
Notas:      Candidato fixo do Wep17/VP70: move EAX,0x1F antes do push.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `WEP17_VP70_CANDIDATE` | `0x002A2ABB` | `B8 1F 00 00 00` | `B8 13 00 00 00` | `B8 17 00 00 00` | `B8 2E 00 00 00` |

Resource table offsets:
```txt
REL ptr file off: 0x00623064 / string off: 0x005CAA54
DAT ptr file off: 0x00623060 / string off: 0x005CAA64
SND ptr file off: 0x0062305C / string off: 0x005CAA74
```

### wep18 / 0x12 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep18.dat
SND:        em\wep18.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x0062306C / string off: 0x005CAA34
SND ptr file off: 0x00623068 / string off: 0x005CAA44
```

### wep19 / 0x13 - HandGre

```txt
REL:        wep19.rel
DAT:        em\wep19.dat
SND:        em\wep19.snd
REL init:   Wep19_init__FP7cPlayer @ 0x00001A00
x86 init:   VA 0x006A3410 / file off 0x002A3410
PMD:        wep1900.pmd
Hit group:  THROWABLE_DYNAMIC_CANDIDATE
Hit off:    0x0030086E
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Candidato: REL não tem call direto extraído, validar com teste.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `THROWABLE_DYNAMIC_CANDIDATE` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x0062307C / string off: 0x005CAA04
DAT ptr file off: 0x00623078 / string off: 0x005CAA14
SND ptr file off: 0x00623074 / string off: 0x005CAA24
```

### wep20 / 0x14 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep20.dat
SND:        em\wep20.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x00623084 / string off: 0x005CA9E4
SND ptr file off: 0x00623080 / string off: 0x005CA9F4
```

### wep21 / 0x15 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep21.dat
SND:        em\wep21.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x00623090 / string off: 0x005CA9C4
SND ptr file off: 0x0062308C / string off: 0x005CA9D4
```

### wep22 / 0x16 - Grenade/incendiary candidate model

```txt
REL:        
DAT:        
SND:        
REL init:    @ 
x86 init:   VA 0x006A3410 | 0x00704570 | 0x00890627 | 0x00891BF7 | 0x00892400 | 0x00892F30 | 0x00896E00 / file off 0x002A3410 | 0x00304570 | 0x00490627 | 0x00491BF7 | 0x00492400 | 0x00492F30 | 0x00496E00
PMD:        wep2200.pmd
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

### wep23 / 0x17 - Grenade/flash candidate model

```txt
REL:        
DAT:        
SND:        
REL init:    @ 
x86 init:   VA 0x005E4390 | 0x006A3410 | 0x00704570 | 0x00890627 | 0x00891BF7 | 0x00892400 | 0x00892F30 | 0x00896E00 / file off 0x001E4390 | 0x002A3410 | 0x00304570 | 0x00490627 | 0x00491BF7 | 0x00492400 | 0x00492F30 | 0x00496E00
PMD:        wep2300.pmd
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

### wep24 / 0x18 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep24.dat
SND:        em\wep24.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x0062309C / string off: 0x005CA9A4
SND ptr file off: 0x00623098 / string off: 0x005CA9B4
```

### wep25 / 0x19 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep25.dat
SND:        em\wep25.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x006230A8 / string off: 0x005CA984
SND ptr file off: 0x006230A4 / string off: 0x005CA994
```

### wep26 / 0x1A - Knife

```txt
REL:        wep26.rel
DAT:        
SND:        
REL init:   Wep26_init__FP7cPlayer @ 0x000019B0
x86 init:   VA 0x00702070 / file off 0x00302070
PMD:        wep2600.pmd
Hit group:  KNIFE_SWEEP_1 | KNIFE_SWEEP_2 | KNIFE_SWEEP_3 | KNIFE_SWEEP_4 | KNIFE_SWEEP_5 | KNIFE_SWEEP_6 | KNIFE_SWEEP_7 | KNIFE_SPECIAL_HIT
Hit off:    0x002D7866 | 0x002D78F0 | 0x002D7942 | 0x002D79BD | 0x002D79FC | 0x002D7A33 | 0x002D7A67 | 0x002DB8A2
Original:   0x10 | 0x10 | 0x10 | 0x10 | 0x10 | 0x10 | 0x10 | 0x03
Patch:      6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX> | 6A <XX>
Notas:      Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Callsite extra com hit 0x03 próximo ao fluxo de knife/close hit.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `KNIFE_SWEEP_1` | `0x002D7866` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_2` | `0x002D78F0` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_3` | `0x002D7942` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_4` | `0x002D79BD` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_5` | `0x002D79FC` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_6` | `0x002D7A33` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SWEEP_7` | `0x002D7A67` | `6A 10` | `6A 13` | `6A 17` | `6A 2E` |
| `KNIFE_SPECIAL_HIT` | `0x002DB8A2` | `6A 03` | `6A 13` | `6A 17` | `6A 2E` |

Resource table offsets:
```txt
REL ptr file off: 0x006234CC / string off: 0x005C96A4
DAT ptr file off:  / string off: 
SND ptr file off:  / string off: 
```

### wep27 / 0x1B - Machinegun

```txt
REL:        wep27.rel
DAT:        
SND:        
REL init:   Wep27_init__FP7cPlayer @ 0x00001578
x86 init:   VA 0x00702B50 / file off 0x00302B50
PMD:        wep2700.pmd
Hit group:  MACHINE_SHARED_A | MACHINE_SHARED_B
Hit off:    0x002E0688 | 0x002E0826
Original:   current_weapon | current_weapon
Patch:      BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90
Notas:      Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `MACHINE_SHARED_A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA 13 00 00 00 90 90 90` | `BA 17 00 00 00 90 90 90` | `BA 2E 00 00 00 90 90 90` |
| `MACHINE_SHARED_B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006234D0 / string off: 0x005C9694
DAT ptr file off:  / string off: 
SND ptr file off:  / string off: 
```

### wep28 / 0x1C - Bow

```txt
REL:        wep28.rel
DAT:        em\wep28.dat
SND:        em\wep28.snd
REL init:   Wep28_init__FP7cPlayer @ 0x00000F70
x86 init:   VA 0x007038D0 / file off 0x003038D0
PMD:        wep2800.pmd
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x006231D8 / string off: 0x005CA524
DAT ptr file off: 0x006231D4 / string off: 0x005CA534
SND ptr file off: 0x006231D0 / string off: 0x005CA544
```

### wep29 / 0x1D - Machinegun

```txt
REL:        wep29.rel
DAT:        em\wep29.dat
SND:        em\wep29.snd
REL init:   Wep29_init__FP7cPlayer @ 0x00002138
x86 init:   VA 0x00696BF0 / file off 0x00296BF0
PMD:        wep2900.pmd
Hit group:  MACHINE_SHARED_A | MACHINE_SHARED_B
Hit off:    0x002E0688 | 0x002E0826
Original:   current_weapon | current_weapon
Patch:      BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90
Notas:      Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `MACHINE_SHARED_A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA 13 00 00 00 90 90 90` | `BA 17 00 00 00 90 90 90` | `BA 2E 00 00 00 90 90 90` |
| `MACHINE_SHARED_B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623178 / string off: 0x005CA6A4
DAT ptr file off: 0x00623174 / string off: 0x005CA6B4
SND ptr file off: 0x00623170 / string off: 0x005CA6C4
```

### wep30 / 0x1E - HandGre

```txt
REL:        wep30.rel
DAT:        em\wep30.dat
SND:        em\wep30.snd
REL init:   Wep30_init__FP7cPlayer @ 0x00001A00
x86 init:   VA  / file off 
PMD:        
Hit group:  THROWABLE_DYNAMIC_CANDIDATE
Hit off:    0x0030086E
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Candidato: REL não tem call direto extraído, validar com teste.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `THROWABLE_DYNAMIC_CANDIDATE` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623158 / string off: 0x005CA724
DAT ptr file off: 0x00623154 / string off: 0x005CA734
SND ptr file off: 0x00623150 / string off: 0x005CA744
```

### wep31 / 0x1F - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep31.dat
SND:        em\wep31.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x006230C0 / string off: 0x005CA944
SND ptr file off: 0x006230BC / string off: 0x005CA954
```

### wep32 / 0x20 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep32.dat
SND:        em\wep32.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x006230CC / string off: 0x005CA924
SND ptr file off: 0x006230C8 / string off: 0x005CA934
```

### wep33 / 0x21 - Shotgun

```txt
REL:        wep33.rel
DAT:        em\wep33.dat
SND:        em\wep33.snd
REL init:   Wep33_init__FP7cPlayer @ 0x00001980
x86 init:   VA 0x007051C0 / file off 0x003051C0
PMD:        wep3300.pmd
Hit group:  SHOTGUN_SHARED
Hit off:    0x002F19A3
Original:   current_weapon
Patch:      B8 <XX> 00 00 00 90 90 90
Notas:      Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `SHOTGUN_SHARED` | `0x002F19A3` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006230DC / string off: 0x005CA8F4
DAT ptr file off: 0x006230D8 / string off: 0x005CA904
SND ptr file off: 0x006230D4 / string off: 0x005CA914
```

### wep34 / 0x22 - Hand

```txt
REL:        wep34.rel
DAT:        em\wep34.dat
SND:        
REL init:   Wep34_init__FP7cPlayer @ 0x000001A0
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x00623104 / string off: 0x005CA874
DAT ptr file off: 0x00623100 / string off: 0x005CA884
SND ptr file off:  / string off: 
```

### wep35 / 0x23 - Hand

```txt
REL:        wep35.rel
DAT:        em\wep35.dat
SND:        
REL init:   Wep35_init__FP7cPlayer @ 0x000001A0
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x0062316C / string off: 0x005CA6D4
DAT ptr file off: 0x00623168 / string off: 0x005CA6E4
SND ptr file off:  / string off: 
```

### wep36 / 0x24 - Hand

```txt
REL:        wep36.rel
DAT:        em\wep36.dat
SND:        
REL init:   Wep36_init__FP7cPlayer @ 0x000001A0
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x006231C0 / string off: 0x005CA584
DAT ptr file off: 0x006231BC / string off: 0x005CA594
SND ptr file off:  / string off: 
```

### wep37 / 0x25 - Hand

```txt
REL:        wep37.rel
DAT:        em\wep37.dat
SND:        
REL init:   Wep37_init__FP7cPlayer @ 0x000001A0
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x0062318C / string off: 0x005CA654
DAT ptr file off: 0x00623188 / string off: 0x005CA664
SND ptr file off:  / string off: 
```

### wep38 / 0x26 - Ruger

```txt
REL:        wep38.rel
DAT:        em\wep38.dat
SND:        em\wep38.snd
REL init:   Wep38_init__FP7cPlayer @ 0x000015D8
x86 init:   VA 0x00706FC0 / file off 0x00306FC0
PMD:        wep3800.pmd
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623110 / string off: 0x005CA844
DAT ptr file off: 0x0062310C / string off: 0x005CA854
SND ptr file off: 0x00623108 / string off: 0x005CA864
```

### wep39 / 0x27 - Machinegun

```txt
REL:        wep39.rel
DAT:        em\wep39.dat
SND:        em\wep39.snd
REL init:   Wep39_init__FP7cPlayer @ 0x00001578
x86 init:   VA 0x00707B40 / file off 0x00307B40
PMD:        wep3900.pmd
Hit group:  MACHINE_SHARED_A | MACHINE_SHARED_B
Hit off:    0x002E0688 | 0x002E0826
Original:   current_weapon | current_weapon
Patch:      BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90
Notas:      Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `MACHINE_SHARED_A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA 13 00 00 00 90 90 90` | `BA 17 00 00 00 90 90 90` | `BA 2E 00 00 00 90 90 90` |
| `MACHINE_SHARED_B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623140 / string off: 0x005CA784
DAT ptr file off: 0x0062313C / string off: 0x005CA794
SND ptr file off: 0x00623138 / string off: 0x005CA7A4
```

### wep40 / 0x28 - HkSniper

```txt
REL:        wep40.rel
DAT:        em\wep40.dat
SND:        em\wep40.snd
REL init:   Wep40_init__FP7cPlayer @ 0x00001770
x86 init:   VA 0x00708470 / file off 0x00308470
PMD:        wep4000.pmd; wep4001.pmd
Hit group:  RIFLE_SHARED
Hit off:    0x002EE77B
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Rifle/sniper usa CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `RIFLE_SHARED` | `0x002EE77B` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623134 / string off: 0x005CA7B4
DAT ptr file off: 0x00623130 / string off: 0x005CA7C4
SND ptr file off: 0x0062312C / string off: 0x005CA7D4
```

### wep41 / 0x29 - HandGre

```txt
REL:        wep41.rel
DAT:        em\wep41.dat
SND:        em\wep41.snd
REL init:   Wep41_init__FP7cPlayer @ 0x00001A00
x86 init:   VA  / file off 
PMD:        
Hit group:  THROWABLE_DYNAMIC_CANDIDATE
Hit off:    0x0030086E
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Candidato: REL não tem call direto extraído, validar com teste.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `THROWABLE_DYNAMIC_CANDIDATE` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623184 / string off: 0x005CA674
DAT ptr file off: 0x00623180 / string off: 0x005CA684
SND ptr file off: 0x0062317C / string off: 0x005CA694
```

### wep42 / 0x2A - HandGre

```txt
REL:        wep42.rel
DAT:        em\wep42.dat
SND:        em\wep42.snd
REL init:   Wep42_init__FP7cPlayer @ 0x00001A00
x86 init:   VA  / file off 
PMD:        
Hit group:  THROWABLE_DYNAMIC_CANDIDATE
Hit off:    0x0030086E
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Candidato: REL não tem call direto extraído, validar com teste.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `THROWABLE_DYNAMIC_CANDIDATE` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006231CC / string off: 0x005CA554
DAT ptr file off: 0x006231C8 / string off: 0x005CA564
SND ptr file off: 0x006231C4 / string off: 0x005CA574
```

### wep43 / 0x2B - Ruger

```txt
REL:        wep43.rel
DAT:        em\wep43.dat
SND:        em\wep43.snd
REL init:   Wep43_init__FP7cPlayer @ 0x000020B0
x86 init:   VA  / file off 
PMD:        
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623198 / string off: 0x005CA624
DAT ptr file off: 0x00623194 / string off: 0x005CA634
SND ptr file off: 0x00623190 / string off: 0x005CA644
```

### wep44 / 0x2C - Government

```txt
REL:        wep44.rel
DAT:        em\wep44.dat
SND:        em\wep44.snd
REL init:   Wep44_init__FP7cPlayer @ 0x000015D8
x86 init:   VA  / file off 
PMD:        
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006231B8 / string off: 0x005CA5A4
DAT ptr file off: 0x006231B4 / string off: 0x005CA5B4
SND ptr file off: 0x006231B0 / string off: 0x005CA5C4
```

### wep45 / 0x2D - HandGre

```txt
REL:        wep45.rel
DAT:        em\wep45.dat
SND:        
REL init:   Wep45_init__FP7cPlayer @ 0x00001A00
x86 init:   VA  / file off 
PMD:        
Hit group:  THROWABLE_DYNAMIC_CANDIDATE
Hit off:    0x0030086E
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Candidato: REL não tem call direto extraído, validar com teste.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `THROWABLE_DYNAMIC_CANDIDATE` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006231A0 / string off: 0x005CA604
DAT ptr file off: 0x0062319C / string off: 0x005CA614
SND ptr file off:  / string off: 
```

### wep46 / 0x2E - Rocket-family

```txt
REL:        wep46.rel
DAT:        em\wep46.dat
SND:        em\wep46.snd
REL init:   Wep46_init__FP7cPlayer @ 0x00001958
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x006234D4 / string off: 0x005C9684
DAT ptr file off: 0x006230B4 / string off: 0x005CA964
SND ptr file off: 0x006230B0 / string off: 0x005CA974
```

### wep47 / 0x2F - HkSniper

```txt
REL:        wep47.rel
DAT:        em\wep47.dat
SND:        em\wep47.snd
REL init:   Wep47_init__FP7cPlayer @ 0x00001770
x86 init:   VA  / file off 
PMD:        
Hit group:  RIFLE_SHARED
Hit off:    0x002EE77B
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Rifle/sniper usa CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `RIFLE_SHARED` | `0x002EE77B` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x006231AC / string off: 0x005CA5D4
DAT ptr file off: 0x006231A8 / string off: 0x005CA5E4
SND ptr file off: 0x006231A4 / string off: 0x005CA5F4
```

### wep48 / 0x30 - Shotgun

```txt
REL:        wep48.rel
DAT:        em\wep48.dat
SND:        em\wep48.snd
REL init:   Wep48_init__FP7cPlayer @ 0x00001980
x86 init:   VA 0x0070C260 / file off 0x0030C260
PMD:        wep4800.pmd
Hit group:  SHOTGUN_SHARED
Hit off:    0x002F19A3
Original:   current_weapon
Patch:      B8 <XX> 00 00 00 90 90 90
Notas:      Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `SHOTGUN_SHARED` | `0x002F19A3` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x00623128 / string off: 0x005CA7E4
DAT ptr file off: 0x00623124 / string off: 0x005CA7F4
SND ptr file off: 0x00623120 / string off: 0x005CA804
```

### wep49 / 0x31 - Tompson

```txt
REL:        wep49.rel
DAT:        em\wep49.dat
SND:        em\wep49.snd
REL init:   Wep49_init__FP7cPlayer @ 0x00002120
x86 init:   VA 0x006A4170 / file off 0x002A4170
PMD:        wep4900.pmd
Hit group:  MACHINE_SHARED_A | MACHINE_SHARED_B
Hit off:    0x002E0688 | 0x002E0826
Original:   current_weapon | current_weapon
Patch:      BA <XX> 00 00 00 90 90 90 | B8 <XX> 00 00 00 90 90 90
Notas:      Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `MACHINE_SHARED_A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA 13 00 00 00 90 90 90` | `BA 17 00 00 00 90 90 90` | `BA 2E 00 00 00 90 90 90` |
| `MACHINE_SHARED_B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 13 00 00 00 90 90 90` | `B8 17 00 00 00 90 90 90` | `B8 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x0062314C / string off: 0x005CA754
DAT ptr file off: 0x00623148 / string off: 0x005CA764
SND ptr file off: 0x00623144 / string off: 0x005CA774
```

### wep50 / 0x32 - AdaBow

```txt
REL:        wep50.rel
DAT:        em\wep50.dat
SND:        em\wep50.snd
REL init:   Wep50_init__FP7cPlayer @ 0x000015D8
x86 init:   VA 0x0070D8D0 / file off 0x0030D8D0
PMD:        wep5000.pmd
Hit group:  SEM_PLWEPHITCHECK2_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off: 0x00623164 / string off: 0x005CA6F4
DAT ptr file off: 0x00623160 / string off: 0x005CA704
SND ptr file off: 0x0062315C / string off: 0x005CA714
```

### wep51 / 0x33 - Laser

```txt
REL:        wep51.rel
DAT:        em\wep51.dat
SND:        em\wep51.snd
REL init:   Wep51_init__FP7cPlayer @ 0x00002850
x86 init:   VA  / file off 
PMD:        
Hit group:  LASER_PRL_1 | LASER_PRL_2 | LASER_PRL_3 | LASER_PRL_4
Hit off:    0x00295477 | 0x0029551D | 0x002955C9 | 0x002957AB
Original:   0x2E | 0x2E | 0x2E | 0x2E
Patch:      6A <XX> | 6A <XX> | 6A <XX> | 6A <XX>
Notas:      PRL/Laser hit. 0x2E é o hit observado do PRL-like. | PRL/Laser hit. 0x2E é o hit observado do PRL-like. | PRL/Laser hit. 0x2E é o hit observado do PRL-like. | PRL/Laser hit. 0x2E é o hit observado do PRL-like.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `LASER_PRL_1` | `0x00295477` | `6A 2E` | `6A 13` | `6A 17` | `6A 2E` |
| `LASER_PRL_2` | `0x0029551D` | `6A 2E` | `6A 13` | `6A 17` | `6A 2E` |
| `LASER_PRL_3` | `0x002955C9` | `6A 2E` | `6A 13` | `6A 17` | `6A 2E` |
| `LASER_PRL_4` | `0x002957AB` | `6A 2E` | `6A 13` | `6A 17` | `6A 2E` |

Resource table offsets:
```txt
REL ptr file off: 0x006230FC / string off: 0x005CA8A4
DAT ptr file off: 0x006230F0 / string off: 0x005CA8B4
SND ptr file off: 0x006230EC / string off: 0x005CA8C4
```

### wep52 / 0x34 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep52.dat
SND:        
REL init:    @ 
x86 init:   VA 0x006961B0 / file off 0x002961B0
PMD:        wep5200.pmd
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x006230F8 / string off: 0x005CA894
SND ptr file off:  / string off: 
```

### wep54 / 0x36 - Xd9

```txt
REL:        wep54.rel
DAT:        em\wep54.dat
SND:        em\wep54.snd
REL init:   Wep54_init__FP7cPlayer @ 0x000015D8
x86 init:   VA  / file off 
PMD:        
Hit group:  HANDGUN_SHARED
Hit off:    0x002D4DAE
Original:   current_weapon
Patch:      B9 <XX> 00 00 00 90 90 90
Notas:      Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX.
```

| Grupo | Offset | Original | Patch para 0x13 | Patch para 0x17 | Patch para 0x2E |
|---|---:|---|---|---|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 13 00 00 00 90 90 90` | `B9 17 00 00 00 90 90 90` | `B9 2E 00 00 00 90 90 90` |

Resource table offsets:
```txt
REL ptr file off: 0x0062311C / string off: 0x005CA814
DAT ptr file off: 0x00623118 / string off: 0x005CA824
SND ptr file off: 0x00623114 / string off: 0x005CA834
```

### wep55 / 0x37 - DAT/SND only or resource-only entry

```txt
REL:        
DAT:        em\wep55.dat
SND:        em\wep55.snd
REL init:    @ 
x86 init:   VA  / file off 
PMD:        
Hit group:  SEM_MAPA_DIRETO
Hit off:    
Original:   
Patch:      
Notas:      Sem mapeamento direto para PlWepHitCheck2.
```

Sem offset direto de `PlWepHitCheck2` mapeado para HxD. Provavelmente usa projétil, ETM/ETS, script, área ambiental ou fluxo próprio.

Resource table offsets:
```txt
REL ptr file off:  / string off: 
DAT ptr file off: 0x006230E4 / string off: 0x005CA8D4
SND ptr file off: 0x006230E0 / string off: 0x005CA8E4
```

## Restore bytes

Use estes bytes para restaurar os patches de hit principais:

| Grupo | Offset | Restore bytes |
|---|---:|---|
| `HANDGUN_SHARED` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` |
| `MACHINE_SHARED_A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` |
| `MACHINE_SHARED_B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` |
| `RIFLE_SHARED` | `0x002EE77B` | `66 0F B6 88 96 5F 00 00` |
| `SHOTGUN_SHARED` | `0x002F19A3` | `66 0F B6 82 96 5F 00 00` |
| `THROWABLE_DYNAMIC_CANDIDATE` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` |
| `KNIFE_SWEEP_1` | `0x002D7866` | `6A 10` |
| `KNIFE_SWEEP_2` | `0x002D78F0` | `6A 10` |
| `KNIFE_SWEEP_3` | `0x002D7942` | `6A 10` |
| `KNIFE_SWEEP_4` | `0x002D79BD` | `6A 10` |
| `KNIFE_SWEEP_5` | `0x002D79FC` | `6A 10` |
| `KNIFE_SWEEP_6` | `0x002D7A33` | `6A 10` |
| `KNIFE_SWEEP_7` | `0x002D7A67` | `6A 10` |
| `KNIFE_SPECIAL_HIT` | `0x002DB8A2` | `6A 03` |
| `WEP17_VP70_CANDIDATE` | `0x002A2ABB` | `B8 1F 00 00 00` |
| `LASER_PRL_1` | `0x00295477` | `6A 2E` |
| `LASER_PRL_2` | `0x0029551D` | `6A 2E` |
| `LASER_PRL_3` | `0x002955C9` | `6A 2E` |
| `LASER_PRL_4` | `0x002957AB` | `6A 2E` |