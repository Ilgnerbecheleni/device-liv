# device-liv


# LIV10 Gateway API — Mapa Completo de Rotas (build atual)

> Base: `http://148.230.76.43:4000`  
> Status: Auth **desligado** (rotas acessíveis sem token nesta instância).  
> Manifesto raiz: `GET /` retorna os prefixos ativos.

---

## Sumário
- [Health & Root](#health--root)
- [Webhook](#webhook)
- [Alexa (Smart Home Skill)](#alexa-smart-home-skill)
- [API Alexa auxiliar](#api-alexa-auxiliar)
- [Clientes (`/api/clients`)](#clientes-apiclients)
- [Ambientes (`/api/environments`)](#ambientes-apienvironments)
- [Dispositivos LIV10 - Novo Modelo (`/api/devices`)](#dispositivos-liv10---novo-modelo-apidevices)
- [Rotas Legadas (“devices legacy”) (`/api`)](#rotas-legadas-devices-legacy-api)
- [Autenticação (existe, mas não montada)](#autenticação-existe-mas-não-montada)
- [Paginação & Padrões](#paginação--padrões)
- [Observações de Segurança](#observações-de-segurança)

---

## Health & Root

### `GET /`
Retorna manifesto com prefixos disponíveis.
**Exemplo de resposta:**
```json
{
  "name": "LIV10 Gateway API",
  "version": "2.0.0",
  "description": "Gateway para integração entre Alexa e dispositivos LIV10 via MQTT com estrutura hierárquica",
  "endpoints": {
    "alexa": "/alexa",
    "devices_legacy": "/api",
    "clients": "/api/clients",
    "environments": "/api/environments",
    "devices": "/api/devices",
    "health": "/health"
  },
  "documentation": "https://github.com/livintech/liv10-gateway"
}
```

### `GET /health`
**200 OK** com status de serviço.
```json
{
  "status": "ok",
  "timestamp": "2025-11-07T19:45:12.732Z",
  "service": "liv10-gateway",
  "version": "1.0.0"
}
```

---

## Webhook

### `POST /webhook`
Recebe eventos do **EMQX** (ex.: `client.connected`, `client.disconnected`, mensagens, etc.).
- **Body**: JSON do webhook EMQX.
- **Respostas**: `200` (processado/ignorado), `400` (body inválido).

---

## Alexa (Smart Home Skill)

Prefixo: **`/alexa`**

- `POST /alexa` — Entrada principal da Skill (Directives).
- `GET /alexa/health` — Health da integração Alexa.
- `GET /alexa/stats` — Métricas básicas.

**Auth**: não exigido nesta build.  
**Uso**: normalmente chamado pela própria Alexa; para testes locais, prefira a API auxiliar abaixo.

---

## API Alexa auxiliar

Prefixo: **`/api/alexa`**

- `GET /api/alexa/discovery` — Simula/força Discovery (lista endpoints).  
  **Resposta**: lista de endpoints com `capabilities`.
- `POST /api/alexa/power/on` — Liga endpoint.  
  **Body**: `{"endpointId":"..."}`
- `POST /api/alexa/power/off` — Desliga endpoint.  
  **Body**: `{"endpointId":"..."}`
- `POST /api/alexa/curtain/control` — Controla cortina.  
  **Body**: `{"endpointId":"...", "action":"open|stop|close"}`
- `GET /api/alexa/state/:endpointId` — Estado atual de um endpoint.
- `GET /api/alexa/health` — Health da API auxiliar.
- `GET /api/alexa/stats` — Estatísticas.

**Auth**: nesta build não exige; há um _stub_ de API key no código (não aplicado).

---

## Clientes (`/api/clients`)

Prefixo: **`/api/clients`**

- `GET /api/clients` — Lista clientes.  
  **Query**: `page`, `limit`, `sort`, `order`.  
  **Resposta**:
  ```json
  {
    "clients": [ { "...campos do cliente..." } ],
    "pagination": { "page": 1, "limit": 20, "total": 1, "totalPages": 1 }
  }
  ```

- `GET /api/clients/:clientId` — Detalhe do cliente.

- `POST /api/clients` — Cria cliente.  
  **Body (schema):**
  ```json
  {
    "name": "string (2-100)", 
    "email": "email válido",
    "phone": "E.164 (opcional)",
    "alexa_user_id": "string (opcional)",
    "is_active": true
  }
  ```
  **Resposta**: cliente criado (com campos como `id`, `created_at`, `subscription_plan` padrão `basic`, etc.).

- `PUT /api/clients/:clientId` — Atualiza cliente.  
  **Body**: mesmos campos do `POST`, todos opcionais.

- `DELETE /api/clients/:clientId` — **Desativa** cliente (soft-delete).

- `GET /api/clients/:clientId/environments` — Ambientes do cliente.

- `GET /api/clients/:clientId/devices` — Devices do cliente.

- `GET /api/clients/:clientId/stats` — Estatísticas consolidadas do cliente.

- `GET /api/clients/email/:email` — Busca cliente por email.
- `GET /api/clients/alexa/:alexaUserId` — Busca por `alexa_user_id`.

**Observações**:
- Unicidade de **email** validada no `POST` (409/400 se duplicado).
- `is_active` padrão `true`.
- `timezone` padrão `America/Sao_Paulo`, `country` padrão `BR` (no model).

---

## Ambientes (`/api/environments`)

Prefixo: **`/api/environments`**

- `GET /api/environments/client/:clientId` — Lista ambientes do cliente.
- `GET /api/environments/:environmentId` — Detalhe do ambiente.
- `POST /api/environments/client/:clientId` — Cria ambiente.  
  **Body (schema):**
  ```json
  {
    "client_id": 6,
    "name": "string (2-100)",
    "description": "string (opcional)",
    "display_order": 0
  }
  ```

- `PUT /api/environments/:environmentId` — Atualiza ambiente (mesmo schema).

- `DELETE /api/environments/:environmentId` — Remove ambiente.

- `GET /api/environments/:environmentId/devices` — Devices do ambiente.
- `GET /api/environments/:environmentId/channels` — Canais do ambiente.  
  **Query** (quando suportado): `channelType`, `isEnabled`, `isAlexaEnabled`.

- `PUT /api/environments/client/:clientId/order` — Reordena ambientes do cliente.  
  **Body**: lista ordenada/pares `{ environment_id, display_order }`.

- `GET /api/environments/:environmentId/stats` — Estatísticas do ambiente.

---

## Dispositivos LIV10 - Novo Modelo (`/api/devices`)

Prefixo: **`/api/devices`**

- `GET /api/devices` — Lista devices (novo modelo).  
  **Query** comuns: `page`, `limit`, filtros como `client_id`, etc.  
  **Resposta (exemplo real):**
  ```json
  {
    "devices": [{ "id": 55, "device_id": "b0a732b73af4", "...": "..." }],
    "pagination": { "page": 1, "limit": 50, "total": 5, "totalPages": 1 }
  }
  ```

- `GET /api/devices/:deviceId` — Detalhe do device.  
  Aceita `device_id` (12 chars) **ou** `MAC` (17 chars).

- `POST /api/devices/client/:clientId/environment/:environmentId` — **Adiciona** device ao cliente/ambiente.  
  **Body (schema):**
  ```json
  {
    "client_id": 6,
    "environment_id": 10,
    "device_id": "string (3-50)",
    "mac_address": "AA:BB:CC:DD:EE:FF",
    "name": "string (2-100)",
    "description": "string opcional",
    "firmware_version": "x.y.z (opcional)",
    "hardware_version": "string (opcional)",
    "ip_address": "IPv4/IPv6 (opcional)",
    "is_online": false
  }
  ```

- `PUT /api/devices/:deviceId` — Atualiza device.  
  **Body**: campos do device (nome, descrição, `is_enabled`, etc.).

- `GET /api/devices/:deviceId/channels` — Lista canais do device.

- `PUT /api/devices/:deviceId/channels/:channelNumber` — Atualiza canal.  
  **Body (schema):**
  ```json
  {
    "device_id": 55,
    "channel_number": 1,
    "channel_type": "input|output",
    "name": "string (2-100)",
    "description": "string (opcional)",
    "is_alexa_enabled": false,
    "alexa_name": "string",
    "alexa_endpoint_id": "string"
  }
  ```

- `POST /api/devices/:deviceId/channels/:channelNumber/control` — **Controla** canal.  
  **Body** depende do tipo:
  - **switch/relay**: `{"action":"on"|"off"|"toggle"}` ou `{"state":1|0}`
  - **dimmer**: `{"level":0..100}`
  - Validação responde 400/422 informando a chave esperada.

- `GET /api/devices/:deviceId/curtains` — Lista pares de cortina.

- `POST /api/devices/:deviceId/curtains/:pairNumber/control` — Controla cortina.  
  **Body**: `{"action":"open"|"stop"|"close"}`

- `POST /api/devices/:deviceId/status` — Callback/status enviado pelo device (atualiza `last_seen`, `is_online`, etc.).  
  **Body**: payload do device.

- `GET /api/devices/:deviceId/events` — Eventos/telemetria do device.
- `GET /api/devices/:deviceId/commands` — Comandos emitidos para o device.

---

## Rotas Legadas (“devices legacy”) (`/api`)

Prefixo: **`/api`** (router legado com “home/devices/channels”)

- `GET /api/homes` — Lista “residências” (modelo antigo).
- `POST /api/homes` — Cria “residência”.  
  **Body**: `{ "name": "...", "address": "...", "timezone": "America/Sao_Paulo" }`

- `GET /api/homes/:homeId/devices` — Devices da residência.
- `POST /api/homes/:homeId/devices` — Adiciona device a uma residência.

- `GET /api/devices/:deviceId` — Detalhe (aceita 12-char id ou MAC 17-char).
- `PUT /api/devices/:deviceId` — Atualiza device (legado).

- `GET /api/devices/:deviceId/channels` — Lista canais (legado).
- `PUT /api/devices/:deviceId/channels/:channelNumber` — Atualiza canal (legado).
- `POST /api/devices/:deviceId/channels/:channelNumber/control` — Controla canal (legado).

- `GET /api/devices/:deviceId/curtains` — Lista pares.
- `PUT /api/devices/:deviceId/curtains/:pairNumber` — Atualiza par.
- `POST /api/devices/:deviceId/curtains/:pairNumber/control` — Controla par.

- `POST /api/devices/:deviceId/status` — Atualização de status do device.
- `POST /api/devices/:deviceId/reboot` — Solicita reboot.
- `POST /api/devices/:deviceId/config` — Atualiza/configura device.

> **Nota**: Esse bloco existe para **compatibilidade**. Prefira o novo modelo em `/api/devices`.

---

## Autenticação (existe, mas não montada)

Há um router em `src/routes/auth.js` com:
- `POST /api/auth/login` — Autentica (bcrypt + JWT) e cria sessão web (`req.session.user`).  
  **Body**: `{"email":"...", "password":"..."}`  
  **Resposta**: `{"success":true, "token":"<JWT>", "user":{...}}`
- `POST /api/auth/logout` — Destroi sessão.
- `GET /api/auth/me` — Retorna usuário da sessão.
- `POST /api/auth/check` — `{ authenticated: true|false, user: {...}|null }`

Para **exigir token** nas APIs, monte o middleware `authenticateToken` e `router.use('/api/...', authenticateToken, ...)`.

---

## Paginação & Padrões

- A maioria das listas aceita `?page=<n>&limit=<n>` e responde com bloco `pagination`.
- Ordenações comuns: `sort=id|name|created_at|updated_at`, `order=asc|desc`.
- IDs de `deviceId` podem ser **12 chars** (ex.: `b0a732b73af4`) **ou** **MAC** (`AA:BB:...:FF`).

---

## Observações de Segurança

- Nesta instância, **auth está desligado** → **NÃO** exponha publicamente.  
- Para produção, monte `auth` e proteja os sub-routers com `authenticateToken`.
- Para scripts de administração, é possível “mintar” tokens (`jwt.sign`) com `JWT_SECRET` — use apenas em ambiente controlado.

---

### Exemplos úteis (copie e cole)

Listar clientes:
```bash
curl "http://148.230.76.43:4000/api/clients?page=1&limit=50"
```

Detalhar device:
```bash
curl http://148.230.76.43:4000/api/devices/b0a732b73af4
```

Controlar canal (on):
```bash
curl -X POST http://148.230.76.43:4000/api/devices/b0a732b73af4/channels/1/control   -H "Content-Type: application/json"   -d '{"action":"on"}'
```

Criar ambiente:
```bash
curl -X POST http://148.230.76.43:4000/api/environments/client/6   -H "Content-Type: application/json"   -d '{"client_id":6,"name":"Sala","description":"andar térreo","display_order":0}'
```
