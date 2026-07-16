# Parte 1

## Bloco 1 — O que é um Middleware

### O problema do comportamento compartilhado entre rotas

Em uma aplicação Express organizada em camadas, existe uma categoria de comportamento que não pertence a nenhuma rota específica — aplica-se à totalidade das rotas, ou a um subconjunto delas. Registrar informações de cada requisição recebida, verificar a presença de um dado obrigatório no corpo da requisição, ou capturar um erro lançado em qualquer ponto do processamento são exemplos dessa categoria. Sem um mecanismo dedicado a esse tipo de responsabilidade transversal, cada controller precisaria repetir a mesma lógica individualmente, duplicando código entre arquivos que deveriam permanecer independentes.

---

### Definição de middleware

Um **middleware**, em Express, é uma função posicionada entre o momento em que uma requisição chega ao servidor e o momento em que a resposta final é enviada. Sua assinatura declara três parâmetros: `request`, `response` e `next`. Os dois primeiros já são conhecidos da camada de controller — carregam, respectivamente, os dados da requisição recebida e o objeto usado para construir a resposta.

O terceiro parâmetro, `next`, é uma função que o próprio Express constrói e entrega a cada middleware. Chamada sem argumentos, `next()` instrui o Express a passar o controle para o próximo elemento da fila de processamento — outro middleware, ou o handler final da rota. Middlewares registrados formam uma fila, na ordem exata de registro: uma requisição percorre essa fila do início ao fim, avançando de um elemento para o próximo a cada chamada de `next()`.

---

### Consequência da ausência de resolução do ciclo

Se uma função de middleware não chamar `next()` e também não enviar nenhuma resposta — nenhum `response.json()`, nenhum `response.send()`, nenhum `response.status().json()` — a requisição permanece suspensa indefinidamente. O cliente que originou a chamada aguarda uma resposta que nunca chega, sem que exista mecanismo automático de tempo limite por parte do Express. Todo middleware resolve o ciclo da requisição de exatamente uma entre duas formas possíveis: chamando `next()` para prosseguir, ou enviando uma resposta para encerrá-lo ali.

Essa disciplina se aplica igualmente a middlewares que decidem condicionalmente prosseguir ou responder, e a middlewares que sempre prosseguem, observando a requisição sem nunca interrompê-la — categoria à qual pertence o middleware construído no Bloco 3.

---

## Bloco 2 — Middlewares Globais e de Rota

### Alcance como propriedade central do middleware

Nem todo middleware precisa se aplicar ao mesmo conjunto de requisições. Um middleware pode ser registrado para intervir sobre a totalidade das requisições recebidas pelo servidor, ou restrito a uma rota específica, sem afetar as demais. Essa distinção de alcance determina o posicionamento de cada regra transversal no código.

---

### Middleware global

Um middleware é registrado como global por meio de `app.use(funcao)`. A partir do ponto em que essa instrução é executada, toda requisição recebida pelo servidor passa primeiro por essa função — independentemente de rota ou método HTTP — desde que a rota final tenha sido registrada depois dessa linha no arquivo. O Express processa os registros na ordem em que aparecem no arquivo, de cima para baixo; um `app.use()` posicionado após uma rota não intercepta requisições que já tenham correspondido a essa rota.

---

### Middleware de rota

Um middleware pode também ser restrito a um subconjunto de rotas. A primeira forma consiste em passá-lo como argumento intermediário entre o caminho e o handler final: `router.post('/', middlewareEspecifico, handlerFinal)`. O Express aceita múltiplas funções nessa posição e as executa em sequência. A segunda forma é `router.use(funcao)` dentro de um arquivo de router específico, aplicando o middleware a todas as rotas daquele router, sem afetar rotas montadas em outros pontos da aplicação.

---

### Critério de escolha entre alcance global e de rota

Um comportamento que se aplica a qualquer requisição — como o registro de um log de acesso — é candidato a middleware global. Um comportamento restrito a uma operação específica — como a verificação de um dado obrigatório no momento de criar um registro — é candidato a middleware de rota. Sem essa distinção de alcance, cada middleware precisaria conter, internamente, uma verificação condicional sobre a natureza da requisição, reintroduzindo a duplicação que o conceito de middleware existe para eliminar.

