# Parte 1

## Bloco 1 — Anotação de Parâmetros e Retorno

### Mudança de foco

Na aula anterior, o foco estava nos tipos como descrição de dados: primitivos, objetos, arrays, tuplas, uniões, interseções, literal types. A pergunta era sempre a mesma — "*qual é a forma deste valor?*"

Agora o foco muda: tipos como descrição de comportamento. Uma função não é um dado estático; ela recebe valores, opera sobre eles e produz um resultado. O TypeScript precisa saber o tipo de cada entrada e o tipo da saída para verificar que a função está sendo usada corretamente — e para verificar que ela mesma está sendo implementada corretamente.

---

### Por que parâmetros não podem ser inferidos

Considere uma função muito simples em JavaScript puro que soma dois valores:

```javascript
function toAdd(a, b) {
  return a + b;
};
```

O TypeScript não tem como saber, apenas olhando para essa declaração, se `a` e `b` são `number`, `string`, ou qualquer outra coisa. A função poderia ser chamada com `toAdd(5, 5)` ou com `toAdd('Hello', ' World!')` — ambos são usos válidos de `+` em JavaScript. Sem anotação, o TypeScript atribui `any` implicitamente a cada parâmetro, e a proteção desaparece.

Parâmetros não têm valor inicial — eles recebem um valor quando a função é chamada, não quando é definida. O TypeScript não pode inferir o tipo a partir de zero chamadas possíveis. A anotação é, portanto, obrigatória em parâmetros quando se quer manter a proteção ativa.

---

### Anotando parâmetros

A sintaxe é direta: depois do nome do parâmetro, dois-pontos e o tipo esperado.

```typescript
function toAdd(a: number, b: number) {
  return a + b;
};
```

**Finalidade**. As anotações `: number` nos dois parâmetros informam ao verificador que esta função aceita exclusivamente valores numéricos. Qualquer chamada com argumento de tipo diferente será rejeitada na verificação.

**Motivação**. Sem as anotações, `toAdd('Hello', ' World!')` compilaria sem erro — `+` é operador válido para strings. O resultado seria a concatenação `'Hello World!'`, não uma soma. O erro passaria para a execução, onde pode causar comportamento silenciosamente incorreto. Com as anotações, a chamada é rejeitada imediatamente na verificação.

**Decisão**. O TypeScript exige anotação explícita em parâmetros quando `strict: true` está ativo no `tsconfig.json` — que será abordado posteriormente. A configuração `strict` ativa a flag `noImplicitAny`, que transforma o `any` implícito de parâmetros sem anotação em erro de verificação. O curso exige anotação explícita em todos os parâmetros, sem exceção.

**Localização**. As anotações de parâmetro aparecem em toda declaração de função do projeto. Elas são a primeira linha de defesa do sistema de tipos.

**Contrafactual**. Sem as anotações `: number`, o TypeScript infere `any` implícito para `a` e `b`. A verificação não protege o chamador nem a implementação. `toAdd(true, null)` seria aceito — e o resultado em JavaScript seria `1` (porque `true` coerce para `1` e `null` coerce para `0`), não um erro.

---

### Anotando o retorno

O retorno de uma função também pode — e muitas vezes deve — ser anotado:

```typescript
function toAdd(a: number, b: number): number {
  return a + b;
};
```

**Finalidade**. A anotação `: number` após os parênteses declara que esta função sempre produz um `number`. O verificador confirmará que todo caminho de execução da função retorna um `number` — incluindo ramificações condicionais.

**Motivação**. O TypeScript pode inferir o tipo do retorno na maioria dos casos simples. Mas para funções com lógica ramificada, a inferência pode ser surpreendente — o tipo inferido pode ser uma união acidental quando o desenvolvedor pretendia um tipo específico. A anotação explícita torna a intenção verificável: se qualquer ramificação retornar algo diferente de `number`, o TypeScript aponta o erro.

**Decisão**. O curso adota a prática de anotar o retorno de toda função que não seja trivialmente inferida. Para funções utilitárias locais e callbacks simples, a inferência é suficiente.

