# claude-delphi-to-go

Uma skill para [Claude Code](https://claude.com/claude-code) que converte código e telas legadas de **Delphi** (Object Pascal, VCL, `.dfm`, FireDAC/FIBPlus sobre Firebird ou InterBase) para **Go idiomático + tela web**, através de leitura semântica assistida por IA em vez de tradução mecânica linha a linha.

## Por que isso existe

Delphi (formulários visuais `.dfm`, componentes `TDataSet`, fluxo modal `ModalResult`) e Go (sem framework de UI, sem classes, sem herança) são paradigmas incompatíveis demais para que um tradutor de sintaxe produza algo utilizável — principalmente na camada de tela. Conversores mecânicos existem (inclusive alguns genéricos multi-linguagem), mas eles traduzem token por token: o resultado compila, mas não faz sentido como aplicação web.

Esta skill parte de outro princípio: em vez de traduzir sintaxe, ela **lê a intenção** de cada handler de evento Delphi — o que aquele botão realmente faz, que validação aquele campo realmente impõe, que estado aquela tela realmente representa — e recria essa intenção como um fluxo web equivalente e idiomático em Go.

## O que ela faz

A skill classifica cada tela num de três arquétipos, porque cada um exige uma conversão estrutural diferente:

| Arquétipo | Convenção de nome | Padrão Delphi | Vira no Go |
|---|---|---|---|
| **Consulta/Busca** | `Cns_*`, `Busca*` | Filtro → `TDBGrid` → seleção devolve um ID via `ModalResult` | `GET /buscar?filtro=...` + lista (JSON/HTML) + seleção via JS |
| **Cadastro/Edição** | `Edit_*` | `TFDQuery` ligada aos campos, máquina de estados `dsBrowse/dsInsert/dsEdit`, validação espalhada em `OnExit` | `GET /editar/{id}` + `POST /salvar` com toda validação centralizada, dentro de uma transação |
| **Relatório** | `Rel_*` | Query fixa/parametrizada gerada como `TQuickRep`/`FastReport` | Endpoint que devolve HTML pronto para impressão ou PDF |

Além da classificação, a skill aplica correções que o código legado normalmente não tem:

- **SQL sempre parametrizado** na conversão, mesmo quando o Delphi original concatenava string diretamente — isso é tratado como correção de segurança obrigatória, não como opção.
- **Sessão e permissão resolvidas pelo mecanismo real do projeto Go de destino**, nunca replicando uma variável global de formulário (`fMain.TipoUsuario` ou equivalente).
- **Código comentado ou morto é sinalizado, nunca traduzido** como se fosse comportamento vivo — muitos formulários legados acumulam blocos inteiros comentados e versões duplicadas do mesmo handler.
- **Nunca inventa dado pessoal (nem fictício) em prévias.** Telas de consulta de pessoas são convertidas mostrando só identificador e metadado não sensível — nunca nome, telefone, e-mail ou documento de exemplo, mesmo fabricado. O mesmo cuidado vale para qualquer detalhe do domínio de negócio de um cliente real (nomes de tabela, de campo ou de regra reconhecíveis) que não deveria aparecer em exemplos ou documentação pública.
- **Nunca assume a estrutura do projeto de destino**: antes de gerar qualquer código, a skill lê módulos Go já existentes no projeto para copiar convenções reais (nomes de pacote, formato de handler, sistema de design/CSS já usado), em vez de inventar um padrão genérico.

## O que ela **não** faz

- Não é um transpilador 100% automático. Telas de cadastro complexas (várias dezenas de campos, regras de negócio específicas do domínio) sempre geram um relatório de decisões que precisam de revisão humana.
- Não commita nada automaticamente — todo resultado é tratado como rascunho.
- Não gera testes de carga, migração de dados históricos, nem lida com componentes visuais de terceiros (grids, combos customizados) além de mapear sua função para o equivalente HTML mais próximo.

## Instalação

Copie o arquivo da skill para o diretório de skills do Claude Code:

**Disponível em qualquer projeto (nível de usuário):**
```bash
mkdir -p ~/.claude/skills/delphi-to-go
cp skills/delphi-to-go/SKILL.md ~/.claude/skills/delphi-to-go/SKILL.md
```

**Ou só num projeto específico:**
```bash
mkdir -p .claude/skills/delphi-to-go
cp skills/delphi-to-go/SKILL.md .claude/skills/delphi-to-go/SKILL.md
```

Reinicie a sessão do Claude Code (a skill é descoberta no início da sessão).

## Uso

Peça diretamente, apontando para o arquivo Delphi:

```
/delphi-to-go converta apps/legacy/Cns_Cliente.pas (com o .dfm) para uma tela Go equivalente
```

Ou em linguagem natural, sem o comando — o Claude Code reconhece o pedido pela descrição da skill:

```
Converte esse formulário Delphi (Edit_Cliente.pas + .dfm) para uma tela Go no nosso padrão atual
```

A skill vai, nessa ordem: (1) inspecionar as convenções do projeto Go de destino, (2) ler o par `.pas`/`.dfm` e identificar o arquétipo, (3) converter a lógica com SQL parametrizado e permissão pelo mecanismo real de sessão do projeto, (4) sinalizar código morto/comentado e decisões de negócio que exigem confirmação, e (5) entregar o handler Go, o template da tela e um relatório de migração — tudo como rascunho para revisão.

## Exemplo

Um formulário de busca `Cns_Cliente.pas` (arquétipo Consulta/Busca) com campos "Nome", "Cidade" e "Status", uma grade de resultado e um botão "Selecionar", vira:

- `GET /api/clientes/buscar?nome=&cidade=&status=` com SQL parametrizado.
- Uma lista de resultados (tabela ou cards, conforme o padrão visual já usado no projeto de destino) com seleção equivalente ao antigo `ModalResult := mrOk`.
- Nenhum dado pessoal de exemplo — só identificador e status, mesmo em dados fictícios de demonstração.

## Contribuindo

Relatos de conversões que não se encaixaram bem nos três arquétipos, ou padrões Delphi comuns que a skill ainda não reconhece, são bem-vindos via issue.

## Precisa de ajuda com uma migração real?

Esta skill resolve boa parte do trabalho mecânico, mas sistemas Delphi legados de verdade sempre têm regras de negócio específicas que exigem decisão humana. Se você tem um sistema Delphi/VCL/Firebird real para migrar e quer ajuda além do que a skill entrega sozinha, abra uma [issue](../../issues) descrevendo o cenário ou entre em contato pelo perfil [@jorgesaba-dev](https://github.com/jorgesaba-dev).

## Licença

MIT — veja [LICENSE](LICENSE).
