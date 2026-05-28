# Parte 2

## Bloco 1 — Anotação de Objetos Literais

### Contexto: objetos em JavaScript

O módulo anterior introduziu objetos como agrupamentos de pares chave-valor. Em JavaScript puro, nada impede a criação de um objeto com `name: 'Alex'` e o posterior acesso a `name.toFixed(2)` — o erro só aparece em tempo de execução, quando o programa já está rodando.

TypeScript resolve isso na etapa de verificação, antes da execução. Para que isso seja possível, é necessário declarar a forma esperada do objeto: quais propriedades existem e qual é o tipo de cada uma.

### A definição

A sintaxe de **anotação de objeto literal** descreve a estrutura esperada diretamente no ponto de declaração da variável:

```typescript
const product: {
  name: string;
  price: number;
} = {
  name: 'Pizza',
  price: 19.99
};
```

Cada propriedade é listada com seu nome, dois-pontos e o tipo esperado. As propriedades são separadas por ponto-e-vírgula dentro da anotação.

### Propriedades opcionais com `?`

Nem toda propriedade de um objeto é obrigatória. O símbolo `?` após o nome da propriedade marca que ela é opcional — ou seja, seu tipo é automaticamente `type | undefined`:

```typescript
const product: {
  name: string;
  price: number;
  description?: string;
} = {
  name: 'Pizza',
  price: 19.99
};
```

**Por que isso importa**: ao tentar acessar `product.description.length` sem verificar se a propriedade existe, o TypeScript emite erro na verificação. O verificador sabe que o valor pode ser `undefined` e exige tratamento explícito do caso antes do uso.

### Propriedades `readonly`

A palavra-chave `readonly` antes de uma propriedade impede que ela seja reatribuída após a criação do objeto:

```typescript
const product: {
  readonly id: number;
  name: string;
  price: number;
  description?: string;
} = {
  id: 1,
  name: 'Pizza',
  price: 19.99
};

product.id = 2; // Cannot assign to 'id' because it is a read-only property.
product.name = 'Juice';
product.price = 2.99;
```

**Por que isso existe**: certos campos de um objeto nunca deveriam ser alterados após a criação — um identificador gerado pelo banco de dados, por exemplo. `readonly` torna essa intenção verificável pelo compilador. Sem ele, qualquer parte do código pode silenciosamente sobrescrever um campo que deveria ser imutável, e o erro só apareceria — se aparecesse — em tempo de execução.

**Contrafactual**: sem `readonly`, o TypeScript trata todas as propriedades como mutáveis. A tentativa de reatribuição `product.id = 2;` compilaria sem erro, e a proteção esperada simplesmente não existiria.

## Bloco 2 — Type Aliases

### O problema

A anotação de objeto literal funciona, mas imagine que o tipo `Product` precise aparecer em dez lugares diferentes: na função que cria um produto, na que atualiza o preço, na que lista todos os produtos, e assim por diante. Repetir `{ readonly id: number; name: string; price: number; description?: string; }` em cada um desses pontos é verboso, propenso a inconsistência e difícil de manter. Se o tipo mudar — por exemplo, se `description` deixar de ser opcional — seria necessário alterar dez lugares.

O *type alias* (apelido de tipo) resolve isso atribuindo um nome reutilizável a um tipo.

### A definição

A palavra-chave `type` cria um *alias* para qualquer tipo:

```typescript
type Product = {
  readonly id: number;
  name: string;
  price: number;
  description?: string;
};
```

A partir daí, `Product` pode ser usado em qualquer ponto onde a anotação completa seria usada:

```typescript
const product: Product = {
  id: 1,
  name: 'Pizza',
  price: 19.99
};
```

**Finalidade**: nomear um tipo para que ele possa ser referenciado em múltiplos pontos sem repetição.

**Motivação**: a repetição de anotações longas é o equivalente tipado de duplicação de código. Quando o tipo muda, a alteração ocorre em um único lugar — a declaração do alias — e todos os usos são atualizados automaticamente na próxima verificação.

**Decisão técnica**: TypeScript também tem `interface`, que serve para descrever a forma de objetos. Este curso adota `type` como padrão por uma razão deliberada: `type` é mais geral — funciona para objetos, mas também para uniões, tuplas, literais e qualquer outra construção de tipo abordada adiante. `interface` só descreve objetos. Adotar `type` consistentemente elimina a decisão caso a caso.

**Convenção do curso**: nomes de *type aliases* seguem PascalCase: `Product`, `Order`, `Delivery`. Isso os distingue visualmente de variáveis e funções, que seguem camelCase. Ao ler `const x: Product`, o PascalCase sinaliza imediatamente que `Product` é um tipo, não um valor.

**Contrafactual**: sem *type aliases*, qualquer mudança na estrutura de `Product` exigiria localizar e atualizar manualmente cada ponto onde a anotação foi repetida. Um campo esquecido produz inconsistência silenciosa — o TypeScript passa a verificar partes do código contra um tipo diferente do que o restante espera.