**Localização**. A anotação de retorno aparece na declaração da função. Qualquer chamador que receba o resultado da função verá o tipo anotado, não o tipo inferido — o que torna a leitura do código mais previsível.

**Contrafactual**. Sem a anotação de retorno em funções complexas, uma instrução `return` esquecida em uma ramificação produz `undefined` em vez do valor esperado. O TypeScript inferirá o retorno como `number | undefined`, e o chamador precisará lidar com `undefined` mesmo que o desenvolvedor nunca pretendesse retorná-lo. A anotação explícita `: number` transformaria a ramificação sem `return` em erro de verificação detectado antes da execução.

---

### Arrow functions

A anotação funciona da mesma forma em arrow functions:

```typescript
const toAdd = (a: number, b: number): number => a + b;
```

A posição da anotação de retorno muda — ela fica entre os parênteses e a seta `=>` — mas a semântica é idêntica.

---

## Bloco 2 — Parâmetros Default e Rest

### Parâmetros default tipados

No módulo anterior, parâmetros default já foram usados em JavaScript:

```javascript
function greet(name = 'visitor') {
  return `Hello, ${name}!`;
};
```

Em TypeScript, quando um parâmetro tem valor default e o valor default tem tipo evidente, o TypeScript infere o tipo do parâmetro a partir do default. No exemplo acima, `name` é inferido como `string` porque `'visitor'` é uma `string`. A anotação explícita é opcional, mas pode ser adicionada:

```typescript
function greet(name: string = 'visitor'): string {
  return `Hello, ${name}!`;
};
```

**Finalidade**. O valor default é o valor usado quando o argumento é omitido na chamada. A anotação de tipo garante que, mesmo quando o argumento é passado, ele seja do tipo esperado.

**Motivação**. O default não dispensa a verificação: `greet(42)` passaria um `number` onde `string` é esperada. Sem anotação, o TypeScript infere o tipo correto a partir do default. Com anotação, a intenção fica explícita no contrato da função.

**Decisão**. Quando o default é um literal simples, a inferência é suficiente e a anotação é redundante. O curso anota explicitamente quando o tipo não é trivialmente óbvio a partir do default — por exemplo, quando o default é `[]` (array vazio, cujo tipo não pode ser inferido sem anotação).

**Localização**. Parâmetros default aparecem em funções utilitárias e em configurações.

**Contrafactual**. Sem o default, o chamador seria obrigado a sempre passar o argumento. Com o default mas sem anotação de tipo em casos de array vazio — `function list(filters = [])` — o TypeScript inferirá `filters` como `never[]`. A anotação `filters: string[] = []` corrige isso.

---

### Rest parameters tipados

Rest parameters capturam todos os argumentos restantes de uma chamada como um array. O tipo do rest parameter é sempre um array:

```typescript
function addThemAll(...numbers: number[]): number {
  return numbers.reduce((total, n) => total + n, 0);
};

console.log(addThemAll(1, 2, 3, 4)); // 10
```

**Finalidade**. `...numbers: number[]` declara que todos os argumentos passados a partir da posição do rest são `number`. O TypeScript rejeita qualquer argumento que não seja `number`.

**Motivação**. Sem a anotação de tipo, `...numbers` seria `any[]`. Qualquer valor seria aceito, e o `reduce` poderia produzir resultados inesperados — por exemplo, `addThemAll(1, '2', 3)` retornaria `'123'` por concatenação implícita em vez de `6` por soma.

**Decisão**. O tipo de um rest parameter é sempre um array, porque ele captura zero ou mais argumentos como coleção. A anotação `number[]` é a forma direta: *"cada um dos argumentos restantes deve ser number"*. A alternativa seria `Array<number>`, que é equivalente e pode ser preferida por legibilidade em tipos mais complexos.

**Localização**. Rest parameters aparecem em funções que aceitam quantidade variável de argumentos.

**Contrafactual**. Sem `...` no parâmetro, a função aceitaria exatamente um argumento do tipo `number[]` — o chamador precisaria passar um array explicitamente: `addThemAll([1, 2, 3])`. Com rest, o chamador passa os valores diretamente: `addThemAll(1, 2, 3)`. A diferença é de ergonomia na chamada.

---

## Bloco 3 — `void` e `never`

