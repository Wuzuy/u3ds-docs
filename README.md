# u3ds — Documentação do ambiente de servidor (Unturned dedicado + SimpleKits)

Documento operacional consolidado de tudo o que foi aprendido durante o desenvolvimento do plugin SimpleKits.
**Atenção: este documento contém caminhos locais e detalhes da sua máquina — NÃO publicar em repositórios públicos.**

---

## 1. Estrutura do servidor

Servidor dedicado SEPARADO do cliente:

| Caminho | Papel |
|---|---|
| `C:\Program Files (x86)\Steam\steamapps\common\u3ds\` | Instalação dedicada do servidor |
| `C:\Program Files (x86)\Steam\steamapps\common\Unturned\` | Cliente (compartilha só o conteúdo vanilla) |
| `u3ds\Unturned.exe` | Binário do servidor |
| `u3ds\ServerHelper.bat` | Launcher (ver seção 2) |
| `u3ds\Bundles\core.masterbundle` (117 MB) | Conteúdo vanilla (items, modelos, texturas) |
| `u3ds\Bundles\Items\...` | Arquivos `.dat` de cada item (GUID, Type, ID, atributos) |
| `u3ds\Servers\Teste\` | Servidor "Teste" (config, mundo, workshop, logs) |
| `u3ds\Servers\Teste\Config.txt` | Configuração do mundo |
| `u3ds\Modules\Rocket.Unturned\` | Rocket (Rocket.API.dll, Rocket.Core.dll, Rocket.Unturned.dll) |
| `u3ds\Rocket\Plugins\` | Plugins Rocket COMPARTILHADOS |
| `u3ds\Servers\Teste\Rocket\Plugins\` | Plugins Rocket DO SERVIDOR (é o que o Teste carrega) |
| `u3ds\Servers\Teste\Workshop\Content\` | Conteúdo workshop do servidor |
| `u3ds\Logs\Server_Teste.log` | Log do servidor (UTC) |

## 2. Iniciar / parar o servidor

### Iniciar (sempre com o argumento do servidor!)

```bat
ServerHelper.bat +LanServer/Teste
```

O `ServerHelper.bat` é só um intermediário:

```bat
start "" "%~dp0Unturned.exe" -batchmode -nographics %*
```

**IMPORTANTE**: sem `+LanServer/Teste` o servidor boota a configuração "Default" (sem plugins/workshop) e parece travado.
Boot completo leva **3–4 minutos** (carrega ~6115 assets; o Rocket só escreve no log depois do level carregar).

Equivalente em PowerShell (sem o bat):

```powershell
Start-Process -FilePath 'C:\Program Files (x86)\Steam\steamapps\common\u3ds\ServerHelper.bat' -ArgumentList '+LanServer/Teste' -WorkingDirectory 'C:\Program Files (x86)\Steam\steamapps\common\u3ds'
```

### Parar (kill limpo)

```powershell
Get-WmiObject Win32_Process -Filter "Name='Unturned.exe'" |
  Where-Object { $_.CommandLine -match 'batchmode' -and $_.ExecutablePath -like '*u3ds*' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

**Nunca matar o cliente** (`common\Unturned` sem `batchmode` na linha de comando).

### Verificar que subiu

```
Servers\Teste\Rocket\Logs\Rocket.log
[Info] SimpleKits >> SimpleKits loaded! Kits: 3
```

`Server_Teste.log` mostra o progresso de assets ("Read X of 6115...", "Loading level: 100%").

## 3. Logs — fusos horários

| Log | Fuso |
|---|---|
| `Servers\Teste\Rocket\Logs\Rocket.log` | **LOCAL** (rotação por boot: `Rocket.<unix>.log`) |
| `u3ds\Logs\Server_Teste.log` | UTC |
| `common\Unturned\Logs\Client.log` | UTC |

## 4. Rocket + SimpleKits

- O servidor Teste carrega o plugin de `Servers\Teste\Rocket\Plugins\SimpleKits.dll`
  (o Rocket.log mostra o caminho exato em "Rocket dependency registered").
- Config: `Servers\Teste\Rocket\Plugins\SimpleKits\SimpleKits.configuration.xml` (gera na 1ª carga).
- Traduções: `SimpleKits.en.translation.xml` (override do `DefaultTranslations` do código; adicionar chaves novas nos DOIS lugares).
- Estado por jogador: `SimpleKits\PlayerSettings.json` (auto-equip, pegar sem espaço); cooldowns em `SimpleKits\Cooldowns.json`.

**Regra de ouro do deploy**: a DLL carregada fica com **arquivo mapeado** — matar o servidor ANTES de copiar a DLL nova.

## 5. Bundle da UI (efeito 47501) — 4 destinos de deploy

| # | Caminho | Quem usa |
|---|---|---|
| 1 | `u3ds\Servers\Teste\Workshop\Content\3782829202\Effects\KitsUI\` | SERVIDOR (dedicado) |
| 2 | `common\Unturned\Bundles\Workshop\Content\Effects\KitsUI\` | CLIENTE (pasta de conteúdo, sem pasta de ID) |
| 3 | `common\Unturned\workshop\content\304930\3782829202\Effects\KitsUI\` | Cliente via workshop assinado |
| 4 | `opencode\KitsUI-Workshop\Content\Effects\KitsUI\` | Repo local para upload SteamCMD |

Cada pasta contém `KitsUI.unity3d` + `KitsUI.dat` (GUID 49b531cd73b94bde82c54ff0d22d0ebc, Type Effect, ID 47501).

- `Servers\Teste\WorkshopDownloadConfig.json` com `File_IDs` vazio → o servidor usa o conteúdo local (não baixa do Steam).
- **Cliente precisa reabrir o jogo para carregar bundle novo** (sem isso vê a UI antiga).

## 6. Builds

### Plugin (Rocket, .NET Framework 4.8)

```bat
dotnet build "C:\Users\ffxtr\OneDrive\Documentos\opencode\SimpleKits\SimpleKits.csproj" -c Release --nologo -v q
```

- MSBuild não existe no PATH; usar `dotnet` (`C:\Program Files\dotnet\dotnet.exe`).
- O csproj referencia DLLs do u3ds (Assembly-CSharp, Rocket.*, UnityEngine.*, Newtonsoft.Json, SDG.NetTransport).
- Saída: `SimpleKits\bin\Release\SimpleKits.dll`.

### UI (Unity 2022.3.62f3)

```bat
"C:\Program Files\Unity\Hub\Editor\2022.3.62f3\Editor\Unity.exe" -batchmode -quit -projectPath "C:\Users\ffxtr\OneDrive\Documentos\opencode\KitsUI-Unity" -executeMethod KitUiBuilder.BuildAndStage -logFile "%TEMP%\opencode\unity-build.log"
```

- Saída: `KitsUI-Unity\Build\Staging\Effects\KitsUI\KitsUI.unity3d` (+ KitsUI.dat gerado).
- Erros de licença no log ("No ULF license found") são NORMais no batchmode.
- No editor: menu `KitsUI > Build Effect Asset (server + workshop)`.

### Módulo do cliente (ícones/ESC/crosshair/clique direito)

```bat
dotnet build "C:\Users\ffxtr\OneDrive\Documentos\opencode\UnturnedItemIcons\src\UnturnedItemIcons\UnturnedItemIcons.csproj" -c Release --nologo -v q
```

- Instalar em `common\Unturned\Modules\UnturnedItemIcons\UnturnedItemIcons.dll` (**cliente fechado**).
- Avisos CS0618 (Dedicator.isDedicated obsoleto etc.) são normais.

## 7. Publicação na Steam Workshop

- Item: **Simple Kits UI** — `publishedfileid 3782829202`, appid 304930.
- Vdf: `opencode\KitsUI-Workshop\kitsui.vdf` (workshopitem: contentfolder/previewfile/visibility/title/description/changenote/tags).
- SteamCMD: `opencode\steamcmd\steamcmd.exe` (já baixado/atualizado).
- Upload: `steamcmd +login <usuario> <senha> +workshop_build_item "…\kitsui.vdf" +quit`
  - Conta: **smurfank** (o login da Steam é diferente do nome de exibição/GitHub).
  - Steam Guard **mobile** exige confirmação no celular a cada login novo (~2 min de janela).
- O conteúdo enviado é a pasta `KitsUI-Workshop\Content` (estrutura `Effects/KitsUI/...`).
- **Limitação conhecida**: steamcmd NÃO muda a visibilidade de item existente de forma confiável; a página pública do item ficou em erro (API `GetPublishedFileDetails` → result 9 = não encontrado/privado). Para publicar: mudar a visibilidade logado em `https://steamcommunity.com/sharedfiles/edit/?id=3782829202` ou recriar o item.
- (Senha da conta Steam não fica neste documento.)

## 8. Runbook — deploy completo de uma versão

1. Build do bundle (Unity batchmode) → copiar para os 4 destinos (seção 5).
2. Build do plugin (`dotnet build`) → kill do servidor → copiar DLL para:
   - `u3ds\Rocket\Plugins\SimpleKits.dll`
   - `u3ds\Servers\Teste\Rocket\Plugins\SimpleKits.dll`
   - (se mudou traduções) `Servers\Teste\Rocket\Plugins\SimpleKits\SimpleKits.en.translation.xml`
3. Build do módulo → copiar para `common\Unturned\Modules\UnturnedItemIcons\` (cliente fechado).
4. `Start-Process ServerHelper.bat +LanServer/Teste`.
5. Aguardar 3–4 min → conferir "SimpleKits loaded!" no Rocket.log.
6. Reenviar o bundle para a Workshop (SteamCMD + confirmação no celular) se mudou a UI.

## 9. Troubleshooting

| Sintoma | Causa provável / solução |
|---|---|
| Servidor "trava" no boot | Faltou `+LanServer/Teste` no ServerHelper.bat |
| DLL não copia ("user-mapped section") | Servidor rodando — matar antes |
| Cliente vê UI antiga | Cliente não reaberto (bundle em cache da sessão) |
| Ícones quebrados/retângulo cinza | Repo de ícones desatualizado; config `ItemIconUrlTemplate` → Akulation/vanilla-icons |
| ESC não fecha a UI | Módulo não instalado ou cliente não reiniciado; conferir Client.log por "[UnturnedItemIcons] ESC detectado" |
| Crosshair aparece na UI | Módulo antigo (fix: SetDirectionalArrowsVisible(false) no KitUIHook) |
| Tradução nova não aparece | Faltou adicionar a chave no XML do servidor (ou só no código) |
| Tradução não muda | XML do servidor sobrescreve DefaultTranslations |
| Arma resgatada sem acessórios | Item entrou pelo campo de texto — state só é salvo via baú virtual |

## 10. Conhecimento técnico do jogo (pesquisas que deram certo)

### Decompiles (referência)

- Ferramenta: `C:\Users\ffxtr\.dotnet\tools\ilspycmd.exe`
- Saídas em `%TEMP%\opencode\`: PlayerUI.cs, Player.cs, PlayerLifeUI.cs, EffectManager.cs, ItemTool.cs, Item.cs, ItemAsset.cs, ItemGunAsset.cs, ItemMagazineAsset.cs, PlayerInventory.cs, Items.cs, InputEx.cs, Crosshair.cs, GraphicsSettings.cs, IconUtils.cs, GlazieruGUITooltip.cs, types.txt, etc.

### Widget flags (`EPluginWidgetFlags`)

`None=0, Modal=1, NoBlur=2, ForceBlur=4, ShowInteractWithEnemy=8, ShowDeathMenu=0x10, ShowHealth=0x20, ShowFood=0x40, ShowWater=0x80, ShowVirus=0x100, ShowStamina=0x200, ShowOxygen=0x400, ShowStatusIcons=0x800, ShowUseableGunStatus=0x1000, ShowVehicleStatus=0x2000, ShowCenterDot=0x4000, ShowReputationChangeNotification=0x8000, ShowLifeMeters=0x7E0, Default=0xFFF8`

- `Player.setPluginWidgetFlag` / `enablePluginWidgetFlag` / `disablePluginWidgetFlag` — aplicam no cliente em tempo real.
- `ShowLifeMeters` = vida/comida/água/vírus/resistência/oxigênio (client reage via `OnLocalPluginWidgetFlagsChanged`).
- `Modal` = cursor visível; `Player.inPluginModal` = `isPluginWidgetFlagActive(Modal)`.

### Cliques de botão na UI de efeito

- O jogo anexa `PluginButtonListener` a TODO `Button` do efeito → envia o **nome do botão** ao servidor (`EffectManager.onEffectButtonClicked`).
- Clique **direito NÃO** dispara o `onClick` do uGUI → para capturá-lo o módulo anexa `IPointerClickHandler` em runtime.
- `EffectManager.sendEffectClicked("nome")` — público; usado pelo módulo para ESC e clique direito.

### ESC

- O servidor nunca recebe ESC. O jogo consome ESC via `InputEx.ConsumeKeyDown` (gate `keyGuard` por frame).
- Solução: módulo no cliente detecta ESC (Input.GetKeyDown + fallback `OnGUI`/`Event.current` com dedupe por frame) → `sendEffectClicked("BExit")` → plugin fecha na ordem: baú real → configurações → preview → baú virtual → editor → UI.

### Crosshair

- `PlayerLifeUI.crosshair` (Crosshair : SleekWrapper, assembly SDG.Glazier.Runtime):
  - Ponto central = `gameWantsCenterDotVisible && pluginAllowsCenterDotVisible` (flag `ShowCenterDot`).
  - Cruz da arma = `SetDirectionalArrowsVisible(bool)` (4 setas).
  - `IsVisible` NÃO existe no tipo — usar os setters públicos acima.

### State de armas (18 bytes)

`[mira ushort][tático ushort][grip ushort][cano ushort][pente ushort][munição byte][firemode byte][qualidades 6x]`
(offsets 0,2,4,6,8 = acessórios; byte 10 = munição; `ItemGunAsset.ammoMax/ammoMin` públicos; `Attachments.parseFromItemState`).

### Munição/pentes

- `EItemType.MAGAZINE`: a munição é o **amount** da pilha (state vazio); pentes/flechas entram como pilha (Type Magazine no .dat).
- `player.inventory.tryAddItem(item, auto)` — **auto=true equipa arma na mão** (causa do bug de autoequip).
- `forceAddItem`/`forceAddItemAuto` dropa no chão se não couber (`ItemManager.dropItem`).
- Inventário: páginas `PlayerInventory.SLOTS (2)..PAGES-2 (7)`; `Items.tryAddItem` limita 200 por página e usa `tryFindSpace(size_x, size_y)`.

### Ícones de itens

- `ItemTool.getIcon(id, skin, quality, state, asset, skinAsset, tags, dynamicProps, w, h, scale, readableOnCPU, cb)` — renderiza o modelo 1 frame após instanciar ("Icon" transform); fila de ícones; cache por asset.
- Pipeline oficial do jogo: `IconUtils.captureAllItemIcons` (Shift+F1 no menu Workshop → `Extras/Icons/{nome}_{id}.png`, resolução 512×size).
- Ícones **flat/silhueta** são normais para acessórios/bandage (comparação com Akulation/vanilla-icons confirma).
- Fontes: `Akulation/vanilla-icons` (`@main/icons/{id}.png`) — oficial e completo.

### Workshop (Steam)

- `GetPublishedFileDetails` (POST, sem auth) → `result 9` = arquivo não encontrado/privado para o anônimo.
- SteamCMD `workshop_build_item` com vdf `"workshopitem"` — fields: appid, publishedfileid, contentfolder, previewfile, visibility (0 privado, 2 público), title, description, changenote, tags.
- Para item existente, mudança de visibilidade pode exigir a página de edição do Steam (não o steamcmd).

### Diversos

- Fonte da UI: `LegacyRuntime.ttf` (não tem glifo "⚙" garantido → sprite de engrenagem gerado proceduralmente no builder).
- Elementos críticos do efeito: botões viram cliques pelo NOME; textos vivem em filhos "Label" RENOMEADOS (padrão `XxxLabel`); `SetActive(false)` no prefab persiste no bundle (painéis ocultos por padrão).
- `GlazieruGUITooltip` existe (hover), mas é `internal` → inacessível para o bundle; tooltips do plugin são por clique (servidor).
- Teste do servidor: SteamID admin `76561198930801026`; sem BattlEye para módulos de cliente.
