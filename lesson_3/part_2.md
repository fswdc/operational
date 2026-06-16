# Parte 2

## Bloco 1 — O Projeto Corcoiote

O Corcoiote é um painel administrativo de contas a receber de uma empresa fictícia. O sistema permite que funcionários internos visualizem e gerenciem clientes e faturas — sem exposição ao público geral.

Do ponto de vista técnico, o Corcoiote é composto por duas partes completamente independentes:

- **`corcoiote-backend`**: o servidor. Expõe uma API REST. É escrito em TypeScript e utiliza Express, Prisma e PostgreSQL.
- **`corcoiote-frontend`**: o painel visual. Consome a API do backend. É construído em React.

Dois repositórios. Dois projetos `npm`. Dois processos distintos em execução simultânea durante o desenvolvimento integrado.

Essa separação reflete o modelo de trabalho adotado por times profissionais: backend e frontend evoluem de forma independente, contanto que o contrato da API seja preservado. A mesma API pode ser consumida por múltiplos clientes — o painel web, um aplicativo mobile, um script de integração — sem que o servidor precise conhecer ou se adaptar a cada um.

Ao final deste módulo, o Corcoiote estará em produção real: backend e frontend publicados, banco de dados hospedado, autenticação, cobertura de testes automatizados e com práticas essenciais de garantia de qualidade de software.

---

## Bloco 2 — Repositório Remoto como Ponto de Partida

O fluxo profissional adotado no curso estabelece uma sequência precisa: o repositório é criado no GitHub primeiro; o clone local vem depois; toda codificação acontece dentro do diretório clonado.

A sequência inversa — criar a pasta localmente, escrever código e só depois conectar ao remoto — compromete o histórico de versões. Commits de configuração inicial, como a criação do `package.json` e do `tsconfig.json`, nunca existiriam no repositório. O histórico começaria em um estado já avançado, com decisões de estrutura invisíveis — como se o projeto tivesse surgido pronto. Qualquer desenvolvedor que inspecione o repositório posteriormente não consegue reconstruir o raciocínio inicial. Além disso, o código existiria apenas localmente durante o período de configuração, sem backup externo.

### Repositório sem inicialização automática

O repositório `corcoiote-backend` é criado no GitHub com visibilidade pública e sem nenhuma opção de inicialização marcada — sem README, sem `.gitignore`, sem licença.

Quando o GitHub cria um repositório com a opção *"Add a README file"* ativada, ele realiza um commit automático no remoto. Ao tentar clonar e inicializar o projeto localmente em seguida, esse commit automático entra em conflito com o primeiro commit local. O repositório vazio evita esse atrito.

### Clonagem e entrada no diretório

```shell
git clone https://github.com/seu-usuario/corcoiote-backend.git
cd corcoiote-backend
```

**`git clone <URL>`**

**Finalidade.** `git clone` copia o repositório remoto para o disco local, criando uma pasta com o nome do repositório e configurando automaticamente o *remote* `origin` apontando para a URL fornecida.

**Motivação.** O repositório precisa existir localmente antes de qualquer trabalho. Clonar a partir do remoto já existente elimina a etapa manual de `git remote add`.

**Decisão.** A clonagem é preferida à inicialização local com `git init` porque configura o *remote* `origin` automaticamente. Com `git init`, seria necessário executar `git remote add origin <URL>` separadamente, com maior risco de erro na URL.

**Localização.** Executado uma única vez por desenvolvedor, no momento em que ingressa no projeto pela primeira vez.

**Contrafactual.** Sem clonar, o diretório local não teria vínculo com o remoto. O `git push` exigiria configuração manual do *remote* antes de funcionar.

---

**`cd corcoiote-backend`**

**Finalidade.** `cd` navega para o diretório recém-criado pelo clone.

**Motivação.** Todos os comandos seguintes precisam ser executados dentro da pasta do projeto. Sem navegar para ela, os comandos seriam executados no diretório pai, criando arquivos no lugar errado.

**Decisão.** Nenhuma alternativa prática existe para este comando em shells padrão — é o mecanismo de navegação de diretório universal.

