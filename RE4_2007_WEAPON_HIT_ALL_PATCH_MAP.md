# RE4 2007 - Weapon Hit Map / PlWepHitCheck2

Base analisada: `BIO4.exe` RE4 2007 PC. Função alvo: `PlWepHitCheck2` via thunk `0x00404B97`.

## Leitura rápida

- Em armas normais, o 4º argumento de `PlWepHitCheck2` é o `hit_kind`.

- Muitos handlers não usam um valor fixo: carregam `PTR_DAT_00A3A280[0x5F96]`, ou seja, `CurrentWeaponKind`.

- Para trocar o hit de uma família inteira, patchar o `movzbw [CurrentWeaponKind]` para `MOV reg, XX`.

- Para hits fixos, patchar `PUSH imm8`: `6A valor`.


## Hit values observados em todos os calls para PlWepHitCheck2

| Hit | Qtde | Interpretação |
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

## Grupos principais de patch por família de arma

| Grupo | WEPs afetadas | Função REL | Offset do hit | Original | Fórmula |
|---|---|---|---:|---|---|
| `HANDGUN_SHARED` | `wep01,wep02,wep04,wep05,wep06,wep15,wep38,wep43,wep44,wep54` | `PlHandgunMove__FP7cPlayer` | `0x002D4DAE` | `66 0F B6 88 96 5F 00 00` | `B9 <XX> 00 00 00 90 90 90` |
| `MACHINE_SHARED_A` | `wep11,wep12,wep27,wep29,wep39,wep49` | `PlMachineMove__FP7cPlayer path A` | `0x002E0688` | `66 0F B6 91 96 5F 00 00` | `BA <XX> 00 00 00 90 90 90` |
| `MACHINE_SHARED_B` | `wep11,wep12,wep27,wep29,wep39,wep49` | `PlMachineMove__FP7cPlayer path B` | `0x002E0826` | `66 0F B6 82 96 5F 00 00` | `B8 <XX> 00 00 00 90 90 90` |
| `RIFLE_SHARED` | `wep09,wep10,wep40,wep47` | `PlRifleMove__FP7cPlayer` | `0x002EE77B` | `66 0F B6 88 96 5F 00 00` | `B9 <XX> 00 00 00 90 90 90` |
| `SHOTGUN_SHARED` | `wep07,wep08,wep33,wep48` | `PlShotgunMove__FP7cPlayer` | `0x002F19A3` | `66 0F B6 82 96 5F 00 00` | `B8 <XX> 00 00 00 90 90 90` |
| `THROWABLE_DYNAMIC_CANDIDATE` | `wep19,wep30,wep41,wep42,wep45` | `PlGrenadeMove / throwable candidate` | `0x0030086E` | `66 0F B6 88 96 5F 00 00` | `B9 <XX> 00 00 00 90 90 90` |
| `KNIFE_SWEEP_1` | `wep16,wep26` | `PlKnifeMove__FP7cPlayer` | `0x002D7866` | `6A 10` | `6A <XX>` |
| `KNIFE_SWEEP_2` | `wep16,wep26` | `PlKnifeMove__FP7cPlayer` | `0x002D78F0` | `6A 10` | `6A <XX>` |
| `KNIFE_SWEEP_3` | `wep16,wep26` | `PlKnifeMove__FP7cPlayer` | `0x002D7942` | `6A 10` | `6A <XX>` |
| `KNIFE_SWEEP_4` | `wep16,wep26` | `PlKnifeMove__FP7cPlayer` | `0x002D79BD` | `6A 10` | `6A <XX>` |
| `KNIFE_SWEEP_5` | `wep16,wep26` | `PlKnifeMove__FP7cPlayer` | `0x002D79FC` | `6A 10` | `6A <XX>` |
| `KNIFE_SWEEP_6` | `wep16,wep26` | `PlKnifeMove__FP7cPlayer` | `0x002D7A33` | `6A 10` | `6A <XX>` |
| `KNIFE_SWEEP_7` | `wep16,wep26` | `PlKnifeMove__FP7cPlayer` | `0x002D7A67` | `6A 10` | `6A <XX>` |
| `KNIFE_SPECIAL_HIT` | `wep16/wep26 candidate` | `knife/special close hit candidate` | `0x002DB8A2` | `6A 03` | `6A <XX>` |
| `WEP17_VP70_CANDIDATE` | `wep17` | `Wep17_move__FP7cPlayer candidate` | `0x002A2ABB` | `B8 1F 00 00 00` | `B8 <XX> 00 00 00` |
| `LASER_PRL_1` | `wep51` | `setBullet__9cObjLaser` | `0x00295477` | `6A 2E` | `6A <XX>` |
| `LASER_PRL_2` | `wep51` | `setBullet__9cObjLaser` | `0x0029551D` | `6A 2E` | `6A <XX>` |
| `LASER_PRL_3` | `wep51` | `setBullet__9cObjLaser` | `0x002955C9` | `6A 2E` | `6A <XX>` |
| `LASER_PRL_4` | `wep51` | `setBullet__9cObjLaser` | `0x002957AB` | `6A 2E` | `6A <XX>` |

