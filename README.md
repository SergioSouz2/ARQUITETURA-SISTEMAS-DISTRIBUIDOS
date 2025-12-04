# Arquitetura de Sistemas Distribuídos - P2P com Balanceamento de Carga Dinâmico

## Descrição do Projeto

Sistema distribuído peer-to-peer (P2P) com balanceamento de carga dinâmico. A arquitetura permite múltiplos servidores (Masters) gerenciarem workers, com capacidade de compartilhamento dinâmico de recursos entre nós e redirecionamento automático de tarefas baseado em carga.

## Estrutura da Arquitetura

### Componentes Principais

1. **Master (Servidor)** - `server/master.py`
   - Gerencia workers locais e remotos
   - Coordena heartbeats com peers
   - Implementa balanceamento de carga dinâmico
   - Monitora fila de tarefas

2. **Worker** - `server/worker.py`
   - Executa tarefas designadas pelo Master
   - Reporta status regularmente
   - Suporta redirecionamento entre Masters
   - Solicita novas tarefas quando ocioso

3. **Configuração** - `server/config.json`
   - Definição de IPs, portas e peers
   - Parâmetros de timing (heartbeat, load balancer)
   - Thresholds de balanceamento de carga

## Protocolo de Comunicação

### SERVIDOR ↔ WORKER

| ID | Comunicação | Payload | Descrição |
|---|---|---|---|
| 1 | Worker → Servidor | `{"WORKER": "ALIVE"}` | Apresentação e solicitação de tarefa |
| 2 | Servidor → Worker | `{"TASK": "QUERY", "USER": "..."}` | Envio de tarefa de consulta |
| 3 | Worker → Servidor | `{"STATUS": "OK", "SALDO": 99.99, ...}` | Retorno de resultado com sucesso |
| 4 | Worker → Servidor | `{"STATUS": "NOK", "TASK": "QUERY", "ERROR": "..."}` | Erro na execução da tarefa |
| 5 | Servidor → Worker | `{"TASK": "REDIRECT", "TARGET_MASTER": {...}, "HOME_MASTER": {...}}` | Redirecionamento para servidor temporário |

### SERVIDOR ↔ SERVIDOR

| ID | Comunicação | Payload | Descrição |
|---|---|---|---|
| 1 | Server A → Server B | `{"SERVER": "ALIVE", "TASK": "REQUEST"}` | Heartbeat (sinal de vida) |
| 2 | Server B → Server A | `{"SERVER": "ALIVE", "TASK": "RECEIVE"}` | Resposta ao heartbeat |
| 3 | Server A → Server B | `{"TASK": "WORKER_REQUEST", "WORKERS_NEEDED": 5}` | Solicitação de workers emprestados |
| 4.1 | Server B → Server A | `{"TASK": "WORKER_RESPONSE", "STATUS": "ACK", "WORKERS": [...]}` | Resposta positiva com workers |
| 4.2 | Server B → Server A | `{"TASK": "WORKER_RESPONSE", "STATUS": "NACK", "WORKERS": []}` | Resposta negativa (sem workers) |
| 4.3 | Worker → Server A | `{"WORKER": "ALIVE", "WORKER_UUID": "..."}` | Worker emprestado conecta ao servidor saturado |

## Configuração

### Arquivo: `server/config.json`

```json
{
  "server": {
    "ip": "127.0.0.1",
    "port": 5000,
    "id_number": 1
  },
  "peers": [
    { "id": "SERVER_1", "ip": "10.62.217.199", "port": 5000 },
    { "id": "SERVER_3", "ip": "10.62.217.16",  "port": 5000 },
    { "id": "SERVER_4", "ip": "10.62.217.209", "port": 5000 },
    { "id": "SERVER_5", "ip": "10.62.217.203", "port": 5000 }
  ],
  "timing": {
    "heartbeat_interval": 2,
    "heartbeat_timeout": 10,
    "heartbeat_retries": 2,
    "heartbeat_retry_delay": 1,
    "load_balancer_interval": 2
  },
  "load_balancing": {
    "threshold_min_tasks": 10,
    "threshold_return_tasks": 15,
    "threshold_window": 30,
    "idle_worker_threshold": 10,
    "min_workers_before_sharing": 1
  }
}
```

