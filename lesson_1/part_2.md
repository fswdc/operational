# Parte 2

## Bloco 1 — Anotação de Objetos Literais

### Contexto

O módulo anterior introduziu objetos como agrupamentos de pares chave-valor. Em JavaScript puro, nada impede a criação de um objeto com `name: 'Alex'` e o posterior acesso a `name.toFixed(2)` — o erro só aparece em tempo de execução, quando o programa já está rodando.

TypeScript resolve isso na etapa de verificação, antes da execução. Para que isso seja possível, é necessário declarar a forma esperada do objeto: quais propriedades existem e qual é o tipo de cada uma.

---

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

---

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

**Implicação prática.** Ao tentar acessar `product.description.length` sem verificar se a propriedade existe, o TypeScript emite erro na verificação. O verificador sabe que o valor pode ser `undefined` e exige tratamento explícito do caso antes do uso.

---

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

**Finalidade.** `readonly` é uma restrição de reatribuição aplicada pelo verificador de tipos: uma propriedade marcada com `readonly` não pode receber novo valor após a inicialização do objeto.

**Motivação.** Certos campos de um objeto nunca deveriam ser alterados após a criação — um identificador gerado pelo banco de dados, por exemplo. `readonly` torna essa intenção verificável em tempo de compilação, em vez de depender de convenção ou disciplina manual.

**Decisão.** O TypeScript oferece `readonly` tanto em anotações de objeto literal quanto em *type aliases* e interfaces. O curso adota `readonly` diretamente na anotação por ser a forma mais explícita e localizada: a restrição está declarada no mesmo ponto em que a propriedade é definida.

**Localização.** `readonly` é aplicado em propriedades que representam identidade ou origem do dado: identificadores gerados externamente, chaves primárias, valores definidos no momento de criação que não devem ser sobrescritos por nenhuma parte do código.

**Contrafactual.** Sem `readonly`, o TypeScript trata todas as propriedades como mutáveis. A tentativa de reatribuição `product.id = 2;` compilaria sem erro, e a proteção esperada simplesmente não existiria.

---

## Bloco 2 — Type Aliases

### O problema

A anotação de objeto literal funciona, mas considere que o tipo `Product` precise aparecer em dez lugares diferentes: na função que cria um produto, na que atualiza o preço, na que lista todos os produtos, e assim por diante. Repetir `{ readonly id: number; name: string; price: number; description?: string; }` em cada um desses pontos é verboso, propenso a inconsistência e difícil de manter. Se o tipo mudar — por exemplo, se `description` deixar de ser opcional — seria necessário alterar dez lugares.

O *type alias* (apelido de tipo) resolve isso atribuindo um nome reutilizável a um tipo.

---

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

**Finalidade.** `type` atribui um nome reutilizável a um tipo, permitindo que ele seja referenciado em múltiplos pontos do código sem repetição da anotação completa.

**Motivação.** A repetição de anotações longas é o equivalente tipado de duplicação de código. Quando o tipo muda, a alteração ocorre em um único lugar — a declaração do *alias* — e todos os usos são atualizados automaticamente na próxima verificação.

**Decisão.** TypeScript também tem `interface`, que serve para descrever a forma de objetos e que será apresentada em momento oportuno. Este curso adota `type` como padrão por uma razão deliberada: `type` é mais geral — funciona para objetos, mas também para outras construções de tipo abordadas adiante. Adotar `type` consistentemente elimina a decisão caso a caso.

**Localização.** *Type aliases* são declarados no nível de módulo, fora de funções e blocos, e referenciados em parâmetros de função, retornos, variáveis e em outros tipos compostos. Nomes de *type aliases* seguem PascalCase: `Product`, `Order`, `Delivery`. Isso os distingue visualmente de variáveis e funções, que seguem camelCase — ao ler `const x: Product`, o PascalCase sinaliza imediatamente que `Product` é um tipo, não um valor.

**Contrafactual.** Sem *type aliases*, qualquer mudança na estrutura de `Product` exigiria localizar e atualizar manualmente cada ponto onde a anotação foi repetida. Um campo esquecido produz inconsistência silenciosa — o TypeScript passa a verificar partes do código contra um tipo diferente do que o restante espera.

---

## Bloco 3 — Arrays Tipados

### Contexto