### O tipo `void`

Em JavaScript, uma função que não tem `return` — ou tem `return` sem expressão — produz `undefined` implicitamente. Em TypeScript, o tipo que descreve o retorno de funções que não produzem um valor útil é `void`:

```typescript
function logError(message: string): void {
  console.error(`[ERRO] ${message}`);
}
```

**Finalidade**. `: void` declara que esta função não retorna um valor que deva ser usado. O TypeScript aceita que a função retorne `undefined`, mas rejeita tentativas de usar o retorno como valor significativo.

**Motivação**. Sem a anotação `: void`, o TypeScript infere o tipo do retorno como `undefined`. A diferença entre `void` e `undefined` como tipo de retorno é sutil mas importante: `void` comunica intenção — *"o chamador não deve usar este retorno"*. `undefined` seria mais específico do que necessário e poderia confundir: `undefined` como retorno explícito é diferente de uma função que simplesmente não retorna nada.

**Decisão**. O curso anota `: void` em funções de efeito colateral puro. Funções que retornam explicitamente `undefined` como valor significativo usam `: undefined`, não `: void`.

**Localização**. `void` aparece em funções de logging, handlers de eventos, e em callbacks passados para operações que não consomem o retorno.

**Contrafactual**. Sem a anotação `: void`, nada impede o chamador de escrever `const result = logError('failed')` e depois tentar usar `result` como `string`. O TypeScript permitiria o acesso, e apenas em tempo de execução o chamador descobriria que `result` é `undefined`. Com `: void`, o TypeScript marca o uso do retorno como erro na verificação.

---

### O tipo `never`

`never` é o tipo de valores que nunca existem. Uma função cujo retorno é `never` nunca completa sua execução normalmente: ou lança um erro, ou entra em loop infinito.

```typescript
function throwError(message: string): never {
  throw new Error(message);
};
```

**Finalidade**. `: never` declara que esta função jamais produz um valor de retorno — ela sempre interrompe o fluxo por exceção ou por não terminar. O verificador usa essa informação para confirmar que qualquer código após a chamada é inalcançável.

**Motivação**. `throw` não retorna um valor — ele interrompe a execução e propaga a exceção pela pilha de chamadas. O tipo `never` captura essa semântica: *"depois desta linha, nada mais executa normalmente"*. Sem `never`, o TypeScript trataria a função como se ela pudesse retornar `undefined`, o que seria semanticamente incorreto.

**Decisão**. O curso usa `never` em duas situações: (1) funções que sempre lançam exceção, e (2) verificação exaustiva de uniões — apresentada a seguir. A alternativa seria anotar como `void`, mas isso seria impreciso: `void` indica *"não retorna valor útil mas termina"*; `never` indica *"não termina"*.

**Localização**. `never` aparece em funções utilitárias de lançamento de erro — úteis para eliminar repetição de `throw new Error(...)` espalhado pelo código.

**Contrafactual**. Sem `never` em uma função que sempre lança, o TypeScript não saberia que o código após a chamada é inalcançável. A verificação de exaustividade abaixo seria impossível — o compilador não teria como confirmar que todos os casos foram cobertos.

---

### `never` na verificação exaustiva

Esta é a aplicação mais sofisticada de `never`. Considere o tipo `Status` da aula anterior:

```typescript
type Status = 'pending' | 'sent' | 'delivered';

function describeStatus(status: Status): string {
  switch (status) {
    case 'pending': return 'Waiting';
    case 'sent': return 'Sending';
    case 'delivered': return 'Delivered';
    default: {
      const verification: never = status;
      return verification;
    };
  };
};
```

**Finalidade**. A linha `const verification: never = status` tenta atribuir `status` ao tipo `never`. Se todos os casos da união foram cobertos nos `case` anteriores, o TypeScript sabe que `status` é do tipo `never` dentro do `default` — porque não há mais valores possíveis. A atribuição é válida. Se um caso foi esquecido — digamos, `'delivered'` foi omitido — o TypeScript sabe que `status` ainda pode ser `'delivered'` dentro do `default`, e a tentativa de atribuir `'delivered'` ao tipo `never` gera erro de verificação.

