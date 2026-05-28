# Parte 1

## Bloco 1 — O que é TypeScript

### O problema

Considere uma função escrita para calcular quantos dias faltam até o vencimento de uma tarefa. A função recebe o número de dias e devolve uma mensagem. Em algum ponto do código, alguém — talvez o próprio autor, em um momento de distração — chamou essa função passando a string `"7"` em vez do número `7`. O JavaScript não reclamou. Executou. Produziu um resultado ligeiramente diferente do esperado. Talvez uma concatenação em vez de uma soma. Talvez um `NaN`. O bug existia no código antes de executar — mas o JavaScript só o revelou em *runtime* (tempo de execução).

TypeScript resolve isso antes da execução.

**O que é TypeScript, de fato**. TypeScript é um *superset* estrito de JavaScript. A palavra *superset* tem um significado preciso aqui: todo arquivo JavaScript válido é, ao mesmo tempo, um arquivo TypeScript válido. TypeScript não é uma linguagem diferente — é JavaScript com uma camada adicional opcional de anotações de tipo. Qualquer arquivo `.js` pode ser renomeado para `.ts` e continuará funcionando sem nenhuma alteração.

O que TypeScript adiciona ao JavaScript é um verificador estático. "Estático" significa: antes de executar. O verificador analisa o código como texto, examina os tipos declarados ou inferidos de cada variável e cada função, e sinaliza contradições antes que o programa rode. Se uma função está anotada para receber `number` e é chamada com `string`, o TypeScript sinalizará o erro antes de qualquer execução.

**Contexto de aplicação**. Em projetos profissionais com múltiplos arquivos e múltiplas pessoas, o verificador estático do TypeScript é o que impede que contratos entre partes do código sejam quebrados silenciosamente. Ao definir que uma função retorna um objeto com as propriedades `id` e `nome`, todo ponto de consumo dessa função sabe exatamente o que esperar — e o verificador garante que ninguém quebre esse contrato sem o código reclamar.

**Por que TypeScript existe**. JavaScript foi criado em 1995 para scripts pequenos em páginas web. Conforme as aplicações cresceram em tamanho e complexidade, a ausência de tipos passou de irrelevante a problemática. TypeScript nasceu em 2012, mantido pela Microsoft, para trazer verificação estática ao ecossistema JavaScript sem exigir abandono da linguagem. Hoje é o padrão de fato em desenvolvimento profissional de aplicações JavaScript de qualquer tamanho.

**O que TypeScript não é**. TypeScript não é uma linguagem de execução. Antes de rodar, as anotações de tipo são removidas e o que sobra é JavaScript puro. Nenhum navegador, nenhum servidor executa TypeScript diretamente — exceto, como será visto no próximo bloco, com um recurso específico do Node.js 24 que dispensa essa etapa manualmente.

## Bloco 2 — *Type Stripping* no Node.js 24

Existe uma pergunta prática imediata que surge ao conhecer TypeScript: se o que executa é sempre JavaScript, como se roda um arquivo `.ts`?

A resposta tradicional envolvia uma etapa extra chamada compilação: executava-se um comando que lia o arquivo `.ts`, removia todas as anotações de tipo, e produzia um arquivo `.js` equivalente. Só então o `.js` era executado. Era um passo obrigatório entre escrever e rodar.

O Node.js 24 eliminou esse passo para o fluxo de desenvolvimento.

**O que é *type stripping***. *Type stripping* (remoção de tipos) é o processo de remover as anotações de tipo de um arquivo TypeScript antes de executá-lo. O resultado é JavaScript puro — funcionalmente idêntico ao que se teria se as anotações nunca tivessem sido escritas. O Node.js 24 realiza esse processo nativamente, sem nenhuma ferramenta intermediária e nenhuma flag adicional.

**Exemplo concreto**. Considere o seguinte conteúdo em um arquivo `playground.ts`:

```typescript
function greet(name: string): string {
  return `Hello, ${name}!`;
};
```

Ao executar `node playground.ts` no Node.js 24, o que o Node.js faz internamente é ler esse arquivo, remover `: string` das duas ocorrências, e executar o JavaScript resultante:

