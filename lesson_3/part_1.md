# Parte 1

## Bloco 1 — Por que Linters e Formatadores Existem

### Distinção entre formatador e linter

Duas ferramentas distintas operam sobre o mesmo código-fonte com objetivos diferentes.

Um **formatador** aplica regras de apresentação visual ao código: indentação, espaçamento, posição de aspas, quebra de linha. O conteúdo do programa não é alterado — o que o código faz permanece idêntico. O que muda é a forma como ele é apresentado no editor.

Um **linter** analisa o código em busca de padrões problemáticos que não são questões de aparência, mas de qualidade ou segurança: variáveis declaradas e nunca utilizadas, comparações com coerção implícita, código morto. O linter reporta ocorrências e pode sugerir ou aplicar correções, mas não reescreve a aparência do arquivo.

---

### Justificativa de existência do formatador

Sem um formatador, cada desenvolvedor escreve conforme o hábito pessoal e a configuração do editor. Em um repositório compartilhado, isso produz diferenças espúrias no `git diff` — alterações de indentação ou aspas que não correspondem a mudanças funcionais. A revisão de código torna-se difícil porque o ruído de formatação se mistura às mudanças reais.

---

### Justificativa de existência do linter

O compilador TypeScript detecta erros de tipo; o linter detecta uma categoria diferente de problema: padrões que são válidos para o compilador mas indicam descuido ou risco. Uma variável declarada e nunca lida pode representar lógica incompleta. Uma comparação com `==` pode produzir resultado inesperado por coerção implícita. Sem o linter, esses padrões se acumulam silenciosamente.

---

### Justificativa de unificação em ferramenta única

Historicamente, o ecossistema JavaScript mantinha ferramentas separadas para os dois papéis, o que criava fricção: configuração dupla, possibilidade de conflito entre regras, e custo de execução sequencial. **Biome** unifica os dois papéis em uma única ferramenta, escrita em Rust, com uma única configuração e sem conflitos internos.

---

## Bloco 2 — Instalação e Inicialização do Biome

### Pré-requisito: inicialização do `package.json`

Biome é um pacote `npm`. Para instalá-lo, o diretório de trabalho precisa de um `package.json`. A inicialização é feita com dois comandos executados na raiz do diretório:

```shell
npm init -y
```

```shell
npm pkg set type="module"
```

**Finalidade.** O primeiro comando cria o `package.json` com valores padrão, sem interação. O segundo adiciona `"type": "module"` ao arquivo, instruindo o Node.js a tratar os arquivos `.js` do projeto como ES Modules.

**Motivação.** Sem o `package.json`, o `npm` não tem onde registrar as dependências instaladas. Sem `"type": "module"`, o Node.js interpretaria os arquivos como CommonJS, divergindo do padrão adotado no curso desde a Parte 2.

**Decisão.** O sinalizador `-y` dispensa a etapa interativa de preenchimento de campos. Em um diretório de experimentação, os valores padrão gerados são suficientes.

**Localização.** O `package.json` fica na raiz do projeto e é o ponto de referência do `npm` para instalação, execução de scripts e registro de dependências.

**Contrafactual.** Sem o `package.json`, qualquer tentativa de `npm install` produziria erro, pois o `npm` não teria como determinar onde registrar a dependência.

---

### Instalação por projeto com `--save-exact`

```shell
npm install --save-dev --save-exact @biomejs/biome
```

**Finalidade.** Este comando instala o Biome como dependência de desenvolvimento e registra a versão exata no `package.json`, sem o prefixo `^`.

**Motivação.** O Biome formata código aplicando regras de uma versão específica. Se desenvolvedores diferentes do mesmo projeto usassem versões distintas, o mesmo arquivo poderia ser formatado de formas diferentes, introduzindo diferenças espúrias no repositório.

**Decisão.** O sinalizador `--save-exact` remove o prefixo `^` do registro no `package.json`. Com ele, toda instalação futura — na mesma máquina, em outra máquina ou em ambiente de CI — instala exatamente a mesma versão. Sem `--save-exact`, o `npm` registraria `"^versão"`, permitindo atualizações automáticas a versões compatíveis que podem alterar sutilmente o comportamento da formatação.