## Bloco 3 — Arrays Tipados

### Contexto: arrays em JavaScript

O módulo anterior trabalhou extensivamente com arrays: `map`, `filter`, `reduce`, `find`, entre outros. Em JavaScript puro, um array pode conter qualquer coisa — números, textos, objetos, elementos misturados sem restrição. TypeScript permite restringir isso: declarar que um array contém exclusivamente elementos de um tipo específico.

### As duas notações

Existem duas formas equivalentes de anotar um array tipado:

```typescript
const prices: number[] = [49.99, 189.99, 320.00];

const prices: Array<number> = [49.99, 189.99, 320.00];
```

**Finalidade de ambas**: declarar que todos os elementos do array são `number`. Qualquer tentativa de inserir uma `string` ou `boolean` nesse array é capturada na verificação.

**Decisão técnica**: o curso usa `number[]` como padrão por ser mais curto e mais legível na maioria dos casos. `Array<number>` é preferível quando o tipo do elemento é complexo — por exemplo, um array de objetos com tipo longo, onde a notação com `<>` separa melhor o tipo do *container* do tipo do conteúdo. Não há diferença funcional entre as duas formas.

**Contrafactual**: sem a anotação de tipo no array, o TypeScript infere o tipo a partir dos valores iniciais. Ao escrever `const prices = [49.99, 189.99];`, ele infere `number[]` corretamente. Mas se o array começa vazio — `const prices = [];` — o TypeScript infere `never[]`, um tipo que não aceita nenhum elemento. Qualquer tentativa de executar `prices.push(49.99);` gera erro. A anotação explícita resolve esse caso.

## Bloco 4 — Tuplas

### O problema

Um array tipado garante que todos os elementos são do mesmo tipo. Mas há situações em que uma lista de posição fixa carrega valores de tipos diferentes, onde a posição tem significado: o primeiro elemento é sempre `string`, o segundo é sempre `number`. Para isso existe a **tupla**.

### A definição

Uma tupla é um array de comprimento fixo onde cada posição tem um tipo declarado:

```typescript
const coordinates: [number, number] = [23.5, -46.6];

const entry: [string, number] = ['altitude', 890];
```

**Finalidade**: descrever uma lista onde a posição importa e o comprimento é conhecido.

**Motivação**: há casos em que é necessário retornar dois valores de uma função. Uma forma comum é retornar um objeto; outra é retornar uma tupla:

```typescript
function divide(n1: number, n2: number): [number, boolean] {
  if (n2 === 0) return [0, false];
  return [n1 / n2, true];
};

const [result, success] = divide(10, 2);
```

A desestruturação (*destructuring*) do módulo anterior funciona diretamente com tuplas. O TypeScript sabe que `result` é `number` e `success` é `boolean` — porque a tupla declarou isso.

**Limitação importante**: o TypeScript não impede chamadas a `.push()` em tuplas sem `readonly`. Ao declarar `const entry: [string, number]`, ainda é possível executar `entry.push('extra')` sem erro de compilação. Para impedir mutações e garantir comprimento fixo em tempo de verificação, a anotação correta é `readonly [string, number]`.

**Diferença em relação a arrays**: um array `number[]` aceita qualquer quantidade de números em qualquer ordem. Uma tupla `[number, boolean]` aceita exatamente dois elementos, na ordem exata, com os tipos exatos. Tentar colocar uma `string` na posição `0` ou adicionar um terceiro elemento por atribuição direta gera erro.

**Contrafactual**: sem a anotação de tupla, `[n1 / n2, true]` seria inferido como `(number | boolean)[]` — um array de comprimento variável contendo números ou booleanos. O TypeScript perderia a informação de que a posição `0` é sempre `number` e a posição `1` é sempre `boolean`. A desestruturação `const [result, success]` produziria variáveis do tipo `number | boolean` em vez dos tipos precisos.

## Bloco 5 — Uniões

### O problema

Nem sempre um valor tem um único tipo possível. Um campo de formulário pode chegar como `string` ou como `null` quando não preenchido. Uma função pode receber um identificador como `number` ou como `string`. TypeScript tem um mecanismo para declarar exatamente isso: o **tipo união**.

### A definição

Uma união declara que um valor pode ser de um tipo **ou** de outro. O operador é `|`:

```typescript
type Id = number | string;

const id: Id = 1;
const newId: Id = crypto.randomUUID();
```

**Finalidade**: representar valores que legitimamente podem assumir mais de um tipo.

**Motivação**: sem uniões, seria necessário escolher entre `any` — que desliga a verificação — ou criar tipos separados para cada caso. A união preserva a verificação: o TypeScript sabe exatamente quais tipos são possíveis.

### Narrowing

Ao declarar uma união, o TypeScript não permite que o valor seja usado como se fosse apenas um dos tipos sem antes verificar qual ele é. Essa verificação se chama ***narrowing*** (estreitamento de tipo).