**Localização.** Executado imediatamente após o clone, antes de qualquer outro comando do projeto.

**Contrafactual.** Sem este comando, o `npm init` e os demais comandos seguintes criariam arquivos no diretório pai, fora do repositório clonado.

---

## Bloco 3 — Estrutura Inicial do Projeto

Após a clonagem, o diretório está vazio. A estrutura mínima de um projeto Node.js com TypeScript requer três elementos: os diretórios que organizam os arquivos, o `package.json` que registra o projeto e a declaração de que o projeto usa ES Modules.

### Diretórios `src/` e `tests/`

```shell
mkdir src
mkdir tests
```

O diretório `src/` concentra todo o código-fonte do servidor. O diretório `tests/` concentra todos os arquivos de teste. Essa separação não é imposta pelo Node.js — tecnicamente, todos os arquivos poderiam estar na raiz. A separação existe porque ferramentas de configuração, como o `tsconfig.json` e o executor de testes, precisam referenciar esses diretórios explicitamente. Diretórios nomeados tornam essa configuração direta e sem ambiguidade.

O diretório `tests/` é criado nesta aula porque o `tsconfig.json` o referencia no campo `include`. Referenciar um diretório inexistente nesse campo gera erro de configuração. O conteúdo do diretório — os primeiros arquivos de teste — é introduzido nas aulas finais da Parte.

**`mkdir src`**

**Finalidade.** `mkdir src` cria o diretório `src/` na raiz do projeto.

**Motivação.** Todo o código-fonte TypeScript do servidor precisa de um diretório dedicado, separado de arquivos de configuração e de testes.

**Decisão.** O nome `src` é convenção universal em projetos Node.js e TypeScript. Qualquer desenvolvedor que abra o projeto reconhece imediatamente o propósito do diretório.

**Localização.** Referenciado no `tsconfig.json` como `rootDir` e em `include`, e no script `dev` como ponto de entrada da execução.

**Contrafactual.** Sem este diretório, os arquivos TypeScript ficariam na raiz, misturados com `package.json`, `tsconfig.json` e `.env`, tornando a estrutura do projeto difícil de navegar e a configuração do TypeScript ambígua.

---

**`mkdir tests`**

**Finalidade.** `mkdir tests` cria o diretório `tests/` na raiz do projeto.

**Motivação.** Os arquivos de teste ficam separados do código de produção desde o início. O `tsconfig.json` referencia este diretório em `include`, e referenciar um diretório inexistente gera erro de configuração imediato.

**Decisão.** A criação antecipada do diretório, antes de qualquer arquivo de teste existir, é a forma de satisfazer a referência no `tsconfig.json` sem precisar revisitar a configuração em uma aula futura.

**Localização.** Referenciado no `tsconfig.json` em `include` e configurado como diretório de testes pelo Jest nas aulas finais da Parte.

**Contrafactual.** Sem este diretório, o `tsconfig.json` criado nesta aula emitiria erro ao referenciar `tests` em `include`, impedindo a verificação de tipos.

---

### Inicialização do `package.json`

```shell
npm init -y
```

**`npm init -y`**

**Finalidade.** `npm init -y` cria o arquivo `package.json` na raiz do projeto com valores padrão, sem perguntas interativas.

**Motivação.** Sem `package.json`, o diretório é apenas uma pasta — não é um projeto Node.js reconhecível pelo `npm`. Nenhuma dependência pode ser instalada e nenhum script pode ser registrado.

**Decisão.** A flag `-y` aceita todos os valores padrão automaticamente. Os valores gerados — nome, versão, descrição — podem ser editados manualmente depois conforme necessário, o que torna a interação desnecessária neste momento.

**Localização.** O `package.json` gerado é o arquivo central do projeto, referenciado por `npm install`, `npm run` e por ferramentas como Biome e Lefthook.

**Contrafactual.** Sem `-y`, o `npm` exibiria um formulário interativo com perguntas sobre nome, versão, descrição e autor — informações que podem ser adicionadas manualmente depois, tornando a interação desnecessária.