**Motivação**. Quando o tipo `Status` cresce e um novo estado `'returned'` é adicionado, sem verificação exaustiva o `switch` continuaria compilando — mas o novo estado cairia no `default` silenciosamente, e a descrição seria incorreta. Com a atribuição para `never`, o TypeScript aponta imediatamente que `describeStatus` precisa tratar `'returned'`.

**Decisão**. Esta é a técnica padrão do curso para garantir que `switch` sobre uniões de literal types seja sempre exaustivo. A alternativa seria um comentário de documentação ou uma revisão manual — nenhuma das quais é verificada pelo compilador.

**Localização**. Essa técnica aparecerá no projeto ao tratar categorias de erro distintas, e ao tratar estados de entidade.

**Contrafactual**. Sem a linha de verificação exaustiva, adicionar um novo valor à união `Status` compilaria sem erro. O `default` seria atingido em execução para o novo valor, e o comportamento dependeria do que estiver no `default` — que frequentemente expressa *"impossível chegar aqui"*, mas sem a verificação, essa suposição não é garantida.

---

## Bloco 4 — Tipo de Função como Valor

No módulo anterior, funções foram passadas como argumentos — em `map`, `filter`, `reduce`, `setTimeout`. Em TypeScript, quando uma variável vai guardar uma função, ou quando um parâmetro vai receber uma função, é necessário anotar o tipo daquela função.

```typescript
type Transformer = (value: number) => string;

const format: Transformer = n => `R$ ${n.toFixed(2)}`;

function apply(value: number, formatNumber: Transformer): string {
  return formatNumber(value);
};

console.log(apply(99.9, format)); // 'R$ 99.90'
```

**Finalidade**. A linha `type Transformer = (value: number) => string;` cria um type alias para o tipo de uma função que recebe um `number` e produz uma `string`. A partir daí, `Transformer` pode ser usado como qualquer outro tipo.

**Motivação**. Sem o tipo de função anotado, o parâmetro `formatNumber` seria `any`. Qualquer função — ou qualquer valor — poderia ser passado. Com a anotação, o TypeScript verifica que a função passada tem a assinatura correta: um parâmetro `number` e retorno `string`.

**Decisão**. A anotação inline `formatNumber: (value: number) => string` é equivalente ao type alias. O curso prefere o type alias quando o mesmo tipo de função aparece em mais de um lugar — centraliza a definição e torna as assinaturas mais legíveis. Para tipos de função usados uma única vez, a anotação inline é aceitável.

**Localização**. Tipos de função aparecem em callbacks, em funções de ordem superior, e em demais situações.

**Contrafactual**. Sem a anotação de tipo da função, `apply(99.9, 'I\'m not a function')` compilaria sem erro. Em execução, a tentativa de chamar a `string` como função lançaria `TypeError: formatNumber is not a function`. Com a anotação, o erro é capturado na verificação.

---

## Bloco 5 — Generics

### O problema

```typescript
function first(list: number[]): number {
  return list[0];
};
```

A função acima opera sobre arrays de `number`. Para arrays de `string`, seria necessário escrever uma segunda função:

```typescript
function firstOfTheString(list: string[]): string {
  return list[0];
};
```

E para arrays de `Product`? Uma terceira. Para arrays de `Task`? Uma quarta. A lógica é idêntica em todos os casos — retornar `list[0]` — mas seria necessário duplicar a função para cada tipo possível. A alternativa ingênua seria usar `any[]`:

```typescript
function first(list: any[]): any {
  return list[0];
};
```

Mas isso remove toda a proteção: o chamador não sabe qual tipo vai receber, e o TypeScript não pode verificar o uso do resultado.

**Generics** resolvem isso: permitem parametrizar o tipo assim como parâmetros parametrizam valores.

---

### A sintaxe

```typescript
function first<T>(list: T[]): T {
  return list[0];
};
```

**Finalidade**. `<T>` declara um parâmetro de tipo chamado `T`. `T` é um espaço reservado — um nome para "qualquer tipo que o chamador fornecer". O parâmetro `list` é um array cujos elementos são do tipo `T`, e a função retorna um valor do mesmo tipo que os elementos do array. A letra `T` é convenção; poderia ser qualquer identificador.

