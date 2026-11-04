---
name: wfaudit
description: Workflow completo de auditoria de projeto (segurança, bugs de lógica, qualidade, arquitetura e organização), com mapeamento de superfície de ataque, divisão por módulo para código grande e verificação de falsos positivos. Use quando o usuário chamar /wfaudit ou pedir "faz o workflow de auditoria do projeto".
argument-hint: "[foco opcional: pasta, módulo, preocupação específica]"
disable-model-invocation: true
---

# Workflow de auditoria do projeto (/wfaudit)

Foco pedido pelo usuário (pode estar vazio): $ARGUMENTS

Siga as etapas **nesta ordem, sem pular nenhuma**. Em cada etapa, carregue a skill indicada com a ferramenta Skill e siga as instruções dela.

## Regras do workflow

- **Etapas 1 a 8 só leem e analisam. Não edite, mova, apague nem renomeie nada nelas.** As skills que normalmente aplicam correções sozinhas devem só listar os achados aqui. A única escrita permitida é a pasta `.wfaudit/` e o `AUDITORIA.md`.
- **Persista os achados em disco, não só na conversa.** Crie `.wfaudit/achados.md` na raiz do projeto e acrescente nele ao fim de cada etapa. Se o contexto for compactado, releia esse arquivo antes de continuar. Se o projeto for um repositório git, avise que `.wfaudit/` pode ir para o `.gitignore`.
- Cada achado tem: ID, severidade (Crítico, Alto, Médio, Baixo), arquivo e linha, problema, como reproduzir ou o caminho do dado até o ponto falho, correção sugerida, etapa de origem e status (`não verificado`, `confirmado`, `descartado`).
- Ao terminar cada etapa, diga em uma linha quantos achados ela teve e qual é o mais grave.
- Se uma skill ou ferramenta não estiver disponível (por exemplo, `semgrep` ou `codeql` não instalado), diga isso em uma linha e siga para a próxima. Não instale nada sem perguntar.
- Se houver foco em $ARGUMENTS, priorize-o, mas ainda rode todas as etapas.
- Cobertura honesta: no relatório final, liste quais pastas e arquivos foram lidos a fundo e quais só por amostragem. Nunca diga que o projeto inteiro foi auditado se não foi.

## 1. Mapa do projeto

Não carrega skill. Leia README, manifestos de dependência (package.json, requirements.txt, pyproject.toml etc.), estrutura de pastas e `git log --oneline -30` se for um repositório git. Conte arquivos de código e linhas aproximadas. Resuma: stack, como roda, partes principais e onde ficam as áreas mais alteradas.

Classifique o tamanho:
- **Pequeno** (até ~150 arquivos de código ou ~30 mil linhas): rode tudo na conversa principal.
- **Grande** (acima disso): use a divisão por módulo da etapa 4.

## 2. Superfície de ataque e fluxo de dados

1. `entry-point-analyzer:entry-point-analyzer`: liste pontos de entrada (rotas, handlers, comandos, jobs, webhooks, leitura de arquivos e e-mails) e quais aceitam dado externo.
2. `trailmark:trailmark-summary`: resumo estrutural e grafo de chamadas. Se o Trailmark não estiver disponível, pule com uma linha.

Resultado: lista priorizada de módulos e funções de maior risco (entrada externa, autenticação, dinheiro, dados pessoais, execução de comandos, acesso a arquivos e banco). As etapas seguintes começam por ela.

## 3. Segurança

Rode, nesta ordem:

1. `cso`: auditoria de segurança do projeto inteiro (segredos, autenticação, injeções, superfície de ataque).
2. `security-reviewer`: varredura de vulnerabilidades com severidade e correção para cada uma.
3. `insecure-defaults:audit`: configurações e padrões inseguros (CORS aberto, debug ligado, senhas padrão, cookies sem flags).
4. `supply-chain-risk-auditor:supply-chain-risk-auditor`: risco das dependências.
5. `static-analysis:semgrep` e `static-analysis:codeql`: análise estática, cada uma só se a ferramenta estiver instalada.
6. `sharp-edges:sharp-edges`: APIs e configurações propensas a uso errado.
7. Se for repositório git com histórico: `differential-review:differential-review` nas áreas mais alteradas listadas na etapa 1.

Junte achados duplicados entre essas skills num só.

## 4. Bugs de lógica e qualidade

