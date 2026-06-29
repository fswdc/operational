# Parte 2

## Bloco 1 — As Três Camadas e Suas Responsabilidades

### O problema da responsabilidade única

Uma API REST com múltiplas entidades — clientes, faturas, usuários — tende a crescer rapidamente. Se cada rota mistura extração de dados da requisição, regras de negócio e formatação de resposta HTTP em um único arquivo, o resultado é um arquivo que acumula responsabilidades incompatíveis e se torna progressivamente difícil de manter.

O **Padrão MVC** resolve esse problema dividindo o processamento de uma requisição em três camadas com responsabilidades exclusivas e não sobreponíveis: **Routes**, **Controllers** e **Services**.

---

### Routes

A camada de **Routes** é responsável exclusivamente por mapear combinações de método HTTP e caminho de URL para funções de controller. Não contém lógica de negócio, não acessa dados e não formata respostas.

A aplicação de middlewares — como validação de entrada e verificação de autenticação — também é responsabilidade desta camada, posicionados como argumentos intermediários no registro da rota.

---

### Controllers

A camada de **Controllers** opera na fronteira entre o protocolo HTTP e a lógica de negócio. Suas responsabilidades são três: extrair os dados relevantes da requisição — parâmetros de caminho em `request.params`, corpo em `request.body`, cabeçalhos quando necessário —, invocar o método correspondente do service com esses dados, e formatar a resposta HTTP com o resultado recebido.

O controller não executa lógica de negócio e não acessa dados diretamente. Ele traduz o mundo HTTP para chamadas de service, e o retorno do service de volta para respostas HTTP.

---

### Services

A camada de **Services** contém a lógica de negócio da aplicação. Recebe dados primitivos — strings, números, objetos planos — e devolve dados primitivos. Não conhece o protocolo HTTP: não recebe objetos `request` ou `response`, não chama `res.json()` e não importa nada do Express.

Quando uma operação falha — recurso não encontrado, conflito de dados —, o service lança um erro. Não devolve `null`, não devolve objetos com campo `sucesso`. O Express 5 captura automaticamente erros lançados em handlers assíncronos e os encaminha para o middleware global de tratamento de erros, dispensando `try/catch` nos controllers.

---

## Bloco 2 — Disciplina Estrita de Dependências

### Verificabilidade pelos imports

A disciplina de separação entre camadas é verificável diretamente pelos imports de cada arquivo, sem necessidade de inspecionar a lógica interna.

Um arquivo `*Service.ts` correto não contém nenhum import proveniente de `'express'`. Um arquivo `*Controller.ts` correto não importa o Prisma Client. Um arquivo `*Router.ts` correto não importa serviços diretamente.

Essa verificabilidade é uma propriedade arquitetural deliberada: qualquer violação da disciplina é imediatamente visível no topo do arquivo.

---

### Regras de dependência entre camadas

A direção das dependências é unidirecional e descendente:

- Routes importam Controllers.
- Controllers importam Services.
- Services não importam nenhuma das outras duas camadas.

As restrições explícitas são:

- **Controllers** nunca importam o Prisma Client. O acesso a dados é responsabilidade exclusiva dos Services.
- **Services** nunca importam `Request`, `Response`, `NextFunction` ou qualquer outro tipo do Express. A camada de negócio é agnóstica ao protocolo de transporte.
- **Services** nunca chamam `res.status()`, `res.json()`, `res.send()`, `res.cookie()` ou `res.redirect()`.

---

### Tratamento de erros entre camadas

Quando uma operação de service não pode ser concluída — cliente inexistente, dado conflitante —, o service lança uma instância de `Error`. O controller não precisa interceptar esse erro: o Express 5 o captura automaticamente e o encaminha para o middleware global de erros, que o traduz em uma resposta HTTP com status adequado.

---

## Bloco 3 — Estrutura de Pastas e Convenção de Nomenclatura

### Organização dos diretórios

O backend do projeto é organizado em três diretórios dentro de `src/`, cada um correspondendo a uma camada:

```
src/
├── controllers/
│   └── customer.controller.ts
├── mocks/
│   └── customer.mock.ts
├── routes/
│   └── customer.router.ts
├── services/
│   └── customer.service.ts
├── types.ts
└── server.ts
```

---

### Convenção de sufixos

Cada entidade do projeto gera três arquivos, um por camada, com sufixos fixos:

- `customer.controller.ts` — camada de controller.
- `customer.service.ts` — camada de service.
- `customer.router.ts` — camada de routes.

A consistência de sufixos permite identificar a responsabilidade de um arquivo pelo nome, sem necessidade de abri-lo.

---

## Bloco 4 — Tipos de Domínio em `types.ts`

### Centralização de tipos

Os tipos de domínio do projeto residem em `src/types.ts`, separados dos arquivos de camada. Essa separação permite que qualquer camada importe os tipos sem criar dependência indireta entre camadas.

```typescript
export type Customer = {
    id: number;
    name: string;
    email: string;
    status: boolean;
};

export type CreateCustomer = Omit<Customer, 'id' | 'status'>;
export type CustomerWithoutId = Omit<Customer, 'id'>;
export type UpdateCustomer = Partial<CustomerWithoutId>;
```

---

### `Customer`

**Finalidade.** `Customer` é o tipo que representa um cliente completo no sistema, com todos os campos que um objeto de cliente possui em memória e, futuramente, no banco de dados.

**Motivação.** Sem um tipo centralizado para `Customer`, cada camada descreveria a forma do objeto de maneira independente, criando divergências silenciosas entre o que o service produz e o que o controller consome.

**Decisão.** O tipo é declarado com `type` e não com `interface`. As duas construções são funcionalmente equivalentes para objetos planos; o projeto adota `type` como padrão consistente.

**Localização.** Importado pelo mock, pelo service e pelo controller sempre que uma função recebe ou devolve um objeto de cliente.

**Contrafactual.** Sem o tipo, o TypeScript inferiria `any` para os objetos do array de mock, desabilitando todas as verificações estáticas sobre propriedades e tipos de campo.

---

### `CreateCustomer`

**Finalidade.** `CreateCustomer` é o tipo dos dados recebidos para a criação de um cliente, derivado de `Customer` pela remoção de `id` e `status` via `Omit<Customer, 'id' | 'status'>`.

**Motivação.** O `id` é gerado internamente pelo service e o `status` é definido como `true` por regra de negócio — clientes são sempre criados ativos. Nenhum dos dois deve ser aceito como entrada externa. O tipo torna essa restrição verificável em tempo de compilação.

**Decisão.** `Omit<Customer, 'id' | 'status'>` deriva o tipo automaticamente de `Customer`, garantindo que qualquer campo adicionado a `Customer` no futuro — exceto `id` e `status` — passe a ser exigido em `CreateCustomer` sem alteração manual.

**Localização.** Usado como tipo do parâmetro da função `insertCustomer` no service e como tipo da asserção em `request.body` no controller de criação.

**Contrafactual.** Sem a exclusão de `status`, o controller poderia receber `status: false` do cliente HTTP e criar um cliente já inativo — transferindo para o requisitante uma decisão que é regra de negócio do service.

---

### `CustomerWithoutId` e `UpdateCustomer`

**Finalidade.** `CustomerWithoutId` é `Customer` sem o campo `id`. `UpdateCustomer` é `Partial<CustomerWithoutId>`, tornando todos os campos de cliente — exceto `id` — opcionais.

**Motivação.** Em uma operação de atualização parcial, nem todos os campos precisam ser enviados. `Partial` expressa essa opcionalidade no nível do tipo, evitando a necessidade de verificações manuais de presença de campo no service.

**Decisão.** A separação em dois tipos — `CustomerWithoutId` e `UpdateCustomer` — permite reutilizar `CustomerWithoutId` em outros contextos que exijam um cliente sem `id` mas com todos os demais campos obrigatórios, sem forçar o uso de `Partial` nesses casos.