**Localização.** O sinalizador é aplicado exclusivamente ao Biome, pois é a ferramenta cujos resultados devem ser bit-a-bit idênticos entre todos os ambientes.

**Contrafactual.** Sem `--save-exact`, um colaborador que fizesse `npm install` semanas depois poderia receber uma versão mais nova do Biome. Um commit subsequente reformataria arquivos não modificados funcionalmente, poluindo o histórico com diferenças sem significado.

---

### Inicialização com `npx biome init`

```shell
npx biome init
```

**Finalidade.** Este comando gera o arquivo `biome.json` na raiz do projeto com uma configuração funcional inicial.

**Motivação.** O Biome precisa de um arquivo de configuração explícito para dois fins: garantir que todos os ambientes usem as mesmas regras, e permitir que ferramentas como o Lefthook encontrem a configuração ao invocar o Biome automaticamente.

**Decisão.** A inicialização via `npx biome init` é preferível à criação manual do `biome.json` porque garante que a estrutura mínima esteja correta e que o campo `$schema` aponte para a versão instalada.

**Localização.** O `biome.json` fica na raiz do projeto, ao lado do `package.json`, e é lido por todas as ferramentas que invocam o Biome: o terminal, o VSCode e os Git hooks configurados pelo Lefthook.

**Contrafactual.** Sem o `biome.json`, o Biome operaria com valores padrão embutidos, que não são versionados no repositório. Colaboradores diferentes poderiam obter comportamentos divergentes dependendo da versão instalada e dos defaults de cada versão.

---

### Anatomia do `biome.json`

O arquivo gerado pelo `npx biome init` contém os seguintes campos:

```json
{
  "$schema": "https://biomejs.dev/schemas/2.5.0/schema.json",
  "vcs": {
    "enabled": true,
    "clientKind": "git",
    "useIgnoreFile": true
  },
  "files": {
    "ignoreUnknown": false
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "tab"
  },
  "linter": {
    "enabled": true,
    "rules": {
      "preset": "recommended"
    }
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double"
    }
  },
  "assist": {
    "enabled": true,
    "actions": {
      "source": {
        "organizeImports": "on"
      }
    }
  }
}
```

**`$schema`**

**Finalidade.** Aponta para o schema JSON oficial do Biome na versão instalada, permitindo que o VSCode valide o arquivo de configuração e ofereça autocomplete.

**Motivação.** Sem o `$schema`, erros de digitação em nomes de campos passariam despercebidos até o momento em que o Biome fosse executado.

**Decisão.** O caminho do schema inclui a versão exata, garantindo que a validação corresponda ao comportamento real da versão instalada.

**Localização.** O campo é lido exclusivamente pelo editor; não afeta a execução do Biome no terminal.

**Contrafactual.** Sem o `$schema`, o arquivo ainda funcionaria, mas o editor não ofereceria validação nem autocomplete — erros de configuração seriam descobertos apenas em execução.

---

**`vcs`**

**Finalidade.** Declara que o Biome opera dentro de um repositório Git e deve respeitar o `.gitignore` do projeto.

**Motivação.** Sem essa configuração, o Biome tentaria processar todos os arquivos do diretório, incluindo os listados no `.gitignore` — como `node_modules/`, que pode conter centenas de milhares de arquivos.

**Decisão.** `"clientKind": "git"` declara o sistema de controle de versão. `"useIgnoreFile": true` instrui o Biome a ler o `.gitignore` e excluir os arquivos listados do processamento.

**Localização.** A configuração tem efeito em todos os comandos do Biome que percorrem o sistema de arquivos.

**Contrafactual.** Sem a seção `vcs`, a execução de `npx biome check --write .` percorreria `node_modules/`, tornando-a extremamente lenta e produzindo avisos de lint em código de bibliotecas que não estão sob controle do projeto.

---

**`files`**

**Finalidade.** Controla o comportamento do Biome ao encontrar arquivos de tipos desconhecidos.

