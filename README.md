# Zabbix Security Monitoring Lab
![YAML](https://img.shields.io/badge/yaml-%23ffffff.svg?style=for-the-badge&logo=yaml&logoColor=151515)

Ambiente de monitoramento de infraestrutura e observação de segurança usando **Zabbix**, rodando localmente via **Docker**, com foco em detectar sinais de segurança em uma máquina Windows (tentativas de login falho, novos serviços instalados, status do Windows Defender) e notificar automaticamente por e-mail.

## Objetivo

Ir além do monitoramento básico de infraestrutura (CPU, memória, disco) e construir uma camada de **observação de segurança**: capturar eventos relevantes do Windows Event Log, transformá-los em triggers acionáveis, e alertar em tempo real quando algo sai do padrão esperado.

## Stack

- **Zabbix 7.0** (Server + Web + Agent) via Docker
- **MySQL 8.0** como banco de dados
- **Zabbix Agent** (active checks) rodando no host Windows monitorado
- **Brevo** como provedor SMTP para notificações por e-mail

## O que é monitorado

**Infraestrutura (baseline):**
- CPU utilization / CPU usage
- Memory utilization
- System uptime
- Disk utilization e filas

**Segurança (itens customizados):**
| Item | Fonte | O que detecta |
|---|---|---|
| Login falho | `eventlog[Security,,,,4625]` | Tentativas de acesso indevido / possível brute-force |
| Novo serviço instalado | `eventlog[System,,,,7045]` | Indicador clássico de persistência / instalação suspeita |
| Status Windows Defender | `system.run[]` + PowerShell (`Get-MpComputerStatus`) | Proteção em tempo real desativada |

## Triggers configuradas

- **Possível tentativa de brute-force**: 3+ eventos de login falho em uma janela de 5 minutos → severidade **High**
- **Novo serviço instalado**: qualquer evento 7045 → severidade **Warning**
- **Windows Defender desativado**: proteção em tempo real reportando `False` → severidade **Disaster**

Todas as triggers disparam uma notificação por e-mail automaticamente via ação configurada no Zabbix.

## Como reproduzir

### 1. Suba o ambiente

```bash
git clone https://github.com/joaovpzdev/<nome-do-repo>.git
cd <nome-do-repo>
cp .env.example .env
# edite o .env com suas próprias senhas
docker compose up -d
```

### 2. Acesse o painel

- URL: `http://localhost:8080`
- Login padrão: `Admin` / `zabbix` (troque a senha assim que entrar)

### 3. Instale o Zabbix Agent no host Windows

Baixe em [zabbix.com/download_agents](https://www.zabbix.com/download_agents), usando a mesma major version do server.

No `zabbix_agentd.conf`, configure:
```
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=<nome_do_host>
AllowKey=system.run[*]
```

> **Atenção:** o `Hostname` aqui precisa bater **exatamente** (case-sensitive) com o nome cadastrado no painel web.

### 4. Cadastre o host no painel

- **Data collection > Hosts > Create host**
- Interface tipo **Agent**, com **Connect to: DNS**, apontando para `host.docker.internal` (necessário porque o server roda em container Docker e `127.0.0.1` de dentro do container não enxerga o host Windows)
- Vincule o template **Windows by Zabbix agent**

### 5. Crie os itens de segurança

Veja a tabela acima — cada item é criado manualmente em **Data collection > Hosts > [host] > Items > Create item**, tipo **Zabbix agent (active)**.

### 6. Configure as triggers e a notificação por e-mail

Crie as 3 triggers de segurança e configure um Media Type de e-mail (Brevo SMTP ou outro provedor) em **Alerts > Media types**, depois vincule ao seu usuário e crie uma Trigger Action em **Alerts > Actions**.

## Desafios encontrados (e como foram resolvidos)

Esse projeto teve bastante troubleshooting real, que documento aqui porque foi a parte mais valiosa do aprendizado:

- **`Row size too large (>8126)` ao criar o banco** → causado pelo charset `utf8`/`utf8_bin` desatualizado. Resolvido trocando para `utf8mb4`/`utf8mb4_bin` no comando do MySQL.
- **`ERROR 1419: You do not have the SUPER privilege...`** → o binary log do MySQL exige `log_bin_trust_function_creators=1` para a criação de triggers do schema do Zabbix.
- **`Zabbix agent is not available`** → o Zabbix Server roda dentro de um container Docker, então `127.0.0.1` aponta para o próprio container, não para o host Windows. Resolvido usando `host.docker.internal` como DNS name na interface do host.
- **Host "not found" nos logs do agent** (`no active checks on server... host not found`) → o `Hostname` no `zabbix_agentd.conf` estava em maiúsculas (`CHAKAL`) enquanto o host cadastrado no painel estava em minúsculas (`chakal`). O Zabbix é case-sensitive nesse campo.
- **Ruído de falsos positivos em "Problems"** (serviços como Google Updater, Intel etc. marcados como "not running") → ajustada a macro `{$SERVICE.NAME.NOT_MATCHES}` no host para excluir serviços não-críticos da descoberta automática.
- **"Login denied" no teste do SMTP (Brevo)** → causado por Connection Security configurado como "None" em vez de STARTTLS, e por uma chave SMTP que precisou ser regenerada.

## Possíveis evoluções futuras

- Habilitar TLS/PSK na comunicação agent ↔ server
- Adicionar mais hosts (Linux, outros dispositivos de rede) para expandir a cobertura
- Criar dashboards separados por categoria (infraestrutura vs. segurança)
- Automatizar a resposta a incidentes (ex: bloquear IP após N tentativas de login falho)