**Localização.** `UpdateCustomer` é usado como tipo do segundo parâmetro da função `modifyCustomer` no service.

**Contrafactual.** Sem `Partial`, a atualização exigiria o envio de todos os campos em toda requisição `PUT`, mesmo quando apenas um campo deve ser alterado.

---

## Bloco 5 — O Service: `customer.service.ts`

### Dados em memória como substituto temporário do banco

```typescript
import { customers } from '../mocks/customer.mock.ts';
```

**Finalidade.** Esta instrução importa o array `customers` do arquivo de mock, que serve como banco de dados em memória enquanto o PostgreSQL e o Prisma não foram introduzidos.

**Motivação.** A arquitetura de três camadas pode ser estabelecida e verificada antes da introdução do banco de dados. O service opera sobre um array com a mesma interface que operará sobre o Prisma Client — recebe dados primitivos e devolve objetos tipados — sem que o controller precise saber a origem dos dados.

**Decisão.** O array é importado de um arquivo dedicado em `mocks/` em vez de declarado diretamente no service. Essa separação permite substituir o mock por outro conjunto de dados sem alterar o service.

**Localização.** Referenciado por todas as funções do service. Quando o Prisma Client substituir o mock, esta importação e as operações sobre o array serão removidas.

**Contrafactual.** Se o array fosse declarado diretamente no service, a substituição pelo Prisma exigiria alterações em múltiplos pontos do mesmo arquivo, aumentando o risco de regressão.

---

### `findAllCustomers`

```typescript
export function findAllCustomers(): Customer[] {
    return customers;
}
```

**Finalidade.** `findAllCustomers` devolve o array completo de clientes.

**Motivação.** A listagem de todos os recursos de uma entidade é a operação mais comum em uma API REST. Encapsulá-la no service mantém o controller desacoplado da origem dos dados.

**Decisão.** O array é devolvido diretamente, sem cópia. Em produção com Prisma, o equivalente será `prisma.customer.findMany()`, cujo resultado também é devolvido diretamente.

**Localização.** Chamada pelo controller na função que trata `GET /customers`.

**Contrafactual.** Se o controller acessasse o array diretamente, qualquer mudança na origem dos dados — de array em memória para banco de dados — exigiria alteração no controller, violando a separação de camadas.

---

### `findCustomerById`

```typescript
export function findCustomerById(id: number): Customer {
    const customer = customers.find((customer) => customer.id === id);

    if (!customer) {
        throw new Error(`Cliente com id ${id} não encontrado.`);
    }

    return customer;
}
```

**Finalidade.** `findCustomerById` localiza e devolve o cliente com o `id` informado, ou lança um erro se não existir.

**Motivação.** O controller que trata `GET /customers/:id` precisa de um cliente específico. O service encapsula a lógica de busca e a decisão sobre o que fazer quando o recurso não existe.

**Decisão.** O parâmetro `id` é do tipo `number`, não `string`. A conversão de `string` para `number` é responsabilidade do controller, que extrai o parâmetro de `request.params` como string e converte antes de chamar o service. Quando o cliente não é encontrado, a função lança `Error` em vez de retornar `undefined`. Retornar `undefined` transferiria para o controller a decisão sobre o status HTTP a responder — misturando lógica de negócio com lógica de protocolo.

**Localização.** Chamada pelo controller nas funções que tratam `GET /customers/:id`.

**Contrafactual.** Se a função retornasse `undefined` em vez de lançar, o controller receberia `undefined` silenciosamente e responderia com `res.json(undefined)` — status `200` com body vazio, como se o recurso existisse.

---

### `insertCustomer`

```typescript
export function insertCustomer({ name, email }: CreateCustomer): Customer {
    const id = customers[customers.length - 1].id;

    const customer: Customer = {
        id: id + 1,
        name,
        email,
        status: true,
    };

    customers.push(customer);

    return customer;
}
```

**Finalidade.** `insertCustomer` cria um novo objeto de cliente, adiciona ao array em memória e devolve o objeto criado.

