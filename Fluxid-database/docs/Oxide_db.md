# Oxide DB

## Banco SQLite de Ingestão Temporária para Dispositivos ESP32

---

# 1. Introdução

O **Oxide DB** é um banco de dados SQLite utilizado como uma camada intermediária entre os dispositivos ESP32 e o banco principal do sistema, o PostgreSQL do FluxID.

Seu objetivo é garantir que nenhuma mensagem enviada pelos dispositivos seja perdida caso ocorram problemas temporários de rede, indisponibilidade do PostgreSQL ou falhas durante a sincronização.

---

# 2. Arquitetura Geral

```text
ESP32
   │
   │ JSON
   ▼
Oxide API
   │
   ▼
SQLite (oxide.db)
   │
   │ Worker de Sincronização
   ▼
PostgreSQL (FluxID)
```

---

# 3. Por que usar SQLite?

Imagine o seguinte cenário:

```text
ESP32 envia uma telemetria
        ↓
PostgreSQL está indisponível
```

Sem uma camada intermediária:

```text
Telemetria perdida
```

Com o SQLite:

```text
ESP32 envia
        ↓
API recebe
        ↓
SQLite armazena
        ↓
PostgreSQL volta
        ↓
Sincronização acontece
```

Resultado:

```text
Nenhuma informação é perdida
```

---

# 4. Responsabilidades do SQLite

O SQLite atua como um **buffer temporário**.

Ele é responsável por:

✅ Receber dados enviados pelos dispositivos

✅ Armazenar mensagens temporariamente

✅ Detectar mensagens duplicadas

✅ Controlar tentativas de sincronização

✅ Registrar erros de sincronização

✅ Enviar os dados para o PostgreSQL posteriormente

---

# 5. O que NÃO fica no SQLite?

O SQLite não contém regras de negócio.

Esses dados continuam sendo responsabilidade do PostgreSQL:

```text
Usuários
Organizações
Perfis
Permissões
Destinatários
Locais de Entrega
Cilindros
Lacres
Dispositivos Oficiais
Alertas
Entregas
Custódias
Movimentações
Auditoria
```

O SQLite existe apenas para suportar a comunicação entre os dispositivos e a API.

---

# 6. Estrutura Geral do Banco

O banco possui quatro tabelas:

```text
devices
telemetries
events
sync_logs
```

---

# 7. Diagrama de Relacionamento

```text
devices
│
├── telemetries
│
└── events

sync_logs
```

---

# 8. Tabela Devices

## Objetivo

Controlar quais dispositivos podem se comunicar com a API.

---

### Estrutura

```sql
CREATE TABLE devices (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    device_id TEXT NOT NULL UNIQUE,

    api_key TEXT NOT NULL,

    firmware_version TEXT,

    active INTEGER NOT NULL DEFAULT 1
);
```

---

### Campos

| Campo | Tipo | Descrição |
|---------|---------|---------|
| id | INTEGER | Identificador interno |
| device_id | TEXT | Código único do dispositivo |
| api_key | TEXT | Chave utilizada para autenticação |
| firmware_version | TEXT | Versão do firmware |
| active | INTEGER | Define se o dispositivo pode enviar dados |

---

### Exemplo

```text
device_id:
DSP-000014
```

```text
firmware_version:
1.0.0
```

---

# 9. Tabela Telemetries

## Objetivo

Receber telemetrias enviadas pelos dispositivos.

Telemetria representa dados operacionais como:

```text
GPS
Velocidade
Bateria
Sinal GSM
```

---

### Estrutura

```sql
CREATE TABLE telemetries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    message_id TEXT NOT NULL UNIQUE,

    device_id TEXT NOT NULL,

    latitude REAL,

    longitude REAL,

    speed_kmh REAL,

    battery_percent REAL,

    gsm_signal INTEGER,

    payload_json TEXT,

    status TEXT DEFAULT 'PENDING',

    attempt_count INTEGER DEFAULT 0,

    last_error TEXT,

    FOREIGN KEY (device_id)
        REFERENCES devices(device_id)
);
```

---

## Chave Estrangeira

```sql
FOREIGN KEY (device_id)
REFERENCES devices(device_id)
```

Relacionamento:

```text
telemetries.device_id
→ devices.device_id
```

---

### Exemplo de Telemetria

```json
{
  "message_id": "MSG-000001",
  "device_id": "DSP-000014",
  "latitude": -7.2091939,
  "longitude": -39.3063666,
  "speed_kmh": 21.98,
  "battery_percent": 57.33,
  "gsm_signal": -64
}
```

---

# 10. Tabela Events

## Objetivo

Registrar eventos importantes gerados pelo dispositivo.

Exemplos:

```text
Violação do lacre
```

```text
Abertura do lacre
```

```text
Fechamento do lacre
```

```text
Bateria crítica
```

```text
Perda de GPS
```

---

### Estrutura

```sql
CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    message_id TEXT NOT NULL UNIQUE,

    device_id TEXT NOT NULL,

    event_type TEXT NOT NULL,

    seal_status TEXT,

    payload_json TEXT,

    status TEXT DEFAULT 'PENDING',

    attempt_count INTEGER DEFAULT 0,

    last_error TEXT,

    FOREIGN KEY (device_id)
        REFERENCES devices(device_id)
);
```

---

## Chave Estrangeira

```sql
FOREIGN KEY (device_id)
REFERENCES devices(device_id)
```