O módulo anterior trabalhou extensivamente com arrays: `map`, `filter`, `reduce`, `find`, entre outros. Em JavaScript puro, um array pode conter qualquer coisa — números, textos, objetos, elementos misturados sem restrição. TypeScript permite restringir isso: declarar que um array contém exclusivamente elementos de um tipo específico.

---

### As duas notações

Existem duas formas equivalentes de anotar um array tipado:

```typescript
const prices: number[] = [49.99, 189.99, 320.00];

const prices: Array<number> = [49.99, 189.99, 320.00];
```

**Finalidade.** Ambas as notações declaram que todos os elementos do array são `number`. Qualquer tentativa de inserir uma `string` ou `boolean` nesse array é capturada na verificação.

**Motivação.** Em JavaScript puro, um array não carrega informação sobre o tipo de seus elementos — nada impede a mistura de números, textos e objetos no mesmo array. A anotação de tipo impõe essa restrição em tempo de compilação, tornando o comportamento do array previsível e verificável.

**Decisão.** O curso usa `number[]` como padrão por ser mais curto e mais legível na maioria dos casos. `Array<number>` é preferível quando o tipo do elemento é complexo — por exemplo, um array de objetos com tipo longo, onde a notação com `<>` separa melhor o tipo do *container* do tipo do conteúdo. Não há diferença funcional entre as duas formas.

**Localização.** Arrays tipados aparecem em parâmetros de função que recebem listas de dados, em variáveis de estado que acumulam resultados, e em retornos de funções que produzem coleções. A escolha da notação segue o padrão do arquivo ou do projeto.

**Contrafactual.** Sem a anotação de tipo no array, o TypeScript infere o tipo a partir dos valores iniciais. Ao escrever `const prices = [49.99, 189.99];`, ele infere `number[]` corretamente. Mas se o array começa vazio — `const prices = [];` — o TypeScript infere `never[]`, um tipo que não aceita nenhum elemento. Qualquer tentativa de executar `prices.push(49.99);` gera erro. A anotação explícita resolve esse caso.

---

## Bloco 4 — Tuplas

### O problema

Um array tipado garante que todos os elementos são do mesmo tipo. Mas há situações em que uma lista de posição fixa carrega valores de tipos diferentes, onde a posição tem significado: o primeiro elemento é sempre `string`, o segundo é sempre `number`. Para isso existe a **tupla**.

---

### A definição

Uma tupla é um array de comprimento fixo onde cada posição tem um tipo declarado:

```typescript
const coordinates: [number, number] = [23.5, -46.6];

const entry: [string, number] = ['altitude', 890];
```

**Finalidade.** Uma tupla descreve uma lista onde a posição importa e o comprimento é conhecido — cada índice carrega um tipo específico e declarado.

**Motivação.** Há casos em que é necessário retornar dois valores de uma função. Uma forma comum é retornar um objeto; outra é retornar uma tupla:

```typescript
function divide(n1: number, n2: number): [number, boolean] {
  if (n2 === 0) return [0, false];
  return [n1 / n2, true];
};

const [result, success] = divide(10, 2);
```

A desestruturação (*destructuring*) do módulo anterior funciona diretamente com tuplas. O TypeScript sabe que `result` é `number` e `success` é `boolean` — porque a tupla declarou isso.

**Decisão.** A alternativa à tupla para retornar múltiplos valores é um objeto com propriedades nomeadas. O objeto é preferível quando os campos têm nomes semânticos relevantes; a tupla é preferível quando a estrutura é simples, posicional e o consumidor usa desestruturação imediata. Para impedir mutações e garantir comprimento fixo em tempo de verificação, a anotação correta é `readonly [string, number]` — sem `readonly`, o TypeScript não impede chamadas a `.push()` em tuplas.

**Localização.** Tuplas aparecem em retornos de funções utilitárias que produzem um par de valores relacionados — resultado e indicador de sucesso, valor e metadado —, e em desestruturação imediata no ponto de chamada.

**Contrafactual.** Sem a anotação de tupla, `[n1 / n2, true]` seria inferido como `(number | boolean)[]` — um array de comprimento variável contendo números ou booleanos. O TypeScript perderia a informação de que a posição `0` é sempre `number` e a posição `1` é sempre `boolean`. A desestruturação `const [result, success]` produziria variáveis do tipo `number | boolean` em vez dos tipos precisos.

A diferença em relação a arrays: um array `number[]` aceita qualquer quantidade de números em qualquer ordem. Uma tupla `[number, boolean]` aceita exatamente dois elementos, na ordem exata, com os tipos exatos. Tentar colocar uma `string` na posição `0` ou adicionar um terceiro elemento por atribuição direta gera erro.

