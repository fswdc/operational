# Parte 1

## Bloco 1 — O que é um Framework e por que o Express Existe

### O problema do roteamento manual

Um servidor HTTP criado com `node:http` funciona a partir de uma única função *handler* registrada em `http.createServer`. Toda requisição que chega ao servidor — independentemente de método HTTP, caminho ou intenção — passa por essa mesma função. É ela que precisa, internamente, inspecionar `request.method` e `request.url` e decidir o que fazer com cada combinação.

Na prática, isso significa uma estrutura de condicionais encadeadas:

```typescript
if (request.method === 'GET' && request.url === '/health') {
	// responder com status da aplicação
} else if (request.method === 'POST' && request.url === '/customers') {
	// criar cliente
} else if (request.method === 'GET' && request.url?.startsWith('/customers/')) {
	// buscar cliente por id
} else {
	// responder 404
}
```

Cada rota nova é um bloco adicional nessa estrutura. Em um projeto com dezesseis rotas distintas, o arquivo de entrada acumula dezesseis verificações condicionais, e a lógica de roteamento — decidir qual código executar — fica entrelaçada com a lógica de cada operação.

A leitura do corpo de uma requisição `POST` acrescenta outra camada de complexidade. O `node:http` não entrega o corpo pronto: ele o transmite em fragmentos, emitindo o evento `data` a cada fragmento recebido e o evento `end` quando a transmissão termina. O código manual para ler esse corpo tem sempre a mesma forma:

```typescript
let body = '';
request.on('data', (chunk) => {
	body += chunk.toString();
});
request.on('end', () => {
	const dados = JSON.parse(body);
	// continuar o processamento
});
```

Esse padrão precisa ser repetido em toda rota que receba corpo — `POST`, `PUT` — porque o `node:http` não oferece atalho para isso. O resultado é código idêntico duplicado em cada ponto de entrada da aplicação.

---

### Definição de framework

Um **framework** é uma biblioteca que impõe uma estrutura e padroniza tarefas recorrentes de um tipo de aplicação, de modo que o desenvolvedor não precise resolver o mesmo problema do zero em cada projeto.

A diferença em relação a uma biblioteca utilitária comum está em quem detém o controle do fluxo: em uma biblioteca utilitária, o código da aplicação chama a biblioteca quando precisa de uma função específica. Em um framework, o framework define a estrutura e chama o código da aplicação nos momentos adequados — o registro de uma rota não a executa; o framework a executa quando a requisição correspondente chega.

---

### Justificativa de existência do Express

O **Express** existe porque o trabalho de roteamento HTTP — decidir qual função executar para cada combinação de método e URL, ler e serializar corpos JSON, compor respostas com *status codes* e cabeçalhos corretos — é repetitivo, idêntico entre projetos e não agrega valor ao produto final.

O `node:http` oferece controle total sobre a camada de rede, mas não oferece convenções para a camada de roteamento. O Express preenche esse espaço: usa `node:http` internamente e oferece uma API declarativa para roteamento e composição de resposta. O produto final — um servidor respondendo a requisições HTTP — é o mesmo; o que muda é a forma como esse comportamento é expresso no código.

---

## Bloco 2 — O Objeto `app` e `app.listen`

### Separação entre configuração e execução

A criação de uma aplicação Express segue uma sequência de dois momentos distintos: a configuração e o início da escuta.

A configuração acontece quando o código é executado de cima para baixo durante a inicialização do processo. Nesse momento, middlewares são registrados e rotas são declaradas. Nenhuma requisição é recebida ainda — o objeto `app` existe apenas em memória, acumulando as regras que serão aplicadas quando o servidor estiver no ar.

O início da escuta acontece quando `app.listen` é chamado. É nesse momento que o Express abre o socket TCP na porta declarada e passa a aceitar conexões. A partir desse ponto, requisições podem chegar e o Express começa a despachar cada uma para os *handlers* registrados.

---

### Criação do objeto `app`

```typescript
const app = express();
```

**Finalidade.** Esta instrução chama a função `express()` e armazena o objeto retornado na variável `app`. O objeto `app` é o centro de controle da aplicação: nele são registrados middlewares e rotas, e a partir dele é iniciada a escuta da porta.

**Motivação.** `express()` não abre nenhuma porta e não inicia nenhum servidor — ela apenas cria e retorna o objeto de controle. Isso permite configurar middlewares e rotas em qualquer ordem antes de o servidor começar a aceitar conexões.

