---
name: "criar-teste-anti-regressao"
description: "Cria testes anti-regressão para o código alterado na branch ou PR atual, organizados na pasta tests/ por tipo e espelhando o código-fonte. Use ao pedirem testes de regressão ou cobertura das mudanças antes do merge."
---

# Criar testes anti-regressão

Objetivo: proteger o comportamento que a branch criou ou alterou, para que uma
mudança futura que o quebre seja pega pelos testes. Um bom teste anti-regressão
falha quando o comportamento muda e continua passando quando só a implementação
muda.

Esta skill costuma vir depois da `revisar-pr`. Se houver um relatório dela na
conversa, use a lista de "Próximo passo recomendado" como ponto de partida.

## Regras fixas

- **Todo teste fica dentro da pasta `tests/`** na raiz do projeto, na
  estrutura descrita em "Estrutura da pasta tests/". Nunca crie testes ao lado
  do código de produção nem soltos na raiz.
- **Só escreva dentro de `tests/`** (testes, fixtures, factories, dados de
  teste) e, se necessário, na configuração do runner (registro de markers,
  paths). Nunca altere código de produção para fazer um teste passar.
- Não mova nem renomeie testes existentes sem pedir.
- Não faça commit nem push, a menos que seja pedido.
- Não adicione dependências novas sem perguntar antes.
- Se o comportamento atual parecer um bug, **não o cimente num teste**: pare,
  descreva o caso e pergunte ao usuário qual é o comportamento correto.

## Estrutura da pasta tests/

Organize por **nível de teste** no primeiro nível e, dentro de cada nível,
**espelhe a estrutura do código-fonte**. Assim qualquer pessoa acha o teste de
um módulo só olhando o caminho dele.

```
tests/
├── conftest.py              # fixtures globais (config, relógio congelado, etc.)
├── unit/                    # rápidos, sem I/O; regra de domínio e casos de uso
│   ├── domain/
│   │   └── honorarios/
│   │       └── test_calculo.py
│   └── application/
│       └── test_emitir_parecer.py
├── integration/             # banco, repositórios, filas, clientes externos
│   ├── conftest.py          # setup de banco/containers só para este nível
│   └── infrastructure/
│       └── repositories/
│           └── test_parecer_repository.py
├── api/                     # contrato dos endpoints (status, schema, erros)
│   ├── conftest.py          # cliente HTTP de teste
│   └── test_pareceres_endpoints.py
├── e2e/                     # fluxos completos (só se o projeto já usar)
├── factories/               # builders/factories de objetos de teste
│   └── parecer_factory.py
└── data/                    # arquivos estáticos (JSON, PDFs, payloads de exemplo)
    └── pareceres/
        └── parecer_valido.json
```

Regras da estrutura:
- **Espelhamento:** `src/app/domain/honorarios/calculo.py` →
  `tests/unit/domain/honorarios/test_calculo.py`. Ignore o prefixo do pacote
  raiz (`src/app/`) e mantenha o resto do caminho.
- **Nível certo para cada teste:**
  - `unit/`: sem banco, rede ou disco; dependências externas substituídas por
    doubles. É onde fica a maior parte dos testes.
  - `integration/`: conversa de verdade com banco, cache, fila ou sistema de
    arquivos.
  - `api/`: requisição HTTP ao app de teste, verificando status, corpo e
    erros.
  - `e2e/`: só se o projeto já tiver; não crie esse nível por conta própria.
- **Fixtures em camadas:** `conftest.py` da raiz só com o que todos usam; o
  que é específico de um nível fica no `conftest.py` daquele nível. Não crie
  um `conftest.py` gigante.
- **Dados de teste:** objetos montados em código vão em `factories/`, com
  valores padrão válidos e sobrescrita por parâmetro
  (`make_parecer(status="arquivado")`). Arquivos fixos vão em `data/`. Nada de
  dados soltos copiados em vários testes.
- **Testes anti-regressão não ganham pasta própria.** Eles ficam no nível e no
  caminho do código que protegem e recebem o marker `regression`, para poderem
  ser rodados em grupo (`pytest -m regression`). Uma pasta `regression/`
  separada duplicaria a organização e esconderia o teste de quem mexe no
  módulo.
- **Nomes:**
  - arquivo: `test_<modulo>.py`
  - agrupamento opcional por unidade: `class TestCalculoHonorarios:`
  - teste: `test_<resultado>_quando_<condicao>`
    (ex.: `test_aplica_teto_quando_valor_excede_limite`)
  - teste de correção de bug: docstring curta com o cenário e a referência
    (`"""Regressão: valor negativo era aceito (PR #123)."""`)
