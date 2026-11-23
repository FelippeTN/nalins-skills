# nalins-skills

Skills para o Claude (claude.ai e Claude Code).

| Skill | O que faz |
|---|---|
| `revisar-pr` | Revisa o PR ou a branch atual: padrões de código, modularização, SOLID, DDD, DRY e overengineering. |
| `criar-teste-anti-regressao` | Cria testes anti-regressão para o código alterado na branch ou PR. |
| `criar-log-rotativo` | Configura logs em arquivo com rotação diária e retenção de 15 dias. |

Cada skill é uma pasta com um `SKILL.md`.

## Como instalar

### Opção 1: pela conta do claude.ai (recomendado)

A skill fica disponível no claude.ai e no Claude Code de quem estiver logado na
mesma conta.

1. Compacte a pasta da skill:
   ```sh
   zip -r revisar-pr.zip revisar-pr
   ```
2. No claude.ai, vá em **Settings → Capabilities → Skills** e envie o `.zip`.
3. No Claude Code, logado na mesma conta, a skill aparece como
   `anthropic-skills:<nome>`. Se ela não aparecer, abra uma nova sessão.

O Claude Code guarda uma cópia sincronizada em `~/.claude/skills/synced/`.
**Não edite essa cópia**: ela é sobrescrita na próxima sincronização.

### Opção 2: só no Claude Code, sem conta

Copie a pasta para o diretório de skills do usuário:

```sh
cp -r revisar-pr ~/.claude/skills/
```

Para usar só em um projeto, copie para `.claude/skills/` dentro dele.

## Como atualizar uma skill

1. Edite o `SKILL.md` neste repositório e faça commit.
2. Envie a nova versão pelo mesmo caminho usado na instalação: substitua a
   skill no claude.ai (opção 1) ou copie a pasta de novo (opção 2).

Alterar o repositório sozinho **não** atualiza a skill instalada.
