# Agentic Dead Code Cleanup Protocol v1.0

A TDD-safe, adversarially verified cleanup workflow for Claude Code agents.

---

## Why this exists

Most AI coding agents are dangerously overconfident when deleting “unused code”.

Static analysis alone is not enough:
- frameworks use dynamic/runtime patterns
- reflection breaks simple dependency tracing
- monorepos confuse linters
- build tags change reachability
- AI agents tend to over-clean aggressively

This protocol adds:
- persistent resumable state
- adversarial verification subagents
- read-only validation agents
- TDD-gated deletion
- rollback-safe workflows
- confidence classification
- framework-aware exceptions
- multi-stack support

---

## Philosophy

This workflow does NOT attempt to mathematically prove semantic deadness.

Instead, it combines:
1. static analysis
2. adversarial search
3. runtime verification
4. characterization testing
5. human approval gates

to reduce the probability of catastrophic cleanup mistakes.

---

## Supported stacks

| Stack | Primary Tool | Secondary Validation |
|---|---|---|
| JS/TS | Knip + ESLint | Adversarial subagent |
| Python | Ruff + Vulture | Pyright contextual |
| Go | Staticcheck U1000 | Build-tag aware validation |
| Rust | Clippy | Macro-aware classification |

---

## Full Prompt

```txt
TAREFA: Auditoria e refatoração de código não utilizado, com TDD, verificação por subagent adversarial, e estado persistente para retomada entre sessões.

================================================
BOOTSTRAP (executar SEMPRE no início da sessão)
================================================

B.1. Verifique se existe CLEANUP_TASKS.md na raiz do projeto.

B.2. SE EXISTE:
   - Leia o arquivo inteiro.
   - Identifique a última fase/step com [~] em andamento ou o próximo [ ] pendente.
   - Reporte: "Sessão anterior detectada. Última posição: FASE X step Y. Continuar daqui? (s/n)".
   - Aguarde confirmação. Pule para a posição correspondente.

B.3. SE NÃO EXISTE:
   - Crie CLEANUP_TASKS.md com o template no final deste prompt.
   - Adicione .cleanup-tmp/ ao .gitignore (criar se não existir).
   - Comece da FASE 1.

B.4. PERSISTÊNCIA (regra dura):
   - Após CADA step concluído, atualize CLEANUP_TASKS.md.
   - Estados: [ ] pendente, [~] em andamento, [x] concluído, [!] bloqueado ou falso positivo.
   - Commit do arquivo a cada checkpoint: "chore(cleanup): checkpoint <fase>.<step>".

================================================
REGRAS GLOBAIS
================================================

R.1. Pare ao fim de cada fase. Aguarde aprovação explícita.
R.2. Nunca afirme "não usado" sem saída de ferramenta estática como evidência.
R.3. Cada finding precisa de: caminho:linha, tipo, saída literal da tool, exit code da tool, confiança (alta/média/baixa).
R.4. Ferramenta com exit code inesperado ou não documentado deve ser marcada como "NÃO VERIFICADA" para aquela categoria.
R.5. "Pronto/feito/implementado" só após testes verdes pós-mudança.
R.6. Nenhuma linha removida sem o ciclo TDD da FASE 3.

================================================
FASE 1: DETECÇÃO E SETUP (leitura apenas)
================================================

1.1. Detecte stack lendo: package.json, pyproject.toml, requirements.txt, go.mod, Cargo.toml, pom.xml, *.csproj.
1.2. Identifique gerenciador de pacotes, comando de teste, comando de build, e branch atual.
1.3. Detecte ferramentas instaladas (knip, ruff, vulture, staticcheck, etc) e versões.
1.4. Para projetos Go, registre GOOS/GOARCH atual e detecte presença de arquivos *_<os>.go, *_<arch>.go, ou diretivas //go:build.
1.5. Escopo padrão:
     INCLUIR: src/, lib/, app/, packages/*/src/
     EXCLUIR: node_modules/, dist/, build/, .next/, out/, .venv/, venv/, __pycache__/, *.test.*, *.spec.*, tests/, __tests__/, e2e/, fixtures/, generated/, migrations/, *.d.ts gerados
1.6. Apresente stack, comandos, branch e escopo. Aguarde aprovação.
1.7. Atualize CLEANUP_TASKS.md.

PARE. Aguarde aprovação.

================================================
FASE 2: AUDIT POR FERRAMENTAS (zero modificação)
================================================

2.1. Crie .cleanup-tmp/. Capture stdout, stderr e exit code separadamente:
   <comando> > .cleanup-tmp/<tool>.stdout 2> .cleanup-tmp/<tool>.stderr; echo $? > .cleanup-tmp/<tool>.exitcode

2.2. JavaScript/TypeScript:

   PRÉ-CHECK Knip (obrigatório antes de classificar findings):
   - Existe knip.json/knip.jsonc/knip.ts/knip.js ou bloco "knip" no package.json?
   - package.json com main, exports, bin, scripts coerentes?
   - Em monorepo, há workspaces["."] e demais workspaces configurados?
   - SE NÃO: rode knip primeiro só para colher Configuration Hints e ajustar config.

   Configuration Hints do Knip:
   Configuration Hints (emitidas no topo do output do Knip) NÃO são findings de código morto. São warnings de configuração que servem apenas para qualificar a confiabilidade dos demais findings. Trate-as assim:
   - Hints presentes e não resolvidas: classifique TODOS os findings de files/exports/dependencies como MÉDIA ou INCERTO, nunca ALTA.
   - Hints resolvidas ou ausentes: classificação normal por confiança.
   - Nunca apresente uma hint como "código não usado". Apresente como "ajuste de configuração necessário antes de avaliar o resto".

   Comandos:
   - npx knip --no-progress --reporter json > .cleanup-tmp/knip.stdout 2> .cleanup-tmp/knip.stderr; echo $? > .cleanup-tmp/knip.exitcode
   - Se ESLint configurado (eslint.config.* ou .eslintrc.*):
     npx eslint . --format json > .cleanup-tmp/eslint.stdout 2> .cleanup-tmp/eslint.stderr; echo $? > .cleanup-tmp/eslint.exitcode

2.3. Python:

   PRIMÁRIO:
   - ruff check <paths_do_escopo> --select F401,F811,F841 --output-format json > .cleanup-tmp/ruff.stdout 2> .cleanup-tmp/ruff.stderr; echo $? > .cleanup-tmp/ruff.exitcode
   - vulture <paths_do_escopo> --min-confidence 80 > .cleanup-tmp/vulture.stdout 2> .cleanup-tmp/vulture.stderr; echo $? > .cleanup-tmp/vulture.exitcode

   COMPLEMENTAR:
   - Pyright só se reportUnusedImport/reportUnusedClass/reportUnusedFunction/reportUnusedVariable estiverem como "warning" ou "error" (default é "none"). Se "none" ou ausentes: registre "Pyright presente, reportUnused* não ativos" e não use como primário.
   - Mypy apenas como complementar via --warn-unused-ignores.

2.4. Go:

   - go test ./... > .cleanup-tmp/gotest.stdout 2>&1; echo $? > .cleanup-tmp/gotest.exitcode
   - go vet ./... > .cleanup-tmp/govet.stdout 2>&1; echo $? > .cleanup-tmp/govet.exitcode  (sanity check apenas)
   - Se staticcheck instalado: staticcheck ./... > .cleanup-tmp/staticcheck.stdout 2>&1; echo $? > .cleanup-tmp/staticcheck.exitcode
   - APENAS staticcheck (U1000) ou ferramenta dedicada de unused é evidência de dead code. go vet não.

   Build tags / GOOS / GOARCH:
   Symbols podem parecer unused em uma configuração de build e estar perfeitamente vivos em outra. Antes de classificar U1000 como ALTA:
   - Detectar presença de: //go:build, // +build, arquivos *_linux.go, *_windows.go, *_darwin.go, *_amd64.go, *_arm64.go, tags customizadas
   - SE qualquer um presente:
     * Idealmente rodar staticcheck -matrix com as configurações relevantes do projeto e mesclar resultados.
     * Se não for prático rodar matrix: classificar U1000 como MÉDIA/INCERTO até que tenha sido verificado em todas as plataformas/tags relevantes para o projeto.
   - SE nenhum presente: classificação normal por confiança.

2.5. Rust:

   - cargo clippy --all-targets -- -W dead_code -W unused_imports > .cleanup-tmp/clippy.stdout 2>&1; echo $? > .cleanup-tmp/clippy.exitcode
   - Classificação:
     * unused_imports = ALTA
     * dead_code = MÉDIA por padrão
     * dead_code = ALTA apenas em função privada simples, sem #[derive], sem macro, sem feature flag, sem build.rs, sem closure capturando o símbolo.

2.6. Outros stacks: identifique o linter idiomático ou marque "NÃO VERIFICADA".

2.7. Ferramenta ausente: registre comando de instalação, marque categoria "NÃO VERIFICADA". Nunca infira manualmente.

2.8. Exceções globais (sempre INCERTO independente da tool):
   - Símbolos exportados em index.* ou no "exports"/"main"/"bin" do package.json
   - Referenciados via string literal (reflexão, rotas, RPC, ORM, DI)
   - Lazy imports, dynamic imports
   - Decorators e classes em DI container
   - Hooks de framework (getStaticProps, generateMetadata, loader, action)
   - Callbacks passados como prop
   - Usados em configs (vite.config.*, next.config.*)
   - Tipos referenciados apenas em .d.ts
   - Símbolos com /** @public */, /** @api */ ou nome iniciado em _
   - Funções de migration, seed, factory

2.9. Código comentado: blocos com mais de 3 linhas. git blame para data/autor. Ignore TODO/FIXME/HACK/NOTE com texto explicativo.

2.10. Gere AUDIT_REPORT.md com:
   - Configuration Hints (se houver, em seção separada como "ajustes de config necessários", não como findings)
   - Sumário
   - Findings por categoria
   - Tabela de INCERTO com motivo

2.11. Atualize CLEANUP_TASKS.md.

PARE. Apresente AUDIT_REPORT.md e aguarde aprovação para iniciar a FASE 2.5.

================================================
FASE 2.5: VERIFICAÇÃO POR SUBAGENT ADVERSARIAL READ-ONLY
================================================

2.5.1. Para CADA finding ALTA, dispare um subagent via Agent tool (anteriormente Task tool, renomeada em Claude Code v2.1.63; alias Task ainda aceito) com configuração READ-ONLY estrita por allowlist:
   - subagent_type: general-purpose
   - tools: Read, Glob, Grep
   - NÃO conceder Bash, Edit, Write, MultiEdit ou NotebookEdit. Allowlist explícita já garante que nada fora de Read/Glob/Grep é acessível, mas a proibição é mantida no prompt como instrução adicional para casos onde o subagent venha a herdar tools por configuração de ambiente.
   - Justificativa: o subagent não precisa de shell para esta auditoria. Issue conhecido anthropics/claude-code#21697 mostra que subagents com Bash tendem a usar sed/cat/find em vez das tools nativas, gerando ruído de permissão e risco de modificação acidental.

   Prompt do subagent (read-only):
   ---
   Você é um auditor hostil. Sua única missão é PROVAR que o símbolo `<NOME>` em `<ARQUIVO:LINHA>` ESTÁ sendo usado em algum lugar do projeto. Você NÃO pode modificar arquivos.

   Use APENAS as tools nativas Read, Glob, Grep. Você não tem acesso a Bash, sed, cat, find, ripgrep via shell ou qualquer tool de edição.

   Buscas obrigatórias com Grep:
   1. Chamadas diretas: padrão "<NOME>\\(" (nome seguido de parêntese)
   2. Imports e re-exports: "from.*<NOME>", "import.*<NOME>", "require.*<NOME>"
   3. String literal contendo o nome (reflexão, JSON, YAML, .env, .toml)
   4. JSDoc/TSDoc/docstrings
   5. Uso em testes (mesmo fora do escopo)
   6. Templates HTML, JSX, Vue, Svelte, Astro
   7. Scripts npm, configs CLI
   8. Migrations, seeds, factories
   9. Strings em arquivos de roteamento
   10. Macros, derives, feature flags (Rust); decorators (Python/TS); código gerado

   Use Glob para enumerar arquivos de extensões relevantes antes de buscar.

   Retorne EXATAMENTE um destes formatos, sem prosa adicional:
   - USADO | <arquivo>:<linha> | <trecho de contexto>
   - NAO_USADO | buscas: <lista das queries executadas>
   ---

2.5.2. Para MÉDIA/INCERTO, mesmo prompt acrescido de: "Considere padrões dinâmicos do framework <NOME_FRAMEWORK>: <listar convenções>".

2.5.3. Cruzamento:
   - Tool=NÃO USADO + Subagent=NAO_USADO = CONFIRMADO_REMOVER
   - Tool=NÃO USADO + Subagent=USADO = FALSO_POSITIVO (remover do relatório, registrar evidência)
   - Discrepância ambígua = ESCALAR_HUMANO

2.5.4. Gere VERIFIED_REPORT.md com três seções: confirmados, falsos positivos, escalados.

2.5.5. Atualize CLEANUP_TASKS.md.

PARE. Apresente VERIFIED_REPORT.md e aguarde aprovação item a item.

================================================
FASE 3: EXECUÇÃO COM DISCIPLINA TDD
================================================

3.1. Pré-condições (aborte se qualquer falhar):
   - git status: working tree limpo
   - <comando de teste>: exit 0
   - <comando de build>: exit 0
   - Branch atual NÃO é main/master/develop sem aprovação explícita
   Registre hash baseline em CLEANUP_TASKS.md.

3.2. Crie branch dedicada:
   - git checkout -b cleanup/dead-code-<YYYYMMDD>
   - Registre nome da branch em CLEANUP_TASKS.md.

3.3. (OPCIONAL, recomendado) Reforço de segurança Claude Code:
   - Se há .claude/settings.json, sugira deny rules para Bash(rm -rf *), Bash(git push --force*), Bash(git push * main), Bash(git push * master) durante a sessão.
   - Sugira PreToolUse hooks para bloquear destrutivos fora do escopo. Apenas sugerir.

3.4. Ciclo TDD para CADA item aprovado:

   PASSO A: Verificar cobertura do contexto adjacente
   - Existe caller, parent module ou consumidor identificável?
   - Existe teste cobrindo esse contexto?
   - NÃO → PASSO B. SIM → PASSO C.

   PASSO B: Escrever teste de caracterização
   - Teste exercendo caller mais próximo ou API pública adjacente.
     * Função: chame com inputs reais dos call sites, asserte output.
     * Componente: smoke render do componente pai.
     * Módulo: import + chamada da API pública.
   - Rode o teste. Deve passar (código intacto).
   - Commit: "test(cleanup): characterization for <símbolo>".

   PASSO C: Remoção verificada
   - Remova o código.
   - Rode TODA a suite.
   - PASSAM: commit "refactor(cleanup): remove unused <símbolo>".
   - FALHA: git reset --hard até antes do commit de remoção (mantendo teste de caracterização). Marque [!] FALSO_POSITIVO_RUNTIME em CLEANUP_TASKS.md com a saída do erro.

   PASSO D: Refactor de órfãos
   - Limpe imports, variáveis, tipos órfãos.
   - Rode testes outra vez.
   - Commit separado se não trivial.

   NOTA sobre limites do TDD aqui: para código verdadeiramente morto, a suite não detecta a remoção porque nada o exercia. Disciplina do PASSO B garante que contexto adjacente segue se comportando como antes. Se nenhum teste exercita o símbolo direta ou transitivamente, registre no commit: "no test coverage existed for <símbolo>, removal verified by adjacent characterization tests only".

3.5. Ordem dos commits por categoria:
   1. Imports não usados
   2. Variáveis locais não usadas
   3. Funções privadas não usadas
   4. Exports não usados (com aprovação item a item)
   5. Dependências não usadas (uninstall do gerenciador correspondente)
   6. Blocos de código comentado aprovados

3.6. Atualize CLEANUP_TASKS.md após cada commit com hash + status.

3.7. NÃO TOQUE em:
   - Arquivos fora do escopo aprovado
   - Itens não aprovados pelo usuário
   - Arquivos com mudanças não commitadas pré-existentes
   - Branch main/master/develop diretamente

3.8. Relatório final em CLEANUP_REPORT.md:
   - Branch criada e lista de commits (hash + mensagem)
   - Linhas removidas vs adicionadas por categoria
   - Falsos positivos pegos pelo TDD (que escaparam do subagent)

   Comandos de reversão (use o cenário aplicável):

   CENÁRIO A (branch ainda não mergeada):
     git checkout <branch_original>
     git branch -D cleanup/dead-code-<YYYYMMDD>

   CENÁRIO B (branch já mergeada na main/master/develop):
     git revert <hash_primeiro_commit_cleanup>..<hash_ultimo_commit_cleanup>
     # ou para um único commit de merge:
     git revert <hash_commit_de_merge> -m 1

   CENÁRIO C (commits parcialmente aplicados, alguns OK outros não):
     git revert <hash_do_commit_específico>

   Em todos os casos, rode a suite de testes após reverter e abra um issue documentando o motivo.

3.9. Marque FASE 3 como [x] em CLEANUP_TASKS.md.

================================================
PROIBIÇÕES
================================================
- Não pule fases nem combine fases.
- Não use "pronto/feito/implementado" sem testes verdes pós-mudança.
- Não invente nomes de arquivo, números de linha, saídas de tool, ou retornos de subagent.
- Não remova código sem o ciclo PASSO A → B (se preciso) → C → D.
- Não delete nem reescreva CLEANUP_TASKS.md sem checkpoint git prévio.
- Não rode comandos destrutivos fora dos passos especificados (rm fora de .cleanup-tmp/, git push --force, npm uninstall fora do passo 3.5.5).
- Não mude para branch main/master/develop durante a execução.
- Não interprete exit code não documentado como "tool falhou silenciosamente". Marque como NÃO VERIFICADA.
- Não dê Bash, Edit, Write, MultiEdit ou NotebookEdit ao subagent adversarial. Use allowlist tools: Read, Glob, Grep.
- Não trate Configuration Hints do Knip como findings.

================================================
TEMPLATE: CLEANUP_TASKS.md
================================================

# Cleanup Tasks

> Estado persistente para auditoria e refatoração de código não utilizado.
> Atualizado após cada step. Não editar manualmente sem commit prévio.

## Metadados
- Iniciado em: <DATA_ISO>
- Stack: <preencher>
- Comando de teste: <preencher>
- Comando de build: <preencher>
- Ferramentas instaladas e versões: <lista>
- Para Go: GOOS/GOARCH atual, presença de build tags: <preencher>
- Branch original: <preencher>
- Branch de cleanup: <preencher>
- Hash baseline (início FASE 3): <preencher>
- Primeiro commit cleanup: <preencher>
- Último commit cleanup: <preencher>

## Escopo aprovado
- Incluir: <lista>
- Excluir: <lista>

## Fase 1: Detecção e Setup
- [ ] 1.1 Stack detectado
- [ ] 1.2 Comandos identificados
- [ ] 1.3 Ferramentas verificadas
- [ ] 1.4 Build configs Go detectadas (se aplicável)
- [ ] 1.5 Escopo definido
- [ ] 1.6 Escopo aprovado pelo usuário

## Fase 2: Audit por Ferramentas
- [ ] 2.1 .cleanup-tmp/ criado, exit codes capturados
- [ ] 2.2 JS/TS: knip config validada, hints separadas de findings, knip+eslint rodados
- [ ] 2.3 Python: ruff+vulture primários, status pyright/mypy registrado
- [ ] 2.4 Go: staticcheck rodado, build tags consideradas na classificação
- [ ] 2.5 Rust: clippy com classificação ajustada
- [ ] 2.6 Outros stacks identificados ou marcados NÃO VERIFICADO
- [ ] 2.7 Exceções globais marcadas INCERTO
- [ ] 2.8 Comentários analisados
- [ ] 2.9 AUDIT_REPORT.md gerado (hints separadas de findings)
- Totais: alta=<A> média=<M> incerto=<B> nao_verificado=<NV>
- Configuration hints pendentes: <quantidade>

## Fase 2.5: Subagent Adversarial Read-Only
- [ ] 2.5.1 Subagents ALTA disparados (tools=Read,Glob,Grep)
- [ ] 2.5.2 Subagents MÉDIA/INCERTO disparados
- [ ] 2.5.3 Cruzamento realizado
- [ ] 2.5.4 VERIFIED_REPORT.md gerado
- Confirmados: <N> | Falsos positivos: <FP> | Escalados: <E>

## Fase 3: Execução com TDD
- [ ] 3.1 Pré-condições verificadas (working tree limpo, tests/build verdes, branch ok)
- [ ] 3.2 Branch cleanup/dead-code-<data> criada
- [ ] 3.3 Reforço de segurança Claude Code sugerido

| ID | Símbolo | Arquivo:Linha | Categoria | Status | Hash Teste | Hash Remoção | Notas |
|----|---------|---------------|-----------|--------|------------|--------------|-------|
| 01 |         |               |           | [ ]    |            |              |       |

## Próxima Ação
<descrever exatamente o que a próxima sessão deve fazer ao retomar>

## Log de Sessões
- Sessão 1 (<DATA>): chegou até <fase>.<step>. Próximo: <step>.
```