**Motivação**. O TypeScript infere `T` a partir do argumento passado na chamada. Ao escrever `first([1, 2, 3])`, o TypeScript vê que o argumento é `number[]`, conclui que `T = number`, e sabe que o retorno é `number`. A duplicação de funções para cada tipo é eliminada sem abrir mão da verificação.

**Decisão**. O nome `T` é convencional para o caso de tipo único. Quando uma função tem múltiplos parâmetros de tipo, o curso usa nomes descritivos: `<TKey, TValue>`, `<TEntry, TExit>`. A letra única `T` é reservada para casos simples onde o contexto torna o significado óbvio.

**Localização**. Generics aparecem em funções utilitárias que operam sobre dados de tipo variável, em tipos de container genérico, e — de forma crucial — nos utility types da próxima seção, que são todos genéricos.

**Contrafactual**. Sem generics, a escolha seria entre duplicação de código por tipo (inviável em larga escala) ou uso de `any` (que remove a proteção). Generics são o mecanismo que preserva a proteção sem exigir duplicação.

---

### Restrições com `extends`

```typescript
function getName<T extends { name: string }>(entity: T): string {
  return entity.name;
};
```

**Finalidade**. `T extends { name: string }` restringe os tipos aceitos por `T`: apenas tipos que tenham, no mínimo, uma propriedade `name` do tipo `string` são válidos. O objeto pode ter outras propriedades além de `name` — a restrição é mínima, não exata.

**Motivação**. Sem a restrição, `entity.name` seria erro de verificação — o TypeScript não sabe que `T` tem `name`. A restrição serve de contrato: *"aceito qualquer `T`, desde que `T` tenha `name`"*. Isso permite usar `entity.name` com segurança dentro da função.

**Decisão**. A restrição com `extends` é a forma padrão de garantir que um tipo genérico satisfaça um conjunto mínimo de requisitos. Para funções genéricas utilitárias, a restrição inline com `extends` e objeto literal é direta.

**Localização**. Restrições aparecem sempre que uma função genérica precisa acessar propriedades do argumento.

**Contrafactual**. Sem `extends { name: string }`, o acesso a `entity.name` gera erro imediato: `Property 'name' does not exist on type 'T'`. A restrição é a condição necessária para que a função possa usar propriedades do argumento genérico.

---

## Bloco 6 — Utility Types Essenciais

Utility types são tipos genéricos pré-construídos que o TypeScript fornece na biblioteca padrão. Todos eles são implementados com os recursos já apresentados — generics e outros operadores de tipo — e são exportados automaticamente em qualquer projeto TypeScript, sem import.

O padrão de uso é sempre o mesmo: `UtilityType<BaseType>`.

---

### `Partial<T>`

Todas as propriedades tornam-se opcionais.

```typescript
type Product = {
  readonly id: number;
  name: string;
  price: number;
  description?: string;
};

type ProductUpdate = Partial<Product>;
```

**Finalidade**. `Partial<T>` produz um novo tipo com todas as propriedades de `T`, mas todas marcadas como opcionais com `?`.

**Motivação**. Em operações de atualização, o chamador fornece apenas os campos que deseja alterar. Uma função que aceita `Product` exigiria todos os campos — inclusive os que não mudam. `Partial<Product>` aceita qualquer subconjunto dos campos.

**Decisão**. A alternativa seria criar manualmente um tipo separado com todos os campos opcionais — mas isso duplicaria a definição e ficaria desatualizado se `Product` mudar. `Partial<T>` deriva o tipo automaticamente do original.

**Localização**. `Partial` aparece em endpoints de atualização de entidades.

**Contrafactual**. Sem `Partial`, ou o tipo de atualização exigiria todos os campos (forçando o chamador a passar campos que não quer alterar), ou usaria `any` (removendo a proteção). `Partial` é o equilíbrio.

---

### `Required<T>`

Todas as propriedades tornam-se obrigatórias.

```typescript
type OptionalConfiguration = {
  timeout?: number;
  retries?: number;
  verbose?: boolean;
};

type CompleteConfiguration = Required<OptionalConfiguration>;
```