**Motivação.** Com `"ignoreUnknown": false`, o Biome reporta arquivos que não sabe processar, mantendo visibilidade sobre o que está sendo incluído ou excluído da análise.

**Decisão.** O valor `false` é o padrão; `true` faria o Biome ignorar silenciosamente arquivos desconhecidos.

**Localização.** Afeta a saída do terminal ao executar qualquer comando que percorra arquivos.

**Contrafactual.** Com `"ignoreUnknown": true`, arquivos de tipos não suportados seriam silenciosamente ignorados, sem nenhuma indicação de que estão fora do escopo da análise.

---

**`formatter` e `linter`**

**Finalidade.** Habilitam o formatador e o linter, respectivamente. `"indentStyle": "tab"` define tabulação como caractere de indentação. `"preset": "recommended"` ativa o conjunto de regras recomendadas pela equipe do Biome.

**Motivação.** A escolha explícita de `"tab"` e `"recommended"` garante que todos os ambientes compartilhem as mesmas decisões de estilo e qualidade, sem depender de defaults implícitos.

**Decisão.** O conjunto `recommended` cobre os padrões problemáticos mais frequentes sem exigir configuração manual de regras individuais.

**Localização.** Estas configurações são aplicadas em todos os comandos operacionais do Biome.

**Contrafactual.** Com `"enabled": false` em qualquer uma das seções, o papel correspondente seria desativado completamente — o Biome só formataria, ou só faria lint, perdendo a vantagem da ferramenta unificada.

---

**`javascript.formatter.quoteStyle`**

**Finalidade.** Define aspas duplas como estilo padrão para strings em arquivos JavaScript e TypeScript.

**Motivação.** Sem uma escolha explícita, diferentes desenvolvedores podem escrever aspas simples e duplas no mesmo arquivo. A configuração impõe consistência: todo arquivo formatado pelo Biome usará aspas duplas.

**Decisão.** A escolha entre aspas simples e duplas é de consistência, não de funcionalidade. O curso adota aspas duplas, alinhadas ao padrão do Biome.

**Localização.** Afeta exclusivamente o formatador para arquivos `.js` e `.ts`.

**Contrafactual.** Sem esta configuração, o Biome usaria o default da versão instalada, que pode diferir entre versões e entre membros da equipe.

---

**`assist`**

**Finalidade.** Habilita ações de organização de código que não são formatação nem lint: neste caso, a reorganização automática da ordem dos imports.

**Motivação.** Imports desordenados são tecnicamente válidos mas reduzem a legibilidade. A organização automática elimina essa fonte de inconsistência sem exigir atenção manual do desenvolvedor.

**Decisão.** Na versão 2.x do Biome, a organização de imports foi movida de `"organizeImports"` na raiz do arquivo para dentro da seção `"assist"`. A funcionalidade é a mesma; a localização na configuração mudou entre versões.

**Localização.** Executado durante `npx biome check --write .` e pelo *format on save* no VSCode.

**Contrafactual.** Sem esta seção, os imports não seriam reorganizados automaticamente, e a ordem dependeria da sequência em que o desenvolvedor os escreveu.

---

## Bloco 3 — Comandos Operacionais do Biome

### `npx biome format --write .`

```shell
npx biome format --write .
```

**Finalidade.** Aplica formatação a todos os arquivos do projeto a partir do diretório atual, reescrevendo-os em disco.

**Motivação.** O comando é útil para formatar todos os arquivos de uma vez — por exemplo, ao configurar o Biome em um projeto que já continha arquivos existentes.

**Decisão.** O sinalizador `--write` autoriza a modificação em disco. Sem ele, o comando exibe o diff de formatação sem aplicar nenhuma mudança. O ponto `.` indica o diretório atual como raiz da varredura.

**Localização.** Utilizado pontualmente, quando se quer garantir que todos os arquivos do projeto estejam formatados, independentemente de quando foram editados pela última vez.

**Contrafactual.** Sem `--write`, o comando opera em modo somente-leitura: reporta o que seria alterado, mas não altera nada.

---

### `npx biome lint .`

```shell
npx biome lint .
```

**Finalidade.** Analisa todos os arquivos do projeto em busca de padrões problemáticos e reporta os avisos no terminal, sem modificar arquivos.