---

## Bloco 3 — Primeiro Middleware Próprio: Log Manual de Requisições

### Motivação de um middleware de log

Sem um mecanismo de observação centralizado, não há registro sistemático de que uma requisição chegou ao servidor, além do que cada controller reportasse manualmente. Como esse comportamento se aplica a qualquer requisição, independentemente da rota, sua implementação corresponde a um middleware global, definido em arquivo próprio dentro de `src/middlewares/`.

---

### Organização de middlewares em diretório próprio

O projeto reserva o diretório `src/middlewares/` para toda função de middleware que não pertença a nenhuma camada de domínio específica — nem controller, nem service, nem router. Essa separação segue o mesmo princípio já aplicado à organização de `controllers/`, `services/` e `routes/`: cada tipo de responsabilidade reside em seu próprio diretório, identificável pelo nome sem necessidade de abrir o arquivo.

---

### Arquivo `requestLogger.ts`

```typescript
import type { NextFunction, Request, Response } from 'express';

export default function requestLogger(
	request: Request,
	_response: Response,
	next: NextFunction
): void {
	console.log(`${request.method} ${request.originalUrl}`);

	next();
}
```

**`import type { NextFunction, Request, Response } from 'express'`**

**Finalidade**. Esta instrução importa exclusivamente os tipos `NextFunction`, `Request` e `Response` do módulo `express`, usados para tipar os três parâmetros da função de middleware.

**Motivação**. Sem esses tipos, os parâmetros da função assumiriam o tipo `any`, eliminando a verificação estática sobre quais propriedades e métodos existem em `request`, `response` e `next`.

**Decisão**. A palavra-chave `type` na importação indica que apenas informação de tipo é trazida ao arquivo, sem gerar nenhum código JavaScript correspondente após a compilação — os três identificadores existem somente durante a checagem de tipo.

**Localização**. Presente no início de todo arquivo de middleware que declare parâmetros tipados de requisição, resposta ou continuação de fluxo.

**Contrafactual**. Sem essa importação, o TypeScript não reconheceria os identificadores `Request`, `Response` e `NextFunction`, e a tentativa de usá-los como anotação de tipo produziria erro de compilação.

---

**`export default function requestLogger(request, _response, next): void`**

**Finalidade**. Esta instrução declara a função `requestLogger` e a exporta como exportação padrão do módulo, disponibilizando-a para importação direta em outro arquivo.

**Motivação**. Middlewares de arquivo único — que exportam apenas uma função — usam exportação padrão por convenção do projeto, dispensando chaves na importação.

**Decisão**. O segundo parâmetro é nomeado `_response`, com prefixo de sublinhado, porque a função nunca envia resposta — apenas observa e prossegue; a posição é obrigatória na assinatura de um middleware, mesmo quando o valor não é utilizado. O tipo de retorno `void` declara explicitamente que a função não devolve nenhum valor utilizável — sua função é produzir efeito colateral (o registro no console) e prosseguir o fluxo.

**Localização**. Importado por padrão em `server.ts` como `import requestLogger from './middlewares/requestLogger.ts'`.

**Contrafactual**. Caso o parâmetro fosse nomeado `response`, sem o prefixo, o Biome — a ferramenta de qualidade de código já configurada no projeto — sinalizaria a variável como declarada e não utilizada, já que nenhum método de `response` é chamado dentro da função.

---

**`console.log(\`${request.method} ${request.originalUrl}\`)`**

**Finalidade**. Esta instrução monta uma string com o método HTTP e a URL da requisição, interpolados por meio de um *template literal*, e a envia à saída do terminal.

**Motivação**. Sem esse registro, não há visibilidade sobre quais requisições chegam ao servidor, em que ordem e com que frequência, à medida que o número de rotas cresce.

**Decisão**. `request.originalUrl` é usado em vez de `request.url` porque preserva o caminho completo da requisição tal como recebido pelo servidor, independentemente de qualquer roteamento interno que reescreva ou fatie o caminho em etapas posteriores da cadeia.