---

### Declaração de ES Modules

Após a criação do `package.json`, adiciona-se manualmente a linha `"type": "module"` ao arquivo:

```json
{
  "name": "corcoiote-backend",
  "version": "1.0.0",
  "type": "module"
}
```

**`"type": "module"`**

**Finalidade.** Esta declaração instrui o Node.js a tratar todos os arquivos `.js` e `.ts` do projeto como ES Modules, habilitando a sintaxe `import` e `export` em todo o código.

**Motivação.** O projeto usa `import` e `export` em todo o código TypeScript. Sem esta declaração, o Node.js tentaria interpretar os arquivos como CommonJS e quebraria em qualquer linha de `import`.

**Decisão.** A declaração no `package.json` aplica a convenção a todos os arquivos do projeto de uma só vez, em vez de renomear cada arquivo para `.mjs` individualmente.

**Localização.** Lido pelo Node.js toda vez que inicia a execução de um arquivo do projeto.

**Contrafactual.** Sem esta linha, a primeira execução de `npm run dev` produziria erro de sintaxe na linha `import http from 'node:http'` do `server.ts`.

---

## Bloco 4 — TypeScript por Projeto

### Instalação local

```shell
npm install --save-dev typescript @types/node
```

A instalação do TypeScript nesta aula é por projeto — não global. A distinção tem consequências práticas diretas.

Quando o TypeScript está instalado globalmente, todos os projetos no sistema compartilham a mesma versão. Ao atualizar o TypeScript global para trabalhar em um projeto, outros projetos passam a ser compilados com uma versão diferente da que foi usada originalmente — o que pode introduzir erros silenciosos ou mudanças de comportamento não solicitadas.

Quando o TypeScript está instalado por projeto, registrado no `package.json`, cada projeto é auto-contido. A versão do TypeScript de um projeto não interfere na versão de outro. Um desenvolvedor que clona o repositório e executa `npm install` obtém exatamente a mesma versão utilizada no desenvolvimento original. O mesmo vale para pipelines de CI/CD.

**`npm install --save-dev typescript @types/node`**

**Finalidade.** Este comando instala o compilador TypeScript e as declarações de tipo para os módulos nativos do Node.js como dependências de desenvolvimento do projeto.

**Motivação.** O projeto precisa do compilador para verificação estática e do pacote `@types/node` para que o TypeScript conheça os tipos de `node:http`, `node:fs` e demais módulos nativos.

**Decisão.** A flag `--save-dev` registra os pacotes em `devDependencies` em vez de `dependencies`, porque TypeScript e `@types/node` são ferramentas de desenvolvimento — não são necessárias no servidor em produção, onde o código já é JavaScript.

**Localização.** O `typescript` é usado pelo `tsc` para verificação de tipos e pelo VSCode para IntelliSense. O `@types/node` é usado pelo TypeScript para verificar qualquer código que importe módulos nativos do Node.js.

**Contrafactual.** Sem `@types/node`, o TypeScript emitiria erro de tipo em qualquer linha que importasse um módulo nativo, tornando o desenvolvimento inviável.

---

## Bloco 5 — `tsconfig.json` para o Projeto

O `tsconfig.json` usado nesta aula é reutilizado das aulas anteriores, com três campos adaptados para o `corcoiote-backend`. Os demais campos permanecem idênticos: `"strict": true`, `"module": "NodeNext"`, `"moduleResolution": "NodeNext"`, `"erasableSyntaxOnly": true` e os demais já justificados.