**Motivação.** O lint permite inspecionar a qualidade do código antes de um commit, identificando variáveis não utilizadas, comparações suspeitas e outros padrões que o compilador TypeScript não detecta.

**Decisão.** O comando não aplica correções automaticamente por padrão, pois algumas correções de lint exigem decisão do desenvolvedor — não são mecânicas como as de formatação.

**Localização.** Utilizado antes de commits ou em pipelines de CI para verificar a qualidade do código sem modificar arquivos.

**Contrafactual.** Sem a execução regular do linter, padrões problemáticos se acumulam no repositório sem aviso, tornando o código progressivamente mais difícil de manter.

---

### `npx biome check --write .`

```shell
npx biome check --write .
```

**Finalidade.** Executa formatação e lint em uma única passagem, aplicando correções automáticas quando possível.

**Motivação.** A combinação dos dois papéis em um único comando reduz a fricção do fluxo de desenvolvimento: um único comando garante que o código está formatado e sem padrões problemáticos antes de um commit.

**Decisão.** Este é o comando padrão do curso e o que o Lefthook invoca automaticamente no hook `pre-commit`. Correções de formatação são sempre aplicadas. Algumas correções de lint também são aplicadas; outras, que exigem decisão do desenvolvedor, apenas são reportadas.

**Localização.** Executado automaticamente pelo hook `pre-commit` antes de cada commit, e manualmente quando se quer verificar e corrigir o projeto inteiro.

**Contrafactual.** Sem `--write`, o comando reporta o que seria corrigido mas não aplica nenhuma mudança — comportamento adequado para inspeção, mas não para correção.

---

## Bloco 4 — Integração com o VSCode

### Extensão oficial do Biome

A extensão do Biome (`biomejs.biome`) é instalada pelo marketplace do VSCode (`Ctrl + Shift + X`, pesquisa por `Biome`). Esta é a primeira instalação de extensão de editor autorizada no curso. Nas Partes anteriores, a regra era zero extensões — o objetivo era desenvolver familiaridade com o editor sem atalhos artificiais. No Módulo Profissionalizar, ferramentas de mercado são adotadas por princípio pedagógico: a meta é replicar o ambiente de trabalho profissional.

A extensão tem dois efeitos: sublinha avisos de lint diretamente no editor em tempo real, sem necessidade de terminal; e aplica formatação automaticamente ao salvar, se configurada para isso.

---

### Configuração de *format on save* por projeto