**Decisão.** O nome `app` é convenção universal em projetos Express. Qualquer desenvolvedor que abra o arquivo reconhece imediatamente o papel da variável sem necessidade de documentação adicional.

**Localização.** `app` é referenciado em todas as instruções seguintes: `app.use`, `app.get`, `app.post` e `app.listen`.

**Contrafactual.** Tentar chamar `express.get(...)` diretamente, sem criar o objeto `app`, produziria erro em tempo de execução: os métodos de rota existem no objeto retornado por `express()`, não na função `express` em si.

---

### `app.listen`

```typescript
app.listen(Number(process.env.PORT), () => {
	console.log(`Servidor rodando na porta ${process.env.PORT}`);
});
```

**Finalidade.** `app.listen` abre o socket TCP na porta declarada e executa o *callback* uma única vez, quando o servidor está pronto para receber conexões.

**Motivação.** Sem `app.listen`, o processo Node.js executa o arquivo inteiro — registrando middlewares e rotas — e encerra imediatamente, pois não há nada mantendo o processo vivo. Nenhuma requisição chega a ser recebida.

**Decisão.** `Number(process.env.PORT)` realiza a conversão explícita de *string* para número. `process.env` sempre retorna *string*; `app.listen` espera número. O *callback* de confirmação é opcional tecnicamente, mas documenta o estado do servidor no terminal durante o desenvolvimento.

**Localização.** Posicionado como última instrução do `server.ts`, após o registro de todos os middlewares e rotas. Nenhuma configuração de `app` deve ocorrer após esta chamada.

**Contrafactual.** Se `process.env.PORT` for `undefined` — por exemplo, pela ausência da flag `--env-file` no script de execução — `Number(undefined)` produz `NaN`, e a tentativa de escutar em `NaN` resulta em erro de porta inválida.

---

### Distinção entre registro e execução

O registro de uma rota com `app.get(...)` e a execução do *handler* correspondente são eventos separados no tempo.

O registro ocorre durante a inicialização do processo, quando o interpretador percorre o arquivo de cima para baixo e encontra a chamada `app.get(...)`. O Express armazena internamente a associação entre o caminho, o método HTTP e a função *handler* — mas não executa a função nesse momento.

A execução ocorre mais tarde, quando uma requisição real chega pela rede, o Express verifica os registros disponíveis e encontra um que corresponde ao método e ao caminho da requisição. Só então a função *handler* é chamada. Esse processo se repete a cada requisição — o registro é único; a execução pode ocorrer zero, uma ou milhares de vezes.

---

## Bloco 3 — Métodos de Rota e a Primeira Rota Declarativa

### Métodos de rota como substitutos dos condicionais

No servidor HTTP nativo, o roteamento é implementado como uma sequência de condicionais que verificam `request.method` e `request.url` explicitamente:

```typescript
if (request.method === 'DELETE' && request.url?.startsWith('/customers/')) {
	const id = request.url.split('/')[2];
	// lógica de exclusão
}
```

O Express substitui esse padrão por métodos declarativos no objeto `app`: `app.get`, `app.post`, `app.put` e `app.delete`. Cada método incorpora implicitamente a verificação do método HTTP correspondente. O desenvolvedor declara o que deve acontecer para cada rota; o Express executa as verificações internamente.

---

### Declaração de rota com parâmetro de caminho

```typescript
app.delete('/customers/:id', (request, response) => {
	const id = request.params.id;
});
```

**`app.delete('/customers/:id', handler)`**

**Finalidade.** Esta instrução registra um *handler* que o Express chama exclusivamente quando uma requisição com método `DELETE` chega em um caminho que corresponde ao padrão `/customers/:id`.

**Motivação.** Sem o registro explícito de método e caminho, qualquer requisição — independentemente de método ou URL — seria encaminhada ao mesmo *handler*, exigindo verificações manuais internas.

**Decisão.** `:id` é um **parâmetro de rota**: um marcador que indica que nessa posição do caminho haverá um valor variável. O Express extrai esse valor automaticamente e o disponibiliza em `request.params.id`, substituindo a extração manual por `request.url.split('/')[2]`.

**Localização.** O padrão `/customers/:id` casa com qualquer caminho que tenha exatamente dois segmentos após a barra inicial, como `/customers/1`, `/customers/42` ou `/customers/abc`. Não casa com `/customers` sem segmento adicional.

**Contrafactual.** Se o método declarado fosse `app.get` em vez de `app.delete`, a rota não responderia a requisições `DELETE` — o Express retornaria `404` automaticamente para esse método nesse caminho.

