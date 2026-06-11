# RE4 2007 PC — Mapa das INITs de WEP

Este relatório organiza as funções de inicialização das armas (`wepXX`) usando duas fontes:

- símbolos reais dos módulos `wepXX.rel` SNR2/MIPS;
- candidatos x86 no `BIO4.exe` encontrados por paths `XFile\Em\wep\wepXXXX.pmd`.

## Resultado geral

- WEP RELs mapeados: **44**
- Símbolos INIT REL extraídos: **126**
- Candidatos x86 por modelo PMD: **73**

## Tabela principal

| WEP | Família/objeto | REL WepXX_init | REL offset | x86 init/model candidate | File off | Modelos | Confiança |
|---|---|---|---:|---:|---:|---|---|
| `wep00` | Hand | `Wep00_init__FP7cPlayer` | `0x000001A0` | `` | `` |  | Sem candidato direto por PMD |
| `wep01` | Fn57 | `Wep01_init__FP7cPlayer` | `0x00002068` | `0x006919E0` | `0x002919E0` | wep0100.pmd; wep0101.pmd | Alta |
| `wep02` | Mauser | `Wep02_init__FP7cPlayer` | `0x00002FE8` | `0x006A0780` | `0x002A0780` | wep0200.pmd; wep0201.pmd | Alta |
| `wep03` |  | `Wep03_init__FP7cPlayer` | `0x000001A0` | `0x00697AA0` | `0x00297AA0` | wep0300.pmd; wep0301.pmd | Alta |
| `wep04` | Xd9 | `Wep04_init__FP7cPlayer` | `0x000015D8` | `0x006FAFA0` | `0x002FAFA0` | wep0400.pmd; wep0401.pmd | Alta |
| `wep05` | Civilian | `Wep05_init__FP7cPlayer` | `0x00001E30` | `0x00690FD0` | `0x00290FD0` | wep0500.pmd | Alta |
| `wep06` | Government | `Wep06_init__FP7cPlayer` | `0x000015D8` | `` | `` |  | Sem candidato direto por PMD |
| `wep07` | Shotgun | `Wep07_init__FP7cPlayer` | `0x00001980` | `0x006FC790` | `0x002FC790` | wep0700.pmd | Alta |
| `wep08` | Striker | `Wep08_init__FP7cPlayer` | `0x00001980` | `0x006FD180` | `0x002FD180` | wep0800.pmd | Alta |
| `wep09` | Sniper | `Wep09_init__FP7cPlayer` | `0x00002080` | `0x006A1150` | `0x002A1150` | wep0900.pmd; wep0901.pmd; wep0902.pmd | Alta |
| `wep10` | HkSniper | `Wep10_init__FP7cPlayer` | `0x00001EA0` | `0x00694760` | `0x00294760` | wep1000.pmd; wep1002.pmd | Alta |
| `wep11` | Machinegun | `Wep11_init__FP7cPlayer` | `0x00002138` | `0x00696BF0` | `0x00296BF0` | wep1100.pmd; wep1101.pmd; wep1102.pmd; wep1103.pmd | Alta |
| `wep12` | Tompson | `Wep12_init__FP7cPlayer` | `0x00002120` | `0x006A4170` | `0x002A4170` | wep1200.pmd | Alta |
| `wep13` | Rocket-family | `Wep13_init__FP7cPlayer` | `0x00001958` | `0x0069FC60` | `0x0029FC60` | wep1300.pmd | Alta |
| `wep14` | Mine | `Wep14_init__FP7cPlayer` | `0x00001090` | `0x00698B80` | `0x00298B80` | wep1400.pmd; wep1401.pmd | Alta |
| `wep15` | Magnum | `Wep15_init__FP7cPlayer` | `0x000015D8` | `0x006FF020` | `0x002FF020` | wep1500.pmd | Alta |
| `wep16` | Knife | `Wep16_init__FP7cPlayer` | `0x000019B0` | `0x006FF540` | `0x002FF540` | wep1600.pmd | Alta |
| `wep17` | Vp70 | `Wep17_init__FP7cPlayer` | `0x00000BD0` | `0x006A6960` | `0x002A6960` | wep1700.pmd | Alta |
| `wep19` | HandGre | `Wep19_init__FP7cPlayer` | `0x00001A00` | `0x006A3410` | `0x002A3410` | wep1900.pmd | Alta |
| `wep26` | Knife | `Wep26_init__FP7cPlayer` | `0x000019B0` | `0x00702070` | `0x00302070` | wep2600.pmd | Alta |
| `wep27` | Machinegun | `Wep27_init__FP7cPlayer` | `0x00001578` | `0x00702B50` | `0x00302B50` | wep2700.pmd | Alta |
| `wep28` | Bow | `Wep28_init__FP7cPlayer` | `0x00000F70` | `0x007038D0` | `0x003038D0` | wep2800.pmd | Alta |
| `wep29` | Machinegun | `Wep29_init__FP7cPlayer` | `0x00002138` | `0x00696BF0` | `0x00296BF0` | wep2900.pmd | Alta |
| `wep30` | HandGre | `Wep30_init__FP7cPlayer` | `0x00001A00` | `` | `` |  | Sem candidato direto por PMD |
| `wep33` | Shotgun | `Wep33_init__FP7cPlayer` | `0x00001980` | `0x007051C0` | `0x003051C0` | wep3300.pmd | Alta |
| `wep34` | Hand | `Wep34_init__FP7cPlayer` | `0x000001A0` | `` | `` |  | Sem candidato direto por PMD |
| `wep35` | Hand | `Wep35_init__FP7cPlayer` | `0x000001A0` | `` | `` |  | Sem candidato direto por PMD |
| `wep36` | Hand | `Wep36_init__FP7cPlayer` | `0x000001A0` | `` | `` |  | Sem candidato direto por PMD |
| `wep37` | Hand | `Wep37_init__FP7cPlayer` | `0x000001A0` | `` | `` |  | Sem candidato direto por PMD |
| `wep38` | Ruger | `Wep38_init__FP7cPlayer` | `0x000015D8` | `0x00706FC0` | `0x00306FC0` | wep3800.pmd | Alta |
| `wep39` | Machinegun | `Wep39_init__FP7cPlayer` | `0x00001578` | `0x00707B40` | `0x00307B40` | wep3900.pmd | Alta |
| `wep40` | HkSniper | `Wep40_init__FP7cPlayer` | `0x00001770` | `0x00708470` | `0x00308470` | wep4000.pmd; wep4001.pmd | Alta |
| `wep41` | HandGre | `Wep41_init__FP7cPlayer` | `0x00001A00` | `` | `` |  | Sem candidato direto por PMD |
| `wep42` | HandGre | `Wep42_init__FP7cPlayer` | `0x00001A00` | `` | `` |  | Sem candidato direto por PMD |
| `wep43` | Ruger | `Wep43_init__FP7cPlayer` | `0x000020B0` | `` | `` |  | Sem candidato direto por PMD |
| `wep44` | Government | `Wep44_init__FP7cPlayer` | `0x000015D8` | `` | `` |  | Sem candidato direto por PMD |
| `wep45` | HandGre | `Wep45_init__FP7cPlayer` | `0x00001A00` | `` | `` |  | Sem candidato direto por PMD |
| `wep46` | Rocket-family | `Wep46_init__FP7cPlayer` | `0x00001958` | `` | `` |  | Sem candidato direto por PMD |
| `wep47` | HkSniper | `Wep47_init__FP7cPlayer` | `0x00001770` | `` | `` |  | Sem candidato direto por PMD |
| `wep48` | Shotgun | `Wep48_init__FP7cPlayer` | `0x00001980` | `0x0070C260` | `0x0030C260` | wep4800.pmd | Alta |
| `wep49` | Tompson | `Wep49_init__FP7cPlayer` | `0x00002120` | `0x006A4170` | `0x002A4170` | wep4900.pmd | Alta |
| `wep50` | AdaBow | `Wep50_init__FP7cPlayer` | `0x000015D8` | `0x0070D8D0` | `0x0030D8D0` | wep5000.pmd | Alta |
| `wep51` | Laser | `Wep51_init__FP7cPlayer` | `0x00002850` | `` | `` |  | Sem candidato direto por PMD |
| `wep54` | Xd9 | `Wep54_init__FP7cPlayer` | `0x000015D8` | `` | `` |  | Sem candidato direto por PMD |