A configuração é feita no arquivo `.vscode/settings.json`, criado na raiz do projeto:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "biomejs.biome"
}
```

**`editor.formatOnSave`**

**Finalidade.** Instrui o VSCode a acionar o formatador automaticamente toda vez que um arquivo é salvo com `Ctrl + S`.

**Motivação.** A formatação manual exige que o desenvolvedor se lembre de executá-la antes de cada commit. O *format on save* elimina essa dependência: o arquivo já está formatado no momento em que é salvo.

**Decisão.** A configuração é feita por projeto para evitar que todos os projetos do desenvolvedor herdem o comportamento, o que causaria conflitos em projetos que usam outros formatadores.

**Localização.** Lido pelo VSCode ao salvar qualquer arquivo do projeto.

**Contrafactual.** Sem esta configuração, a formatação automática não ocorre ao salvar. O desenvolvedor precisaria formatar manualmente pelo terminal antes de cada commit — ou depender do hook `pre-commit` do Lefthook como única garantia.

---

**`editor.defaultFormatter`**

**Finalidade.** Declara qual extensão o VSCode deve usar como formatador para os arquivos do projeto.

**Motivação.** Sem esta configuração, o VSCode pode utilizar um formatador diferente do Biome — ou solicitar uma escolha toda vez que o arquivo for salvo — produzindo resultados inconsistentes.

**Decisão.** O valor `"biomejs.biome"` é o identificador único da extensão oficial. A configuração é feita por projeto, em `.vscode/settings.json`, para não afetar outros projetos que possam usar formatadores diferentes.

**Localização.** Lido exclusivamente pelo VSCode ao abrir o projeto. Não afeta a execução do Biome pelo terminal ou pelo Lefthook.

**Contrafactual.** Sem esta configuração, a extensão do Biome estaria instalada mas inativa para formatação: sublinharia avisos de lint, mas não formataria automaticamente ao salvar.

---

## Bloco 5 — Por que Git Hooks Existem

### Conceito de hook

Um **Git hook** é um script executado automaticamente pelo Git em momentos específicos do fluxo de trabalho. O script é invocado pelo Git antes ou depois de uma operação; se retornar código de erro, a operação é interrompida.

Os hooks residem no diretório `.git/hooks/` do repositório, como scripts de shell. Por residir dentro de `.git/`, esse diretório não é versionado — o que historicamente obrigava cada desenvolvedor a configurar os hooks manualmente após clonar o repositório. O Lefthook resolve esse problema ao gerenciar os hooks como arquivos versionáveis fora do `.git/` e sincronizá-los automaticamente.

---

### Hook `pre-commit`

O hook `pre-commit` é executado pelo Git imediatamente antes de registrar um commit — após o comando `git commit`, mas antes que o commit seja gravado no histórico. Se o script retornar erro, o commit é cancelado. Este é o hook onde o Biome é invocado: qualquer problema de formatação não corrigível automaticamente, ou aviso de lint grave, bloqueia o commit.

---

## Bloco 6 — Instalação e Configuração do Lefthook

### Instalação

```shell
npm install --save-dev lefthook
```

**Finalidade.** Instala o Lefthook como dependência de desenvolvimento do projeto.

**Motivação.** O Lefthook gerencia os Git hooks como arquivos versionáveis, eliminando a necessidade de cada desenvolvedor configurar os hooks manualmente após clonar o repositório.

**Decisão.** A instalação é por projeto, sem `--save-exact`. O Lefthook é uma ferramenta de orquestração de scripts — não processa arquivos de código — e pequenas variações de versão não alteram o resultado do código produzido.

**Localização.** Registrado em `devDependencies` do `package.json`; não é incluído no bundle de produção.

**Contrafactual.** Sem o Lefthook, os hooks precisariam ser configurados manualmente por cada desenvolvedor dentro do diretório `.git/hooks/`, que não é versionado. Novos membros da equipe não receberiam os hooks ao clonar o repositório.

---

### Inicialização com `npx lefthook install`

```shell
npx lefthook install
```

**Finalidade.** Cria os scripts dentro de `.git/hooks/` que apontam para o Lefthook, tornando os hooks declarados no `lefthook.yml` efetivos.

**Motivação.** A instalação via `npm` disponibiliza o Lefthook como executável, mas o Git não o conhece. O `npx lefthook install` é o passo que conecta o Lefthook ao Git.

**Decisão.** O comando deve ser executado uma vez por ambiente, após o `npm install`. Em projetos profissionais, é comum adicioná-lo como script `postinstall` no `package.json` para automatizar a etapa.

**Localização.** Deve ser executado na raiz do repositório, no mesmo nível em que o diretório `.git/` existe.

**Contrafactual.** Sem `npx lefthook install`, o `lefthook.yml` pode existir e estar correto, mas nenhum hook será executado. O Git criará commits normalmente, ignorando completamente o Lefthook. Este é o erro mais comum ao configurar a ferramenta pela primeira vez.

---

### O arquivo `lefthook.yml`

```yaml
pre-commit:
  commands:
    biome:
      run: npx biome check --write .
      stage_fixed: true