---

## Bloco 4 — Middleware e `express.json()`

### Definição de middleware

Um **middleware** é uma função com três parâmetros — `request`, `response` e `next` — executada pelo Express antes do *handler* final de uma rota. A função `next` é um *callback* que, quando chamado, instrui o Express a passar o controle para o próximo middleware ou *handler* na cadeia. Se `next` não for chamado e nenhuma resposta for enviada, a requisição fica suspensa indefinidamente.

A analogia com controle de acesso em um prédio é precisa: antes de chegar à sala de reuniões (*handler* da rota), a requisição passa por etapas de verificação (middlewares), cada uma com a capacidade de autorizar o prosseguimento — chamando `next()` — ou de encerrar o fluxo enviando uma resposta diretamente.

---

### Middleware global com `app.use`

```typescript
app.use(express.json());
```

**Finalidade.** Esta instrução registra o middleware `express.json()` como middleware global: ele é executado em todas as requisições que chegam ao servidor, antes de qualquer *handler* de rota.

**Motivação.** Por padrão, o Express não lê o corpo da requisição automaticamente. Sem este middleware, `request.body` é `undefined` em qualquer *handler* que tente acessá-lo.

**Decisão.** `app.use(fn)` registra o middleware para todas as rotas e todos os métodos. A alternativa — registrar o middleware individualmente em cada rota que o necessite — é adequada para middlewares seletivos, como autenticação, mas redundante para o parse de JSON, que se aplica a todo o projeto.

**Localização.** Deve ser posicionado antes de qualquer declaração de rota que leia `request.body`. O Express executa middlewares e *handlers* na ordem de registro; um middleware registrado após uma rota não é executado para requisições que correspondam a essa rota.

**Contrafactual.** Se omitido, ou se registrado após as rotas que leem `request.body`, o valor de `request.body` será `undefined` nesses *handlers*. O servidor não emite erro — o processo continua normalmente — mas os dados enviados pelo cliente nunca chegam ao *handler*.

---

### O que `express.json()` faz internamente

No `node:http`, ler o corpo de uma requisição exige código explícito: o corpo não está disponível de imediato quando o *handler* é chamado. Ele chega pela rede em fragmentos, e o Node.js emite um evento `data` a cada fragmento recebido. O desenvolvedor precisa acumular esses fragmentos em uma variável, aguardar o evento `end` — que sinaliza que a transmissão terminou — e só então converter o conteúdo acumulado em um objeto JavaScript com `JSON.parse`:

```typescript
let body = '';
request.on('data', (chunk) => {
	body += chunk.toString();
});
request.on('end', () => {
	const dados = JSON.parse(body);
	// prosseguir com os dados
});
```

O middleware `express.json()` executa exatamente esse trabalho de forma centralizada. Ele escuta os eventos `data` e `end` do objeto `request`, acumula os fragmentos recebidos, executa `JSON.parse` sobre o resultado e atribui o objeto JavaScript resultante a `request.body`. Quando o *handler* da rota é chamado, `request.body` já contém os dados prontos para uso — sem nenhum código adicional no *handler*.

---

## Bloco 5 — `res.json()` e `res.status()`

### Composição de resposta no servidor HTTP nativo

No `node:http`, compor e enviar uma resposta JSON envolve três operações distintas, executadas em duas chamadas separadas.

A primeira chamada é `response.writeHead`, que define o *status code* e os cabeçalhos da resposta. O cabeçalho `Content-Type` precisa ser declarado explicitamente como `application/json` para que o cliente saiba interpretar o corpo como JSON — sem essa declaração, o cliente pode tratar o conteúdo como texto simples.

A segunda chamada é `response.end`, que envia o corpo da resposta e encerra a conexão. O método aceita *string* ou `Buffer`, não objetos JavaScript diretamente — por isso `JSON.stringify` precisa ser chamado antes, convertendo o objeto para a representação textual em JSON.

```typescript
response.writeHead(201, { 'content-type': 'application/json' });
response.end(JSON.stringify({ id: 1, name: 'Ana' }));
```

As três operações — definir *status*, declarar `Content-Type` e serializar o objeto — são invariavelmente necessárias juntas. Omitir `JSON.stringify` envia `[object Object]` como corpo. Omitir o cabeçalho `Content-Type` envia JSON sem declaração de tipo. Omitir `response.end` deixa a requisição sem resposta, e o cliente aguarda até o *timeout*.

---

### `res.json()`

