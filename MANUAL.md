# pickle_waypoints — Manual

Exibe o waypoint do mapa como um indicador na tela com distância e um marker 3D no mundo, e permite que admins enviem um waypoint para um jogador específico ou para todos.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Permissões](#permissões)
4. [Configuração](#configuração)
5. [Comandos](#comandos)
6. [Waypoint de admin](#waypoint-de-admin)
7. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
8. [Localização](#localização)
9. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `ox_lib` | Sim | `inputDialog`, `lib.math.torgba` |
| `qb-core` | Não | Bridge automática. Habilita permissão por job e por grupo |
| `es_extended` | Não | Bridge automática |

A bridge é escolhida em runtime: se `es_extended` ou `qb-core` estiverem iniciados, o arquivo correspondente em `bridge/` é usado; caso contrário cai no `bridge/custom`, que só valida permissões por ACE.

---

## Instalação

1. Copie a pasta `pickle_waypoints` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure ox_lib
   ensure pickle_waypoints
   ```
3. Não há SQL nem itens.

As preferências do jogador são salvas localmente no client via KVP (`pickle_waypoints:settings`), não no banco.

---

## Permissões

O comando `/adminwaypoint` e o evento `pickle_waypoints:sendAdminWaypoint` são validados por `Config.AdminWaypointPermissions`. Basta atender a **um** dos três critérios.

Para liberar por ACE:

```
add_ace group.admin pickle_waypoints.set allow
```

Com a bridge do `qb-core` ou do `es_extended`, também valem o job (com nível mínimo) e o grupo definidos no config. Com a bridge `custom`, apenas o ACE é verificado.

---

## Configuração

Arquivo: `config.lua`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.Language` | string | Sim | Idioma ativo. Deve existir em `locales/translations/` |
| `Config.UseImperial` | bool | Sim | `true` exibe a distância em pés/milhas; `false`, em metros/quilômetros |
| `Config.RenderDistance` | number | Sim | Distância máxima, em metros, para desenhar o marker 3D no mundo |
| `Config.EnterDistance` | number | Sim | Distância que remove automaticamente o waypoint, se ele foi criado com `clearEnter` |
| `Config.WaypointSettings.icon` | string / `nil` | Não | Imagem do indicador na tela. `nil` usa o ícone embutido. Usar imagem alternativa impede a aplicação da cor |
| `Config.WaypointSettings.enabled` | bool | Sim | Padrão do indicador do waypoint pessoal. O jogador pode desligar em `/waypointsettings` |
| `Config.WaypointSettings.color` | `{r,g,b,a}` | Sim | Cor padrão do indicador e do marker |
| `Config.AdminWaypointPermissions.jobs` | tabela | Sim | `["nome_do_job"] = nivel_minimo` |
| `Config.AdminWaypointPermissions.groups` | array | Sim | Grupos do framework |
| `Config.AdminWaypointPermissions.ace` | array | Sim | Permissões ACE |

---

## Comandos

| Comando | Permissão | Descrição |
|---|---|---|
| `/waypointsettings` | Todos | Abre o diálogo de preferências: liga/desliga o indicador e escolhe a cor. Salvo em KVP no client |
| `/clearwaypoints` | Todos | Remove todos os waypoints ativos na tela do jogador, inclusive os recebidos de um admin |
| `/adminwaypoint` | `AdminWaypointPermissions` | Abre o diálogo para enviar um waypoint a um jogador ou a todos |

---

## Waypoint de admin

O fluxo do `/adminwaypoint` exige que o admin **já tenha um waypoint marcado no mapa** — as coordenadas vêm dele. Sem waypoint marcado, o comando apenas notifica e sai.

O diálogo pede:

| Campo | Descrição |
|---|---|
| Rótulo | Texto exibido no blip criado no mapa do destinatário |
| Alvo | ID do jogador (server id) ou `all` para todos |
| Blip | ID do sprite do blip |
| Cor do blip | ID da cor do blip |
| Ícone | Imagem do indicador. `default` usa o ícone embutido |
| Cor | Cor RGBA do indicador e do marker |
| Limpar ao chegar | Se marcado, o waypoint some quando o jogador chega a `Config.EnterDistance` |

Ao confirmar, o waypoint do admin no mapa é desmarcado e o evento é enviado ao servidor, que revalida a permissão antes de repassar aos clients.

---

## Entrypoints para outros recursos

### Exports do client

```lua
-- Cria um waypoint e retorna seu índice.
-- settings aceita: icon, color = {r,g,b,a}, clearEnter (bool), blipId, blipColor
local index = exports['pickle_waypoints']:AddWaypoint('Entrega', coords, {
    color = {0, 230, 153, 255},
    blipId = 38,
    clearEnter = true
})

-- Remove o waypoint (e o blip, se houver).
exports['pickle_waypoints']:RemoveWaypoint(index)

-- Retorna a tabela do waypoint: { label, coords, icon, color, clearEnter, blip }
local wp = exports['pickle_waypoints']:GetWaypoint(index)

-- Move o waypoint sem recriá-lo.
exports['pickle_waypoints']:UpdateWaypointCoords(index, coords)

-- Troca ícone e cor de um waypoint existente.
exports['pickle_waypoints']:UpdateWaypointSettings(index, { color = {255, 0, 0, 200} })
```

### Eventos

```lua
-- Servidor -> client: cria um waypoint no destinatário.
TriggerClientEvent('pickle_waypoints:addWaypoint', target, label, coords, settings)

-- Client -> servidor: envia um waypoint de admin. Revalida a permissão no servidor.
TriggerServerEvent('pickle_waypoints:sendAdminWaypoint', coords, label, target, blipId, blipColor, icon, color, clearEnter)

-- Servidor -> client: abre o diálogo de waypoint de admin.
TriggerClientEvent('pickle_waypoints:adminWaypointMenu', source)
```

O waypoint pessoal (o que o jogador marca no mapa) é gerenciado por um loop interno e não deve ser criado via `AddWaypoint` — ele é recriado sozinho a cada segundo enquanto houver um blip de waypoint ativo.

---

## Localização

As strings ficam em `locales/translations/`, uma tabela `Language["<codigo>"]` por arquivo. Idiomas incluídos:

- `en.lua` — inglês
- `pt.lua` — português

O idioma ativo é escolhido pelo `Config.Language` (não pela convar `ox:locale`). Para adicionar um idioma, crie `locales/translations/<codigo>.lua` seguindo a estrutura dos existentes e aponte o `Config.Language` para ele.

---

## Estrutura de arquivos

```
pickle_waypoints/
├── config.lua                      — todas as opções
├── core/
│   ├── client.lua                  — helper CreateBlip
│   └── shared.lua                  — utilitários (lerp, round, v3)
├── modules/
│   ├── main/
│   │   ├── client.lua              — render do indicador e do marker, comando /clearwaypoints, exports
│   │   └── server.lua              — comando /adminwaypoint e validação de permissão
│   └── waypoint/
│       └── client.lua              — waypoint pessoal, KVP de preferências, /waypointsettings
├── bridge/
│   ├── qb/                         — permissões por job/grupo do qb-core
│   ├── esx/                        — permissões por job/grupo do es_extended
│   └── custom/                     — fallback, só ACE
├── locales/
│   ├── locale.lua                  — função _L
│   └── translations/
│       ├── en.lua
│       └── pt.lua
├── nui/
│   ├── index.html                  — indicador na tela
│   └── assets/                     — CSS, fonte, ícone do waypoint
└── fxmanifest.lua
```
