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

Sem um mecanismo de observação centralizado, não há registro sistemático de que uma requisição chegou ao servidor, além do que cada controller reportasse manualmente. Como esse comportamento se aplica a qualquer requisição, independentemente da rota, sua implementação corresponde a um middleware global.

```typescript
import type { NextFunction, Request, Response } from 'express';

export function requestLogger(
	request: Request,
	_response: Response,
	next: NextFunction
): void {
	console.log(`${request.method} ${request.originalUrl}`);

	next();
}
```

**Finalidade**. `requestLogger` é uma função de middleware que registra, na saída do terminal, o método HTTP e a URL de cada requisição recebida pelo servidor.

**Motivação**. Sem esse registro centralizado, não há visibilidade sistemática sobre quais requisições chegam ao servidor, em que ordem e com que frequência, à medida que o número de rotas cresce.

**Decisão**. O segundo parâmetro é nomeado `_response` porque a função nunca envia resposta — apenas observa e prossegue; o prefixo sinaliza que a posição é obrigatória na assinatura, mas o valor não é utilizado. `request.originalUrl` é usado em vez de `request.url` para preservar o caminho completo da requisição, independentemente de roteamento interno posterior.

**Localização**. Registrado como `app.use(requestLogger)` no início do arquivo de entrada do servidor, antes da montagem de qualquer router.

**Contrafactual**. Se a chamada a `next()` fosse omitida, ou condicionada a alguma verificação, toda requisição que não satisfizesse essa condição ficaria suspensa indefinidamente, sem resposta.

---

## Bloco 4 — Express 5 e Promise Rejections

### Distinção entre lançamento síncrono e rejeição assíncrona

Um erro lançado de forma síncrona, no exato instante em que um handler é chamado, sempre foi capturado automaticamente pelo Express, em qualquer versão do framework — o Express invoca cada handler dentro de seu próprio controle interno, e um lançamento nesse instante é detectado imediatamente. Esse comportamento não é uma novidade da versão 5.

A situação é diferente quando o handler é declarado `async`. Nesse caso, a chamada ao handler retorna imediatamente uma `Promise` pendente; o erro real só se manifesta depois, no momento em que essa `Promise` é rejeitada — em um ponto posterior ao controle interno síncrono original do Express.

---

### Comportamento do Express 5

Em versões do Express anteriores à 5, esse instante posterior de rejeição não era observado automaticamente: era necessário um `try/catch` explícito dentro de cada handler `async`, com uma chamada manual a `next(err)` dentro do bloco `catch` para encaminhar o erro. Sem isso, a rejeição não tratada deixava a requisição suspensa, sem resposta.

No Express 5, esse acompanhamento passou a ser automático: o Express monitora a `Promise` retornada por um handler `async` e, caso ela seja rejeitada, encaminha o erro para o mesmo destino que já recebia erros síncronos — o middleware de tratamento de erros. Essa capacidade dispensa o `try/catch` manual dentro de handlers assíncronos.

---

### Sustentação da disciplina entre camadas

Essa capacidade de captura automática, tanto síncrona quanto assíncrona, é o que permite que um controller permaneça sem qualquer `try/catch`, independentemente de a chamada ao service ser síncrona ou envolver `await`. Sem essa capacidade, o controller precisaria conhecer, em detalhe, cada tipo de falha que o service pudesse produzir, o que comprometeria a separação de responsabilidades entre as camadas.

---

## Bloco 5 — Middleware de Erro com Quatro Parâmetros

### Reconhecimento por aridade de parâmetros

O Express distingue um middleware comum de um middleware de tratamento de erros exclusivamente pela quantidade de parâmetros declarados na assinatura da função — não por nome, palavra-chave ou forma de registro. Um middleware comum declara três parâmetros: `request`, `response` e `next`. Um middleware de erro declara quatro: `err`, `request`, `response` e `next`, nessa ordem, com o objeto de erro interceptado ocupando a primeira posição.

Uma função destinada a tratar erros, mas declarada com menos de quatro parâmetros, é tratada pelo Express como middleware comum, e nunca é chamada quando um erro é interceptado.

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

```typescript
export class NotFoundError extends Error {
	statusCode = 404;

	constructor(message: string) {
		super(message);
	}
}
```

**Finalidade**. `NotFoundError` é uma classe que representa a ausência de um recurso solicitado, carregando o `statusCode` `404` como propriedade própria.

**Motivação**. Sem uma classe própria para essa categoria de falha, o service não teria como comunicar que tipo específico de erro ocorreu sem depender de comparações de texto na mensagem.

**Decisão**. `super(message)` é chamado no construtor para inicializar a propriedade `message` herdada de `Error`; essa chamada é obrigatória em uma classe derivada que declara construtor próprio, antes de qualquer uso de `this`.

**Localização**. Lançada dentro das funções de service que precisam sinalizar a ausência de um recurso, e verificada com `instanceof` pelo middleware de tratamento de erros.

**Contrafactual**. Sem `statusCode` declarado como propriedade da classe, o middleware de erro não teria, a partir do próprio objeto de erro, nenhuma informação sobre qual status HTTP corresponde àquela falha.

As demais classes de erro do domínio — `ValidationError`, com `statusCode` `400`; `UnauthorizedError`, com `statusCode` `401`; e `ConflictError`, com `statusCode` `409` — seguem estrutura idêntica, herdando de `Error` e adicionando cada uma o `statusCode` correspondente à sua categoria de falha.

---

## Bloco 7 — Disciplina entre Camadas: Lançar versus Responder

### A regra de disciplina

Services lançam exceções; nunca respondem. Controllers não capturam essas exceções — não há `try/catch` nessa camada. O middleware de tratamento de erros, e somente ele, traduz a exceção em uma resposta HTTP.