**Motivação.** A criação de um recurso exige a geração de um identificador único e a definição de campos com valores padrão determinados por regra de negócio. Ambas as responsabilidades pertencem ao service.

**Decisão.** O parâmetro usa desestruturação direta com o tipo `CreateCustomer`, que exclui `id` e `status` — os campos gerados internamente. O `status` é sempre definido como `true` porque clientes são criados ativos por regra de negócio; essa decisão não é delegada ao requisitante. O `id` é calculado a partir do `id` do último elemento do array. Esta abordagem tem uma limitação conhecida: se elementos forem removidos, pode haver colisão de `id`. A limitação é aceitável para dados em memória temporários — quando o Prisma substituir o array, a geração de `id` será responsabilidade do banco.

**Localização.** Chamada pelo controller na função que trata `POST /customers`.

**Contrafactual.** Se o controller recebesse `status` do body e o passasse ao service, o requisitante poderia criar clientes inativos diretamente — transferindo para a entrada externa uma decisão que é regra de negócio interna.

---

### `modifyCustomer`

```typescript
export function modifyCustomer(
    id: number,
    { name, email, status }: UpdateCustomer
): Customer {
    const customer = customers.find((customer) => customer.id === id);

    if (!customer) {
        throw new Error(`Cliente com id ${id} não encontrado.`);
    }

    if (name) customer.name = name;
    if (email) customer.email = email;
    if (status !== undefined) customer.status = status;

    return customer;
}
```

**Finalidade.** `modifyCustomer` localiza o cliente com o `id` informado, atualiza os campos presentes no objeto de atualização e devolve o objeto modificado.

**Motivação.** A atualização parcial — onde apenas os campos enviados são alterados — exige verificação individual de presença de cada campo antes de aplicar a modificação.

**Decisão.** Os campos `name` e `email` são verificados com `if (name)` e `if (email)`, aproveitando que strings vazias são falsy — uma string vazia não deve substituir um valor existente. O campo `status` usa `if (status !== undefined)` porque `false` é falsy: verificar `if (status)` impediria a desativação de um cliente, já que `status: false` seria tratado como ausência do campo. A distinção entre *campo ausente* (`undefined`) e *campo com valor `false`* exige a comparação explícita com `undefined`. A modificação ocorre diretamente no objeto referenciado pelo array, aproveitando a mutabilidade por referência do JavaScript — sem necessidade de remover e reinserir o elemento.

**Localização.** Chamada pelo controller na função que trata `PUT /customers/:id`.

**Contrafactual.** Usar `if (status)` em vez de `if (status !== undefined)` faria com que uma requisição com `{ status: false }` não alterasse o campo — o cliente permaneceria ativo após uma tentativa explícita de desativação.

---

### `removeCustomer`

```typescript
export function removeCustomer(id: number): void {
    const index = customers.findIndex((customer) => customer.id === id);

    if (index === -1) {
        throw new Error(`Cliente com id ${id} não encontrado.`);
    }

    customers.splice(index, 1);
}
```

**Finalidade.** `removeCustomer` localiza o cliente com o `id` informado pelo índice no array e o remove. Lança erro se o cliente não existir.

**Motivação.** A remoção de um elemento de array requer o índice da posição, não a referência ao elemento. `findIndex` é o método adequado porque devolve o índice — diferente de `find`, que devolve o elemento.

**Decisão.** `findIndex` devolve `-1` quando nenhum elemento satisfaz o predicado. A verificação `if (index === -1)` precisa ocorrer antes de `splice`: chamar `splice(-1, 1)` sem verificação removeria silenciosamente o último elemento do array, comportamento incorreto e sem erro visível. O retorno é `void` porque a exclusão bem-sucedida não tem dado a devolver — o controller responde com `204 No Content`.

**Localização.** Chamada pelo controller na função que trata `DELETE /customers/:id`.

**Contrafactual.** Omitir a verificação de `index === -1` faria com que a tentativa de remover um cliente inexistente removesse o último cliente do array, corrompendo os dados silenciosamente.

---