- **Colisão de nomes (pytest):** com estrutura espelhada é comum ter dois
  `test_models.py`. Use `--import-mode=importlib` na configuração do pytest,
  ou `__init__.py` em cada pasta de teste, seguindo o que o projeto já fizer.
- **Markers registrados:** registre `unit`, `integration`, `api` e
  `regression` na configuração (`[tool.pytest.ini_options] markers` no
  `pyproject.toml`, ou `pytest.ini`), se ainda não existirem, para evitar
  avisos e permitir filtrar a execução.

Equivalentes em outras stacks (mesma ideia: nível + espelhamento):
- **TypeScript/JavaScript (jest/vitest):** `tests/unit/...`,
  `tests/integration/...`, `tests/api/...`; arquivos `<modulo>.test.ts`;
  factories em `tests/factories/`; setup em `tests/setup/`. Marque regressão
  com `describe` ou tag no nome (`[regression]`) conforme o runner suportar.
- **PHP (PHPUnit / Laravel):** `tests/Unit/...`, `tests/Integration/...`,
  `tests/Feature/...` (API), em PascalCase espelhando o namespace; classes
  `<Classe>Test.php`; regressão com `#[Group('regression')]`.

### Quando o projeto já tem testes
- Se `tests/` já tem uma organização clara e consistente por nível e/ou
  espelhamento, siga a do projeto: ela já é o padrão da equipe.
- Se os testes existentes estão soltos, misturados ou fora de `tests/`, crie
  os novos na estrutura acima, **sem mover os antigos**, e sugira no relatório
  uma migração gradual.
- Se não houver nenhuma infraestrutura de testes, proponha a estrutura acima
  com a configuração mínima do runner e peça confirmação antes de criar.

## 1. Levantar o escopo

1. Branch atual: `git branch --show-current`
2. Branch base, nesta ordem:
   - se houver PR aberto: `gh pr view --json baseRefName,title,body,number`
   - senão: `git symbolic-ref refs/remotes/origin/HEAD`
   - se o usuário indicar outra base, use a dele.
3. Diff a partir do merge-base:
   - `git fetch origin <base>`
   - `git diff --stat origin/<base>...HEAD`
   - `git diff origin/<base>...HEAD`
4. Se o usuário apontar arquivos ou comportamentos específicos, restrinja o
   escopo a eles.
5. Classifique a mudança, porque isso define como validar os testes (passo 5):
   - **Refatoração**: o comportamento deveria ser o mesmo da base.
   - **Funcionalidade nova**: comportamento que não existia na base.
   - **Correção de bug**: comportamento que estava errado na base.
   Uma branch pode misturar os três; classifique por trecho.

## 2. Entender a infraestrutura de testes do projeto

Antes de escrever qualquer teste:
- Framework e runner (`pytest`, `unittest`, `jest`, `vitest`, `phpunit`…) e
  como é executado (Makefile, `package.json`, `pyproject.toml`, CI).
- Estado atual da pasta `tests/` e qual caso de "Quando o projeto já tem
  testes" se aplica.
- Fixtures, factories, `conftest.py` e helpers de banco ou de cliente HTTP já
  existentes. **Reaproveite antes de criar.**
- Como o projeto lida com I/O: banco de teste, containers, mocks, doubles.
- Leia dois ou três testes existentes parecidos com o que você vai escrever e
  imite o estilo.

## 3. Mapear o que proteger

Liste os comportamentos alterados, observáveis pela interface pública:
- Pontos de entrada mexidos: endpoints, casos de uso, funções e métodos
  públicos, comandos, jobs.
- Regras de negócio e cálculos.
- Contratos: formato de resposta, códigos de status, exceções lançadas,
  eventos emitidos, efeitos persistidos.
- Casos de borda: vazio, nulo, limites, duplicados, permissões, entradas
  inválidas.
- Caminhos de erro: falhas de dependência externa, timeouts, validação.

Priorize por risco: regra de negócio e contrato público primeiro; detalhes
internos e getters triviais por último (ou nunca). Prefira poucos testes que
pegam regressões reais a muitos testes que só aumentam a cobertura.

Antes de escrever, mostre ao usuário o plano priorizado: comportamento →
nível (unit/integration/api) → caminho do arquivo em `tests/`. Se o usuário
não estiver acompanhando, siga com o plano e registre-o no relatório final.

