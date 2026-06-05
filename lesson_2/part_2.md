# Parte 2

## Bloco 1 — O que é o `tsconfig.json`

O ponto de partida é o problema concreto, antes da definição.

Considere o gerente de uma gráfica. Toda vez que um designer entrega um arquivo para imprimir, ele chega sem nenhuma instrução: sem especificar papel, tamanho, cor, acabamento. O gerente precisa adivinhar. Às vezes acerta, às vezes imprime no tamanho errado, às vezes usa papel brilhante quando era para ser fosco. Agora, considere que existe um formulário padrão que o designer preenche antes de entregar o arquivo: *"papel A4, cor preto-e-branco, sem acabamento"*. A gráfica lê o formulário e age com precisão.

O `tsconfig.json` é esse formulário. Ele fica na raiz do projeto e diz ao TypeScript como tratar o código — antes que qualquer verificação ou execução aconteça.

---

### Leitores do arquivo

Quatro atores distintos leem o `tsconfig.json`, e isso é importante porque revela o alcance da configuração:

O `tsc` — o compilador do TypeScript — lê para saber como verificar tipos e, opcionalmente, como emitir arquivos JavaScript. O VSCode lê para alimentar o **IntelliSense**: sublinhados vermelhos, sugestões de autocompletar, avisos de tipo. Linters — que serão introduzidos na próxima aula — leem para entender as mesmas regras que o compilador aplicaria. Executores de teste — que aparecerão nas aulas finais — leem para configurar o ambiente de análise de tipos nos testes.

Um único arquivo, quatro consumidores. Por isso, configurar corretamente tem impacto em toda a experiência de desenvolvimento.

### Não-leitores do arquivo

O runtime do Node.js não lê o `tsconfig.json`. Ao executar `node playground.ts`, o Node.js faz **type stripping** — remove as anotações TypeScript e executa o JavaScript resultante. Ele não consulta o `tsconfig.json`. Não verifica tipos. Não aplica nenhuma das restrições declaradas lá.

Isso consolida algo que apareceu nas últimas aulas: verificar e executar são operações separadas, feitas por ferramentas separadas, com responsabilidades separadas. O `tsconfig.json` configura a verificação. O Node.js gerencia a execução.

---

## Bloco 2 — Criação Manual

Existe um comando chamado `tsc --init` que gera um `tsconfig.json` automaticamente. O curso não usa esse comando, e a razão é pedagógica mas também prática: o arquivo gerado tem mais de 100 linhas, a maioria comentada, com opções que não foram justificadas. O arquivo é copiado, o VSCode para de reclamar, e ele se torna um artefato mágico no projeto — presente, mas opaco.

Construir linha por linha força uma decisão consciente sobre cada opção. Ao final desta aula, cada linha do `tsconfig.json` terá uma razão que pode ser articulada.

### Estrutura mínima

O `tsconfig.json` é um arquivo JSON com uma chave obrigatória no nível raiz:

```json
{
  "compilerOptions": {}
}
```

Só isso. Tudo que configura o comportamento do TypeScript vai dentro de `"compilerOptions"`. O objeto pode estar vazio inicialmente — o TypeScript aceita e usa seus defaults — mas será preenchido completamente ao longo dos blocos seguintes.

A localização é sempre a raiz do projeto: o mesmo diretório onde fica o `package.json`. Quando o `tsc` é invocado sem argumentos, ele sobe no sistema de arquivos a partir do diretório atual procurando esse arquivo. Quando o VSCode abre uma pasta, ele também o procura na raiz.

---

## Bloco 3 — Opções de Saída e Execução

Este bloco cobre quatro opções que definem como o TypeScript se relaciona com o JavaScript gerado — e uma delas tem uma decisão que só faz sentido no contexto do Node.js 24.

### `"target": "esnext"`

