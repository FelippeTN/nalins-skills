---
name: "criar-log-rotativo"
description: "Configura logs em arquivo com rotação diária e exclusão automática após 15 dias, na aplicação (Python/FastAPI, PHP, Node) ou via logrotate no servidor. Use ao pedirem logs, rotação ou retenção de logs."
---

# Criar log com rotação e retenção de 15 dias

Objetivo: o projeto passa a gravar logs em arquivo, com um arquivo por dia, e
os arquivos com mais de 15 dias são apagados automaticamente. A configuração
fica centralizada, parametrizável por variável de ambiente e segura para o
jeito como a aplicação realmente roda em produção.

Padrões, salvo pedido diferente do usuário:
- rotação **diária, à meia-noite** (horário local do servidor)
- **retenção de 15 dias**; o que for mais antigo é apagado
- pasta `logs/` na raiz do projeto em desenvolvimento e `/var/log/<app>/` em
  servidor
- encoding UTF-8 e timestamp ISO 8601 em toda linha

## Regras fixas

- Nunca registre segredos ou dados sensíveis: senhas, tokens, chaves, headers
  `Authorization`, CPF e dados pessoais completos. Se o código atual fizer
  isso, aponte no relatório.
- Não reescreva as chamadas de log existentes além do necessário; o foco é a
  configuração.
- Não crie nem altere nada fora do repositório (ex.: `/etc/logrotate.d/`,
  `/var/log/`) sem confirmação explícita: isso exige `sudo` e mexe no
  servidor. Gere o arquivo no repositório (ex.: `deploy/logrotate/<app>`) e
  mostre os comandos de instalação.
- Não adicione dependências novas sem perguntar antes.
- Não faça commit nem push, a menos que seja pedido.
- Garanta que `logs/` e `*.log` estejam no `.gitignore`.

## 1. Levantar o contexto

Antes de escolher a estratégia, descubra:
1. **Stack:** Python (FastAPI, Django, scripts), PHP (Laravel, puro), Node.
2. **Como roda em produção** (procure em `Dockerfile`, `docker-compose.yml`,
   `gunicorn.conf.py`, unidades `systemd`, scripts de deploy, comando do
   `uvicorn`):
   - um único processo
   - vários processos/workers (`gunicorn -w N`, `uvicorn --workers N`,
     PHP-FPM)
   - container Docker
3. **Logging existente:** já há `logging.basicConfig`, `dictConfig`, Monolog,
   winston, `print` espalhado? Onde é configurado?
4. **Servidor:** se for RHEL/CentOS, considere SELinux para logs em
   `/var/log`.

Se não der para descobrir como roda em produção, pergunte. A escolha da
estratégia depende disso.

## 2. Escolher a estratégia

| Como roda | Estratégia | Por quê |
|---|---|---|
| Processo único (script, job, app com 1 worker) | Rotação na própria aplicação | Simples e sem dependência do servidor |
| Vários processos/workers no mesmo servidor | App grava em arquivo **sem rotacionar**; **logrotate** rotaciona e apaga | Rotação feita por vários processos ao mesmo tempo corrompe ou perde logs |
| Container Docker | Log para **stdout** e rotação no runtime do Docker; ou volume + logrotate no host se exigirem arquivo | O padrão de container é stdout |

Isto é importante: `TimedRotatingFileHandler` (Python) e equivalentes **não
são seguros com vários processos** gravando no mesmo arquivo. Cada worker
tenta rotacionar sozinho e o resultado são arquivos sobrescritos ou logs
perdidos. Com múltiplos workers, use sempre logrotate.

## 3. Implementar

### Configuração comum (todas as stacks)
Variáveis de ambiente, com padrões seguros, documentadas no `.env.example`:
```
LOG_LEVEL=INFO
LOG_DIR=logs
LOG_RETENTION_DAYS=15
```
Formato de linha:
`2026-11-23T10:34:00-0300 | INFO | app.modulo | mensagem`

### Python (logging da stdlib)
Centralize num único módulo, chamado uma vez na inicialização
(ex.: `app/core/logging_config.py`, seguindo a estrutura do projeto). Use
`logging.config.dictConfig`.

**Processo único** — rotação pela aplicação:
```python
import logging.config
import os
from pathlib import Path

def setup_logging() -> None:
    log_dir = Path(os.getenv("LOG_DIR", "logs"))
    log_dir.mkdir(parents=True, exist_ok=True)
    retention_days = int(os.getenv("LOG_RETENTION_DAYS", "15"))
    level = os.getenv("LOG_LEVEL", "INFO").upper()

    logging.config.dictConfig({
        "version": 1,
        "disable_existing_loggers": False,
        "formatters": {
            "default": {
                "format": "%(asctime)s | %(levelname)s | %(name)s | %(message)s",
                "datefmt": "%Y-%m-%dT%H:%M:%S%z",
            },
        },
        "handlers": {
            "console": {
                "class": "logging.StreamHandler",
                "formatter": "default",
            },
            "file": {
                "class": "logging.handlers.TimedRotatingFileHandler",
                "formatter": "default",
                "filename": str(log_dir / "app.log"),
                "when": "midnight",
                "interval": 1,
                "backupCount": retention_days,  # apaga o que passar de 15 arquivos/dias
                "encoding": "utf-8",
                "delay": True,
            },
        },
        "root": {"level": level, "handlers": ["console", "file"]},
    })
```
`backupCount` com `when="midnight"` mantém 15 arquivos diários e apaga o mais
antigo a cada rotação.