---

## Bloco 5 — Uniões

### O problema

Nem sempre um valor tem um único tipo possível. Um campo de formulário pode chegar como `string` ou como `null` quando não preenchido. Uma função pode receber um identificador como `number` ou como `string`. TypeScript tem um mecanismo para declarar exatamente isso: o **tipo união**.

---

### A definição

Uma união declara que um valor pode ser de um tipo **ou** de outro. O operador é `|`:

```typescript
type Id = number | string;

const id: Id = 1;
const newId: Id = crypto.randomUUID();
```

**Finalidade.** O operador `|` declara que uma variável ou parâmetro aceita mais de um tipo, preservando a verificação para cada um dos tipos possíveis.

**Motivação.** Sem uniões, seria necessário escolher entre `any` — que desliga a verificação — ou criar tipos separados para cada caso. A união preserva a verificação: o TypeScript sabe exatamente quais tipos são possíveis e rejeita qualquer valor fora desse conjunto.

**Decisão.** O operador `|` é a forma nativa de união em TypeScript. A alternativa mais próxima seria `any`, que remove toda verificação, ou sobrecarga de função, que é mais verbosa e limitada a declarações de função. O curso adota `|` como padrão para todo caso em que um valor legitimamente assume mais de um tipo.

**Localização.** Uniões aparecem em parâmetros de função que aceitam mais de um tipo de entrada, em campos de formulário que podem estar preenchidos ou nulos, e em retornos de API que podem produzir dado ou indicador de ausência.

**Contrafactual.** Sem a união, seria necessário usar `any` para aceitar múltiplos tipos — o que remove toda a proteção do TypeScript — ou criar funções separadas para cada tipo, o que duplica código. A união é o mecanismo que preserva a verificação sem forçar a separação.

---

### *Narrowing*

Ao declarar uma união, o TypeScript não permite que o valor seja usado como se fosse apenas um dos tipos sem antes verificar qual ele é. Essa verificação se chama ***narrowing*** (estreitamento de tipo).

O mecanismo mais direto é o `typeof`:

```typescript
function process(value: number | string): string {
  if (typeof value === 'number') return value.toFixed(2);

  return value.toUpperCase();
};
```

**Finalidade.** `typeof` inspeciona o tipo de um valor em tempo de execução. Dentro de um bloco condicional com `typeof`, o TypeScript estreita o tipo da variável para o tipo verificado, permitindo o uso de métodos específicos daquele tipo.

**Motivação.** `toFixed` existe em `number` mas não em `string`. `toUpperCase` existe em `string` mas não em `number`. Ao tentar chamar `value.toFixed(2)` sem o `if`, o TypeScript emitiria erro — porque `value` poderia ser `string`, e `string` não tem `toFixed`. O *narrowing* é o mecanismo que prova ao compilador, dentro de um bloco específico, que o tipo é o declarado.

**Decisão.** O curso adota `typeof` como mecanismo primário de *narrowing* para uniões de tipos primitivos. Para uniões de objetos, outros mecanismos de *narrowing* serão apresentados quando necessários.

**Localização.** O *narrowing* com `typeof` aparece em funções que recebem parâmetros de união e precisam executar operações específicas de cada tipo — verificação antes do uso é o padrão obrigatório sempre que a variável for de tipo union.

**Contrafactual.** Sem *narrowing*, chamar qualquer método específico de um dos tipos da união gera erro de verificação. E sem a união em si — usando `any` no lugar — o TypeScript aceitaria o código sem reclamar, mas toda a proteção seria perdida: uma `string` poderia chegar onde apenas `number` faz sentido, e o erro só apareceria em execução.

---

## Bloco 6 — Interseções

### A intuição

A união diz *"ou"*: o valor é do tipo A ou do tipo B. A **interseção** diz *"e"*: o valor tem todas as propriedades do tipo A e todas as propriedades do tipo B simultaneamente.

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

**Finalidade.** O operador `&` compõe um novo tipo que reúne todas as propriedades de dois ou mais tipos existentes. O tipo resultante é mais restritivo que qualquer um dos tipos originais individualmente.

**Motivação.** Em sistemas reais, muitas entidades compartilham campos comuns — *timestamps* de criação e atualização, por exemplo. Em vez de repetir esses campos em cada tipo, declara-se uma vez e compõe-se via `&` onde forem necessários. A mudança se propaga automaticamente para todos os tipos compostos.