**Finalidade**. `Required<T>` é o inverso de `Partial<T>`: remove o `?` de todas as propriedades.

**Motivação**. Um sistema pode aceitar configuração parcial do usuário, preencher os defaults internamente, e depois operar com a configuração completa. `Required` expressa que, após o preenchimento dos defaults, nenhum campo é mais opcional — todos estão presentes.

**Decisão**. `Required` é menos frequente que `Partial`, mas essencial para tipos que passam por uma fase de "completar defaults" antes do uso interno.

**Localização**. `Required` aparecerá em funções que normalizam as opções de configuração do servidor antes de iniciá-lo.

**Contrafactual**. Sem `Required`, o código que opera com a configuração após o preenchimento de defaults precisaria tratar cada campo como potencialmente `undefined` — mesmo sabendo que todos foram preenchidos. O TypeScript não teria como saber que a normalização aconteceu.

---

### `Pick<T, K>`

Seleciona um subconjunto de propriedades.

```typescript
type ProductSummary = Pick<Product, 'id' | 'name'>;
```

**Finalidade**. `Pick<T, K>` produz um tipo com apenas as propriedades listadas em `K`, extraídas de `T`. `K` é uma união de literal types de string correspondendo aos nomes das propriedades desejadas.

**Motivação**. APIs bem projetadas não expõem todos os campos de uma entidade em todas as respostas. Uma listagem de produtos pode precisar apenas de `id` e `name` — sem o `price` ou a `description`. `Pick` expressa exatamente esse contrato de resposta.

**Decisão**. A alternativa seria criar um tipo separado com os campos desejados, o que não fica automaticamente sincronizado com o tipo original. `Pick` deriva o subconjunto do tipo original, incluindo modificadores como `readonly` e `?`.

**Localização**. `Pick` aparece em tipos de respostas de endpoints de projetos. Cada endpoint retorna um subconjunto específico dos campos de uma entidade, e `Pick` expressa esse contrato com precisão.

**Contrafactual**. Sem `Pick`, o endpoint poderia retornar campos que não deveriam ser expostos — como senhas hash, tokens, ou dados internos — porque não há como expressar no tipo que apenas um subconjunto deve ser incluído.

---

### `Omit<T, K>`

Remove propriedades.

```typescript
type ProductWithoutID = Omit<Product, 'id'>;
```

**Finalidade**. `Omit<T, K>` é o complemento de `Pick`: produz um tipo com todas as propriedades de `T`, exceto as listadas em `K`.

**Motivação**. Na criação de um produto, o `id` é gerado pelo banco de dados — o chamador não fornece um `id`. `Omit<Product, 'id'>` expressa o tipo do objeto de entrada para criação: todos os campos de `Product`, exceto o identificador.

**Decisão**. `Omit` e `Pick` são complementares. `Pick` é preferível quando o subconjunto desejado é pequeno; `Omit` é preferível quando o subconjunto a remover é pequeno. Se uma entidade tem 10 campos e se quer 9 deles, `Omit` é mais conciso que `Pick`. Se se quer 2 de 10, `Pick` é mais conciso.

**Localização**. `Omit` aparece em tipos de entrada dos endpoints de criação — `Omit<Product, 'id' | 'createdAt'>` expressaria os campos que o chamador fornece ao criar um recurso.

**Contrafactual**. Sem `Omit`, o tipo de entrada para criação seria escrito manualmente — e ficaria desatualizado se a entidade ganhar novos campos obrigatórios.

---

### `Record<K, V>`

Dicionário tipado.

```typescript
type CountByCategory = Record<string, number>;

const count: CountByCategory = {
  electronics: 42,
  clothes: 17,
  foods: 93
};
```

**Finalidade**. `Record<K, V>` cria o tipo de um objeto cujas chaves são do tipo `K` e cujos valores são do tipo `V`. É o tipo para dicionários.

**Motivação**. Sem `Record`, um dicionário precisaria ser anotado como `{ [key: string]: number }`. `Record<string, number>` é equivalente mas mais legível, especialmente quando combinado com literal types como chaves: `Record<'electronics' | 'clothes', number>` restringe as chaves válidas.