O TypeScript pode *compilar para baixo*: transformar JavaScript moderno em JavaScript compatível com versões mais antigas. Por exemplo, ao usar `async/await` com `target: "es5"`, o TypeScript transforma tudo em código com callbacks e `Promise.then`, porque o ES5 não tem `async/await` nativo.

`"esnext"` diz o oposto: use o JavaScript mais moderno disponível, sem nenhuma transformação para baixo. Isso faz sentido aqui porque o Node.js 24 suporta tudo que o JavaScript moderno oferece — não há razão para degradar.

### `"module": "nodenext"`

Define como o TypeScript deve tratar o sistema de módulos. O JavaScript tem dois sistemas: **CommonJS** (o antigo, com `require` e `module.exports`) e **ES Modules** (o moderno, com `import` e `export`). O Node.js 24 suporta os dois, mas tem suas próprias regras sobre como resolvê-los — por exemplo, imports com extensão explícita `.js` ou `.ts`, e o campo `"type": "module"` no `package.json`.

`"nodenext"` instrui o TypeScript a seguir exatamente as regras do Node.js. Não inventar. Não assumir. Replicar o comportamento que o runtime vai ter.

### `"moduleResolution": "nodenext"`

`module` e `moduleResolution` são opções distintas. A primeira diz qual formato de módulo usar; a segunda diz como resolver os caminhos dos imports — ou seja, como o TypeScript vai encontrar o arquivo ao processar `import { something } from './utils.ts'`. Com `"nodenext"`, ele segue o algoritmo de resolução do próprio Node.js 24. As duas opções caminham juntas: quando `module` é `"nodenext"`, `moduleResolution` deve ser `"nodenext"` também.

### `"noEmit": true`

Esta é a decisão mais importante deste bloco, e ela só existe porque o Node.js 24 faz type stripping.

Normalmente, o fluxo TypeScript clássico é: escrever `.ts` → compilar com `tsc` → obter `.js` → executar o `.js`. O TypeScript emite os arquivos JavaScript. Nesse fluxo, `noEmit` seria falso: os arquivos emitidos são o objetivo.

No Node.js 24, o fluxo é diferente: escrever `.ts` → executar diretamente com `node`. O Node.js remove as anotações em memória e executa. Não há etapa de compilação separada. Não há arquivos `.js` gerados.

Se `noEmit` for `false` nesse contexto, o `tsc` vai gerar arquivos `.js` ao lado dos arquivos `.ts`. O diretório acumulará duplicatas. Pior: ao editar o `.ts`, o `.js` desatualizado ainda estará lá. Com `"noEmit": true`, o `tsc` apenas verifica os tipos e para. Ele não toca no sistema de arquivos. Essa é a única função relevante neste contexto: verificação estática sem efeitos colaterais.

Se omitida ou definida como `false`, a flag `--noEmit` ainda funciona quando passada explicitamente na linha de comando via `tsc --noEmit`. Mas qualquer outro consumidor poderia acionar a emissão e gerar arquivos `.js` indesejados no projeto.

---

## Bloco 4 — Opções de Rigor

### `"strict": true`

`strict` não é uma opção única. É um atalho que habilita um conjunto de verificações estritas de uma só vez. As principais são:

- `strictNullChecks`: por padrão, sem essa flag, `null` e `undefined` são atribuíveis a qualquer tipo. Uma variável anotada como `string` poderia receber `null` sem reclamação. Com `strictNullChecks`, isso vira erro. É necessário declarar explicitamente `string | null` para aceitar `null`.

- `noImplicitAny`: sem essa flag, quando o TypeScript não consegue inferir o tipo de um parâmetro, ele silenciosamente assume `any`. Com essa flag, isso vira erro — a anotação explícita passa a ser obrigatória.

- `strictFunctionTypes`: verifica a contravariância de tipos em funções, impedindo atribuições que parecem compatíveis mas não são de forma segura.

- `useUnknownInCatchVariables`: em blocos `catch`, o erro capturado passa a ter tipo `unknown` em vez de `any`, forçando o **narrowing** antes do uso.