## Leituras importantes

### Handguns / pistolas
- `wep01`: `Wep01_init__FP7cPlayer` / objetos `ObjFn57_init__FP4cObj | init__8cObjFn57PC6cModel` / x86 `0x006919E0` / modelos `wep0100.pmd; wep0101.pmd`
- `wep02`: `Wep02_init__FP7cPlayer` / objetos `ObjMauser_init__FP4cObj | init__10cObjMauserPC6cModel | ObjRuger_init__FP4cObj | init__9cObjRugerPC6cModel` / x86 `0x006A0780` / modelos `wep0200.pmd; wep0201.pmd`
- `wep04`: `Wep04_init__FP7cPlayer` / objetos `ObjXd9_init__FP4cObj | init__7cObjXd9PC6cModel` / x86 `0x006FAFA0` / modelos `wep0400.pmd; wep0401.pmd`
- `wep05`: `Wep05_init__FP7cPlayer` / objetos `ObjCivilian_init__FP4cObj | init__12cObjCivilianPC6cModel` / x86 `0x00690FD0` / modelos `wep0500.pmd`
- `wep06`: `Wep06_init__FP7cPlayer` / objetos `ObjGovernment_init__FP4cObj | init__14cObjGovernmentPC6cModel` / x86 `` / modelos ``
- `wep15`: `Wep15_init__FP7cPlayer` / objetos `ObjMagnum_init__FP4cObj | init__10cObjMagnumPC6cModel` / x86 `0x006FF020` / modelos `wep1500.pmd`
- `wep17`: `Wep17_init__FP7cPlayer` / objetos `ObjVp70_init__FP4cObj | init__8cObjVp70PC6cModel` / x86 `0x006A6960` / modelos `wep1700.pmd`
- `wep38`: `Wep38_init__FP7cPlayer` / objetos `init__9cObjRugerPC6cModel` / x86 `0x00706FC0` / modelos `wep3800.pmd`
- `wep43`: `Wep43_init__FP7cPlayer` / objetos `ObjRuger_init__FP4cObj | init__9cObjRugerPC6cModel` / x86 `` / modelos ``
- `wep44`: `Wep44_init__FP7cPlayer` / objetos `ObjGovernment_init__FP4cObj | init__14cObjGovernmentPC6cModel` / x86 `` / modelos ``
- `wep54`: `Wep54_init__FP7cPlayer` / objetos `ObjXd9_init__FP4cObj | init__7cObjXd9PC6cModel` / x86 `` / modelos ``

