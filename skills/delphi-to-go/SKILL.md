---
name: delphi-to-go
description: Converte units e telas legadas de Delphi (.pas/.dfm, VCL, FireDAC/FIBPlus sobre Firebird) para Go idiomático e telas web, usando leitura semântica assistida por IA em vez de transpilação mecânica. Use quando o pedido envolver migrar, portar ou converter código ou telas Delphi (Object Pascal) para Go, ou analisar um par .pas/.dfm.
---

# Delphi → Go

Esta skill não é um transpilador de sintaxe. Delphi (VCL, formulários `.dfm`, `TDataSet`/`ModalResult`) e Go (sem framework de UI, sem classes/herança) são paradigmas incompatíveis demais para que uma tradução mecânica produza algo útil — principalmente na camada de tela. O valor real está em entender a *intenção* de cada handler de evento e recriá-la como um fluxo web equivalente, não linha por linha.

Isso significa: sem IA, dá para automatizar apenas a extração de metadados (lista de campos, queries SQL, nomes de botões) — não a tela em si. Com IA (esta skill), o handler de evento é lido, entendido e reescrito.

## Passo 0 — Antes de tocar em código Delphi

Descubra as convenções do projeto Go de destino antes de gerar qualquer arquivo:

1. Leia um ou dois módulos Go já existentes e equivalentes (ex.: `modules/<algo-parecido>/*.go`) para copiar o padrão de nome de pacote, injeção de conexão/tenant, formato de handler HTTP e como sessão/permissão são lidas do contexto.
2. Verifique se a tabela do banco referenciada no `.pas` **já existe** no schema do projeto de destino, consultando via o script oficial de acesso ao banco do projeto (não assuma que precisa criar a tabela — o Delphi e o Go costumam apontar para o mesmo banco).
3. Verifique se já existe um validador equivalente no projeto Go (ex. CPF, telefone) antes de escrever um novo — reaproveite.
4. Ache o sistema de design/CSS real já usado nas telas web do projeto de destino (tokens de cor, tipografia e, principalmente, o **componente de lista/card já existente**, se houver) e use exatamente isso. Uma tela de resultado de busca no Delphi (`TDBGrid`) não precisa virar uma tabela HTML — se o projeto já lista registros em cards, a conversão deve usar os mesmos cards. Gerar uma prévia num estilo genérico que não existe em lugar nenhum do projeto real é o erro mais fácil de cometer aqui: sempre confirme o tema visual real antes de montar a tela, não invente um.

Nunca gere código assumindo uma estrutura de projeto genérica: adapte-se ao que já existe.

## Dados sensíveis — nunca exibir dado pessoal, nem fictício, em prévia

Muitas telas Delphi legadas são consultas de pessoas (clientes, contatos, cadastros). Ao montar qualquer prévia/mockup dessas telas — mesmo só para mostrar o layout ao usuário, sem nenhum banco real conectado — **não invente nomes, telefones, e-mails ou documentos fictícios que pareçam reais**. Dado fabricado, numa tela cujo propósito é consulta de pessoas, é o tipo errado de exemplo a mostrar em qualquer contexto de proteção de dados (LGPD, GDPR ou equivalente). Use o próprio identificador interno do sistema (ID/número de controle) como referência do registro na prévia, e mostre só metadado não identificável (status, categoria, tempo de cadastro) — nunca nome, contato ou documento, nem de mentira.

O mesmo cuidado vale para qualquer detalhe do domínio de negócio do cliente real cujo código está sendo convertido: nomes de tabelas, de campos ou de regras que sejam específicos e reconhecíveis daquele cliente não devem aparecer em exemplos, documentação ou prévias que serão compartilhados fora do projeto dele.

## Passo 1 — Ler o par `.pas` / `.dfm`

- O `.dfm` é a árvore de componentes (`object X: TClasse ... end`), com posição (`Left`/`Top`/`Width`/`Height` — geralmente irrelevante para a versão web, exceto para inferir agrupamento visual), `Caption`/`Text` (vira label ou placeholder) e vínculos de evento (`OnClick = X`, aponta para o handler no `.pas`).
- O `.pas` é o código por trás da tela: cada `procedure TfNome.XxxClick` é a lógica real de um botão ou campo.
- Descubra o "arquétipo" da tela antes de converter — isso muda a estrutura do resultado:

### Arquétipo 1 — Consulta/Busca (`Cns_*`, `Busca*`)
Padrão: campos de filtro → botão "Consultar" monta SQL dinamicamente (`SQL.Add`) → grade (`TDBGrid`) lista o resultado → duplo-clique ou "Selecionar" devolve um ID (`ModalResult := mrOk`, campo público do tipo `NRP_ISN*`) para o formulário chamador.
Vira: endpoint `GET /api/.../buscar?filtro=...` que devolve uma lista (JSON ou um trecho de HTML para uma tabela ou lista de cards), chamado por um modal/typeahead na tela que abriu a busca. **Nunca** transforme o botão "Selecionar" em algo modal nativo — é uma seleção que devolve um ID e fecha um diálogo web (ou preenche um campo via JavaScript).