## 4. Escrever os testes

- Coloque cada teste no nível e no caminho espelhado definidos na estrutura,
  com o marker do nível e o marker `regression`.
- **Teste comportamento, não implementação.** Exercite pela interface
  pública; não teste métodos privados nem a ordem interna de chamadas.
- **Um comportamento por teste**, com o padrão de nome definido acima.
- Estrutura Arrange / Act / Assert bem separada.
- Use `pytest.mark.parametrize` (ou equivalente) para variações do mesmo
  comportamento, em vez de copiar o teste.
- **Determinismo:** nada de rede real, relógio real ou aleatoriedade sem
  semente. Congele o tempo, fixe seeds, isole o sistema de arquivos.
- **Mocks só nas fronteiras** (HTTP, fila, serviço externo, relógio). Não
  mocke o próprio código sob teste nem objetos de domínio.
- Dados via `factories/` e `data/`, nunca valores mágicos repetidos.
- Asserções específicas: verifique os valores que importam, não um snapshot
  gigante de tudo.
- Testes independentes entre si: nenhum depende da ordem de execução nem do
  estado deixado por outro.
- Para refatorações, escreva **testes de caracterização**: fixe o
  comportamento observável que já existia, inclusive casos de borda.
- Para correções de bug, inclua um teste que reproduz exatamente o cenário
  do bug, com a docstring de regressão.

## 5. Validar que os testes realmente protegem

1. Rode os testes novos na branch atual. Todos precisam passar.
2. Rode a suíte afetada (ou a suíte inteira, se for rápida) para garantir que
   nada quebrou.
3. Valide contra a base usando um worktree temporário, sem tocar na árvore de
   trabalho do usuário:
   - `git worktree add /tmp/base-check origin/<base>`
   - copie os testes novos (e factories/dados que eles usam) para lá e rode-os
   - resultado esperado:
     - **Refatoração** → os testes **passam** na base (o comportamento foi
       preservado).
     - **Correção de bug** → o teste do bug **falha** na base (prova que ele
       pega o problema).
     - **Funcionalidade nova** → não se aplica; pule.
   - remova ao final: `git worktree remove /tmp/base-check --force`
   - Se a estrutura mudou demais para os testes rodarem na base, registre
     isso no relatório em vez de forçar.
4. Teste de sanidade (opcional, para os testes mais críticos): no worktree
   temporário, introduza uma alteração pequena no comportamento protegido e
   confirme que o teste falha. Nunca faça isso na árvore de trabalho do
   usuário.

Se um teste passar em tudo, inclusive quando o comportamento é alterado, ele
não está protegendo nada: reescreva ou remova.

## 6. Relatório

Em português, tom profissional e direto:

### Resumo
- **Branch:** <branch> → <base>
- **Testes criados:** <n> testes em <m> arquivos
- **Resultado:** ✅ todos passando / ⚠️ <detalhe>

### Estrutura criada ou alterada
Árvore apenas dos arquivos novos ou modificados em `tests/` (e configuração
do runner, se mexida).

### Testes criados
| Arquivo | Teste | Comportamento protegido | Nível |
|---|---|---|---|
| `tests/unit/...` | `test_...` | … | unit / integration / api |

### Validação
- Branch atual: resultado
- Base (refatoração/bug): resultado esperado e obtido
- Sanidade: o que foi testado, se feito

### Como rodar
- Só os testes novos: comando exato
- Todos os testes de regressão: `pytest -m regression` (ou equivalente)

### Lacunas
Comportamentos que ficaram sem teste e por quê (custo alto, depende de
ambiente externo, precisa de decisão do usuário).

### Possíveis bugs encontrados
Comportamentos que pareceram incorretos durante a escrita dos testes, com
arquivo e linha. Não foram corrigidos nem fixados em teste.

### Sugestões de organização
Só se os testes antigos estiverem fora do padrão: o que migrar e para onde,
sem ter movido nada.

## Princípios de conduta
- Um teste que nunca falha não vale nada; um teste que falha a cada
  refatoração atrapalha. Mire no meio: comportamento observável.
- O caminho do teste deve dizer o que ele testa e em que nível.
- Convenção consistente do projeto acima de preferência pessoal.
- Poucos testes certeiros valem mais que cobertura inflada.
- Na dúvida sobre o comportamento correto, pergunte em vez de presumir.