---

### Justificativa a partir da disciplina de dependências já estabelecida

Um service que respondesse diretamente com um status HTTP dependeria do protocolo de transporte para comunicar uma informação de domínio, violando a regra de que services não conhecem `Request`, `Response` nem qualquer tipo do Express. Lançar uma exceção — `throw new NotFoundError(...)` — comunica a falha sem que o service precise conhecer como ela será traduzida em resposta.

Do lado do controller, a captura automática de erros do Express 5 dispensa qualquer `try/catch`: se o service lançar, o erro sobe sem que o controller escreva uma linha para tratá-lo. Concentrar a tradução exclusivamente no middleware de erro evita que a relação entre tipo de falha e status HTTP precise ser duplicada em cada controller.

---

### Refatoração de lançamentos genéricos para classes customizadas

```typescript
throw new NotFoundError(`Cliente com id ${id} não encontrado.`);
```

**Finalidade**. Esta instrução substitui o lançamento de um `Error` genérico por uma instância de `NotFoundError`, mantendo a mesma mensagem interpolada com o identificador do cliente.

**Motivação**. A mudança de tipo do erro é o que permite ao middleware de tratamento de erros distinguir esse caso de qualquer outra falha, sem alterar onde ou quando a verificação de ausência do cliente acontece.

**Decisão**. A mensagem permanece um template literal interpolado com o `id`; apenas o tipo do objeto lançado muda, preservando a lógica de validação já existente na função.

**Localização**. Presente nas funções de service que verificam a existência de um cliente antes de retornar, atualizar ou remover um registro.

**Contrafactual**. Se a função continuasse lançando `Error` genérico, o middleware de erro não reconheceria essa instância como `NotFoundError`, tratando-a como falha não categorizada e respondendo com status `500` em vez de `404`.

---

## Bloco 8 — Implementação do `errorHandler.ts`

### Lógica de tradução centralizada

```typescript
import type { NextFunction, Request, Response } from 'express';
import {
	ConflictError,
	NotFoundError,
	UnauthorizedError,
	ValidationError
} from '../errors/index.ts';

export function errorHandler(
	err: unknown,
	_request: Request,
	response: Response,
	_next: NextFunction
): void {
	if (
		err instanceof NotFoundError ||
		err instanceof ValidationError ||
		err instanceof UnauthorizedError ||
		err instanceof ConflictError
	) {
		response.status(err.statusCode).json({ message: err.message });
		return;
	}

	console.error(err);

	response.status(500).json({ message: 'Erro interno do servidor.' });
}
```

**Finalidade**. `errorHandler` é o middleware de quatro parâmetros que centraliza a tradução de qualquer erro interceptado em uma resposta HTTP.

**Motivação**. Sem essa centralização, a relação entre categoria de erro e status HTTP precisaria ser reproduzida em cada controller, reintroduzindo a duplicação que a arquitetura em camadas existe para eliminar.

**Decisão**. `err` é tipado como `unknown`, e não como `Error`, porque um `throw` pode, em JavaScript, lançar qualquer valor; a verificação com `instanceof` antes do uso de `err.statusCode` e `err.message` autoriza o acesso a essas propriedades.

**Localização**. Registrado como último `app.use` do arquivo de entrada do servidor, depois de todos os routers e de qualquer middleware de rota não encontrada.

**Contrafactual**. Sem este middleware, o Express recorre ao seu próprio middleware de erro padrão, que responde sempre com status `500` genérico, independentemente da categoria real da falha.

---

### Ramo de erro não categorizado

**Finalidade**. O `return` posicionado após a resposta do `if` encerra a execução da função quando o erro corresponde a uma das classes conhecidas; as instruções seguintes tratam exclusivamente erros que não correspondem a nenhuma delas.

**Motivação**. Sem essa separação, um erro não categorizado — como um erro de programação não previsto — provocaria uma tentativa de leitura de `statusCode` em um objeto que não possui essa propriedade.

**Decisão**. O ramo não categorizado registra o erro com `console.error` antes de responder, preservando visibilidade sobre falhas que não foram previstas por nenhuma classe customizada.

**Localização**. Executado sempre que `err` não é instância de `NotFoundError`, `ValidationError`, `UnauthorizedError` ou `ConflictError`.

**Contrafactual**. Sem esse ramo, um erro não categorizado alcançaria as mesmas linhas de resposta do bloco anterior, tentando acessar uma propriedade inexistente e produzindo uma falha adicional dentro do próprio middleware de tratamento de erros.

---

### Posicionamento do middleware de log e do middleware de erro

```typescript
import express from 'express';
import { errorHandler } from './middleware/errorHandler.ts';
import { requestLogger } from './middleware/requestLogger.ts';
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

**Finalidade**. Esta sequência registra `requestLogger` como primeiro middleware do arquivo e `errorHandler` como último, com a montagem dos routers entre os dois.

**Motivação**. `requestLogger` precisa observar toda requisição recebida, incluindo aquelas que não correspondem a nenhuma rota; `errorHandler` precisa capturar erros provenientes de qualquer rota já registrada.

**Decisão**. A ordem de registro determina a ordem de tentativa no Express; posicionar `requestLogger` no início e `errorHandler` no final é a única ordem que satisfaz as duas exigências de alcance simultaneamente.

**Localização**. Ambas as instruções fazem parte da configuração global do arquivo de entrada do servidor, junto ao registro de `express.json()` e à montagem de `CustomerRouter`.

**Contrafactual**. Se `errorHandler` fosse registrado antes da montagem de `CustomerRouter`, erros provenientes das rotas de clientes nunca alcançariam essa posição, e o Express recorreria ao seu tratamento de erro padrão.