Esta etapa busca falhas de comportamento, não só estilo. Foque em: condições de corrida, tratamento de erro ausente, casos de borda (vazio, nulo, limites, fuso horário, encoding), validação de entrada, estados inconsistentes, recursos não liberados e regras de negócio contraditórias.

**Projeto pequeno:**
1. `health`: saúde geral (testes, lint, tipos, código morto).
2. `audit-context-building:audit-context-building` nos módulos de maior risco da etapa 2, depois `code-reviewer` em todos os arquivos de código, não só nos principais.

**Projeto grande:**
1. Divida o código em módulos (por pasta ou domínio) com cerca de 15 a 30 arquivos cada.
2. Antes de disparar, mostre ao usuário o plano: quantos módulos, quantos subagentes e que o custo em tokens é alto. Só dispare depois de ele confirmar (é a única confirmação do workflow).
3. Rode um subagente por módulo, em paralelo, cada um com `audit-context-building` e `code-reviewer`, escrevendo seus achados em `.wfaudit/modulo-<nome>.md` no formato de achado acima.
4. Rode `health` na conversa principal enquanto os subagentes trabalham.
5. Junte os arquivos dos módulos em `.wfaudit/achados.md`.

Depois, para cada achado Alto ou Crítico, rode `variant-analysis:variant-analysis` para procurar o mesmo padrão em outros pontos do código e acrescente as variantes.

## 5. Verificação de falsos positivos

Para cada achado Crítico ou Alto, use `fp-check:fp-check` (ou, sem ela, releia o código e tente refutar o achado: o dado realmente chega ao ponto falho? existe validação antes? o caminho é alcançável?). Atualize o status para `confirmado` ou `descartado` com uma linha de justificativa. Achados Médios e Baixos ficam como `não verificado`, e isso deve constar no relatório.

## 6. Qualidade dos testes

1. Liste o que tem e o que não tem teste nos módulos de maior risco.
2. Se houver testes e as ferramentas estiverem disponíveis, use `mutation-testing:mutation-testing` em um ou dois módulos críticos para medir se os testes pegam defeitos. Se o custo for alto, só proponha e pergunte.

## 7. Arquitetura

A skill `improve-codebase-architecture` só pode ser chamada pelo usuário, então leia o arquivo `~/.claude/skills/improve-codebase-architecture/SKILL.md` (e o `HTML-REPORT.md` ao lado dele) e siga as etapas 1 e 2 dele: explorar e gerar o relatório HTML. **Pare antes da etapa 3 (grilling)**; ela fica para o final, se o usuário quiser.

## 8. Organização

1. `file-organizer`: proponha uma estrutura de pastas e nomes mais clara, **só como plano**, com a lista do que mover para onde. Não mova nada.
2. Liste código duplicado, arquivos mortos e funções grandes demais que valem simplificar.

## 9. Relatório final

Escreva `AUDITORIA.md` na raiz do projeto com:

1. Resumo de 5 linhas: estado geral e os 3 problemas mais urgentes.
2. Cobertura: o que foi lido a fundo, o que foi amostrado, o que ficou de fora e quais ferramentas não estavam disponíveis.
3. Tabela de achados ordenada por severidade (Crítico primeiro), com a coluna de status (`confirmado`, `descartado`, `não verificado`). Achados descartados vão numa seção separada curta.
4. Plano de correção em fases: primeiro segurança crítica e alta, depois bugs confirmados, depois testes, depois organização e arquitetura.
5. Caminho do relatório HTML de arquitetura da etapa 7.

Mostre o resumo e pergunte: **"Quer que eu aplique as correções? Por qual fase começo?"** Não aplique nada antes da resposta.

## 10. Correções (só com aprovação)

Quando o usuário aprovar:

- Se for repositório git, crie uma branch nova antes de editar. Se não for, avise que não há como desfazer pelo git e sugira criar uma cópia ou `git init` antes.
- Aplique uma fase por vez. Use `secure-code-guardian` para as correções de segurança, `caveman:safe-refactor` para refatorações e reorganização de pastas, e `simplify` para limpar duplicação.
- Corrija só achados `confirmado` ou `não verificado` que o usuário aceitar. Nunca os `descartado`.
- Depois de cada fase, rode os testes e o build do projeto e mostre o resultado. Se algo quebrar, pare e avise.
- Para melhorar a arquitetura, use a etapa 3 do `improve-codebase-architecture` (grilling) com o candidato que o usuário escolher.