**Localização**. Única instrução do corpo da função anterior à chamada de `next()`.

**Contrafactual**. Se `request.url` fosse usado no lugar de `request.originalUrl`, o valor registrado poderia divergir do caminho originalmente solicitado pelo cliente, caso algum middleware anterior alterasse essa propriedade.

---

**`next()`**

**Finalidade**. Esta instrução invoca a função `next`, recebida como terceiro parâmetro, sem nenhum argumento, encerrando a execução do middleware e liberando a passagem para o próximo elemento da cadeia.

**Motivação**. Como `requestLogger` apenas observa a requisição, sem nunca decidir interrompê-la, a chamada precisa ocorrer sempre — sem qualquer condição que a impeça em algum caminho de execução.

**Decisão**. A chamada é feita fora de qualquer estrutura condicional, na última linha do corpo da função, garantindo que seja alcançada em toda execução possível.

**Localização**. Última instrução do corpo da função, executada após o registro no console.

**Contrafactual**. Se essa chamada fosse removida, ou condicionada a alguma verificação, toda requisição — ou apenas aquelas que não satisfizessem a condição — ficaria suspensa indefinidamente, sem chegar ao router de clientes nem a nenhuma resposta.

---

## Bloco 4 — Express 5 e Promise Rejections

### Distinção entre lançamento síncrono e rejeição assíncrona

Um erro lançado de forma síncrona, no exato instante em que um handler é chamado, sempre foi capturado automaticamente pelo Express, em qualquer versão do framework — o Express invoca cada handler dentro de seu próprio controle interno, e um lançamento nesse instante é detectado imediatamente. Esse comportamento não é uma novidade da versão 5.

A situação é diferente quando o handler é declarado `async`. Nesse caso, a chamada ao handler retorna imediatamente uma `Promise` pendente; o erro real só se manifesta depois, no momento em que essa `Promise` é rejeitada — em um ponto posterior ao controle interno síncrono original do Express.

---

### Comportamento do Express 5

Em versões do Express anteriores à 5, esse instante posterior de rejeição não era observado automaticamente: era necessário um `try/catch` explícito dentro de cada handler `async`, com uma chamada manual a `next(error)` dentro do bloco `catch` para encaminhar o erro. Sem isso, a rejeição não tratada deixava a requisição suspensa, sem resposta.

No Express 5, esse acompanhamento passou a ser automático: o Express monitora a `Promise` retornada por um handler `async` e, caso ela seja rejeitada, encaminha o erro para o mesmo destino que já recebia erros síncronos — o middleware de tratamento de erros. Essa capacidade dispensa o `try/catch` manual dentro de handlers assíncronos.

---

### Aplicação direta no controller de clientes

As funções do controller de clientes — `getCustomerById`, `updateCustomer`, `deleteCustomer` — não declaram `try/catch` em nenhum ponto, e nenhuma delas é declarada `async`. Cada uma chama diretamente uma função do service, que pode lançar de forma síncrona. A captura automática de lançamentos síncronos, presente em qualquer versão do Express, já é suficiente para que esses erros alcancem o middleware de tratamento de erros sem nenhuma instrução adicional no controller.

---

### Sustentação da disciplina entre camadas

Essa capacidade de captura automática, tanto síncrona quanto assíncrona, é o que permite que um controller permaneça sem qualquer `try/catch`, independentemente de a chamada ao service ser síncrona ou envolver `await`. Sem essa capacidade, o controller precisaria conhecer, em detalhe, cada tipo de falha que o service pudesse produzir, o que comprometeria a separação de responsabilidades entre as camadas.

---

## Bloco 5 — Middleware de Erro com Quatro Parâmetros

### Reconhecimento por aridade de parâmetros

O Express distingue um middleware comum de um middleware de tratamento de erros exclusivamente pela quantidade de parâmetros declarados na assinatura da função — não por nome, palavra-chave ou forma de registro. Um middleware comum declara três parâmetros: `request`, `response` e `next`. Um middleware de erro declara quatro, com o objeto de erro interceptado ocupando a primeira posição.