### Shotguns
- `wep07`: `Wep07_init__FP7cPlayer` / `ObjShotgun_init__FP4cObj | init__11cObjShotgunPC6cModel` / x86 `0x006FC790` / modelos `wep0700.pmd`
- `wep08`: `Wep08_init__FP7cPlayer` / `ObjStriker_init__FP4cObj | init__11cObjStrikerPC6cModel` / x86 `0x006FD180` / modelos `wep0800.pmd`
- `wep33`: `Wep33_init__FP7cPlayer` / `ObjShotgun_init__FP4cObj | init__11cObjShotgunPC6cModel` / x86 `0x007051C0` / modelos `wep3300.pmd`
- `wep48`: `Wep48_init__FP7cPlayer` / `ObjShotgun_init__FP4cObj | init__11cObjShotgunPC6cModel` / x86 `0x0070C260` / modelos `wep4800.pmd`

### Rifles
- `wep09`: `Wep09_init__FP7cPlayer` / `ObjSniper_init__FP4cObj | init__10cObjSniperPC6cModel` / x86 `0x006A1150` / modelos `wep0900.pmd; wep0901.pmd; wep0902.pmd`
- `wep10`: `Wep10_init__FP7cPlayer` / `ObjHkSniper_init__FP4cObj | init__12cObjHkSniperPC6cModel` / x86 `0x00694760` / modelos `wep1000.pmd; wep1002.pmd`
- `wep40`: `Wep40_init__FP7cPlayer` / `init__12cObjHkSniperPC6cModel | ObjHkSniper_init__FP4cObj` / x86 `0x00708470` / modelos `wep4000.pmd; wep4001.pmd`
- `wep47`: `Wep47_init__FP7cPlayer` / `init__12cObjHkSniperPC6cModel` / x86 `` / modelos ``