O mecanismo mais direto é o `typeof`:

```typescript
function process(value: number | string): string {
  if (typeof value === 'number') return value.toFixed(2);

  return value.toUpperCase();
};
```

**Por que o *narrowing* é obrigatório**: `toFixed` existe em `number` mas não em `string`. `toUpperCase` existe em `string` mas não em `number`. Ao tentar chamar `value.toFixed(2)` sem o `if`, o TypeScript emitiria erro — porque `value` poderia ser `string`, e `string` não tem `toFixed`. O *narrowing* é o mecanismo que prova ao compilador, dentro de um bloco específico, que o tipo é o declarado.

**Contrafactual**: sem *narrowing*, chamar qualquer método específico de um dos tipos da união gera erro de verificação. E sem a união em si — usando `any` no lugar — o TypeScript aceitaria o código sem reclamar, mas toda a proteção seria perdida: uma `string` poderia chegar onde apenas `number` faz sentido, e o erro só apareceria em execução.

## Bloco 6 — Interseções

### A intuição

A união diz "ou": o valor é do tipo A ou do tipo B. A **interseção** diz "e": o valor tem todas as propriedades do tipo A e todas as propriedades do tipo B simultaneamente.

```typescript
type Registration = {
  barcode: string;
  validity: string;
};

type Product = {
  readonly id: number;
  name: string;
  price: number;
  category?: string;
};

type ProductWithRegistration = Product & Registration;
```

`ProductWithRegistration` tem todas as propriedades de `Product` e todas as propriedades de `Registration`. Um objeto desse tipo precisa satisfazer os dois tipos ao mesmo tempo.

**Finalidade**: compor tipos a partir de partes reutilizáveis.

**Motivação**: em sistemas reais, muitas entidades compartilham campos comuns — *timestamps* de criação e atualização, por exemplo. Em vez de repetir esses campos em cada tipo, declara-se uma vez e compõe-se via `&` onde forem necessários. A mudança se propaga automaticamente para todos os tipos compostos.

**Contrafactual**: sem interseções, as propriedades seriam repetidas em cada tipo que as necessitasse. Uma mudança nesses campos — renomear ou alterar o tipo — exigiria atualização manual em cada ocorrência.

## Bloco 7 — Literal Types e `as const`

### O problema

`enum` não será utilizado neste curso por conta da configuração `erasableSyntaxOnly` — ele gera código JavaScript e não pode ser removido por *type stripping*. Mas `enum` existe por uma razão legítima: restringir um valor a um conjunto fixo de opções. Um estado de pedido não é qualquer `string` — é exatamente `'pending'`, `'sent'` ou `'delivered'`. **Literal types** e `as const` são os substitutos adotados.

### Literal types

Um **literal type** restringe um valor a uma opção específica. Combinado com união, forma um conjunto fechado de valores permitidos:

```typescript
type Status = 'pending' | 'sent' | 'delivered';

const firstOrder: Status = 'sent';
const secondOrder: Status = 'canceled'; // Type '"canceled"' is not assignable to type 'Status'.
```

**Finalidade**: garantir que um campo só aceite valores de um conjunto predefinido, com verificação em tempo de compilação.

**Motivação**: sem literal types, `firstOrder` e `secondOrder` seriam `string` — qualquer `string` passaria. Com literal types, o TypeScript rejeita na verificação qualquer valor fora do conjunto. Erros de digitação são capturados antes da execução.

### `as const`

Ao declarar um objeto ou array sem anotação, o TypeScript infere os tipos de forma ampla. Um campo `'pending'` é inferido como `string`, não como o literal `'pending'`. O `as const` instrui o TypeScript a preservar os literais exatos:

```typescript
const STATUS = {
  PENDING: 'pending',
  SENT: 'sent',
  DELIVERED: 'delivered'
} as const;
```

Sem `as const`, `STATUS.PENDING` seria do tipo `string`. Com `as const`, é do tipo `'pending'` — o literal exato. Isso permite usar os valores do objeto como literal types:

```typescript
type Status = typeof STATUS[keyof typeof STATUS]; // equivale a: 'pending' | 'sent' | 'delivered'
```

**Por que `as const` substitui `enum`**: `enum` gera um objeto JavaScript em tempo de execução — é código, não apenas tipo. `as const` sobre um objeto literal é apenas uma instrução para o verificador de tipos; o objeto já existiria de qualquer forma. Nenhum código adicional é gerado. O *type stripping* remove apenas as anotações, e o objeto `STATUS` permanece como JavaScript válido.

**Contrafactual**: sem `as const`, o TypeScript infere `STATUS.PENDING` como `string`. A capacidade de usar os valores como literal types é perdida, assim como a verificação exaustiva — o compilador não consegue detectar que um `switch` sobre `Status` deixou de tratar o caso `'delivered'`.
