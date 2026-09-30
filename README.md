# FluxID Database

Banco de dados oficial da plataforma **FluxID**.

Responsável por armazenar os dados operacionais relacionados à:

- Segurança
- Rastreabilidade
- Controle Operacional

de cilindros e lacres inteligentes.

---

# Objetivo

Garantir a gestão completa do ciclo de vida dos ativos monitorados através de:

- Controle de cilindros
- Controle de lacres inteligentes
- Telemetria GPS
- Eventos de segurança
- Alertas
- Entregas
- Custódias
- Auditoria
- Conformidade operacional

---

# Arquitetura

```text
ESP32
   │
   ▼
Oxide API
   │
   ▼
Oxide DB (SQLite)
   │
   ▼
FluxID Database (PostgreSQL)
```

O banco PostgreSQL é a fonte oficial dos dados do sistema.

---

# Tecnologias

- PostgreSQL
- SQL
- PostGIS *(planejado para evolução futura)*
- UUID
- JSONB

---

# Estrutura

```text
schema/
seed/
migrations/
views/
triggers/
docs/
```

### Schema

Contém a definição estrutural das tabelas.

### Seed

Massa de dados para desenvolvimento e testes.

### Migrations

Controle de evolução do banco de dados.

### Views

Consultas otimizadas para dashboards e relatórios.

### Triggers

Automação de regras operacionais e auditoria.

### Docs

Documentação técnica e funcional.

---

# Principais Módulos

## Identidade e Acesso

```text
organizacoes
organizacao_contatos

usuarios

perfis
permissoes

perfil_permissoes
usuario_perfis
```

---

## Clientes e Locais

```text
destinatarios
locais_entrega
```

---

## Ativos

```text
cilindros
lacres
dispositivos
```

---

## Histórico de Vínculos

```text
vinculos_cilindro_lacre

vinculos_dispositivo_lacre
```

---

## Logística

```text
movimentacoes
movimentacao_itens

entregas
entrega_itens

custodias
```

---

## IoT e Segurança

```text
telemetrias

eventos_lacre

alertas
```

---

## Conformidade

```text
testes_hidrostaticos

inspecoes_lacre
```

---

## Governança

```text
auditoria
```

---

# Estado Atual do Projeto

## Banco de Dados

- [x] Modelagem concluída
- [x] Schema criado
- [x] Massa de testes carregada
- [x] Relacionamentos definidos
- [x] Auditoria modelada

## Dados de Teste

| Tabela | Registros |
|---------|---------:|
| organizacoes | 3 |
| usuarios | 3 |
| destinatarios | 20 |
| locais_entrega | 20 |
| cilindros | 50 |
| lacres | 50 |
| dispositivos | 50 |
| telemetrias | 200 |
| eventos_lacre | 10 |
| alertas | 10 |
| testes_hidrostaticos | 50 |
| inspecoes_lacre | 50 |
| auditoria | 3 |

---

# Próximas Etapas

## API

- NestJS
- TypeScript
- JWT
- Swagger
- Prisma (em avaliação)

---

## Banco

- Views operacionais
- Views de dashboard
- Triggers de auditoria
- Índices avançados
- Evolução para PostGIS

---

# Documentação

A documentação técnica completa encontra-se na pasta:

```text
docs/
```

---

# Licença

Projeto interno do ecossistema FluxID.