## Bloco 6 — O Controller: `customer.controller.ts`

### Imports do controller

```typescript
import type { Request, Response } from 'express';
import * as CustomerService from '../services/customer.service.ts';
import type { CreateCustomer } from '../types.ts';
```

**`import type { Request, Response } from 'express'`**

**Finalidade.** Esta instrução importa os tipos `Request` e `Response` do Express exclusivamente para uso em anotações de tipo, sem incluir nenhum valor em runtime.

**Motivação.** Sem essas anotações, os parâmetros `request` e `response` das funções do controller seriam inferidos como `any`, desabilitando todas as verificações estáticas sobre o acesso a `request.params`, `request.body` e os métodos de `response`.

**Decisão.** O uso de `import type` — e não `import` simples — é exigido pela configuração `verbatimModuleSyntax: true` do `tsconfig.json`. Essa configuração proíbe importações de tipo sem o modificador `type`, pois importações de valor do Express dentro do controller introduziriam uma dependência de runtime desnecessária.

**Localização.** Os tipos `Request` e `Response` são usados como anotações em todos os parâmetros `request` e `response` das funções do controller.

**Contrafactual.** Usar `import { Request, Response }` sem o modificador `type` viola a configuração `verbatimModuleSyntax` e produz erro de compilação.

---

**`import * as CustomerService from '../services/customer.service.ts'`**

**Finalidade.** Esta instrução importa todas as funções exportadas do service como propriedades de um único objeto nomeado `CustomerService`.

**Motivação.** O controller precisa invocar as funções do service. Esta é a única importação de outro módulo de camada permitida em um controller — além de tipos.

**Decisão.** O namespace `import * as CustomerService` é preferido à desestruturação individual das funções porque torna explícita a origem de cada chamada: `CustomerService.findAllCustomers()` identifica imediatamente que a operação pertence ao service, sem ambiguidade em arquivos que eventualmente cresçam.

**Localização.** Referenciado em todas as cinco funções do controller.

**Contrafactual.** Um caminho sem a extensão `.ts` — como `'../services/customer.service'` — impede a resolução do módulo pelo Node.js 24 e produz erro em runtime.

---

### Extração de dados da requisição

O controller é responsável por extrair todos os dados da requisição antes de chamar o service. Há dois pontos de extração principais: `request.params` para parâmetros de caminho e `request.body` para o corpo da requisição.

---

#### Conversão de parâmetro de caminho

```typescript
const id = Number(request.params.id);
```

**Finalidade.** Esta instrução extrai o parâmetro `id` do caminho da URL e o converte de `string` para `number`.

**Motivação.** O Express disponibiliza todos os parâmetros de caminho como strings em `request.params`, independentemente do valor original. A função `findCustomerById` do service espera `number`. A conversão é responsabilidade do controller — a fronteira entre o protocolo HTTP e o domínio tipado.

**Decisão.** `Number(string)` converte a string para número. Se a string não representar um número válido — como `"abc"` —, `Number` devolve `NaN`. O tratamento explícito de `NaN` será introduzido na aula de validação com Zod; por ora, um `id` inválido resulta em `NaN`, que não casa com nenhum `id` do array, e o service lança o erro de recurso não encontrado.

**Localização.** Presente nas funções `getCustomerById`, `updateCustomer` e `deleteCustomer` — todas as que recebem `id` pelo caminho da URL.

**Contrafactual.** Passar `request.params.id` diretamente ao service sem conversão faria a comparação `customer.id === "3"` — comparação entre `number` e `string` que nunca é verdadeira em TypeScript com verificação estrita, e resultaria em nenhum cliente sendo encontrado.

---

#### Extração do corpo da requisição

```typescript
const { name, email } = request.body as CreateCustomer;
```

**Finalidade.** Esta instrução extrai os campos `name` e `email` do corpo da requisição e afirma ao TypeScript que o objeto tem a forma de `CreateCustomer`.