Uma função destinada a tratar erros, mas declarada com menos de quatro parâmetros, é tratada pelo Express como middleware comum, e nunca é chamada quando um erro é interceptado.

---

### Nomenclatura do primeiro parâmetro

O primeiro parâmetro do middleware de erro do projeto é nomeado `error`, e não `err`. O nome do parâmetro é irrelevante para o reconhecimento do Express — apenas a posição e a quantidade de parâmetros importam. A escolha do nome é de legibilidade: `error` é o nome completo da palavra, sem abreviação, consistente com a preferência por nomes de variável descritos por extenso já adotada em `request` e `response`.

---

### Posição obrigatória no arquivo

Um middleware de erro só é capaz de capturar falhas provenientes de rotas já registradas antes dele na cadeia. Por essa razão, precisa ser posicionado como o último `app.use` do arquivo de entrada do servidor — depois da montagem de todos os routers. Um middleware de erro registrado antes de uma rota nunca intercepta erros provenientes dela, porque a rota ainda não existia na cadeia no momento do registro.

---

## Bloco 6 — Classes de Erro Customizadas

### Limitação do erro genérico

Um objeto `Error` nativo não carrega nenhuma informação sobre a categoria da falha que representa — do ponto de vista do tipo, é indistinguível entre um erro de domínio esperado, como um recurso inexistente, e um erro de programação não previsto. Essa ausência de identidade impede que o middleware de tratamento de erros diferencie um caso do outro.

---

### Herança da classe Error

`Error` é uma classe nativa do JavaScript e pode ser estendida como qualquer outra classe. Uma classe declarada com `extends Error` herda o comportamento nativo de um erro — pode ser lançada com `throw`, carrega uma `message` — e adiciona, além disso, uma propriedade própria que identifica a categoria da falha e o status HTTP correspondente.

---

### Arquivo `errors/index.ts`

```typescript
export class NotFoundError extends Error {
	statusCode = 404;

	constructor(message: string) {
		super(message);
	}
}
```

**`export class NotFoundError extends Error { statusCode = 404; }`**

**Finalidade**. Esta declaração cria a classe `NotFoundError`, que representa a ausência de um recurso solicitado, e define nela um campo próprio, `statusCode`, com valor fixo `404`.

**Motivação**. Sem uma classe própria para essa categoria de falha, o service não teria como comunicar que tipo específico de erro ocorreu sem depender de comparações de texto na mensagem — uma verificação frágil e sujeita a divergência entre o texto lançado e o texto verificado.

**Decisão**. `statusCode` é declarado como campo de classe, com valor atribuído diretamente, e não como parâmetro do construtor. Como toda instância de `NotFoundError` representa sempre a mesma categoria de falha, o valor `404` não varia entre instâncias, tornando desnecessário recebê-lo como argumento.

**Localização**. Exportada do arquivo `errors/index.ts`, e importada tanto pelo service — para lançar a exceção — quanto pelo middleware de tratamento de erros — para verificá-la com `instanceof`.

**Contrafactual**. Sem `statusCode` declarado como propriedade da classe, o middleware de erro não teria, a partir do próprio objeto de erro, nenhuma informação sobre qual status HTTP corresponde àquela falha, exigindo uma tabela externa de mapeamento entre tipo de erro e status.

---

**`constructor(message: string) { super(message); }`**

**Finalidade**. Este construtor recebe uma `string` de mensagem e a repassa para o construtor da classe-mãe `Error`, por meio de `super(message)`.

**Motivação**. A mensagem específica de cada ocorrência — qual cliente, com qual identificador, não foi encontrado — só existe no momento em que o erro é lançado, dentro do service; o construtor é o que permite que essa informação varie entre instâncias, ainda que `statusCode` permaneça fixo.

**Decisão**. O parâmetro `message` é tipado como `string`, obrigatório, e não opcional. Essa exigência é herdada da assinatura declarada explicitamente aqui — sem este construtor próprio, a classe herdaria a assinatura nativa de `Error`, que declara `message` como opcional, permitindo `new NotFoundError()` sem nenhum argumento.