### TMP / metralhadoras
- `wep11`: `Wep11_init__FP7cPlayer` / `ObjMachinegun_init__FP4cObj | init__14cObjMachinegunPC6cModel` / x86 `0x00696BF0` / modelos `wep1100.pmd; wep1101.pmd; wep1102.pmd; wep1103.pmd`
- `wep12`: `Wep12_init__FP7cPlayer` / `ObjTompson_init__FP4cObj | init__11cObjTompsonPC6cModel` / x86 `0x006A4170` / modelos `wep1200.pmd`
- `wep27`: `Wep27_init__FP7cPlayer` / `init__14cObjMachinegunPC6cModel | ObjKlauMGun_init__FP4cObj` / x86 `0x00702B50` / modelos `wep2700.pmd`
- `wep29`: `Wep29_init__FP7cPlayer` / `ObjMachinegun_init__FP4cObj | init__14cObjMachinegunPC6cModel` / x86 `0x00696BF0` / modelos `wep2900.pmd`
- `wep39`: `Wep39_init__FP7cPlayer` / `init__14cObjMachinegunPC6cModel` / x86 `0x00707B40` / modelos `wep3900.pmd`
- `wep49`: `Wep49_init__FP7cPlayer` / `ObjTompson_init__FP4cObj | init__11cObjTompsonPC6cModel | ObjTompsonAda_init__FP4cObj | init__14cObjTompsonAdaPC6cModel` / x86 `0x006A4170` / modelos `wep4900.pmd`

### Granadas / explosivos / rockets / bow
- `wep13`: `Wep13_init__FP7cPlayer` / `` / x86 `0x0069FC60` / modelos `wep1300.pmd`
- `wep14`: `Wep14_init__FP7cPlayer` / `ObjMine_init__FP4cObj | init__8cObjMinePC6cModel` / x86 `0x00698B80` / modelos `wep1400.pmd; wep1401.pmd`
- `wep19`: `Wep19_init__FP7cPlayer` / `ObjHandGre_init__FP4cObj | init__11cObjHandGrePC6cModel` / x86 `0x006A3410` / modelos `wep1900.pmd`
- `wep30`: `Wep30_init__FP7cPlayer` / `ObjHandGre_init__FP4cObj | init__11cObjHandGrePC6cModel` / x86 `` / modelos ``
- `wep41`: `Wep41_init__FP7cPlayer` / `ObjHandGre_init__FP4cObj | init__11cObjHandGrePC6cModel` / x86 `` / modelos ``
- `wep42`: `Wep42_init__FP7cPlayer` / `ObjHandGre_init__FP4cObj | init__11cObjHandGrePC6cModel` / x86 `` / modelos ``
- `wep45`: `Wep45_init__FP7cPlayer` / `ObjHandGre_init__FP4cObj | init__11cObjHandGrePC6cModel` / x86 `` / modelos ``
- `wep46`: `Wep46_init__FP7cPlayer` / `` / x86 `` / modelos ``
- `wep28`: `Wep28_init__FP7cPlayer` / `init__7cObjBowPC6cModel | ObjKlauBow_init__FP4cObj | init__9cObjAllowPC6cModel | ObjKlauAllow_init__FP4cObj` / x86 `0x007038D0` / modelos `wep2800.pmd`
- `wep50`: `Wep50_init__FP7cPlayer` / `init__10cObjAdaBowPC6cModel | init__12cObjAdaAllowPC6cModel | ObjAdaBow_init__FP4cObj | ObjAdaAllow_init__FP4cObj` / x86 `0x0070D8D0` / modelos `wep5000.pmd`
- `wep51`: `Wep51_init__FP7cPlayer` / `ObjLaser_init__FP4cObj | init__9cObjLaserPC6cModel` / x86 `` / modelos ``

## Alvos x86 mais fortes