**Finalidade.** `res.json(objeto)` serializa o objeto JavaScript com `JSON.stringify`, define o cabeçalho `Content-Type: application/json` e envia a resposta ao cliente, encerrando a conexão.

**Motivação.** As três operações manuais do `node:http` — serialização com `JSON.stringify`, definição do cabeçalho e encerramento com `response.end` — são invariavelmente realizadas juntas. Encapsulá-las em um único método elimina a possibilidade de omitir qualquer uma por esquecimento.

**Decisão.** O método é usado em vez de `res.send()`, que também aceita objetos mas cujo comportamento exato depende do tipo do argumento. `res.json()` tem semântica inequívoca: sempre serializa para JSON e sempre define o cabeçalho correto.

**Localização.** Chamado como última operação sobre `res` em qualquer *handler* que devolva dados ao cliente. Não pode ser chamado duas vezes na mesma requisição.

**Contrafactual.** Chamar `res.json()` duas vezes na mesma requisição produz o erro `ERR_HTTP_HEADERS_SENT`: os cabeçalhos já foram enviados na primeira chamada e não podem ser redefinidos na segunda.

---

### `res.status()`

**Finalidade.** `res.status(codigo)` define o *status code* HTTP da resposta e retorna o próprio objeto `res`, permitindo o encadeamento com `res.json()`.

**Motivação.** O Express usa `200` como *status code* padrão quando `res.status()` não é chamado explicitamente. Para respostas com outros códigos — como `201` em criações bem-sucedidas — a definição explícita é necessária.

**Decisão.** O retorno de `res` pelo método habilita o encadeamento `res.status(201).json(objeto)`, que é idiomático no Express e equivalente a chamar os dois métodos sequencialmente em linhas separadas.

**Localização.** Chamado antes de `res.json()` quando o *status code* da resposta deve ser diferente de `200`.

**Contrafactual.** Omitir `res.status()` em uma criação bem-sucedida faz o cliente receber `200` em vez de `201`. Semanticamente, `200` indica sucesso genérico; `201` indica criação de recurso. A distinção é relevante para clientes que interpretam o *status code* para determinar a natureza da resposta.

---

## Bloco 6 — Captura Automática de Erros em Handlers Assíncronos

### O problema no servidor HTTP nativo

No servidor HTTP nativo, toda função *handler* assíncrona exige um bloco `try/catch` explícito. Uma função `async` que lança um erro sem `try/catch` produz uma `UnhandledPromiseRejection`: o Node.js registra o aviso no console, a requisição fica sem resposta e o cliente aguarda até o *timeout*.

Esse padrão exige que o desenvolvedor lembre de envolver cada *handler* assíncrono em `try/catch` — uma responsabilidade manual repetida em cada arquivo de rota.

---

### Comportamento do Express 5

O Express 5 captura automaticamente erros lançados dentro de *handlers* assíncronos. Quando uma função `async` registrada como *handler* lança um erro — seja por `throw` explícito, seja por uma `Promise` rejeitada em um `await` — o Express intercepta a rejeição e a encaminha para o *middleware* global de tratamento de erros, sem necessidade de `try/catch` no *handler*.

```typescript
app.post('/customers', async (request, response) => {
	const customer = await customerService.create(request.body);
	response.status(201).json(customer);
});
```

Se `customerService.create` lançar um erro, o Express 5 o captura e o encaminha para o *middleware* de erro. O *handler* não precisa de `try/catch`.

---

### Distinção em relação ao Express 4

No Express 4, a captura automática de erros em *handlers* assíncronos não existia. Cada *handler* `async` precisava de `try/catch` explícito, ou o erro se tornava uma `UnhandledPromiseRejection`. O Express 5 elimina essa obrigatoriedade por construção, tornando o padrão de lançamento de erros nos *services* viável sem exigir disciplina manual de `try/catch` em cada *handler*.

---

### Papel do middleware global de erros

O Express reconhece um *middleware* de erro pela assinatura de quatro parâmetros: `(err, req, res, next)`. Quando um erro é encaminhado — seja pelo Express 5 a partir de um *handler* assíncrono, seja por uma chamada explícita a `next(err)` — o Express ignora os middlewares comuns e chama o primeiro *middleware* de erro registrado.

Esse *middleware* é o ponto centralizado de tradução de erros em respostas HTTP: recebe o objeto de erro, determina o *status code* e o corpo da resposta e os envia ao cliente. A arquitetura resultante — *services* que lançam erros, *handlers* que não capturam, *middleware* global que traduz — é estabelecida em detalhes nas aulas seguintes.