```

**`pre-commit`**

**Finalidade.** Declara o bloco de comandos a ser executado pelo hook `pre-commit` — imediatamente antes de cada commit.

**Motivação.** O hook `pre-commit` é o ponto de interceção onde a verificação de qualidade é garantida, independentemente de o desenvolvedor ter lembrado de rodar o Biome manualmente.

**Decisão.** O Lefthook permite múltiplos hooks no mesmo arquivo, cada um com seu bloco separado. O `pre-commit` é o primeiro a ser configurado por ser o mais crítico para a qualidade do repositório.

**Localização.** Lido pelo Lefthook toda vez que o Git executa o hook `pre-commit`.

**Contrafactual.** Sem este bloco, o Lefthook estaria instalado mas sem nenhum hook configurado — o comportamento do Git seria idêntico ao de não ter o Lefthook.

---

**`biome`**

**Finalidade.** Nome atribuído ao comando dentro do hook `pre-commit`. Identifica este comando específico nas mensagens de saída do Lefthook.

**Motivação.** Um hook pode ter múltiplos comandos. O nome permite distinguir qual comando está sendo executado e qual falhou, quando há mais de um.

**Decisão.** O nome pode ser qualquer string; `biome` descreve com precisão o que o comando faz.

**Localização.** Aparece nas mensagens de saída do Lefthook durante a execução do hook.

**Contrafactual.** O nome não afeta a execução — qualquer nome válido em YAML funcionaria.

---

**`run: npx biome check --write .`**

**Finalidade.** Define o comando executado pelo Lefthook quando o hook `pre-commit` é acionado.

**Motivação.** Este é o mesmo comando que seria executado manualmente no terminal. O Lefthook o executa automaticamente, eliminando a dependência da memória do desenvolvedor.

**Decisão.** O comando `check --write .` foi escolhido por combinar formatação e lint em uma única passagem, com aplicação automática de correções.

**Localização.** Executado no diretório raiz do projeto, no contexto do hook `pre-commit`.

**Contrafactual.** Se o valor de `run` fosse omitido ou apontasse para um comando inexistente, o hook falharia com erro de execução e bloquearia todos os commits.

---

**`stage_fixed: true`**

**Finalidade.** Instrui o Lefthook a adicionar ao staging area do Git os arquivos modificados pelo Biome durante a execução do hook.

**Motivação.** Quando o Biome corrige formatação automaticamente, ele modifica arquivos em disco. Sem `stage_fixed: true`, essas correções existiriam no sistema de arquivos mas não entrariam no commit — o repositório receberia a versão não formatada, e as correções ficariam como alterações pendentes não commitadas.

**Decisão.** A opção é específica do Lefthook e não existe nativamente no Git. Ela foi adicionada para resolver exatamente esse caso: ferramentas que modificam arquivos durante o hook `pre-commit`.

**Localização.** Afeta exclusivamente o comportamento do Lefthook após a execução do comando declarado em `run`.

**Contrafactual.** Sem `stage_fixed: true`, o Biome formataria os arquivos em disco, mas o commit registraria a versão anterior — não formatada. O desenvolvedor precisaria fazer `git add` manualmente e commitar novamente, o que anula a automação do hook.

---

## Bloco 7 — Conventional Commits

### O padrão

**Conventional Commits** é uma convenção de mercado para mensagens de commit com estrutura padronizada e legível por humanos e máquinas:

```
tipo(escopo): descrição
```

---

### Tipos principais

| Tipo | Uso |
|---|---|
| `feat` | Funcionalidade nova adicionada |
| `fix` | Correção de bug |
| `refactor` | Reorganização de código sem mudança de comportamento externo |
| `docs` | Alteração exclusiva de documentação |
| `test` | Adição ou modificação de testes |
| `chore` | Tarefa de manutenção sem impacto no código de produção nem nos testes |

---

### Escopo

O escopo é opcional e identifica a área do sistema afetada pela mudança: `feat(auth): implementar middleware de autenticação`. A presença do escopo melhora a rastreabilidade do histórico em projetos com múltiplos módulos.

---

### Regras da descrição

A descrição é redigida em letras minúsculas, no imperativo, sem ponto final: *"adicionar"*, *"corrigir"*, *"extrair"* — não *"adicionado"*, *"corrigi"* ou *"extrai"*.

---

### Justificativa de existência

A convenção serve dois propósitos concretos. O primeiro é legibilidade imediata: ao inspecionar o `git log`, o tipo de cada commit classifica instantaneamente a natureza da mudança sem exigir leitura do corpo. O segundo é automação: ferramentas de geração de changelog podem construir documentos de mudanças automaticamente a partir do histórico, porque os tipos são padronizados e interpretáveis por máquina.