**Localização**. Executado implicitamente a cada `new NotFoundError(mensagem)`, antes de a instância ficar disponível para uso.

**Contrafactual**. Se `super(message)` fosse omitido, o TypeScript recusaria a compilação: uma classe derivada que declara construtor próprio precisa chamar o construtor da classe-mãe antes de qualquer uso de `this`, e a propriedade `message`, herdada de `Error`, não seria inicializada corretamente.

---

## Bloco 7 — Disciplina entre Camadas: Lançar versus Responder

### A regra de disciplina

O service lança exceções; nunca responde. O controller não captura essas exceções — não há `try/catch` nessa camada. O middleware de tratamento de erros, e somente ele, traduz a exceção em uma resposta HTTP.

---

### Justificativa a partir da disciplina de dependências já estabelecida

Um service que respondesse diretamente com um status HTTP dependeria do protocolo de transporte para comunicar uma informação de domínio, violando a regra de que services não conhecem `Request`, `Response` nem qualquer tipo do Express. Lançar uma exceção — `throw new NotFoundError(...)` — comunica a falha sem que o service precise conhecer como ela será traduzida em resposta.

Do lado do controller, a captura automática de erros do Express dispensa qualquer `try/catch`: se o service lançar, o erro sobe sem que o controller escreva uma linha para tratá-lo. Concentrar a tradução exclusivamente no middleware de erro evita que a relação entre tipo de falha e status HTTP precise ser duplicada em cada controller.

---

### Aplicação nas três funções do service

```typescript
export function findCustomerById(id: number): Customer {
	const customer = customers.find((customer) => customer.id === id);

	if (!customer) {
		throw new NotFoundError(`Cliente com id ${id} não encontrado.`);
	}

	return customer;
}
```

**`throw new NotFoundError(\`Cliente com id ${id} não encontrado.\`)`**

**Finalidade**. Esta instrução lança uma instância de `NotFoundError`, com uma mensagem que interpola o identificador não encontrado, interrompendo a execução da função no ponto em que é lançada.

**Motivação**. A verificação `if (!customer)` já existia antes da criação da classe customizada; a mudança não altera onde nem quando a ausência do cliente é detectada, apenas o tipo do objeto lançado quando isso acontece.

**Decisão**. A mensagem permanece um *template literal* interpolado com o `id`, preservando a informação específica da ocorrência; apenas o construtor do objeto lançado muda, de `Error` genérico para `NotFoundError`.

**Localização**. Presente de forma idêntica em `findCustomerById`, `modifyCustomer` e `removeCustomer` — as três funções do service que dependem da existência prévia de um cliente para prosseguir.

**Contrafactual**. Se qualquer uma dessas três funções lançasse `Error` genérico em vez de `NotFoundError`, o middleware de tratamento de erros não reconheceria a instância na verificação `instanceof`, tratando-a como falha não categorizada e respondendo com status `500` em vez de `404`.

---

## Bloco 8 — Implementação do `errorHandler.ts` e Montagem no `server.ts`

### Arquivo `errorHandler.ts`

```typescript
import type { NextFunction, Request, Response } from 'express';
import { NotFoundError } from '../errors/index.ts';

export default function errorHandler(
	error: unknown,
	_request: Request,
	response: Response,
	_next: NextFunction
): void {
	if (error instanceof NotFoundError) {
		response.status(error.statusCode).json({ message: error.message });
		return;
	}

	console.log(error);

	response.status(500).json({ message: 'Erro interno do servidor.' });
}
```

**`export default function errorHandler(error, _request, response, _next): void`**

**Finalidade**. Esta declaração define `errorHandler`, o middleware de quatro parâmetros que centraliza a tradução de qualquer erro interceptado em uma resposta HTTP, e o exporta como exportação padrão do módulo.

**Motivação**. Sem essa centralização, a relação entre categoria de erro e status HTTP precisaria ser reproduzida em cada controller, reintroduzindo a duplicação que a arquitetura em camadas existe para eliminar.