**Motivação.** O corpo de uma requisição chega como tipo `any` em `request.body` porque o Express não conhece a estrutura esperada. A asserção de tipo instrui o TypeScript a tratar o objeto como `CreateCustomer`, habilitando verificações nos campos extraídos.

**Decisão.** O `as CreateCustomer` é uma asserção de tipo, não uma validação em runtime. O TypeScript aceita a asserção sem verificar o conteúdo real do objeto. Se o corpo chegar sem o campo `name`, o valor será `undefined` — e o service receberá `undefined` onde espera `string`. A validação real do corpo será implementada com Zod na aula seguinte; por ora, a asserção é suficiente para o desenvolvimento da arquitetura.

**Localização.** Presente na função `createCustomer`. A função `updateCustomer` realiza extração análoga com o tipo inline `{ name: string; email: string; status: boolean }`.

**Contrafactual.** Sem a asserção, `request.body` permanece como `any`, e a desestruturação compila sem verificações — erros de propriedade ausente ou de tipo incorreto só seriam detectados em runtime.

---

### Formatação da resposta

Após invocar o service, o controller formata e envia a resposta HTTP com `response.status().json()` ou `response.status().send()`.

```typescript
response.status(201).json(customer);
```

O encadeamento de `res.status()` com `res.json()` é idiomático no Express: `res.status()` define o código de status e devolve o próprio objeto `res`, permitindo a chamada imediata de `res.json()`.

Para a exclusão de recursos, a resposta não contém body:

```typescript
response.status(204).send();
```

**Finalidade.** `response.status(204).send()` envia uma resposta com status `204 No Content` e sem corpo.

**Motivação.** O status `204` é a resposta semanticamente correta para exclusões bem-sucedidas: confirma a operação sem dados a devolver.

**Decisão.** `send()` é usado em vez de `json()` porque `json()` sempre serializa um valor e define o cabeçalho `Content-Type: application/json`. Uma resposta `204` não deve ter corpo nem cabeçalho de conteúdo.

**Localização.** Exclusivo da função `deleteCustomer`.

**Contrafactual.** Chamar `response.status(204).json({})` enviaria um corpo JSON vazio — tecnicamente inconsistente com a semântica do `204`, que define a ausência de corpo como parte do contrato.

---

## Bloco 7 — O Router: `customer.router.ts`

### Criação do router

```typescript
import { Router } from 'express';
import * as CustomerController from '../controllers/customer.controller.ts';

const router = Router();
```

**`import { Router } from 'express'`**

**Finalidade.** Esta instrução importa a função `Router` do Express para criar um objeto de roteamento isolado.

**Motivação.** O router agrupa as rotas de uma entidade em um objeto independente, que pode ser montado no `app` com um prefixo de caminho. Sem ele, todas as rotas precisariam ser registradas diretamente no objeto `app` do `server.ts`.

**Decisão.** A importação desestruturada `{ Router }` é preferida a `import express from 'express'` seguido de `express.Router()` porque importa apenas o necessário para o arquivo — o router não usa nenhum outro recurso do Express.

**Localização.** Usado exclusivamente na linha de criação do objeto `router`.

**Contrafactual.** Sem o router, as rotas de `customers` precisariam ser declaradas no `server.ts`, misturando a configuração de entidades distintas em um único arquivo e dificultando a manutenção à medida que o projeto cresce.

---

**`const router = Router()`**

**Finalidade.** Esta instrução cria o objeto de roteamento que agrupa todas as rotas da entidade `customers`.

**Motivação.** O router permite montar um grupo de rotas com um prefixo comum no `server.ts`, de modo que os caminhos declarados internamente — como `'/'` e `'/:id'` — sejam interpretados relativamente ao prefixo de montagem.

**Decisão.** `Router()` é chamado sem argumentos. A opção `{ strict: true }`, que diferencia `/customers/` de `/customers`, não é necessária para o projeto.

**Localização.** Todas as declarações de rota do arquivo usam o objeto `router`. Ao final do arquivo, `router` é exportado para montagem no `server.ts`.

**Contrafactual.** Registrar rotas diretamente no `app` importado do `server.ts` criaria dependência circular entre arquivos e impossibilitaria a organização por entidade.