Com `"strict": true`, todas essas são habilitadas de uma vez. Em projetos reais, é o padrão. O custo é que mais anotações precisam ser escritas; o benefício é que a verificação realmente protege.

Sem ele, o TypeScript é consideravelmente mais permissivo. Parâmetros sem anotação viram `any` sem aviso. `null` pode aparecer onde não deveria. A ferramenta existe, mas opera com metade da capacidade.

---

## Bloco 5 — Opção de Escrita

### `erasableSyntaxOnly`

Esta é a opção mais específica desta parte. Para entendê-la, é necessário retomar como o Node.js 24 executa TypeScript.

Type stripping é uma operação de remoção. O Node.js pega o arquivo `.ts`, apaga as anotações de tipo, e executa o JavaScript que sobra. É como apagar lápis de um texto: o papel (o código JavaScript) fica intacto; só as marcações a lápis (os tipos) somem.

Esse processo funciona perfeitamente para anotações puras — `: string`, `: number`, `type Product = ...`, `: Tarefa[]`. Essas anotações não geram nenhum código JavaScript. Elas existem apenas na análise estática e desaparecem sem deixar rastro.

O TypeScript, porém, tem algumas construções que não são apenas anotações. Elas requerem geração de código JavaScript real para funcionar:

- `enum`: em TypeScript, `enum Color { Red, Blue }` gera um objeto JavaScript complexo em runtime. Não é uma anotação — é código executável. Se o Node.js fizer apenas type stripping, o `enum` simplesmente desaparece, e o código que dependia dele quebra.

- `namespace`: similar — gera código de agrupamento em JavaScript.

`"erasableSyntaxOnly": true` instrui o compilador a rejeitar qualquer uso dessas construções. Ao escrever `enum` no projeto, `tsc --noEmit` acusa erro imediatamente. Isso garante que todo o código TypeScript do projeto é compatível com type stripping — ou seja, que o Node.js 24 pode executá-lo corretamente sem uma etapa de compilação separada.

É por isso que na aula anterior o `enum` foi declarado proibido e o padrão `as const` foi apresentado como substituto. A proibição não é arbitrária: ela decorre diretamente desta configuração.

Sem `erasableSyntaxOnly`, seria possível escrever `enum`, o TypeScript não reclamaria, o VSCode não sublinharia nada em vermelho — mas ao executar o arquivo, o `enum` estaria ausente e o programa quebraria em runtime, provavelmente com `ReferenceError`. O erro apareceria tarde, longe da causa.

---

## Bloco 6 — Opções de Módulos

### `"verbatimModuleSyntax": true`

Ao importar apenas um tipo — por exemplo, `import { Product } from './types.ts'` onde `Product` é um type alias — essa importação não precisa existir em runtime. Tipos desaparecem no type stripping. Se o import sobrar no JavaScript gerado, ele tentará resolver um módulo que pode não existir ou pode ter efeitos colaterais indesejados ao ser importado.

`verbatimModuleSyntax` exige a declaração explícita de que um import é apenas de tipo: `import type { Product } from './types.ts'`. Com a palavra `type`, o TypeScript sabe que esse import deve ser completamente removido. Sem ela, ele assume que o import precisa sobreviver.

Isso elimina uma categoria inteira de bugs silenciosos onde imports de tipos *vazam* para o JavaScript gerado.

### `"allowImportingTsExtensions": true`

Por padrão, o TypeScript não permite escrever `import { something } from './utils.ts'` com a extensão `.ts` literal. A convenção histórica era escrever `.js` mesmo no código TypeScript — o TypeScript resolveria para o arquivo `.ts` correspondente.

No Node.js 24, a recomendação mudou: use a extensão real. `import { something } from './utils.ts'` é o que o Node.js espera ver. Esta opção habilita essa sintaxe.

### `"rewriteRelativeImportExtensions": true`