## Exemplos de patch

Para trocar Shotgun/Striker para hit de granada/explosão `0x13`:
```txt
Offset 0x002F19A3
Original: 66 0F B6 82 96 5F 00 00
Patch:    B8 13 00 00 00 90 90 90
```
Para trocar TMP/Machinegun para hit de PRL/Laser `0x2E`, aplique os dois paths:
```txt
Offset 0x002E0688
Patch: BA 2E 00 00 00 90 90 90

Offset 0x002E0826
Patch: B8 2E 00 00 00 90 90 90
```
Para trocar Handgun-like para hit de granada `0x13`:
```txt
Offset 0x002D4DAE
Patch: B9 13 00 00 00 90 90 90
```

## Mapa por WEP

| WEP | Família | REL funcs de hit | Grupos/offsets | Observação |
|---|---|---|---|---|
| `wep00` | `Hand` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep01` | `Fn57` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep02` | `Mauser` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep03` | `nan` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep04` | `Xd9` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep05` | `Civilian` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep06` | `Government` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep07` | `Shotgun` | `PlShotgunMove__FP7cPlayer` | `SHOTGUN_SHARED / `0x002F19A3` | Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX. |
| `wep08` | `Striker` | `PlShotgunMove__FP7cPlayer` | `SHOTGUN_SHARED / `0x002F19A3` | Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX. |
| `wep09` | `Sniper` | `PlRifleMove__FP7cPlayer` | `RIFLE_SHARED / `0x002EE77B` | Rifle/sniper usa CurrentWeaponKind via ECX. |
| `wep10` | `HkSniper` | `PlRifleMove__FP7cPlayer` | `RIFLE_SHARED / `0x002EE77B` | Rifle/sniper usa CurrentWeaponKind via ECX. |
| `wep11` | `Machinegun` | `PlMachineMove__FP7cPlayer` | `MACHINE_SHARED_A | MACHINE_SHARED_B / `0x002E0688 | 0x002E0826` | Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson. |
| `wep12` | `Tompson` | `PlMachineMove__FP7cPlayer` | `MACHINE_SHARED_A | MACHINE_SHARED_B / `0x002E0688 | 0x002E0826` | Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson. |
| `wep13` | `Rocket-family` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep14` | `Mine` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep15` | `Magnum` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep16` | `Knife` | `PlKnifeMove__FP7cPlayer` | `KNIFE_SWEEP_1 | KNIFE_SWEEP_2 | KNIFE_SWEEP_3 | KNIFE_SWEEP_4 | KNIFE_SWEEP_5 | KNIFE_SWEEP_6 | KNIFE_SWEEP_7 | KNIFE_SPECIAL_HIT / `0x002D7866 | 0x002D78F0 | 0x002D7942 | 0x002D79BD | 0x002D79FC | 0x002D7A33 | 0x002D7A67 | 0x002DB8A2` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Callsite extra com hit 0x03 próximo ao fluxo de knife/close hit. |
| `wep17` | `Vp70` | `Wep17_move__FP7cPlayer` | `WEP17_VP70_CANDIDATE / `0x002A2ABB` | Candidato fixo do Wep17/VP70: move EAX,0x1F antes do push. |
| `wep19` | `HandGre` | `` | `THROWABLE_DYNAMIC_CANDIDATE / `0x0030086E` | Candidato: REL não tem call direto extraído, validar com teste. |
| `wep26` | `Knife` | `PlKnifeMove__FP7cPlayer` | `KNIFE_SWEEP_1 | KNIFE_SWEEP_2 | KNIFE_SWEEP_3 | KNIFE_SWEEP_4 | KNIFE_SWEEP_5 | KNIFE_SWEEP_6 | KNIFE_SWEEP_7 | KNIFE_SPECIAL_HIT / `0x002D7866 | 0x002D78F0 | 0x002D7942 | 0x002D79BD | 0x002D79FC | 0x002D7A33 | 0x002D7A67 | 0x002DB8A2` | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Um dos sete traços/hits do knife. Patchar todos para trocar o hit do knife por completo. | Callsite extra com hit 0x03 próximo ao fluxo de knife/close hit. |
| `wep27` | `Machinegun` | `PlMachineMove__FP7cPlayer` | `MACHINE_SHARED_A | MACHINE_SHARED_B / `0x002E0688 | 0x002E0826` | Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson. |
| `wep28` | `Bow` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep29` | `Machinegun` | `PlMachineMove__FP7cPlayer` | `MACHINE_SHARED_A | MACHINE_SHARED_B / `0x002E0688 | 0x002E0826` | Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson. |
| `wep30` | `HandGre` | `` | `THROWABLE_DYNAMIC_CANDIDATE / `0x0030086E` | Candidato: REL não tem call direto extraído, validar com teste. |
| `wep33` | `Shotgun` | `PlShotgunMove__FP7cPlayer` | `SHOTGUN_SHARED / `0x002F19A3` | Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX. |
| `wep34` | `Hand` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep35` | `Hand` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep36` | `Hand` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep37` | `Hand` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep38` | `Ruger` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep39` | `Machinegun` | `PlMachineMove__FP7cPlayer` | `MACHINE_SHARED_A | MACHINE_SHARED_B / `0x002E0688 | 0x002E0826` | Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson. |
| `wep40` | `HkSniper` | `PlRifleMove__FP7cPlayer` | `RIFLE_SHARED / `0x002EE77B` | Rifle/sniper usa CurrentWeaponKind via ECX. |
| `wep41` | `HandGre` | `` | `THROWABLE_DYNAMIC_CANDIDATE / `0x0030086E` | Candidato: REL não tem call direto extraído, validar com teste. |
| `wep42` | `HandGre` | `` | `THROWABLE_DYNAMIC_CANDIDATE / `0x0030086E` | Candidato: REL não tem call direto extraído, validar com teste. |
| `wep43` | `Ruger` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep44` | `Government` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |
| `wep45` | `HandGre` | `` | `THROWABLE_DYNAMIC_CANDIDATE / `0x0030086E` | Candidato: REL não tem call direto extraído, validar com teste. |
| `wep46` | `Rocket-family` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep47` | `HkSniper` | `PlRifleMove__FP7cPlayer` | `RIFLE_SHARED / `0x002EE77B` | Rifle/sniper usa CurrentWeaponKind via ECX. |
| `wep48` | `Shotgun` | `PlShotgunMove__FP7cPlayer` | `SHOTGUN_SHARED / `0x002F19A3` | Loop de pellets da shotgun/striker usa CurrentWeaponKind via EAX. |
| `wep49` | `Tompson` | `PlMachineMove__FP7cPlayer` | `MACHINE_SHARED_A | MACHINE_SHARED_B / `0x002E0688 | 0x002E0826` | Primeiro callsite de machine/TMP/Tompson. | Segundo callsite de machine/TMP/Tompson. |
| `wep50` | `AdaBow` | `` | `SEM_PLWEPHITCHECK2_DIRETO / `` | Sem call direto para PlWepHitCheck2 nos RELs; pode usar projectile/ETM/ITA/estado próprio. |
| `wep51` | `Laser` | `setBullet__9cObjLaser` | `LASER_PRL_1 | LASER_PRL_2 | LASER_PRL_3 | LASER_PRL_4 / `0x00295477 | 0x0029551D | 0x002955C9 | 0x002957AB` | PRL/Laser hit. 0x2E é o hit observado do PRL-like. | PRL/Laser hit. 0x2E é o hit observado do PRL-like. | PRL/Laser hit. 0x2E é o hit observado do PRL-like. | PRL/Laser hit. 0x2E é o hit observado do PRL-like. |
| `wep54` | `Xd9` | `PlHandgunMove__FP7cPlayer` | `HANDGUN_SHARED / `0x002D4DAE` | Troca o hit de armas handgun-like que usam CurrentWeaponKind via ECX. |

## Observação sobre burn

Trocar `hit_kind` altera dano/reação/colisão, mas não garante estado de fogo nos Ganados. O burn real parece depender da cadeia de tocha/fogo/área ou de um helper específico de inimigo, não apenas do valor do hit.


## Arquivos da análise

- `RE4_2007_WEAPON_HIT_PATCH_GROUPS.csv`: grupos práticos por família de arma.

- `RE4_2007_WEAPON_HIT_BY_WEP.csv`: mapa por `wepXX.rel`.

- `RE4_2007_ALL_PLWEP_HIT_CALLS_PATCH_VALUES.csv`: todos os 100 callsites encontrados com patch bytes para vários hit values.

- `RE4_2007_OBSERVED_HIT_VALUES.csv`: contagem dos valores observados.