---

### Mapeamento de rotas

```typescript
router.get('/', CustomerController.getAllCustomers);
router.get('/:id', CustomerController.getCustomerById);
router.post('/', CustomerController.createCustomer);
router.put('/:id', CustomerController.updateCustomer);
router.delete('/:id', CustomerController.deleteCustomer);
```

**Finalidade.** Estas instruções registram no router as associações entre métodos HTTP, caminhos relativos e funções de controller.

**Motivação.** O router não sabe o que cada função de controller faz — apenas para onde encaminhar cada combinação de método e caminho. Essa separação permite alterar a lógica de uma operação sem modificar o registro da rota.

**Decisão.** O segundo argumento de cada chamada é a referência à função do controller — sem parênteses. Passar `CustomerController.getAllCustomers()` com parênteses chamaria a função imediatamente durante o registro, não quando a requisição chegasse. O Express armazena a referência e a invoca no momento correto.

**Localização.** Quando o router for montado no `server.ts` com o prefixo `'/customers'`, os caminhos `'/'` e `'/:id'` serão interpretados como `/customers` e `/customers/:id`, respectivamente.

**Contrafactual.** A ordem dos registros importa quando dois padrões podem casar com a mesma URL. O padrão `'/:id'` casaria com qualquer segmento, incluindo caminhos como `'/stats'` se houvesse uma rota nesse caminho. Registrar a rota mais específica antes da mais genérica garante que o Express encontre o match correto.

---

### Exportação do router

```typescript
export default router;
```

**Finalidade.** Esta instrução exporta o objeto `router` como exportação padrão do módulo.

**Motivação.** Sem exportação, o objeto `router` é local ao arquivo e inacessível para o `server.ts`, onde será montado.

**Decisão.** `export default` é usado em vez de `export const router` porque o arquivo exporta um único objeto. A exportação padrão é a convenção para módulos com responsabilidade única.

**Localização.** Importado no `server.ts` como `import CustomerRouter from './routes/customer.router.ts'`.

**Contrafactual.** A ausência de exportação não produz erro de compilação — o arquivo é válido sem exportações. O erro só aparece em runtime, quando a importação no `server.ts` resulta em um objeto vazio e nenhuma rota de `customers` responde.

---

## Bloco 8 — Montagem no `server.ts`

### Estado do `server.ts` após a refatoração

```typescript
import express from 'express';
import CustomerRouter from './routes/customer.router.ts';

const app = express();

app.use(express.json());

app.use('/customers', CustomerRouter);

app.use((_request, response) => {
    response.status(404).json({
        message: 'Not found!'
    });
});

app.listen(Number(process.env.PORT));
```

O `server.ts` passa a ter responsabilidade exclusiva de configuração global: registro de middlewares e montagem de routers. A lógica de cada entidade migra para os respectivos arquivos de camada.

---

### Montagem do router

```typescript
app.use('/customers', CustomerRouter);
```

**Finalidade.** Esta instrução monta o `CustomerRouter` no prefixo `/customers`, fazendo com que toda requisição cuja URL começa com `/customers` seja encaminhada ao router.

**Motivação.** Sem a montagem, o router existe em memória mas nunca é acionado. O Express não descobre routers automaticamente — cada um precisa ser registrado explicitamente no `app`.

**Decisão.** O prefixo `/customers` é declarado aqui, no `server.ts`, e não dentro do router. Essa separação permite que o mesmo router seja reutilizado sob um prefixo diferente sem modificação interna. A responsabilidade de *onde* um grupo de rotas vive na API pertence ao ponto de montagem.

**Localização.** Deve ser posicionado após `app.use(express.json())`. O Express processa middlewares e rotas na ordem de registro; o middleware de parse de JSON precisa executar antes dos handlers que leem `request.body`.

**Contrafactual.** Omitir esta linha faz com que todas as requisições para `/customers/*` cheguem ao handler de `404` no final do arquivo — o router existe mas é invisível para o Express.