### Arquétipo 2 — Cadastro/Edição (`Edit_*`)
Padrão: uma consulta (`TFDQuery`) ligada diretamente aos campos (`TDBEdit`/`TSF_ComboBox`, data binding); botões Novo/Gravar/Cancelar/Excluir/Sair controlam um estado (`dsBrowse`/`dsInsert`/`dsEdit`) via `DS_XxxStateChange`, habilitando ou desabilitando campos e botões; validações ficam espalhadas em eventos `OnExit` de cada campo e, muitas vezes, repetidas de novo no botão Gravar antes do `Post`/`ApplyUpdates`.
Vira: **não** replique a máquina de estados de widgets. Colapse em:
  - Um `GET /editar/{id}` (ou uma rota vazia para "novo") que carrega os dados e renderiza o formulário.
  - Um `POST /salvar` que roda **todas** as validações de uma vez no servidor — inclusive as que hoje estão espalhadas em `OnExit` de cada campo e as que estão repetidas (às vezes de forma incompleta) dentro do botão Gravar; unifique tudo numa única função de validação.
  - Envolva o insert/update numa transação (`tx.Begin`/`Commit`/`Rollback`), preservando o mesmo cuidado que o código Delphi original já tinha ao abrir e fechar transação.
  - Os valores padrão definidos num evento `AfterInsert` viram valores padrão do formulário ou da struct Go, não lógica condicional espalhada.

### Arquétipo 3 — Relatório (`Rel_*`)
Padrão: query fixa ou parametrizada, gerada como relatório (`TQuickRep`/`FastReport`) para impressão.
Vira: endpoint que devolve HTML pronto para impressão pelo navegador (`window.print()`) ou PDF gerado no servidor — decida com o usuário antes qual dos dois, não assuma PDF automaticamente (é uma dependência extra).

## Passo 2 — Converter as queries (atenção: aqui mora um bug de segurança comum)

O código legado costuma concatenar SQL como string (`SQL.Add('...' + Edit1.Text)`) e, às vezes, usa `ParamByName` corretamente em outros pontos do mesmo arquivo. **Ao converter, sempre parametrize** (`$1`/`?`), mesmo quando o original concatenava direto — isso não é fidelidade ao original, é uma correção de segurança que deve ser feita durante a conversão, não depois. Preserve a sintaxe específica do banco de origem quando for proposital (ex. `FIRST` no Firebird), mas sempre via placeholder, nunca por concatenação.

## Passo 3 — Permissões e sessão

Handlers legados costumam checar o tipo de usuário e permissões direto de uma variável global do formulário principal. **Nunca** replique variável global — resolva a sessão e a permissão reais do usuário pelo mecanismo do projeto Go de destino (sessão, token, contexto de tenant). Se o projeto de destino ainda não tiver um conceito de permissão equivalente, pare e pergunte ao usuário em vez de inventar um novo.

## Passo 4 — Código morto e comentado

Formulários legados acumulam blocos inteiros comentados, handlers praticamente vazios (por exemplo, um botão "Excluir" cujo corpo real está todo comentado, sobrando só uma mensagem de erro) e duplicação de validação entre versões sucessivas do mesmo handler. **Não traduza código comentado como se fosse comportamento vivo.** Liste esses trechos à parte no relatório de migração como "não fica claro se é intencional — confirmar com o usuário"; nunca decida sozinho se deveriam ou não funcionar.

## Saída esperada

Para cada tela convertida, produza:

1. O handler Go, seguindo a convenção já detectada no passo 0.
2. O template/HTML da tela, reaproveitando o CSS e o layout já usados no projeto de destino — nunca um estilo novo inventado na hora.
3. Um relatório de migração (arquivo ou seção na resposta) listando: campos e regras que exigiram uma decisão de produto (regra de negócio real, cujo equivalente atual deve ser confirmado com o usuário antes de portar), código morto encontrado, e qualquer chamada a serviço externo embutida na tela que precisa de uma chave ou endpoint próprio no ambiente Go.

Trate o resultado sempre como um rascunho para revisão humana — não commite automaticamente.

## Exemplo ilustrativo (fictício)

Um formulário de busca chamado `Cns_Cliente.pas`/`.dfm` (arquétipo 1) com campos "Nome", "Cidade" e "Status", uma grade de resultado e um botão "Selecionar", vira:

- `GET /api/clientes/buscar?nome=&cidade=&status=` — SQL parametrizado, nunca concatenado.
- Uma lista (tabela ou cards, conforme o padrão visual já usado no projeto) mostrando os resultados, com um clique/duplo-clique equivalente ao antigo `ModalResult := mrOk`.
- Nenhum dado pessoal de exemplo na prévia — só identificador e status, mesmo em dados fictícios de demonstração.