**Decisão**. O curso usa `Record` quando as chaves são um conjunto conhecido de literals. Para dicionários com qualquer chave `string`, os dois estilos são aceitos.

**Localização**. `Record` aparece em mapeamentos de códigos de status HTTP para mensagens de erro em projetos, dentre outros lugares.

**Contrafactual**. Sem `Record`, a anotação manual de dicionários é verbosa e menos legível. Com literal types como chaves — `Record<Status, string>` — o TypeScript verifica que todos os estados têm uma entrada correspondente.

---

### `Awaited<T>`

Extrai o tipo resolvido de uma promessa.

```typescript
type asyncResult = Promise<number>;
type Result = Awaited<asyncResult>;
```

**Finalidade**. `Awaited<T>` extrai o tipo do valor que uma `Promise<T>` produz quando resolvida. Se `T` não é uma `Promise`, `Awaited<T>` retorna `T` inalterado.

**Motivação**. Funções assíncronas retornam `Promise<T>`. Quando o resultado é passado adiante, frequentemente é o valor resolvido, não a `Promise`, que importa para o tipo. `Awaited` extrai esse tipo sem precisar desfazer a `Promise` manualmente.

**Decisão**. `Awaited` é particularmente útil em código que combina tipos inferidos de funções assíncronas com outros tipos. O curso o usa na tipagem dos retornos de funções que retornam `Promise`.

**Localização**. `Awaited` aparece no backend, quando funções assíncronas são compostas com outras funções.

**Contrafactual**. Sem `Awaited`, derivar o tipo do valor resolvido de uma `Promise` exigiria anotação manual repetida — e ficaria desatualizado se o tipo retornado pela função mudar.

---

## Bloco 7 — Asserção com `as`

### Asserção de tipo com `as`

Há situações em que o desenvolvedor sabe algo sobre o tipo de um valor que o TypeScript não consegue verificar automaticamente. A asserção de tipo com `as` instrui o verificador a tratar um valor como se fosse de um tipo específico, sobrepondo a inferência:

```typescript
const input = document.querySelector('#field') as HTMLInputElement;

input.value;
```

**Finalidade**. `as HTMLInputElement` instrui o TypeScript a tratar o resultado de `querySelector` como `HTMLInputElement`. Sem a asserção, o tipo inferido seria `HTMLElement | null` — que não tem a propriedade `.value`.

**Motivação**. `document.querySelector` pode retornar qualquer elemento HTML ou `null`. O TypeScript não tem como saber, a partir da string `'#field'`, que o elemento em questão é especificamente um input. O desenvolvedor sabe — porque escreveu o HTML e conhece a estrutura da página — e `as` é o mecanismo de comunicar esse conhecimento ao verificador.

**Decisão**. A asserção com `as` é uma substituição da verificação, não uma conversão de valor. Em tempo de execução, nenhuma coerção acontece — o valor é exatamente o que era. O TypeScript apenas para de verificar e passa a confiar na declaração do desenvolvedor. Isso torna `as` perigoso: se o desenvolvedor estiver errado, o erro só aparece em execução.

**Localização**. O curso restringe `as` a dois casos: (1) interação com APIs do DOM que retornam tipos amplos quando o desenvolvedor conhece o tipo específico, e (2) conversão de `unknown` após validação explícita — quando o código verificou o tipo manualmente mas o TypeScript não consegue inferir isso. Fora desses casos, `as` é proibido como prática.

**Contrafactual**. Sem `as`, acessar `.value` em um `HTMLElement | null` gera erro de verificação — `.value` não existe em `HTMLElement`, apenas em `HTMLInputElement`. O desenvolvedor precisaria de narrowing com verificação de tipo (mais seguro) ou de `as` (mais rápido, mas arriscado).

---

### Limitação da asserção: double assertion

```typescript
const n = '42' as unknown as number;

console.log(n.toFixed(2)); // n.toFixed is not a function
```

Este padrão — chamado de *double assertion* — é a forma de forçar qualquer tipo para qualquer outro, passando por `unknown` como intermediário. O TypeScript aceita em verificação, mas o erro aparece em execução. O curso proíbe explicitamente esse padrão.