**Decisão**. O parâmetro `error` é tipado como `unknown`, e não como `Error`, porque um `throw` pode, em JavaScript, lançar qualquer valor; a verificação com `instanceof`, realizada antes de qualquer acesso a propriedades, é o que autoriza o uso de `error.statusCode` e `error.message` dentro do bloco condicional. `_request` e `_next` recebem o prefixo de sublinhado pelo mesmo motivo já aplicado em `requestLogger`: são posições obrigatórias na assinatura, não utilizadas no corpo da função.

**Localização**. Exportado por padrão do arquivo `middlewares/errorHandler.ts`, e importado em `server.ts` como `import errorHandler from './middlewares/errorHandler.ts'`.

**Contrafactual**. Se a assinatura declarasse menos de quatro parâmetros, o Express deixaria de reconhecer a função como middleware de erro, e ela nunca seria chamada quando um erro fosse interceptado — sem nenhum aviso de erro de sintaxe que denunciasse o problema.

---

**`if (error instanceof NotFoundError) { response.status(error.statusCode).json({ message: error.message }); return; }`**

**Finalidade**. Este bloco verifica se `error` é uma instância de `NotFoundError` e, em caso afirmativo, responde com o `statusCode` e a `message` que a própria instância carrega, encerrando a execução da função com `return`.

**Motivação**. É essa verificação que permite distinguir uma falha de domínio conhecida — recurso não encontrado — de qualquer outro tipo de erro, respondendo com o status HTTP semanticamente correto para o caso específico.

**Decisão**. `error.statusCode` e `error.message` são lidos diretamente da instância, sem nenhuma tabela externa de mapeamento entre tipo de erro e status; a verificação `instanceof` é o que autoriza, em tempo de compilação, o acesso a `statusCode` — propriedade que não existe em um `Error` comum.

**Localização**. Primeiro bloco executado dentro de `errorHandler`, antes de qualquer outra instrução.

**Contrafactual**. Sem o `return` ao final do bloco, a execução prosseguiria até as linhas seguintes, tentando responder novamente com status `500` depois de uma resposta já enviada — produzindo o erro de execução `Cannot set headers after they are sent`.

---

**`console.log(error)` e resposta de status `500`**

**Finalidade**. Estas duas instruções, executadas quando `error` não é uma instância de `NotFoundError`, registram o erro no console e respondem ao cliente com status `500` e uma mensagem genérica.

**Motivação**. Um erro que não corresponde a nenhuma categoria conhecida — como um erro de programação não previsto — ainda precisa de alguma resposta; sem esse ramo, a requisição ficaria sem tratamento algum para esse caso.

**Decisão**. `console.log` é usado para registrar o erro no terminal, dando visibilidade ao desenvolvedor sobre uma falha não categorizada, antes de compor a resposta enviada ao cliente.

**Localização**. Últimas duas instruções do corpo de `errorHandler`, alcançadas apenas quando o bloco `if` anterior não captura o erro.

**Contrafactual**. Sem este ramo final, um erro não categorizado alcançaria o fim da função sem que nenhuma resposta fosse enviada, deixando a requisição suspensa exatamente como uma requisição sem `next()` e sem resposta.

---

### Arquivo `server.ts`

```typescript
import express from 'express';
import errorHandler from './middlewares/errorHandler.ts';
import requestLogger from './middlewares/requestLogger.ts';
import CustomerRouter from './routes/customer.router.ts';

const app = express();

app.use(requestLogger);
app.use(express.json());

app.use('/customers', CustomerRouter);

app.use((_request, response) => {
	response.status(404).json({
		message: 'Not found!'
	});
});

app.use(errorHandler);

app.listen(Number(process.env.PORT));
```

**Importações de `errorHandler` e `requestLogger`**

**Finalidade**. As instruções `import errorHandler from './middlewares/errorHandler.ts'` e `import requestLogger from './middlewares/requestLogger.ts'` trazem as duas funções de middleware para o escopo do arquivo de entrada do servidor.