Os três campos adaptados são `rootDir`, `outDir` e `include`, que descrevem a localização dos arquivos específicos deste projeto.

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "noEmit": true,
    "strict": true,
    "erasableSyntaxOnly": true,
    "verbatimModuleSyntax": true,
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src", "tests"]
}
```

### `outDir`, `rootDir` e `include`

**`"outDir": "dist"`**

**Finalidade.** `outDir` define o diretório de destino dos arquivos JavaScript gerados pelo `tsc` quando ele é invocado para compilar.

**Motivação.** Sem esta declaração, o `tsc` geraria os arquivos `.js` ao lado dos `.ts`, misturando código-fonte e código gerado no mesmo diretório.

**Decisão.** O nome `dist` é convenção universal para arquivos de distribuição gerados — qualquer desenvolvedor reconhece o propósito sem documentação adicional.

**Localização.** Usado quando `tsc` é invocado para geração de arquivos. Não afeta a execução direta pelo Node.js 24 com *type stripping*.

**Contrafactual.** Sem `outDir`, os arquivos `.js` seriam gerados dentro de `src/`, ao lado dos `.ts`, tornando impossível distinguir código-fonte de código gerado.

---

**`"rootDir": "src"`**

**Finalidade.** `rootDir` declara o diretório raiz dos arquivos TypeScript de origem, garantindo que a estrutura interna de `src/` seja espelhada corretamente dentro de `dist/`.

**Motivação.** Sem esta declaração, o TypeScript poderia incluir a raiz do projeto inteira como parte da estrutura de origem, gerando hierarquias de pasta inesperadas dentro de `dist/`.

**Decisão.** Aponta para `src/` em vez de para a raiz do projeto porque arquivos de configuração na raiz — como `tsconfig.json` e `package.json` — não são código TypeScript compilável.

**Localização.** Usado pelo `tsc` para determinar a estrutura relativa dos arquivos de saída em `dist/`.

**Contrafactual.** Sem `rootDir`, um arquivo em `src/services/clientes.ts` poderia ser compilado para `dist/src/services/clientes.js` em vez de `dist/services/clientes.js`, dependendo do comportamento inferido pelo compilador.

---

**`"include": ["src", "tests"]`**

**Finalidade.** `include` declara explicitamente quais diretórios o TypeScript deve processar durante a verificação de tipos.

**Motivação.** Sem esta declaração, o TypeScript usaria heurísticas próprias para decidir o que incluir — o que pode variar entre versões e produzir comportamento inesperado, como incluir arquivos temporários ou excluir arquivos de teste.

**Decisão.** Lista explícita em vez de depender do comportamento padrão. Inclui `tests/` desde o início porque o diretório é referenciado desde agora, mesmo que os primeiros arquivos de teste sejam criados em aulas futuras.

**Localização.** Lido pelo `tsc` e pelo VSCode ao inicializar o serviço de linguagem TypeScript.

**Contrafactual.** Sem `include`, arquivos em `tests/` poderiam não ser verificados pelo TypeScript, permitindo que erros de tipo em testes passassem despercebidos.

---

## Bloco 6 — Script de Desenvolvimento

### Variáveis de ambiente com `.env`

Configurações que mudam entre ambientes — como a porta em que o servidor escuta — não devem estar escritas diretamente no código. Se a porta estivesse escrita como `3001` em um arquivo TypeScript, mudar para outra porta exigiria editar o código-fonte, o que é um erro de design.

A solução padrão da indústria é o arquivo `.env`: um arquivo de texto simples na raiz do projeto onde cada linha declara uma variável no formato `NOME=valor`.

```
PORT=3001
```

O Node.js 24 oferece suporte nativo ao carregamento desse arquivo via flag `--env-file`, sem necessidade da biblioteca `dotenv`. O uso de `dotenv` é proibido no curso: quando o *runtime* resolve nativamente, adicionar uma dependência externa para o mesmo fim é regressão, não progresso.

O arquivo `.env` nunca entra no repositório — em aulas futuras conterá senhas de banco de dados e chaves secretas. Por isso, é adicionado ao `.gitignore`. Em seu lugar, o arquivo `.env.example` é versionado: tem a mesma estrutura, mas com valores vazios, documentando quais variáveis o projeto espera.

```
PORT=
```

### Flag `--watch`

Durante o desenvolvimento, o código é modificado com frequência. Sem automação, cada modificação exigiria parar o servidor manualmente, executar o comando novamente e aguardar a inicialização. A flag `--watch` instrui o Node.js a monitorar os arquivos do projeto e reiniciar o processo automaticamente quando detecta uma modificação salva.

Essa flag é nativa do Node.js — não é um pacote externo. Assim como `dotenv` é desnecessário no Node.js 24, ferramentas como `nodemon` são substituídas pela flag nativa `--watch`, sem dependências adicionais.

### Script `dev`

O comando completo que une execução nativa de TypeScript, carregamento de variáveis de ambiente e reinício automático é:

```shell
node --watch --env-file=.env src/server.ts
```

Esse comando é registrado como script no `package.json`:

```json
"scripts": {
  "dev": "node --watch --env-file=.env src/server.ts"
}
```

E executado com `npm run dev`. O `npm` lê o `package.json`, localiza o script `dev` e executa o comando registrado. Isso padroniza a forma de iniciar o servidor: qualquer desenvolvedor no projeto executa `npm run dev`, independentemente do comando técnico subjacente.

**`node`**

**Finalidade.** `node` invoca o *runtime* Node.js para executar o arquivo especificado.

**Motivação.** É o executor — sem ele, o arquivo TypeScript é apenas texto em disco.

**Decisão.** `node` direto em vez de `npx tsx` ou `npx ts-node` porque o Node.js 24 executa TypeScript nativamente via *type stripping*, eliminando a necessidade de executores externos.

**Localização.** Invocado pelo `npm run dev` durante todo o desenvolvimento da Parte.

**Contrafactual.** Usar `npx tsx src/server.ts` produziria o mesmo resultado visível, mas adicionaria uma dependência externa e uma camada de indireção desnecessária no Node.js 24.

---

**`--watch`**

**Finalidade.** `--watch` ativa o modo de observação de arquivos: o Node.js monitora o projeto e reinicia o processo automaticamente quando detecta alterações salvas.

**Motivação.** Sem reinício automático, cada mudança no código exige parar o servidor manualmente e iniciá-lo novamente, interrompendo o fluxo de trabalho.

**Decisão.** Flag nativa do Node.js em vez de `nodemon` — elimina uma dependência de desenvolvimento sem perda de funcionalidade.

**Localização.** Ativo durante toda a execução do script `dev`. Desativado em produção, onde o código não muda.

**Contrafactual.** Sem `--watch`, salvar `server.ts` não teria efeito imediato: o servidor continuaria executando o código anterior até ser reiniciado manualmente.

---

**`--env-file=.env`**

**Finalidade.** `--env-file=.env` instrui o Node.js a ler o arquivo `.env` e carregar seu conteúdo em `process.env` antes de executar qualquer código.

**Motivação.** As variáveis de ambiente precisam estar disponíveis desde a primeira linha do código. O arquivo `.env` no disco é apenas texto — o Node.js não o lê automaticamente sem esta flag.

**Decisão.** Flag nativa do Node.js 24 em vez da biblioteca `dotenv`. Quando o *runtime* oferece suporte nativo, adicionar uma dependência para o mesmo fim é redundante e desnecessário.

**Localização.** Executado pelo Node.js antes do *type stripping* e da execução do código, disponibilizando as variáveis em `process.env` para todo o processo.

**Contrafactual.** Sem esta flag, `process.env.PORT` seria `undefined` mesmo com o arquivo `.env` presente no disco. O arquivo existe, mas nunca é lido.

---

**`src/server.ts`**

**Finalidade.** `src/server.ts` especifica o arquivo de entrada que o Node.js deve executar.

**Motivação.** O Node.js precisa saber por onde começar. Sem o caminho do arquivo de entrada, o comando não tem destino.

**Decisão.** O arquivo `.ts` é referenciado diretamente em vez do `.js` compilado. O Node.js 24 aplica *type stripping* e executa o TypeScript sem etapa de compilação prévia.

**Localização.** Lido pelo Node.js no início da execução. A flag `--watch` usa este arquivo como âncora para determinar quais arquivos relacionados monitorar.

**Contrafactual.** Referenciar `dist/server.js` exigiria compilar manualmente com `tsc` antes de cada execução — adicionando uma etapa que o Node.js 24 elimina.

---

## Bloco 7 — Servidor HTTP Nativo em TypeScript

### Propósito da versão nativa

O servidor criado nesta aula usa `node:http` diretamente, sem Express. Essa escolha não é limitação — é decisão pedagógica deliberada. A aula seguinte introduz o Express como abstração sobre `node:http`. Para que essa abstração seja compreendida, e não apenas utilizada, é necessário ter construído a versão sem abstração imediatamente antes.

O servidor desta aula será refatorado na próxima. O que muda não é o comportamento observável — o servidor continua respondendo a requisições HTTP — mas a forma como esse comportamento é expresso no código.

### Anotação de tipos em `req` e `res`

O handler do servidor em `node:http` recebe dois parâmetros: o objeto de requisição e o objeto de resposta. Em JavaScript puro, esses parâmetros não têm tipo declarado. O TypeScript exige que cada valor tenha tipo anotado ou inferido.

O pacote `@types/node`, instalado no Bloco 4, fornece os tipos corretos para os objetos do módulo `node:http`. O parâmetro de requisição é do tipo `http.IncomingMessage`; o parâmetro de resposta é do tipo `http.ServerResponse`. Com essas anotações, o TypeScript conhece todas as propriedades e métodos disponíveis em cada objeto e emite erro imediato se o código tentar acessar algo inexistente.

### Leitura da porta via `process.env`

A porta não é escrita como número literal no código. Ela é lida de `process.env.PORT` — a variável carregada do `.env` pela flag `--env-file`. Como `process.env.PORT` é sempre uma *string*, a conversão para número é feita com `Number()`. O valor padrão é declarado com `||`:

```typescript
const PORT = Number(process.env.PORT) || 3001;
```

`Number(undefined)` produz `NaN`, não `undefined`. `NaN` é *falsy*, então `NaN || 3001` avalia para `3001`. O servidor sempre dispõe de uma porta válida, mesmo que o `.env` esteja ausente.

### Arquivo `src/server.ts`

```typescript
import http from 'node:http';