| WEP | x86 VA | File off | Função sugerida | Modelos |
|---|---:|---:|---|---|
| `wep01` | `0x006919E0` | `0x002919E0` | `BIO4_2007_WEP01_InitModel_Fn57` | wep0100.pmd; wep0101.pmd |
| `wep02` | `0x006A0780` | `0x002A0780` | `BIO4_2007_WEP02_InitModel_Mauser` | wep0200.pmd; wep0201.pmd |
| `wep03` | `0x00697AA0` | `0x00297AA0` | `BIO4_2007_WEP03_InitModel_` | wep0300.pmd; wep0301.pmd |
| `wep04` | `0x006FAFA0` | `0x002FAFA0` | `BIO4_2007_WEP04_InitModel_Xd9` | wep0400.pmd; wep0401.pmd |
| `wep05` | `0x00690FD0` | `0x00290FD0` | `BIO4_2007_WEP05_InitModel_Civilian` | wep0500.pmd |
| `wep07` | `0x006FC790` | `0x002FC790` | `BIO4_2007_WEP07_InitModel_Shotgun` | wep0700.pmd |
| `wep08` | `0x006FD180` | `0x002FD180` | `BIO4_2007_WEP08_InitModel_Striker` | wep0800.pmd |
| `wep09` | `0x006A1150` | `0x002A1150` | `BIO4_2007_WEP09_InitModel_Sniper` | wep0900.pmd; wep0901.pmd; wep0902.pmd |
| `wep10` | `0x00694760` | `0x00294760` | `BIO4_2007_WEP10_InitModel_HkSniper` | wep1000.pmd; wep1002.pmd |
| `wep11` | `0x00696BF0` | `0x00296BF0` | `BIO4_2007_WEP11_InitModel_Machinegun` | wep1100.pmd; wep1101.pmd; wep1102.pmd; wep1103.pmd |
| `wep12` | `0x006A4170` | `0x002A4170` | `BIO4_2007_WEP12_InitModel_Tompson` | wep1200.pmd |
| `wep13` | `0x0069FC60` | `0x0029FC60` | `BIO4_2007_WEP13_InitModel_Rocket-family` | wep1300.pmd |
| `wep14` | `0x00698B80` | `0x00298B80` | `BIO4_2007_WEP14_InitModel_Mine` | wep1400.pmd; wep1401.pmd |
| `wep15` | `0x006FF020` | `0x002FF020` | `BIO4_2007_WEP15_InitModel_Magnum` | wep1500.pmd |
| `wep16` | `0x006FF540` | `0x002FF540` | `BIO4_2007_WEP16_InitModel_Knife` | wep1600.pmd |
| `wep17` | `0x006A6960` | `0x002A6960` | `BIO4_2007_WEP17_InitModel_Vp70` | wep1700.pmd |
| `wep19` | `0x006A3410` | `0x002A3410` | `BIO4_2007_WEP19_InitModel_HandGre` | wep1900.pmd |
| `wep26` | `0x00702070` | `0x00302070` | `BIO4_2007_WEP26_InitModel_Knife` | wep2600.pmd |
| `wep27` | `0x00702B50` | `0x00302B50` | `BIO4_2007_WEP27_InitModel_Machinegun` | wep2700.pmd |
| `wep28` | `0x007038D0` | `0x003038D0` | `BIO4_2007_WEP28_InitModel_Bow` | wep2800.pmd |
| `wep29` | `0x00696BF0` | `0x00296BF0` | `BIO4_2007_WEP29_InitModel_Machinegun` | wep2900.pmd |
| `wep33` | `0x007051C0` | `0x003051C0` | `BIO4_2007_WEP33_InitModel_Shotgun` | wep3300.pmd |
| `wep38` | `0x00706FC0` | `0x00306FC0` | `BIO4_2007_WEP38_InitModel_Ruger` | wep3800.pmd |
| `wep39` | `0x00707B40` | `0x00307B40` | `BIO4_2007_WEP39_InitModel_Machinegun` | wep3900.pmd |
| `wep40` | `0x00708470` | `0x00308470` | `BIO4_2007_WEP40_InitModel_HkSniper` | wep4000.pmd; wep4001.pmd |
| `wep48` | `0x0070C260` | `0x0030C260` | `BIO4_2007_WEP48_InitModel_Shotgun` | wep4800.pmd |
| `wep49` | `0x006A4170` | `0x002A4170` | `BIO4_2007_WEP49_InitModel_Tompson` | wep4900.pmd |
| `wep50` | `0x0070D8D0` | `0x0030D8D0` | `BIO4_2007_WEP50_InitModel_AdaBow` | wep5000.pmd |

## Observações de engenharia

- `WepXX_init__FP7cPlayer` nos RELs é a init lógica/player-side da arma no módulo MIPS original.
- `Obj*_init__FP4cObj` e `init__*cObj*PC6cModel` são as inits do objeto visual/modelo da arma.
- Os candidatos x86 encontrados por `wepXXXX.pmd` normalmente correspondem à init visual/model bind, não necessariamente à rotina completa de disparo/recarga.
- Para achar `moveFire`, `moveReload`, `setCartridge`, suba as XREFs do candidato x86 e compare com os nomes do REL.
- `wep02` está confirmado: `wep0200.pmd` é Handgun normal e `wep0201.pmd` é Handgun + Silenciador; os IDs são `23 00`, `3F 00`, `24 00`.

## Arquivos gerados

- `RE4_2007_WEP_INIT_MAP.csv` — mapa principal.
- `RE4_2007_WEP_INIT_REL_SYMBOLS.csv` — todos os símbolos REL de init.
- `RE4_2007_WEP_INIT_X86_CANDIDATES.csv` — todos os candidatos x86 por PMD.
- `RE4_2007_WepInitLabels.java` / `NewScript_WepInitLabels.java` — labels para Ghidra.