**Decisão.** A alternativa à interseção é repetir manualmente as propriedades em cada tipo que as necessita. A interseção é preferível porque centraliza a definição e garante que todos os tipos compostos permaneçam sincronizados quando um dos tipos-base for alterado.

**Localização.** Interseções aparecem quando uma entidade do domínio precisa satisfazer mais de um contrato de tipo — por exemplo, um produto que também carrega dados de registro, ou um usuário que também carrega dados de sessão. O tipo composto é usado nos pontos onde a entidade completa é necessária.

**Contrafactual.** Sem interseções, as propriedades seriam repetidas em cada tipo que as necessitasse. Uma mudança nesses campos — renomear ou alterar o tipo — exigiria atualização manual em cada ocorrência.

---

## Bloco 7 — Literal Types e `as const`

### O problema

`enum` não será utilizado neste curso por conta da configuração `erasableSyntaxOnly` — ele gera código JavaScript e não pode ser removido por *type stripping*. Mas `enum` existe por uma razão legítima: restringir um valor a um conjunto fixo de opções. Um estado de pedido não é qualquer `string` — é exatamente `'pending'`, `'sent'` ou `'delivered'`. **Literal types** e `as const` são os substitutos adotados.

---

### Literal types

Um **literal type** restringe um valor a uma opção específica. Combinado com união, forma um conjunto fechado de valores permitidos:

```typescript
type Status = 'pending' | 'sent' | 'delivered';

const firstOrder: Status = 'sent';
const secondOrder: Status = 'canceled'; // Type '"canceled"' is not assignable to type 'Status'.
```

**Finalidade.** Um literal type restringe o tipo de uma variável a um valor exato — não apenas `string`, mas a `string` específica `'pending'`. Combinado com `|`, forma um conjunto fechado de valores válidos verificado em tempo de compilação.

**Motivação.** Sem literal types, `firstOrder` e `secondOrder` seriam `string` — qualquer `string` passaria. Com literal types, o TypeScript rejeita na verificação qualquer valor fora do conjunto. Erros de digitação são capturados antes da execução.

**Decisão.** A alternativa seria usar `enum`, que gera código JavaScript em tempo de execução — incompatível com a configuração `erasableSyntaxOnly` adotada no curso. Literal types com união são equivalentes funcionais de `enum` sem geração de código: existem apenas como instrução para o verificador de tipos.

**Localização.** Literal types aparecem em campos que representam estado, categoria ou papel: estado de um pedido, tipo de usuário, modo de operação. São declarados como *type aliases* e referenciados nos tipos de objeto onde o campo correspondente aparece.

**Contrafactual.** Sem literal types, o campo `status` seria `string` — o TypeScript aceitaria qualquer valor, incluindo `'cancelado'` em vez de `'canceled'`, sem emitir erro. A verificação exaustiva de estados também seria perdida: o compilador não detectaria que um `switch` deixou de tratar um dos casos possíveis.

---

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

**Finalidade.** `as const` é uma instrução para o verificador de tipos que preserva os literais exatos de todas as propriedades de um objeto ou array, impedindo que o TypeScript amplie os tipos para `string`, `number` ou `boolean`.

**Motivação.** Sem `as const`, um objeto usado como fonte de constantes perde o valor dos literais para o verificador — `STATUS.PENDING` passa a ser `string`, e não `'pending'`. A capacidade de usar os valores do objeto como tipos depende diretamente dessa preservação.

**Decisão.** `as const` é preferível a `enum` neste curso porque não gera código JavaScript em tempo de execução — é apenas uma instrução para o verificador de tipos. O objeto `STATUS` já existiria como JavaScript válido de qualquer forma; `as const` apenas adiciona precisão tipográfica sem custo de compilação. O *type stripping* remove apenas as anotações, e o objeto permanece intacto.

**Localização.** `as const` é aplicado a objetos de constantes nomeadas que servem como fonte de valores para literal types. O padrão típico é declarar o objeto com `as const`, derivar o tipo com `typeof` e `keyof`, e usar o tipo derivado nas anotações do domínio.

**Contrafactual.** Sem `as const`, o TypeScript infere `STATUS.PENDING` como `string`. A capacidade de usar os valores como literal types é perdida, assim como a verificação exaustiva — o compilador não consegue detectar que um `switch` sobre `Status` deixou de tratar o caso `'delivered'`.
