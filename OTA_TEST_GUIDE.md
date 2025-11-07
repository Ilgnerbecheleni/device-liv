# Guia de Teste - Sistema OTA ESP32 LIV10-CORTINA

## Resumo da Implementação

O sistema OTA foi implementado com sucesso no firmware ESP32 LIV10-CORTINA-MQTT com as seguintes funcionalidades:

### ✅ Arquivos Implementados

1. **Cabeçalhos (include/)**:
   - `OTAManager.h` - Gerenciador principal OTA
   - `OTAConfig.h` - Configurações e constantes
   - `OTAUtils.h` - Utilitários auxiliares

2. **Implementações (src/)**:
   - `OTAManager.cpp` - Lógica principal do OTA
   - `OTAConfig.cpp` - Configurações do dispositivo
   - `OTAUtils.cpp` - Funções auxiliares

3. **Integração**:
   - `main.cpp` - Inicialização e integração MQTT
   - `platformio.ini` - Dependências adicionadas

### ✅ Funcionalidades Implementadas

- **Download de firmware via HTTPS**
- **Verificação de integridade SHA256**
- **Instalação automática de firmware**
- **Sistema de rollback (estrutura preparada)**
- **Relatório de progresso via MQTT**
- **Validação pós-instalação**
- **Integração completa com sistema MQTT existente**

## Tópicos MQTT Configurados

O sistema utiliza os seguintes tópicos MQTT:

```
/ota/{device_id}/command  - Recebe comandos OTA
/ota/{device_id}/status   - Publica status do OTA
/ota/{device_id}/progress - Publica progresso do download/instalação
```

Onde `{device_id}` é gerado automaticamente como `LIV10_` + MAC Address (sem `:`)

## Comandos OTA Suportados

### 1. Verificar Atualizações
```json
{
  "command": "check"
}
```

### 2. Iniciar Atualização
```json
{
  "command": "update",
  "url": "https://servidor.com/firmware.bin",
  "version": "1.1.0",
  "checksum": "sha256_hash_do_firmware"
}
```

### 3. Cancelar Atualização
```json
{
  "command": "cancel"
}
```

### 4. Solicitar Status
```json
{
  "command": "status"
}
```

### 5. Rollback (preparado para implementação futura)
```json
{
  "command": "rollback"
}
```

## Estados do Sistema OTA

- `OTA_IDLE` (0) - Sistema inativo, pronto para comandos
- `OTA_CHECKING` (1) - Verificando atualizações no servidor
- `OTA_DOWNLOADING` (2) - Baixando firmware
- `OTA_INSTALLING` (3) - Instalando firmware
- `OTA_SUCCESS` (4) - Atualização concluída com sucesso
- `OTA_ERROR` (5) - Erro durante o processo
- `OTA_ROLLBACK` (6) - Executando rollback

## Códigos de Erro

- `OTA_NO_ERROR` (0) - Sem erro
- `OTA_NETWORK_ERROR` (1) - Erro de rede
- `OTA_SERVER_ERROR` (2) - Erro do servidor
- `OTA_DOWNLOAD_ERROR` (3) - Erro no download
- `OTA_CHECKSUM_ERROR` (4) - Erro de verificação
- `OTA_INSTALL_ERROR` (5) - Erro na instalação
- `OTA_TIMEOUT_ERROR` (6) - Timeout
- `OTA_MEMORY_ERROR` (7) - Erro de memória
- `OTA_NOT_SUPPORTED` (8) - Funcionalidade não suportada

## Teste Manual

### Pré-requisitos
1. Firmware compilado e carregado no ESP32
2. Dispositivo conectado ao WiFi
3. Cliente MQTT configurado (ex: MQTT Explorer)
4. Servidor OTA funcionando (opcional para teste completo)

### Passos de Teste

#### 1. Verificar Inicialização
- Conectar ao monitor serial
- Verificar logs de inicialização do OTA Manager
- Confirmar que os tópicos MQTT foram configurados

#### 2. Teste de Conectividade MQTT
- Usar cliente MQTT para se conectar ao broker
- Verificar se o dispositivo está publicando heartbeat no tópico status
- Confirmar recepção de mensagens no tópico command

#### 3. Teste de Comando Status
```bash
# Publicar no tópico command
Tópico: /ota/LIV10_XXXXXXXXXXXX/command
Payload: {"command": "status"}

# Verificar resposta no tópico status
Tópico: /ota/LIV10_XXXXXXXXXXXX/status
```

#### 4. Teste de Comando Check
```bash
# Publicar no tópico command
Tópico: /ota/LIV10_XXXXXXXXXXXX/command
Payload: {"command": "check"}

# Verificar resposta indicando se há atualizações disponíveis
```

#### 5. Teste de Comando Update (simulado)
```bash
# Publicar no tópico command
Tópico: /ota/LIV10_XXXXXXXXXXXX/command
Payload: {
  "command": "update",
  "url": "https://httpbin.org/status/404",
  "version": "1.1.0",
  "checksum": "test"
}

# Verificar que o sistema reporta erro de download
```

### Logs Esperados

Durante a inicialização:
```
[INFO][OTA] OTA Manager initialized for device: LIV10_XXXXXXXXXXXX
[INFO][OTA] OTA Topics configured:
[INFO][OTA]   Command: /ota/LIV10_XXXXXXXXXXXX/command
[INFO][OTA]   Status: /ota/LIV10_XXXXXXXXXXXX/status
[INFO][OTA]   Progress: /ota/LIV10_XXXXXXXXXXXX/progress
```

Durante operação:
```
[INFO][OTA] Received OTA command: {"command":"status"}
[INFO][OTA] OTA Status: idle - Status requested
```

## Configurações Importantes

### Timeouts e Limites
- Timeout de atualização: 5 minutos (300.000ms)
- Tamanho do chunk: 1KB (1024 bytes)
- Tentativas de retry: 3
- Intervalo de heartbeat: 30 segundos
- Intervalo de progresso: 1 segundo

### Uso de Memória
- Buffer de firmware: 70% da heap disponível
- Alocação dinâmica durante download
- Limpeza automática após operações

## Próximos Passos

1. **Implementar servidor OTA completo** para testes end-to-end
2. **Adicionar funcionalidade de rollback** automático
3. **Implementar assinatura digital** para validação de firmware
4. **Adicionar logs persistentes** para auditoria
5. **Otimizar uso de memória** para dispositivos com pouca RAM

## Compatibilidade

✅ **Mantém total compatibilidade** com sistema existente:
- Sistema MQTT original preservado
- Callbacks e tópicos existentes não afetados
- Funcionalidades de cortina e Alexa intactas
- Configurações de rede preservadas

## Status da Implementação

- ✅ Compilação bem-sucedida
- ✅ Integração MQTT completa
- ✅ Estrutura de comandos implementada
- ✅ Sistema de status e progresso
- ✅ Tratamento de erros robusto
- ⏳ Testes em hardware real pendentes
- ⏳ Servidor OTA de produção pendente