Complementa a anterior. Quando o `tsc` encontra um import com `.ts`, ele pode reescrever para `.js` em um arquivo de saída. Como `noEmit` é `true` e não há arquivo de saída, essa reescrita afeta o que algumas ferramentas enxergam. A combinação das duas opções mantém os imports com `.ts` coerentes em todo o conjunto de ferramentas.

### `"esModuleInterop": true`

Algumas bibliotecas do ecossistema Node.js ainda são publicadas em CommonJS — o formato antigo com `module.exports`. Por padrão, importar um módulo CommonJS com sintaxe ES Module pode gerar incompatibilidades. `esModuleInterop` habilita um mecanismo de compatibilidade que permite `import lib from 'lib'` funcionar mesmo que internamente use `module.exports`. Sem isso, seria necessário escrever `import * as lib from 'lib'`, que é mais verboso e menos idiomático.

---

## Bloco 7 — Opções de Robustez

### `"skipLibCheck": true`

Ao instalar uma biblioteca via `npm`, ela vem com arquivos `.d.ts` — arquivos de declaração de tipos. O TypeScript pode verificar esses arquivos também. Na prática, isso raramente encontra problemas relevantes para o código do projeto, mas pode ser lento — especialmente em projetos com muitas dependências — e pode gerar erros de tipos em bibliotecas de terceiros que não estão sob controle do projeto e não podem ser corrigidas.

`skipLibCheck` instrui o TypeScript a pular a verificação de arquivos de declaração de bibliotecas. Ele ainda usa as informações de tipo das bibliotecas para verificar o código do projeto; ele apenas não verifica se os arquivos `.d.ts` das bibliotecas são internamente consistentes. É a decisão padrão em projetos profissionais.

### `"forceConsistentCasingInFileNames": true`

Sistemas de arquivos se comportam de forma diferente em diferentes sistemas operacionais. O Linux é **case-sensitive**: `Utils.ts` e `utils.ts` são arquivos distintos. O macOS, por padrão, é **case-insensitive**: os dois nomes apontam para o mesmo arquivo. O Windows também é case-insensitive por padrão.

Isso cria um problema clássico: o desenvolvimento ocorre no Windows, o import é escrito como `import { something } from './Utils.ts'`, mas o arquivo no disco se chama `utils.ts`. No Windows, funciona. O código vai para um servidor Linux, e `Utils.ts` não é encontrado.

`forceConsistentCasingInFileNames` instrui o TypeScript a tratar imports como case-sensitive independentemente do sistema operacional. Se o arquivo no disco se chama `utils.ts`, o import deve usar `utils.ts`. Isso previne a categoria inteira de bugs que só aparecem em produção.

---

## Bloco 8 — Verificação versus Execução

Este bloco consolida a distinção que atravessou toda a aula.

Dois comandos operam sobre o mesmo arquivo `.ts`, mas fazem coisas completamente diferentes:

`tsc --noEmit` verifica tipos. Lê o código, analisa as anotações, verifica coerência, reporta erros estáticos. Não executa nenhuma linha. Não produz saída. Termina com código `0` se tudo estiver correto, código `1` se houver erros de tipo. É a ferramenta do TypeScript operando como TypeScript.

`node file.ts` executa o código. Faz type stripping, remove as anotações, executa o JavaScript resultante. Não verifica tipos. Não reporta erros estáticos. Se houver um erro de tipo — por exemplo, uma `string` passada onde era esperado `number` — o Node.js não sabe disso e executa assim mesmo, com o comportamento que o JavaScript puro teria.

As duas operações são complementares. Em um projeto real, ambas são utilizadas: `tsc --noEmit` no desenvolvimento para identificar erros antes da execução, e `node` para executar.

Sem o `tsconfig.json`, o `tsc --noEmit` opera com defaults que podem não corresponder ao Node.js 24. Com o `tsconfig.json` correto, as duas ferramentas operam com as mesmas premissas sobre o código.