**Vários workers** — troque o handler de arquivo por
`logging.handlers.WatchedFileHandler` (sem `when`/`backupCount`). Ele reabre o
arquivo quando o logrotate o move. A rotação e a exclusão ficam com o
logrotate (abaixo).

**FastAPI/Uvicorn:** faça os loggers `uvicorn`, `uvicorn.error` e
`uvicorn.access` usarem a mesma configuração (declare-os no `dictConfig` com
`propagate: True` e sem handlers próprios, ou passe `log_config` ao
Uvicorn), senão os logs de acesso ficam fora do arquivo.

No resto do código: `logger = logging.getLogger(__name__)` por módulo; nunca
`print` para log. Use `logger.exception(...)` dentro de `except` para manter o
stack trace.

### PHP
- **Laravel:** em `config/logging.php`, canal `daily` com `'days' =>
  env('LOG_RETENTION_DAYS', 15)` e `LOG_CHANNEL=daily` (ou dentro de um
  `stack`).
- **Monolog puro:** `new RotatingFileHandler($dir.'/app.log', 15, Level::Info)`.
- Com PHP-FPM e vários processos, prefira logrotate, pela mesma razão dos
  workers Python.

### Node
- Use a biblioteca que o projeto já tiver. Com winston, o transporte
  `winston-daily-rotate-file` com `maxFiles: '15d'`. Com pino, `pino-roll` ou
  log para stdout + logrotate. Pergunte antes de instalar.

### logrotate (servidor, múltiplos workers)
Gere no repositório `deploy/logrotate/<app>`:
```
/var/log/<app>/*.log {
    daily
    rotate 15
    maxage 15
    dateext
    dateformat -%Y-%m-%d
    compress
    delaycompress
    missingok
    notifempty
    create 0640 <usuario> <grupo>
}
```
- `rotate 15` e `maxage 15` garantem a exclusão depois de 15 dias, mesmo se a
  rotação falhar algum dia.
- Com `WatchedFileHandler` (Python) não é preciso sinalizar o processo. Se a
  aplicação não reabrir o arquivo sozinha, use `copytruncate` (pode perder
  poucas linhas no instante da cópia) ou um `postrotate` que recarregue o
  serviço.
- Instalação (só com confirmação do usuário):
  ```
  sudo install -m 0644 deploy/logrotate/<app> /etc/logrotate.d/<app>
  sudo mkdir -p /var/log/<app> && sudo chown <usuario>:<grupo> /var/log/<app>
  sudo restorecon -Rv /var/log/<app>   # RHEL com SELinux
  sudo logrotate -d /etc/logrotate.d/<app>   # simulação, não altera nada
  ```

### Docker
- Aplicação loga em stdout. No `docker-compose.yml`:
  ```yaml
  logging:
    driver: json-file
    options:
      max-size: "20m"
      max-file: "15"
  ```
- O driver `json-file` rotaciona por **tamanho**, não por dia. Se a retenção
  por 15 dias for obrigatória, grave em arquivo num volume e use logrotate no
  host. Explique essa diferença ao usuário.

## 4. Verificar

1. Suba a aplicação (ou rode um script mínimo que chame `setup_logging()`) e
   confirme que o arquivo é criado no caminho certo, com o formato certo, e
   que logs do framework (ex.: acesso do Uvicorn) também aparecem.
2. Rotação pela aplicação: adicione um teste unitário da configuração
   (handler correto, `when="midnight"`, `backupCount` igual a
   `LOG_RETENTION_DAYS`, leitura das variáveis de ambiente), seguindo a
   estrutura de `tests/` do projeto (ex.:
   `tests/unit/core/test_logging_config.py`). Use pasta temporária; não
   grave em `logs/` real.
3. logrotate: rode `logrotate -d` para validar a sintaxe e mostre a saída.
   Só rode `logrotate -f` (forçar rotação) com confirmação.
4. Confirme que `logs/` está no `.gitignore`.

## 5. Relatório

Em português, tom profissional e direto:

### Resumo
- **Estratégia escolhida:** rotação na aplicação / logrotate / Docker, e o
  motivo (como a aplicação roda)
- **Retenção:** 15 dias, rotação diária à meia-noite
- **Onde ficam os logs:** caminho em desenvolvimento e em produção

### Arquivos criados ou alterados
Lista com uma linha sobre cada um.

### Variáveis de ambiente
`LOG_LEVEL`, `LOG_DIR`, `LOG_RETENTION_DAYS` e seus padrões.

### Passos manuais no servidor
Comandos exatos que o usuário precisa rodar com `sudo`, se houver. Nada
disso foi executado sem confirmação.

### Verificação
O que foi testado e o resultado.

### Pontos de atenção
Logs com dados sensíveis encontrados no código, `print` usados como log,
loggers de bibliotecas que ficaram de fora.