### Parâmetros Principais

| Parâmetro | Descrição | Padrão |
|---|---|---|
| **heartbeat_interval** | Intervalo entre heartbeats (segundos) | 2 |
| **heartbeat_timeout** | Tempo limite para resposta de heartbeat | 10 |
| **load_balancer_interval** | Intervalo de verificação de carga | 2 |
| **threshold_min_tasks** | Tamanho mínimo de fila para solicitar workers | 10 |
| **threshold_return_tasks** | Tamanho máximo de fila para retornar workers | 15 |
| **min_workers_before_sharing** | Workers mínimos mantidos localmente | 1 |

## Estrutura de Diretórios

```
ARQUITETURA-SISTEMAS-DISTRIBUIDOS/
├── README.md                 # Este arquivo
├── server/
│   ├── config.json          # Configuração do sistema
│   ├── master.py            # Implementação do Master (Servidor)
│   ├── worker.py            # Implementação do Worker
│   └── logs/                # Diretório de logs
├── Diagrama Sprint 1.png    # Documentação visual
├── Diagrama Sprint 2.png
├── Diagrama Sprint 3.png
├── Diagrama Sprint 4.png
└── Diagrama Sprint 5.png
```

## Como Executar

### Pré-requisitos
- Python 3.7+
- Configuração de rede entre os peers definida em `server/config.json`

### Iniciar Master
```bash
cd server
python master.py
```

### Iniciar Worker
```bash
cd server
python worker.py
```

Os workers se conectarão automaticamente ao Master configurado no `config.json`.

## Balanceamento de Carga Dinâmico

O sistema monitora continuamente:

- **Tamanho da fila local** de tarefas
- **Disponibilidade de workers** em cada servidor
- **Status de peers** conectados na rede

### Algoritmo de Balanceamento

1. **Monitoramento**: A cada `load_balancer_interval` segundos, cada Master verifica sua fila
2. **Detecção de Sobrecarga**: Se a fila > `threshold_min_tasks`, solicita workers a peers
3. **Solicitação Dinâmica**: Envia `WORKER_REQUEST` aos peers com capacidade ociosa
4. **Redirecionamento**: Workers são redirecionados para o Master sobrecarregado
5. **Retorno**: Quando a fila cai abaixo de `threshold_return_tasks`, workers retornam ao servidor original

## Logs

Logs detalhados são salvos em `server/logs/`:

- **master.log** - Atividades do Master, heartbeats, balanceamento de carga
- **worker.log** - Atividades dos Workers, execução de tarefas, redirecionamentos

Formato dos logs:
```
HH:MM:SS [NIVEL] mensagem
```

## Fluxo de Operação

```
┌─────────────────────────────────────┐
│  1. Heartbeat Contínuo              │
│     Masters trocam sinais de vida   │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  2. Monitoramento de Carga          │
│     Verifica tamanho da fila        │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  3. Solicitação Dinâmica            │
│     Pede workers a peers se saturado│
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  4. Redirecionamento                │
│     Workers mudam de servidor       │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│  5. Execução e Retorno              │
│     Workers executam tarefas        │
│     Retornam resultados ao Master   │
└─────────────────────────────────────┘
```

## Recursos de Resiliência

- **Heartbeat com Retries**: Servidores tentam se reconectar em caso de falha
- **Timeout de Heartbeat**: Detecta servidores offline
- **Redirecionamento com Fallback**: Workers possuem lista de Masters alternativos
- **Logging Detalhado**: Rastreamento de todas operações

## Métricas Monitoradas

O sistema coleta:
- Número de workers ativos
- Tamanho da fila de tarefas
- Taxa de execução de tarefas
- Latência de comunicação entre peers
- Status de heartbeats

## Sprints

Documentação visual dos sprints está disponível nos diagramas inclusos no repositório:
- **Sprint 1-5**: Evolução da arquitetura e implementação de features
