---
name: "revisar-pr"
description: "Revisa o PR ou a branch atual contra a base: padrões de código, modularização, SOLID, DDD, DRY e overengineering. Use ao pedirem para revisar ou analisar um PR, branch ou diff."
---

# Revisão de PR

Faça uma revisão profissional, baseada em evidências e proporcional ao problema
que o PR resolve. O objetivo não é aplicar princípios por aplicar, e sim
responder: "este código está na medida certa para o que precisa fazer, e vai
ser fácil de manter?"

Regra central: **SOLID, DDD e DRY são ferramentas, não metas.** Cobrar
abstrações num CRUD simples é tão errado quanto aceitar um god-object num
domínio complexo. Toda crítica precisa ser proporcional à complexidade real
do problema.

Não use esta skill para revisar um arquivo isolado sem relação com uma branch.

## 1. Levantar o contexto

Não altere código, não faça commit, não faça push e não comente no PR, a menos
que seja pedido. Esta skill só lê.

1. Branch atual: `git branch --show-current`
2. Branch base, nesta ordem:
   - se houver PR aberto: `gh pr view --json baseRefName,title,body,number,url`
   - senão: `git symbolic-ref refs/remotes/origin/HEAD` (geralmente main/master)
   - se o usuário indicar outra base (ex.: develop), use a dele.
3. Atualize e gere o diff a partir do merge-base (três pontos, para pegar só o
   que a branch introduziu):
   - `git fetch origin <base>`
   - `git diff --stat origin/<base>...HEAD`
   - `git diff origin/<base>...HEAD`
   - `git log --oneline origin/<base>..HEAD`
4. Entenda a intenção: título e descrição do PR, mensagens de commit, issue
   vinculada. Se a intenção não estiver clara, diga isso no relatório. Não
   invente um objetivo.
5. **Leia além do diff.** Abra os arquivos alterados inteiros e o código
   vizinho (quem chama, o que é chamado, módulos irmãos). Não dá para julgar
   modularização, DRY ou consistência olhando só as linhas alteradas.
6. Identifique as convenções do repositório (estrutura de pastas, camadas,
   nomes, tratamento de erros, padrão de testes, linters configurados). O
   padrão de referência é **o do próprio projeto**; preferências genéricas
   vêm depois.

Se o diff for muito grande (mais de ~1500 linhas), avise, priorize os arquivos
de domínio e regra de negócio e diga explicitamente o que não foi revisado a
fundo.

## 2. Verificações automáticas (se existirem)

Rode só o que o projeto já tem configurado e que não altera estado:
testes (`pytest`, `npm test`, `phpunit`…), lint e tipagem (`ruff`, `mypy`,
`eslint`, `tsc --noEmit`, `phpstan`…). Restrinja aos arquivos afetados quando
der. Se algo falhar, entra como achado. Se não houver ferramenta ou não for
possível rodar, registre isso; não invente resultado.

## 3. Análise por dimensão

Para cada dimensão, procure sinais concretos no código. Só registre um achado
se conseguir apontar arquivo e linha.

### Correção (sempre primeiro)
Bugs, condições de corrida, erros engolidos, validação ausente, falhas de
segurança (injeção, segredos no código, autorização), quebra de contrato com
quem consome a API. Um problema de correção vale mais que qualquer questão de
estilo.

### Padrões de código
- Nomes que revelam intenção e seguem a linguagem do domínio
- Funções curtas com um nível de abstração cada; aninhamento raso
- Tratamento de erros consistente com o resto do projeto
- Consistência com as convenções existentes (não introduzir um segundo jeito
  de fazer a mesma coisa)
- Testes: cobrem o comportamento novo e os casos de borda? Testam
  comportamento ou detalhes de implementação?
- Código morto, comentários desatualizados, TODOs sem dono
- **Comentários desnecessários no código:** aponte comentários que só repetem
  o que o código já diz (ex.: `# incrementa o contador` acima de `i += 1`),
  comentários de seção óbvios, código comentado e explicações que deveriam
  ser um nome melhor. Comentário só se justifica quando explica o **porquê**
  que o código não consegue mostrar (regra de negócio, workaround, decisão
  não óbvia).

### Cobertura anti-regressão
Verifique se o PR cria ou altera testes que protejam o comportamento mexido:
- Procure arquivos de teste no diff (`test_*.py`, `*_test.py`, `*.spec.ts`,
  `*.test.js`, `tests/`, `__tests__/`, `*Test.php` etc.) e testes existentes
  que exercitem o código alterado.
- Considere a alteração **grande** quando qualquer um destes valer:
  - mais de ~300 linhas de código de produção alteradas (sem contar testes,
    lockfiles, migrations geradas, arquivos gerados e documentação);
  - mexe em regra de negócio, cálculo, fluxo de dados ou contrato de API
    usado por outros módulos;
  - refatora ou move lógica existente entre módulos ou camadas.
- Se a alteração for grande **e** nenhum teste anti-regressão cobrir o
  comportamento alterado (nenhum teste novo ou modificado no diff, e nenhum
  teste existente que exercite o código mexido), registre um achado
  🟠 **Importante** e inclua no relatório a seção "Próximo passo
  recomendado" sugerindo rodar a skill `criar-teste-anti-regressao`,
  listando os arquivos e comportamentos que mais precisam de cobertura.
  Não crie os testes você mesmo; apenas recomende.

### Modularização
- Coesão: cada módulo e classe tem um motivo claro para existir?
- Acoplamento: dependências fluem na direção certa? Há imports circulares ou
  um módulo que conhece detalhes internos de outro?
- Fronteiras: a lógica nova está na camada certa (ex.: regra de negócio fora
  de controller/rota, SQL fora do domínio)?