Relacionamento:

```text
events.device_id
→ devices.device_id
```

---

### Eventos Permitidos

```text
VIOLATION
OPEN
CLOSE
BATTERY_LOW
GPS_LOST
GPS_RESTORED
TAMPER_DETECTED
DEVICE_RESTART
```

---

### Exemplo

```json
{
  "message_id": "EVT-000001",
  "device_id": "DSP-000014",
  "event_type": "VIOLATION",
  "seal_status": "OPEN"
}
```

---

# 11. Tabela Sync Logs

## Objetivo

Registrar o histórico das sincronizações realizadas.

Essa tabela é usada apenas para fins técnicos e monitoramento.

---

### Estrutura

```sql
CREATE TABLE sync_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,

    started_at TEXT,

    finished_at TEXT,

    telemetries_sent INTEGER DEFAULT 0,

    events_sent INTEGER DEFAULT 0,

    errors INTEGER DEFAULT 0,

    status TEXT,

    log_message TEXT
);
```

---

### Exemplo

```text
Sincronização iniciada:
14:00
```

```text
Telemetrias enviadas:
200
```

```text
Eventos enviados:
15
```

```text
Erros:
0
```

---

# 12. Estados das Filas

Cada registro possui um estado que identifica em qual etapa está.

### PENDING

Mensagem recebida.

Ainda não enviada ao PostgreSQL.

---

### PROCESSING

Mensagem em processo de sincronização.

---

### SYNCED

Mensagem enviada com sucesso ao PostgreSQL.

---

### ERROR

Falha durante a sincronização.

---

# 13. Controle de Duplicidade

Cada mensagem possui um identificador único:

```sql
message_id
```

A coluna possui a restrição:

```sql
UNIQUE
```

Consequentemente:

```text
Mesmo message_id
=
Mesmo registro
```

---

### Exemplo

Primeira mensagem:

```text
MSG-000001
```

Resultado:

```text
INSERT realizado
```

---

Segunda tentativa:

```text
MSG-000001
```

Resultado:

```text
Ignorar duplicata
```

---

# 14. Controle de Tentativas

Sempre que ocorrer uma falha:

```sql
attempt_count
```

será incrementado.

---

Exemplo:

```text
Tentativa 1
↓
Falha
↓
attempt_count = 1
```

```text
Tentativa 2
↓
Falha
↓
attempt_count = 2
```

---

# 15. Controle de Erros

Caso uma sincronização falhe:

```sql
last_error
```

armazenará o motivo.

---

Exemplos:

```text
Timeout PostgreSQL
```

```text
Conexão recusada
```

```text
JSON inválido
```

---

# 16. Fluxo Completo de Funcionamento

## Passo 1

ESP32 envia uma mensagem.

```text
ESP32
↓
Oxide API
```

---

## Passo 2

A API valida:

```text
API Key
Estrutura do JSON
message_id
```

---

## Passo 3

Mensagem é gravada no SQLite.

```text
status = PENDING
```

---

## Passo 4

O Worker busca mensagens pendentes.

```sql
SELECT *
FROM telemetries
WHERE status = 'PENDING';
```

---

## Passo 5

O Worker envia ao PostgreSQL.

```text
SQLite
↓
FluxID PostgreSQL
```

---

## Passo 6

Após sucesso:

```text
status = SYNCED
```

---

## Passo 7

Em caso de erro:

```text
status = ERROR
attempt_count += 1
last_error atualizado
```

---

# 17. Índices Recomendados

## Telemetrias

```sql
CREATE INDEX idx_telemetries_device
ON telemetries(device_id);
```

```sql
CREATE INDEX idx_telemetries_status
ON telemetries(status);
```

---

## Eventos

```sql
CREATE INDEX idx_events_device
ON events(device_id);
```

```sql
CREATE INDEX idx_events_status
ON events(status);
```

---

# 18. Política de Limpeza

Após sincronização bem-sucedida, os dados podem ser removidos periodicamente.

Prazo definido:

```text
31 dias
```

---

### Telemetrias

```sql
DELETE FROM telemetries
WHERE status = 'SYNCED';
```

---

### Eventos

```sql
DELETE FROM events
WHERE status = 'SYNCED';
```

---

# 19. Benefícios da Arquitetura

✅ Evita perda de dados

✅ Funciona mesmo com falhas temporárias

✅ Permite reprocessamento

✅ Detecta duplicidade

✅ Banco leve e rápido

✅ Simples de manter

✅ Reduz carga sobre o PostgreSQL

✅ Separa a camada IoT da regra de negócio

✅ Facilita o desenvolvimento do MVP

---

# 20. Resumo Executivo

O banco **oxide.db** é um componente técnico de suporte à comunicação entre os dispositivos ESP32 e o FluxID.

Ele não substitui o PostgreSQL.

Sua função é:

```text
Receber
↓
Armazenar temporariamente
↓
Sincronizar
↓
Descartar após período configurado
```

---

## Tabelas

```text
devices
telemetries
events
sync_logs
```

---

## Relacionamentos

```text
telemetries.device_id
→ devices.device_id
```

```text
events.device_id
→ devices.device_id
```

---

## Banco Temporário

```text
SQLite (oxide.db)
```

---

## Banco Definitivo

```text
PostgreSQL (FluxID)
```

---

## Objetivo Final

Garantir que nenhuma informação enviada pelos dispositivos seja perdida antes de chegar ao banco principal do FluxID.