```javascript
function greet(name) {
  return `Hello, ${name}!`;
};
```

Esse JavaScript intermediário não é visível nem gravado em disco. O Node.js processa e executa diretamente.

**Relevância para este módulo**. No módulo anterior, o Node.js era usado para executar arquivos `.js`. O fluxo deste módulo é o mesmo — os arquivos agora têm extensão `.ts`. O comando `node file.ts` funciona diretamente. Nenhum passo extra é necessário.

**O que o *type stripping* não faz**. *Type stripping* remove as anotações e executa. Ele não verifica se os tipos estão corretos. Se uma função está anotada para receber `number` mas é chamada com `string`, o Node.js executará mesmo assim — da mesma forma que JavaScript executaria. A verificação dos tipos é responsabilidade de um comando separado, o `tsc --noEmit`, que será apresentado no Bloco 8 desta parte.

Isso significa que execução e verificação são duas operações distintas no TypeScript. Executar com `node` roda o código. Verificar com `tsc --noEmit` analisa os tipos. As duas operações são realizadas em momentos diferentes, com propósitos diferentes.

**Contrafactual**. Sem o *type stripping* nativo do Node.js 24, seria necessária uma ferramenta intermediária — historicamente `ts-node` ou `tsx` — para executar arquivos `.ts` diretamente. Essas ferramentas existem e funcionam, mas adicionam uma dependência ao projeto que o Node.js 24 tornou desnecessária. Por isso o curso proíbe explicitamente o uso de `ts-node` e `tsx`: o recurso nativo é suficiente e mais simples.

## Bloco 3 — Instalação Global Temporária

Antes de escrever qualquer código TypeScript, é necessária a instalação de uma ferramenta: o compilador `tsc`.

O `tsc` é o compilador oficial do TypeScript, mantido pela Microsoft. É ele que executa o comando `tsc --noEmit` responsável pela verificação de tipos. Sem ele instalado, esse comando não estará disponível.

**Instalação global**. Nesta aula, o TypeScript será instalado globalmente na máquina. "Global" significa que o comando `tsc` ficará disponível em qualquer diretório do sistema, sem a necessidade de estar dentro de um projeto específico.

O comando para instalação é:

```shell
npm install -g typescript
```

**O que este comando faz**: instala o pacote `typescript` no diretório global do Node.js na máquina, tornando o comando `tsc` disponível em qualquer diretório do sistema.

**Por que ele existe**: sem o compilador instalado, não há como executar a verificação de tipos com `tsc --noEmit`. O npm, por padrão, instala dependências dentro do projeto corrente; a flag `-g` redireciona essa instalação para o escopo global.

**Decisão técnica**: a instalação global é adotada exclusivamente nesta aula porque não há projeto configurado com `package.json`. A partir da Aula 2, o TypeScript será instalado localmente por projeto — o que garante que qualquer pessoa que clonar o repositório obtenha exatamente a mesma versão, sem depender do que está instalado globalmente em cada máquina.

**Onde é usado**: o comando `tsc --noEmit` executado no Bloco 8 desta aula depende diretamente desta instalação. Sem ela, o terminal retorna erro de comando não encontrado.

**Contrafactual**: sem a flag `-g` e sem um `package.json` no diretório corrente, o npm não teria onde registrar a dependência e a instalação falharia. Se a instalação fosse omitida inteiramente, o Bloco 8 desta parte falharia com comando não encontrado.

Ao concluir, execute:

```shell
tsc --version
```

O terminal exibirá a versão instalada, algo como `Version 6.x.x`.

Agora crie o diretório de experimentação:

```shell
mkdir typescript-playground
```

Em seguida, abra esse diretório no VSCode:

```shell
code typescript-playground
```

Dentro do VSCode, crie um arquivo chamado `playground.ts` na raiz desse diretório.

## Bloco 4 — Tipos Primitivos

Os tipos primitivos do JavaScript são reconhecidos pelo TypeScript — que adiciona a capacidade de declarar explicitamente qual tipo uma variável ou parâmetro deve conter.

Os cinco tipos primitivos com os quais o TypeScript trabalha são: `string`, `number`, `boolean`, `null` e `undefined`.