- Tamanho: arquivos ou classes que cresceram demais com este PR

### SOLID
- **S**: classe ou função com mais de um motivo para mudar (ex.: validar,
  persistir e notificar no mesmo método)
- **O**: `if/elif` ou `switch` por tipo que vai crescer a cada caso novo,
  *quando* existe variação real esperada
- **L**: subclasse que lança `NotImplementedError`, ignora parâmetros ou
  muda o contrato da base
- **I**: interface ou classe base que obriga implementações a ter métodos que
  não usam
- **D**: regra de negócio instanciando diretamente infraestrutura (cliente
  HTTP, ORM, SDK) quando isso dificulta testar ou trocar

### DDD (só onde houver domínio de verdade)
Avalie apenas se o projeto adota DDD ou se o PR mexe em regra de negócio
relevante. Não cobre DDD de scripts, CRUDs triviais ou código de
infraestrutura.
- Domínio dependendo de infraestrutura (models do ORM vazando para o domínio)
- Modelo anêmico: entidades que só guardam dados, com toda a regra em
  services
- Invariantes não protegidas (estado inválido pode ser criado)
- Fronteiras de agregado e de contexto respeitadas
- Linguagem ubíqua: nomes do código batem com os termos do negócio

### DRY
- Duplicação dentro do PR e **com código que já existe no repositório**
  (procure helpers e utilitários existentes antes de apontar)
- Cuidado com o falso DRY: duas coisas que hoje são parecidas mas mudam por
  motivos diferentes *não* devem ser unificadas. Duplicação acidental é
  melhor que a abstração errada.
- Regra de três: duas ocorrências geralmente ainda não justificam abstrair

### Overengineering (KISS / YAGNI)
Pergunta-guia: "uma solução mais simples resolveria o mesmo problema, com o
mesmo nível de manutenção?" Sinais:
- Interface ou classe abstrata com uma única implementação e sem segunda
  prevista
- Factory, strategy, builder ou registry para um único tipo
- Camadas que só repassam a chamada sem acrescentar nada
- Configuração, parâmetros ou generics para variações que não existem
- Generalização especulativa ("caso um dia precise…")
- Dependência nova para algo que a stdlib ou o projeto já resolve
- Mais arquivos e indireção do que o tamanho do problema justifica

Ao apontar overengineering, mostre a alternativa mais simples em código.
Também aponte o contrário (underengineering) quando a simplicidade estiver
gerando acoplamento ou duplicação real.

## 4. Relatório

Escreva em português, com tom profissional, direto e construtivo. Critique o
código, nunca a pessoa. Sem elogios vazios e sem conselhos genéricos: todo
achado precisa citar o código.

Formato:

### Resumo
- **PR / branch:** <título ou branch> → <base> (<n> arquivos, +X/−Y)
- **Objetivo entendido:** uma ou duas frases
- **Veredito:** ✅ Aprovar · ⚠️ Aprovar com ressalvas · ❌ Solicitar mudanças
- **Justificativa:** duas ou três frases com os pontos decisivos

### Avaliação por dimensão
| Dimensão | Avaliação | Comentário curto |
|---|---|---|
| Correção | 🟢/🟡/🔴 | … |
| Padrões de código | | |
| Cobertura anti-regressão | | |
| Modularização | | |
| SOLID | | |
| DDD | (ou "não se aplica") | |
| DRY | | |
| Proporcionalidade / overengineering | | |

### Achados
Agrupe por severidade, da mais grave para a menos grave:
- 🔴 **Bloqueante**: bug, risco de segurança, quebra de contrato
- 🟠 **Importante**: problema de design que vai custar caro para manter
- 🟡 **Sugestão**: melhoria clara, mas não impede o merge
- ⚪ **Nit**: detalhe de estilo (agrupe vários num item só)

Cada achado:
> **[Severidade] Título curto** — `caminho/arquivo.py:42-58`
> **Princípio:** SRP / DRY / YAGNI / …
> **Problema:** o que está acontecendo e por que isso importa neste contexto.
> **Sugestão:** a correção concreta, com trecho de código quando ajudar.

Nos trechos de código sugeridos, **não adicione comentários desnecessários**.
Deixe só o comentário que explica um porquê importante; o resto deve ficar
claro pelos nomes e pela estrutura.

### Pontos positivos
Só o que for genuinamente bom e específico (ex.: "boa separação entre o caso
de uso e o repositório em X"). Pode ficar vazio.

### Verificações executadas
O que foi rodado e o resultado, e o que não foi possível verificar.

### Próximo passo recomendado
Inclua esta seção somente quando a alteração for grande e não tiver testes
anti-regressão (ver "Cobertura anti-regressão"). Exemplo:
> Esta alteração é grande (<motivo: +N linhas em X / mexe na regra Y>) e não
> tem testes anti-regressão. Recomendo rodar a skill
> **`criar-teste-anti-regressao`** antes do merge, priorizando:
> - `caminho/arquivo.py` — <comportamento a proteger>
> - `caminho/outro.py` — <comportamento a proteger>

### Limitações da revisão
O que não foi analisado a fundo e por quê (tamanho, falta de contexto,
intenção do PR não documentada).

## Princípios de conduta
- Evidência acima de opinião: sem arquivo e linha, não é achado.
- Proporcionalidade: a régua é a complexidade real do problema.
- Convenção do projeto acima de preferência pessoal.
- Prefira poucos achados relevantes a uma lista longa de detalhes.
- Comentário no código só quando for importante: nunca adicione nem aprove
  comentários que apenas descrevem o óbvio.
- Na dúvida sobre uma intenção, pergunte no relatório em vez de presumir erro.