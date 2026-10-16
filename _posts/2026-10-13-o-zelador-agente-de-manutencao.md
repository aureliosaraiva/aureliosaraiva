---
title: "O Zelador: como a Creditas colocou um agente de IA para cuidar do que ninguém prioriza"
description: "Manutenção não fica para trás por falta de consciência: ela disputa capacity com o roadmap, e perde. Construí um agente que opera fora dessa disputa. Em quatro semanas ele entregou 224 PRs, com 99% do esforço em teste, correção, refatoração e segurança, e levou a cobertura de um serviço de 62% para 96,7%."
tags: [ia, agentes-de-ia, engenharia, manutencao, automacao]
---


*Publicado originalmente no [LinkedIn](https://www.linkedin.com/pulse/o-zelador-como-creditas-colocou-um-agente-de-ia-para-cuidar-saraiva-8bxrf/) em 13/10/2026.*

Todo time de tecnologia tem débito técnico, e todo mundo sabe que ele existe. Vulnerabilidade nova aparece toda semana. Biblioteca desatualizada acumula. Refatoração adiada vira comentário em code review que ninguém volta para ler. Cenário crítico sem teste fica sem teste até o dia em que quebra em produção.

Nada disso acontece por ignorância. Acontece porque manutenção disputa capacity com iniciativa de negócio, e perde, quase sempre, e com razão aparente, porque o custo de não fazer só aparece depois. É uma questão de priorização, não de compreensão.

O que muda a escala do problema é o crescimento. Com 10 serviços, dá para carregar o débito técnico na cabeça de quem os mantém. Com 500, não dá. O backlog de manutenção deixa de ser uma lista e vira um passivo distribuído, invisível, que só ganha prioridade quando já virou incidente.

**Minha tese é que esse problema não se resolve priorizando melhor. Ele se resolve tirando o trabalho da fila onde ele sempre perde.** Um agente autônomo não disputa sprint, não é realocado para a entrega do trimestre e não precisa convencer ninguém de que atualizar o Node ou Kotlin é mais importante que a próxima feature. Ele opera fora dessa disputa.

Foi essa aposta que virou o Zelador, o agente de manutenção que roda em alguns repositórios da Creditas. O nome é o que ele faz: zelador é quem mantém o condomínio funcionando, conserta o que quebra e melhora o que ninguém repara, sem que ninguém precise pedir.

Abaixo está como ele foi construído e o que quatro semanas de funcionamento mostraram.

## O problema estava claro; o como, não

Eu sabia o que precisava ser feito. A dificuldade inteira estava na execução, e ela se resumia a três restrições que eliminaram quase todas as alternativas.

**A primeira era escala sem infraestrutura nova.** Se a ideia é atender a empresa toda, não pode depender da máquina do desenvolvedor ligada rodando agente. E não pode ser trabalho local: um agente que só existe no notebook de quem o instalou é uma ferramenta pessoal, não uma capacidade de plataforma.

**A segunda era o custo de operar isso.** Imagine 50 pull requests sendo produzidos ao mesmo tempo, cada um com um build de Kotlin e Gradle rodando junto. Que tamanho tem essa máquina? Quanta CPU e memória eu reservo, e para qual pico? Dava para escalar pods sob demanda, claro. Mas gerenciar essa infraestrutura era exatamente o trabalho que eu não queria assumir no começo de um projeto cujo valor ainda está por provar.

**A terceira era não ficar preso a um fornecedor.** Eu queria liberdade para trocar de modelo sem reescrever a plataforma.

Foi procurando por essas três restrições ao mesmo tempo que encontrei o [OpenHands](https://github.com/All-Hands-AI/OpenHands), um harness de desenvolvimento open source. Ele conecta em AWS Bedrock, que é a infraestrutura de modelo que a Creditas já usa, e aceita proxy reverso para um LiteLLM customizado quando for necessário trocar de provedor. Resolve a terceira restrição por padrão.

E resolve a primeira e a segunda por um detalhe que parecia menor: o OpenHands roda em GitHub Actions. Isso significa que a plataforma não tem servidor. Cada execução do agente é um job efêmero, que nasce quando há trabalho e morre quando termina. Não há máquina ociosa esperando pico, não há dimensionamento a acertar, não há cluster a manter. E quando o volume justificar infraestrutura própria, o mesmo harness roda em servidor dedicado sem reescrita.

Para quem adota, a consequência prática é que não se instala nada. O engenheiro interage pelo GitHub: abre issue, aplica label, revisa o PR que o agente abriu.

## Um agente de manutenção precisa fabricar o próprio backlog

Colocar um agente para executar tarefas de manutenção resolve metade do problema. A outra metade é que ninguém escreveu essas tarefas.

Esse é o ponto que eu subestimei no começo. Catalogar o que precisa ser feito é tão difícil quanto fazer. Se o Zelador dependesse de alguém abrir as issues, ele herdaria exatamente a fila que já não é priorizada, e o projeto inteiro teria sido um jeito caro de não resolver nada.

Por isso uma premissa virou requisito: **o agente alimenta o próprio backlog.** Ele consome sinais que já existem na empresa e os transforma em trabalho executável.

| Fonte | O que ela entrega |
|---|---|
| Código-fonte | Achados de teste faltando, refatoração pendente, problema estrutural |
| Dependabot | Biblioteca desatualizada e dependência com vulnerabilidade conhecida |
| Semgrep | Findings de segurança da análise estática |
| Datadog | Erro recorrente em log e trace de produção, correlacionado com o código |

A fonte do Datadog é a que menos se parece com as outras, e é a mais interessante: ela não olha para o código, olha para o comportamento em produção. O agente pega o erro que está acontecendo agora, correlaciona com o trecho de código que o causa e abre issue quando a correção é local e mecânica. É o caminho mais curto entre um erro que ninguém abriu ticket e um PR que o conserta.

## O funil, e por que a máquina de estados vive nas labels

O Zelador é um GitHub App mais um conjunto de workflows reutilizáveis. Não há banco de dados de estado, não há painel de controle próprio, não há um segundo lugar para consultar. **O estado de cada trabalho vive na label da issue**, e a transição entre etapas acontece por label.

Essa decisão não é estética. Ela significa que qualquer pessoa entende o que o agente está fazendo olhando para o GitHub, com a interface que já usa, sem aprender ferramenta nova. E significa que intervir é aplicar uma label.

O fluxo tem cinco momentos:

**Descoberta.** Um scan somente-leitura do repositório, mais a leitura do Datadog quando habilitada, abre issues para os achados de alto sinal. Há teto diário para não inundar ninguém.

**Refino.** O agente investiga o repositório e reescreve a issue como uma spec pronta: contexto, arquivos afetados, passos de execução ordenados, critérios de aceite, o que está fora de escopo. Achado que a triagem julga inválido é fechado com o motivo escrito, em vez de ficar apodrecendo no backlog.

O plano ficar dentro da issue é deliberado, e é a parte do desenho de que mais gosto. Ele é revisável por uma pessoa antes de existir uma linha de código. Em repositório crítico, é exatamente aí que a liberação humana entra. E ele sobrevive ao contexto do agente de desenvolvimento, que relê a issue em vez de confiar na própria memória.

**Orquestração.** Um scheduler promove issues prontas para execução respeitando prioridade, urgência e ordem de chegada, e segurando o limite de trabalho em curso. Ele também reconcilia o que travou: issue marcada como em execução, sem nada rodando e sem PR aberto, volta para a fila.

**Desenvolvimento.** O agente executa o plano tarefa por tarefa, na ordem dada. Ele não planeja por conta própria e desvio precisa ser declarado. Cada tarefa passa por um ciclo de implementar, verificar com o comando de build do próprio repositório, revisar e corrigir, com teto de tentativas. Uma issue rende um commit e um PR, por mais tarefas que o plano tenha.

**Convergência.** Depois do PR aberto, dois fluxos reagem sozinhos: um conserta CI vermelho, outro aplica sugestões de review e responde os comentários. Se a branch protection for satisfeita, o merge acontece pelo mecanismo nativo do GitHub.

Um detalhe que só aparece quando se opera: a execução é em série por repositório, de propósito, para não estourar o rate limit do Bedrock. O limite de PRs abertos e o paralelismo de execução são dimensões diferentes, e confundir as duas é o erro mais fácil de cometer aqui.

## O que o agente nunca faz

Autonomia sem limite é um problema, não uma feature. Algumas proteções são parte do desenho e não dependem de configuração de quem adota:

- **Não edita `.github/`.** Nem workflow, nem action, nem `CODEOWNERS`. O agente não mexe nas regras que o governam.
- **Não faz commit nem push, e não chama a API do GitHub.** O token fica escondido dele. Quem escreve no repositório é o pipeline determinístico, em Python e Bash, que não é o agente.
- **Não desabilita nem pula teste ou lint**, e não afrouxa a configuração de build. O caminho mais curto para o verde está bloqueado.
- **Respeita os paths protegidos** que cada repositório declara.
- **Tem teto de iterações e de budget por tarefa.** Execução cara é interrompida, não descoberta na fatura.

E todo trabalho é rastreável: os PRs carregam label de proveniência e os commits trazem os trailers que declaram que aquilo foi feito por IA, com qual ferramenta e com qual modelo.

## Quatro semanas de operação

Entre 22 de julho e 19 de agosto de 2026, o Zelador mergeou **224 pull requests em diversos repositórios**, de sete times diferentes.

**Volume não é o que importa aqui**, e o motivo é simples. Mais código é passivo antes de ser ativo: o que se acrescenta também se mantém. Contagem de PR é a métrica mais fácil de inflar e a que menos diz.

O que importa é a composição. **99% desses PRs são teste, correção de bug, refatoração ou segurança.** Manutenção, quase por inteiro. Feature é resíduo, e é assim que tem que ser.

### Sobre comparar o agente com a engenharia

Essa comparação pede cuidado, e o cuidado é o seguinte: **não é uma comparação de produtividade, e não faria sentido que fosse.**

Um agente não participa de reunião, não faz descoberta de produto, não desenha solução com stakeholder, não negocia escopo e não carrega contexto de negócio. Ele executa uma fatia estreita de trabalho, escolhida de propósito para ser estreita. Ler o número de PRs dele ao lado do de uma pessoa compara coisas que não são a mesma.

A diferença real é de **foco**, não de velocidade. No mesmo período, pouco menos da metade do trabalho da engenharia foi manutenção, o que é exatamente o esperado de quem entrega roadmap e precisa escolher. A do agente foi quase toda. Ele não está acelerando o que já era feito; está cobrindo a faixa que a fila de prioridade não alcança.

Teste é o exemplo mais claro. É o trabalho que mais sofre com a disputa de capacity, porque o custo de não escrever aparece meses depois e em outro lugar. Que o agente escreva teste em quase todo repositório onde entra não é mérito dele sobre ninguém: é o efeito de existir uma fila onde esse trabalho não compete com prazo.

### Onde a cadeia fecha ponta a ponta

Em um dos serviços dá para medir do esforço ao resultado. É o repositório em que o agente mais atuou, e nele **a cobertura de teste saiu de 62% para 96,7% em quatro semanas**, sobre uma suíte que antes era quase inexistente.

Isso sai da faixa de melhoria incremental. É criação de suíte de teste onde praticamente não havia uma. Nos demais repositórios o movimento é incremental, e em alguns não há instrumentação de cobertura para dizer coisa alguma.

### Segurança

Nove PRs de segurança no período, todos rotulados e rastreáveis. A maior parte é atualização de dependência vulnerável, uma delas fechando dois CVEs nomeados. O restante é higiene: tratamento de erro e de log.

É uma categoria que, no período, não vinha sendo endereçada por nenhum outro caminho.

## O gargalo mudou de lugar

O dado que mais me interessa operacionalmente não é sobre o agente. É sobre nós.

O tempo entre a revisão e o merge dos PRs do agente ficou em 96 minutos, cerca de seis vezes o de um PR comum. Novos revisores entraram no período e a degradação parou, mas o patamar continua alto.

A leitura é direta. **A geração deixou de ser o limite; a revisão virou o limite.** A capacidade já existe e já foi paga. O que a destrava é ampliar o time que revisa, não aumentar o volume que o agente produz.

O efeito de segunda ordem é este: um agente que produz mais do que a empresa consegue revisar não está entregando valor, está transferindo trabalho de um lugar para outro. O número de PRs em aberto é o termômetro disso, e é por isso que existe um teto de trabalho em curso no desenho.

## Conclusão

O Zelador não é uma aposta de que IA escreve código melhor que gente. É uma aposta de que existe uma categoria de trabalho que toda empresa reconhece como necessária e nunca prioriza, e que essa categoria não precisa competir por atenção humana para ser feita.

O que sustenta o caso não é volume. É que a quase totalidade do que ele entregou é teste, correção, refatoração e segurança, a faixa que a fila não alcançava, e que, onde havia instrumentação para medir, o efeito apareceu.

Os próximos passos são de escala, não de invenção. Expandir o número de aplicações cobertas. Continuar aprendendo com o que cada repositório novo ensina, porque nenhum deles se parece com o anterior. E conectar novos estímulos externos: fontes de sinal que ainda não alimentam o backlog dele e que podem virar issue sem ninguém escrever uma.

Expandir e revisar andam juntos, pelo motivo da seção anterior. Capacidade de geração que a revisão não acompanha não vira entrega, vira fila.

Manutenção não some do backlog porque alguém finalmente entendeu que ela importa. Ela some quando deixa de depender disso.