Esses tipos já eram usados em JavaScript sem necessidade de nomeá-los. Em TypeScript, passam a ser nomeados quando necessário.

**A sintaxe de anotação**. Para anotar o tipo de uma variável, escrevem-se dois-pontos após o nome da variável, seguido do tipo:

```typescript
let name: string = 'Alex';
let age: number = 30;
let active: boolean = true;
```

O que está acontecendo em cada linha:

A anotação `: string` na primeira linha é uma declaração de contrato. Ela diz ao TypeScript: "esta variável deve conter exclusivamente valores do tipo `string`". A partir do momento em que essa anotação existe, o verificador passa a monitorar todo lugar onde `name` é usado ou atribuído — e sinalizará erro se for atribuído um `number` ou `boolean` a ela.

A anotação `: number` na segunda linha faz o mesmo para números. Tanto inteiros quanto decimais são `number` em TypeScript — não existe `int` ou `float` como tipos separados, assim como no JavaScript.

A anotação `: boolean` na terceira linha cobre `true` e `false`.

**Por que essas anotações existem**. Uma variável declarada com `let` e valor inicial já tem seu tipo evidente — o TypeScript consegue inferir que `let name = 'Alex';` é uma `string` sem que se escreva `: string`. Mas há situações onde a inferência não é possível: quando uma variável é declarada sem valor inicial, o TypeScript não tem como saber o que virá depois. É nesses casos que a anotação se torna necessária, não opcional:

```typescript
let description: string;
description = 'Here contains a description.';
```

Sem a anotação `: string` na primeira linha, o TypeScript assumiria o tipo `any` para `description` — o que desabilita a verificação. Com a anotação, o contrato está declarado desde a linha de declaração, antes mesmo de qualquer valor ser atribuído.

**`null` e `undefined`**. Esses dois também são tipos em TypeScript. Um valor pode ser anotado como `null` quando representa ausência intencional de valor, e `undefined` quando representa ausência de atribuição.

## Bloco 5 — Anotação em Variáveis

A sintaxe de anotação foi apresentada no Bloco 4. Este bloco trata de quando anotar e quando deixar o TypeScript inferir sozinho — escrever anotações em todo lugar não é a prática correta.

**Inferência**. Quando uma variável é declarada com valor inicial, o TypeScript examina esse valor e determina o tipo automaticamente, sem necessidade de escrita explícita:

```typescript
let name = 'Alex';
```

O TypeScript infere que `name` é `string` pelo fato de o valor inicial ser uma `string`. Tentar atribuir um número a `name` posteriormente produz erro do verificador — mesmo sem nenhuma anotação explícita. O contrato existe, só foi estabelecido pela inferência em vez de pela escrita direta.

**Por que a inferência é suficiente nesses casos**. A anotação explícita e a inferência produzem o mesmo resultado do verificador. Escrever `: string` quando o valor inicial já é uma `string` é redundante — é informar ao TypeScript algo que ele já sabe. A prática profissional é deixar a inferência trabalhar quando ela tem informação suficiente, e anotar explicitamente apenas quando ela não tem.

**Quando a anotação é necessária**. Há dois casos principais onde a inferência não tem informação suficiente e a anotação se torna necessária.

O primeiro é a declaração sem valor inicial:

```typescript
let description: string;
description = 'Here contains a description.';
```

Sem `: string`, o TypeScript não tem como saber o que virá nessa variável. Sem a anotação, ele atribui o tipo `any` — que desabilita a verificação. Com a anotação, o contrato está declarado desde o início.

O segundo caso é o parâmetro de função — mas esse tem bloco próprio, o Bloco 6. Por ora, o critério prático para variáveis é direto: se há valor inicial, deixe inferir. Se não há valor inicial, anote.

**Contrafactual**. Anotar em todo lugar torna o código mais verboso sem ganho real de segurança. Não anotar onde a inferência não alcança faz o TypeScript assumir `any` e a verificação deixa de funcionar para aquele valor — o pior dos dois mundos: TypeScript no projeto com parte do código sem verificação.