const PORT = Number(process.env.PORT) || 3001;

const server = http.createServer(
  (req: http.IncomingMessage, res: http.ServerResponse) => {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok' }));
  }
);

server.listen(PORT, () => {
  console.log(`Servidor rodando na porta ${PORT}`);
});
```

**`import http from 'node:http'`**

**Finalidade.** Esta instrução importa o módulo nativo de HTTP do Node.js, que fornece a capacidade de criar servidores e lidar com requisições e respostas.

**Motivação.** Sem este módulo, não há como criar um servidor HTTP — ele é a única interface entre o código e a camada de rede do sistema operacional.

**Decisão.** O prefixo `node:` torna explícito que se trata de um módulo nativo — não de um pacote instalado via `npm`. É a convenção moderna e recomendada pelo Node.js para importação de módulos nativos.

**Localização.** Usado nas linhas seguintes para chamar `http.createServer()` e para referenciar os tipos `http.IncomingMessage` e `http.ServerResponse`.

**Contrafactual.** Sem este `import`, qualquer referência a `http` produziria `ReferenceError: http is not defined`.

---

**`const PORT = Number(process.env.PORT) || 3001`**

**Finalidade.** Esta linha lê a variável de ambiente `PORT`, converte de *string* para número e define `3001` como valor padrão caso a variável esteja ausente ou inválida.

**Motivação.** A porta não deve ser um valor literal no código — precisa ser configurável por ambiente sem alterar o código-fonte.

**Decisão.** `Number()` realiza a conversão explícita, porque `process.env.PORT` é sempre *string* e `.listen()` espera número. O `|| 3001` garante o *fallback* quando `Number()` produz `NaN`.

**Localização.** Passado para `server.listen()` e interpolado na mensagem de log.

**Contrafactual.** Sem a conversão `Number()`, o TypeScript emitiria erro de tipo ao passar a *string* `'3001'` diretamente para `.listen()`, que espera `number`.

---

**`const server = http.createServer(...)`**

**Finalidade.** `http.createServer()` cria uma instância de servidor HTTP, registrando a função *handler* que será chamada a cada requisição recebida.

**Motivação.** O servidor precisa existir como objeto antes de poder escutar em uma porta. Criação e início da escuta são duas operações separadas.

**Decisão.** O *handler* é passado diretamente no construtor em vez de registrado via `.on('request', handler)`. As duas formas são equivalentes, mas passar no construtor é mais conciso.

**Localização.** A variável `server` é usada na linha seguinte para chamar `.listen()`.

**Contrafactual.** Sem atribuir a `const server`, a chamada `.listen()` precisaria ser encadeada diretamente em `http.createServer(...).listen(...)`, o que funciona, mas reduz a legibilidade.

---

**`(req: http.IncomingMessage, res: http.ServerResponse)`**

**Finalidade.** Estas anotações declaram os tipos dos parâmetros do *handler*, informando ao TypeScript o que `req` e `res` representam.

**Motivação.** Com os tipos anotados, o TypeScript conhece todas as propriedades e métodos disponíveis em cada objeto e emite aviso imediato se o código tentar acessar algo inexistente.

**Decisão.** Os tipos específicos de `node:http` são usados em vez do tipo genérico `any`. Com `"strict": true` ativo, o TypeScript emitiria erro de tipo se os parâmetros ficassem sem anotação.

**Localização.** Ativo durante toda a verificação de tipos pelo TypeScript. Sem efeito em tempo de execução após o *type stripping*.

**Contrafactual.** Sem as anotações, o TypeScript inferiria `any` implícito para `req` e `res` e, com `"strict": true`, emitiria erro imediato exigindo anotação explícita.

---

**`res.writeHead(200, { 'Content-Type': 'application/json' })`**

**Finalidade.** `res.writeHead()` define o *status code* da resposta como `200` e o cabeçalho `Content-Type` como `application/json`.

**Motivação.** O cliente precisa saber que o corpo da resposta é JSON para interpretá-lo corretamente. Sem `Content-Type`, o cliente pode tratar o corpo como texto simples.

**Decisão.** `writeHead` define *status* e cabeçalhos em uma única chamada, em vez de `res.statusCode = 200` seguido de `res.setHeader(...)` separadamente.

**Localização.** Chamado antes de `res.end()` — os cabeçalhos devem ser enviados antes do corpo da resposta.

**Contrafactual.** Se `res.end()` fosse chamado antes de `res.writeHead()`, o Node.js enviaria os cabeçalhos padrão automaticamente e a tentativa posterior de definir novos cabeçalhos produziria o erro *"Cannot set headers after they are sent"*.

---

**`res.end(JSON.stringify({ status: 'ok' }))`**

**Finalidade.** `res.end()` serializa o objeto `{ status: 'ok' }` para *string* JSON e o envia como corpo da resposta, encerrando a conexão.

**Motivação.** `res.end()` é a chamada que envia efetivamente a resposta ao cliente e fecha a conexão. Sem ela, a requisição fica pendente indefinidamente.

**Decisão.** `JSON.stringify()` é necessário porque `res.end()` aceita *string* ou `Buffer` — não objeto JavaScript diretamente.

**Localização.** Sempre a última operação sobre `res` em qualquer *handler*.

**Contrafactual.** Omitir `res.end()` faz o navegador aguardar a resposta até o *timeout*. A requisição trava e nunca é concluída.

---

**`server.listen(PORT, () => { ... })`**

**Finalidade.** `server.listen()` instrui o servidor a começar a aceitar conexões na porta declarada, executando o *callback* quando estiver pronto.

**Motivação.** `createServer` apenas cria o servidor — ele não começa a aceitar conexões até que `.listen()` seja chamado.

**Decisão.** O *callback* de `listen` é usado para o log de confirmação. Ele é executado uma única vez, quando o servidor está de fato pronto para receber requisições.

**Localização.** Chamado uma única vez na inicialização do processo, como última instrução do arquivo.

**Contrafactual.** Sem `.listen()`, o processo Node.js iniciaria, executaria o arquivo inteiro e encerraria imediatamente — sem nenhum servidor em execução.