**Motivação**. Sem essas importações, as funções `errorHandler` e `requestLogger`, declaradas em seus próprios arquivos, permaneceriam inacessíveis ao `server.ts`, onde precisam ser registradas.

**Decisão**. A importação é feita sem chaves, porque ambas as funções são exportação padrão de seus respectivos módulos — a mesma convenção já aplicada a `CustomerRouter`.

**Localização**. Posicionadas junto às demais importações, no início do `server.ts`.

**Contrafactual**. Uma importação sem chaves de um módulo que não possui exportação padrão resultaria em `undefined`, e a tentativa de registrar esse valor com `app.use` produziria erro em tempo de execução.

---

**`app.use(requestLogger)`**

**Finalidade**. Esta instrução registra `requestLogger` como middleware global, aplicado à totalidade das requisições recebidas pelo servidor.

**Motivação**. O registro de cada requisição no console é um comportamento transversal, que não pertence a nenhuma rota específica — deve se aplicar mesmo a requisições que não correspondam a nenhum router registrado.

**Decisão**. A posição — primeira instrução após `const app = express()` — garante que nenhuma requisição escape desse registro, independentemente do resultado final do processamento.

**Localização**. Primeira chamada a `app.use` do arquivo, antes de `express.json()` e antes da montagem de qualquer router.

**Contrafactual**. Se `app.use(requestLogger)` fosse registrado depois de `app.use('/customers', CustomerRouter)`, requisições que a rota de clientes já respondesse nunca alcançariam esse middleware, porque o ciclo da requisição já teria se encerrado antes de chegar a essa posição.

---

**`app.use((_request, response) => { response.status(404).json({ message: 'Not found!' }); })`**

**Finalidade**. Esta instrução registra um middleware anônimo, sem `next` na assinatura, que responde com status `404` a qualquer requisição que não tenha correspondido a nenhuma rota registrada anteriormente na cadeia.

**Motivação**. Sem esse middleware, uma requisição para um caminho inexistente — que nenhum router reconhece — chegaria ao fim da cadeia de registros sem que nenhuma resposta fosse enviada, deixando o cliente sem retorno algum.

**Decisão**. Este middleware não lança nenhuma exceção e não é acionado pela captura automática de erros do Express — é alcançado simplesmente porque nenhuma rota anterior respondeu à requisição. Por essa razão, sua natureza é distinta da do `errorHandler`: um trata ausência de correspondência de rota; o outro traduz uma falha efetivamente lançada durante o processamento. Essa distinção justifica manter os dois middlewares separados, ainda que ambos produzam, hoje, o mesmo status `404` em situações diferentes.

**Localização**. Posicionado depois da montagem de `CustomerRouter` e antes de `app.use(errorHandler)`.

**Contrafactual**. Se este middleware fosse removido, uma requisição para um caminho não registrado por nenhum router seguiria até `errorHandler` sem que nenhum erro tivesse sido lançado; como `errorHandler` não chama `next()` em nenhum de seus ramos, mas também não é acionado sem um erro interceptado, a requisição ficaria suspensa, sem resposta alguma.

---

**`app.use(errorHandler)`**

**Finalidade**. Esta instrução registra `errorHandler` como o middleware de tratamento de erros da aplicação, reconhecido pelo Express por sua assinatura de quatro parâmetros.

**Motivação**. Sem esse registro, qualquer erro capturado automaticamente pelo Express — síncrono ou assíncrono — seria tratado pelo middleware de erro padrão embutido no framework, que responde sempre com status `500` genérico.

**Decisão**. A posição — última instrução antes de `app.listen` — garante que `errorHandler` só seja alcançável depois de todas as rotas e do middleware de rota não encontrada, permitindo que capture erros provenientes de qualquer um deles.

**Localização**. Última chamada a `app.use` do arquivo.

**Contrafactual**. Se `app.use(errorHandler)` fosse posicionado antes da montagem de `CustomerRouter`, erros lançados pelas rotas de clientes nunca alcançariam essa posição na cadeia, e o Express recorreria ao seu tratamento de erro padrão, respondendo `500` mesmo para um `NotFoundError`.