## Bloco 6 — Anotação em Parâmetros e Retorno de Função

Parâmetros de função são o caso mais importante de anotação obrigatória em TypeScript — e o mais diretamente conectado ao problema que abriu esta aula.

**Por que parâmetros não podem ser inferidos**. Quando uma variável é declarada com valor inicial, o TypeScript vê o valor e infere o tipo. Parâmetros de função não têm valor inicial visível no momento da declaração — o valor só chega quando a função é chamada, em outro lugar do código. O TypeScript não pode examinar todos os lugares onde a função será chamada para inferir o tipo do parâmetro. Por isso, parâmetros precisam ser anotados explicitamente.

Sem anotação, o TypeScript atribui `any` ao parâmetro — e `any` desabilita a verificação para aquele valor dentro da função inteira. A função passa a aceitar qualquer coisa silenciosamente, o que é exatamente o comportamento que TypeScript existe para eliminar.

**A sintaxe**. A anotação de parâmetro segue o mesmo padrão da variável — dois-pontos após o nome:

```typescript
function greet(name: string): string {
  return `Hello, ${name}!`;
};
```

Há duas anotações nessa linha. A primeira, `: string` após `name`, declara que esta função aceita exclusivamente strings nesse parâmetro. A segunda, `: string` após o parêntese de fechamento, declara o tipo do valor que a função retorna.

**A anotação de retorno**. O retorno de função pode ser inferido pelo TypeScript — ele examina o que a função retorna e determina o tipo. Mas anotar o retorno explicitamente tem uma vantagem prática: o verificador passa a confirmar que o que é retornado é compatível com o que foi prometido retornar. Se o retorno prometido é `: string` e em algum caminho da função é retornado um `number`, o verificador sinaliza o erro na própria função — não no lugar que a chamou.

**`void` como retorno**. Quando uma função não retorna nenhum valor, o tipo de retorno é `void`:

```typescript
function register(message: string): void {
  console.log(message);
};
```

`void` significa "esta função executa e não produz valor utilizável". É diferente de `undefined` — `void` é uma declaração de intenção, não um valor concreto.

**Contrafactual**. Omitir a anotação do parâmetro faz o TypeScript assumir `any` e a função passa a aceitar qualquer valor sem verificação. O erro que abriu esta aula — passar `"7"` onde se esperava `7` — voltaria a ser possível silenciosamente, mesmo com TypeScript no projeto.

## Bloco 7 — `any` e `unknown`

Os blocos anteriores trabalharam com tipos precisos: `string`, `number`, `boolean`. TypeScript tem dois tipos especiais que lidam com situações onde o tipo não é conhecido com antecedência. Eles parecem similares à primeira vista — ambos representam "tipo desconhecido" — mas têm comportamentos fundamentalmente diferentes, e a distinção entre eles é uma das decisões de segurança mais importantes no uso de TypeScript.

**`any`**. Quando uma variável tem tipo `any`, o TypeScript desabilita completamente a verificação para aquele valor. É possível atribuir qualquer coisa a ela, passá-la para qualquer função, acessar qualquer propriedade — o verificador não reclama.

```typescript
let value: any = 'value';
value = 0;
value = false;
value.anyProperty;
```

Nenhuma dessas linhas produz erro de tipo. O `any` é um buraco na verificação — um ponto cego onde o TypeScript simplesmente para de trabalhar.

**Por que `any` existe**. `any` existe principalmente para migrações. Ao pegar um projeto JavaScript existente e começar a converter para TypeScript, é inviável anotar tudo de uma vez. O `any` permite migrar gradualmente: converte-se arquivo por arquivo, e os valores ainda não anotados ficam temporariamente como `any` sem quebrar o projeto inteiro. É uma ferramenta de transição, não de uso permanente.

**O perigo do `any` como padrão**. Usar `any` livremente significa ter TypeScript no projeto sem verificação real. É o pior cenário possível: a complexidade adicional da linguagem sem o benefício que a justifica. No projeto desenvolvido neste módulo, `any` explícito é evitado. Quando aparecer, será sinalizado.

**`unknown`**. `unknown` também representa "tipo desconhecido" — mas com uma diferença crucial: antes de usar um valor `unknown`, é obrigatório verificar o que ele é. O TypeScript não permite operar sobre um `unknown` sem antes confirmar o tipo.

```typescript
const entry: unknown = 'value';

console.log(entry.toUpperCase());
```

Essa linha produz erro: `Object is of type 'unknown'`. O TypeScript recusa a chamada de `.toUpperCase()` porque não foi confirmado que `entry` é uma `string`. Para usar o valor, é necessária uma verificação explícita:

```typescript
const entry: unknown = 'value';

if (typeof entry === 'string') console.log(entry.toUpperCase());
```

Com o operador `typeof`, dentro do bloco `if`, o TypeScript sabe que `entry` é `string` — porque a verificação acabou de ocorrer. Esse processo de verificar o tipo antes de usar é chamado de *narrowing* (estreitamento de tipo), e é o mecanismo central de segurança do `unknown`.

**Contexto de aplicação de `unknown`**. `unknown` é o tipo correto para valores que chegam de fora do controle do código: o corpo de uma requisição HTTP, o resultado de um `JSON.parse`, dados lidos de um arquivo. O tipo da entrada não é conhecido com antecedência — pode ser qualquer coisa. `unknown` força a verificação antes do uso, o que é exatamente o comportamento correto para entrada externa.

**A distinção prática**. `any` diz "confie em mim, eu sei o que é isso" — e o TypeScript obedece sem questionar. `unknown` diz "eu não sei o que é isso ainda" — e o TypeScript exige prova antes de permitir o uso. Em código profissional, quando o tipo é desconhecido, a escolha correta é `unknown`.

## Bloco 8 — Verificação por `tsc --noEmit`

O comando `tsc --noEmit` foi utilizado nos blocos anteriores. Este bloco trata formalmente do que ele faz, por que existe nessa forma específica, e qual é o seu papel no fluxo de trabalho deste módulo.

**O que `tsc` faz normalmente**. O compilador TypeScript, invocado como `tsc file.ts`, tem dois comportamentos simultâneos: verifica os tipos do arquivo e produz um arquivo `.js` correspondente com as anotações removidas. O arquivo `.js` é o artefato de saída — o que seria entregue para execução em um ambiente que não suporta TypeScript nativamente.

**Por que `--noEmit`**. A flag `--noEmit` instrui o `tsc` a executar apenas a verificação de tipos, sem produzir nenhum arquivo de saída. O nome é literal: *no emit* — não emite arquivos.

No contexto deste módulo, `--noEmit` é o modo correto por dois motivos. Primeiro, o Node.js 24 já executa arquivos `.ts` diretamente via *type stripping* — não há necessidade de um `.js` gerado para executar. Segundo, produzir arquivos `.js` ao lado dos `.ts` criaria duplicatas desnecessárias no projeto, poluindo o diretório de trabalho.

**O que o comando reporta**. Quando há erros de tipo, `tsc --noEmit` lista cada erro com três informações: localização exata no formato `arquivo:linha:coluna`, código do erro no formato `TS` seguido de número, e descrição do problema em linguagem natural. Quando não há erros, o comando termina silenciosamente — nenhuma saída, código de saída zero. Silêncio é sucesso.

**Contrafactual**. Executar `tsc --noEmit` sem o TypeScript instalado faz o terminal retornar erro de comando não encontrado. Executar `tsc file.ts` sem a flag `--noEmit` faz o TypeScript produzir um arquivo `file.js` no mesmo diretório — o que neste módulo é desnecessário e indesejado.

**O fluxo de trabalho deste módulo**. Ao longo das 20 aulas, dois comandos serão usados para trabalhar com TypeScript:

`node file.ts` — para executar. O Node.js 24 faz o *type stripping* e roda o código.

`tsc --noEmit file.ts` — para verificar tipos. O compilador analisa os contratos e reporta violações.

A partir da Aula 2, quando o `tsconfig.json` for configurado, o comando de verificação simplifica para `tsc --noEmit` sem especificar arquivo — o `tsconfig.json` declara quais arquivos verificar. Por ora, o arquivo é especificado explicitamente.
