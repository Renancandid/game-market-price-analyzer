# Destaques técnicos

O **Muito Duro** é uma aplicação web que consome uma API pública de preços em tempo real, cruza esses preços com um banco de ~7.000 receitas e calcula onde há lucro. Ela compara milhares de itens em 8 mercados e 5 qualidades.

Este documento mostra os **problemas reais** que apareceram no desenvolvimento e **como cada um foi resolvido**. O código citado está nos arquivos indicados.

| | |
|---|---|
| **Stack** | HTML, CSS e JavaScript puro (sem framework) · Node.js para o pipeline de dados · Cloudflare para o deploy |
| **Dados externos** | API REST do Albion Online Data Project (preços atuais e histórico) · dumps oficiais do jogo (JSON e XML) |
| **Escala** | ~12 mil itens · ~7 mil receitas · 8 mercados × 5 qualidades por item |

---

## Sumário

1. [Integração com a API](#1-integração-com-a-api)
2. [Limite de requisições (HTTP 429)](#2-limite-de-requisições-http-429)
3. [Cache no navegador](#3-cache-no-navegador)
4. [Comparação de preços e qualidade dos dados](#4-comparação-de-preços-e-qualidade-dos-dados)
5. [Cálculo de custo: comprar ou produzir?](#5-cálculo-de-custo-comprar-ou-produzir)
6. [Pipeline de dados (ETL)](#6-pipeline-de-dados-etl)
7. [Experiência do usuário e robustez](#7-experiência-do-usuário-e-robustez)
8. [Segurança](#8-segurança)
9. [Deploy](#9-deploy)
10. [Como o projeto foi feito](#10-como-o-projeto-foi-feito)

---

## 1. Integração com a API

**Arquivos:** `js/arbitragem.js` (`montarLotes`, `baixarLotes`), `js/liquidez.js`

A aplicação usa dois endpoints:

| Endpoint | Para quê |
|---|---|
| `GET /api/v2/stats/prices/{itens}.json?locations=…&qualities=…` | menor preço de venda e maior preço de compra, por cidade e qualidade |
| `GET /api/v2/stats/history/{itens}.json?time-scale=24&…` | volume e preço médio por dia (últimos 7 dias) |

### Problema: a URL tem limite de tamanho
A API recebe vários itens na mesma URL, separados por vírgula, mas recusa URLs com mais de **4096 caracteres**. Uma varredura de milhares de itens não cabe numa requisição só.

**Solução: lotes calculados pelo tamanho real da URL.** Em vez de um número fixo de itens por lote, o código mede a URL base e vai somando o tamanho de cada código até chegar perto do limite (3800 caracteres, com folga):

```js
const tamanhoBase = urlDoLote([], "1,2,3,4,5").length;
// ...
if (atual.length > 0 && tamanho + codigo.length + 1 > URL_MAX) {
  lotes.push(atual); atual = []; tamanho = tamanhoBase;
}
```

Assim cada requisição leva o **máximo possível de itens**, com códigos curtos ou longos, e o número total de chamadas cai.

### Problema: muitas requisições em sequência são lentas
**Solução: um pool de "trabalhadores" concorrentes.** Uma fila de lotes é consumida por N funções `async` em paralelo (4 para preços, 2 para histórico). Isso limita a concorrência sem precisar de biblioteca:

```js
async function trabalhador() {
  while (!falhou) {
    const i = proximo++;
    if (i >= lotes.length) return;
    const r = await fetch(urlDoLote(lotes[i], qualidades), { signal });
    // ...
  }
}
await Promise.all(Array.from({ length: CONCORRENCIA }, trabalhador));
```

### Problema: trocar um filtro não deveria gerar nova consulta
**Solução:** a consulta sempre pede **todas as cidades** de uma vez. Quando o usuário muda a rota (comprar em X, vender em Y), o resultado é **recalculado em memória**, sem nova chamada à API.

### Cancelamento
Varreduras longas podem ser interrompidas pelo botão "Interromper", com `AbortController`. Até as esperas entre tentativas (o `sleep`) são canceláveis: o `setTimeout` é limpo quando o sinal de abort dispara.

---

## 2. Limite de requisições (HTTP 429)

**Arquivos:** `js/arbitragem.js`, `js/liquidez.js`, `js/automatico.js`

A API é gratuita e mantida pela comunidade, então limita o número de chamadas. Respeitar esse limite foi um requisito, não um detalhe.

- **Retentativa com backoff exponencial:** ao receber `429`, o cliente espera **15 s, depois 30 s, depois 60 s** antes de tentar de novo, e avisa na tela quanto tempo vai esperar. O número de tentativas é limitado por consulta.
  ```js
  const seg = 15 * Math.pow(2, tentativa++);   // 15 s, 30 s, 60 s...
  ```
- **Circuit breaker simples no histórico:** depois de um `429`, o módulo de liquidez para de pedir por **1 minuto** e agenda uma nova tentativa sozinho.
- **Falha rápida:** se um lote falha de vez, uma flag compartilhada para os outros trabalhadores, em vez de continuar disparando chamadas que também vão falhar.
- **Atualização automática com limites:** só roda com no mínimo 5 minutos de intervalo e **pausa quando a aba do navegador está oculta** (`visibilitychange`). Varreduras com mais de 300 itens só rodam quando o usuário clica.

---

## 3. Cache no navegador

**Arquivo:** `js/cache-precos.js`

Para não buscar de novo o que acabou de chegar, os preços ficam guardados no `localStorage`.

| Decisão | Por quê |
|---|---|
| **Validade configurável (TTL)**, padrão 10 min | preço de mercado envelhece rápido |
| **Chave por item + conjunto de qualidades** | uma consulta "só Normal" não serve para uma consulta "todas as qualidades" |
| **Cache negativo**: itens *sem* preço também são guardados | evita perguntar de novo, a cada consulta, por itens que ninguém negocia |
| **Teto de ~2,5 MB com descarte dos mais antigos** | o limite do `localStorage` costuma ser 5 MB e é compartilhado com o resto do site |
| **Gravação mesmo se a consulta falhar no meio** (`finally`) | o que já chegou não se perde |
| **Tudo em `try/catch`** | em janela anônima ou com armazenamento bloqueado, o site funciona sem cache |

### Problema de desempenho encontrado e corrigido
A primeira versão apagava **uma entrada por vez** e convertia o cache inteiro para texto (`JSON.stringify`) **a cada exclusão**, para medir o tamanho. Numa varredura grande, isso **travava a página por minutos** (custo quadrático).

**Correção:** cada entrada é medida **uma vez só**, por estimativa, sem serializar. As entradas são ordenadas da mais nova para a mais antiga, e o código fica com as mais novas até 80% do limite. A operação passou a ser **uma ordenação e uma passada**.

O usuário também vê de onde veio cada dado: *"Última consulta: 120 da API, 480 do cache (dados de há 4 min)"*.

---

## 4. Comparação de preços e qualidade dos dados

**Arquivos:** `js/arbitragem.js`, `js/liquidez.js`, `js/qualidades.js`

Os preços são **enviados por jogadores**, então vêm com ruído: ordens isoladas com preços absurdos, dados de horas atrás e itens que quase ninguém compra. Um "lucro" calculado em cima disso engana. Por isso a comparação passa por várias camadas de validação:

1. **Filtro de outliers pela mediana:** um preço de venda maior que **3× a mediana** do mesmo item nas outras cidades é descartado como ordem isolada. Só vale quando há pelo menos 3 cidades com preço.
2. **Comparação com a média ponderada de 7 dias:** o histórico gera a média de preço ponderada pelo volume. Preço atual acima de **2×** ou abaixo de **50%** dessa média recebe um selo de alerta.
3. **Liquidez:** o volume médio diário vira um selo 🔴 quando fica abaixo do mínimo configurado. A quantidade sugerida para comprar é limitada a um **percentual do volume diário** do destino, porque não adianta comprar 500 unidades se só 20 vendem por dia.
4. **Idade do dado:** cada preço mostra há quanto tempo foi visto (verde até 15 min, amarelo até 1 h, vermelho depois). Dados mais velhos que o limite configurado são ignorados.

### Regras de negócio no cálculo do lucro
O lucro não é só "preço de venda − preço de compra". Entram:
- imposto de venda diferente para conta **Premium (4%)** e **sem Premium (8%)**
- **taxa de ordem (2,5%)** só quando se anuncia uma ordem de venda (não na venda imediata nem no Black Market)
- **peso do item × carga da montaria** → unidades por viagem e **lucro por viagem**
- **capital disponível** → quantidade, investimento, lucro total e **ROI**
- marcação de **rotas de risco** (cidades que exigem passar por zona de PvP)

Na comparação **entre qualidades**, o "prêmio" de cada qualidade sobre a Normal é comparado com a **mediana** desse prêmio em todos os itens da tabela. O que foge do padrão aparece destacado como possível compra ou venda.

Também há um **mini gráfico de 7 dias** (sparkline) desenhado em **SVG gerado à mão**, sem biblioteca de gráficos, com a linha da média e o ponto do preço atual.

---

## 5. Cálculo de custo: comprar ou produzir?

**Arquivo:** `js/craft.js` (`custoUnidade`, `custoProducao`)

Um item final, como uma espada, usa intermediários (barras), que por sua vez usam matéria-prima (minério). Em cada nível vale a pena **comprar pronto** ou **produzir**, e a resposta muda conforme os preços.

**Solução: avaliação recursiva da árvore de receitas com memoização.**

```js
function custoUnidade(id, o, memo, pilha) {
  if (memo[id]) return memo[id];                           // memoização
  const mercado = precoIngrediente(id, o);                 // custo de comprar
  if (o.cadeia && IDX[id] && !pilha.has(id) && pilha.size < 8) {   // proteção contra ciclos e profundidade
    pilha.add(id);
    producao = custoProducao(IDX[id], o, memo, pilha);    // custo de produzir (recursivo)
    pilha.delete(id);
  }
  // fica com o mais barato e guarda o motivo da decisão
}
```

- **Memoização** (`memo`): cada ingrediente é calculado uma vez por análise, mesmo aparecendo em centenas de receitas.
- **Detecção de ciclos** (`pilha`): impede recursão infinita se o dump tiver uma receita circular, com limite de profundidade.
- **Explicabilidade:** cada decisão guarda *por que* escolheu comprar ou produzir e quanto economizou. O usuário clica na linha e vê a conta completa.

O modelo de custo, documentado no topo do arquivo, inclui retorno de recursos (RRR), taxa da estação, bônus de cidade especialista, foco gasto e o valor dos diários. Os resultados saem por unidade, por hora, como ROI e como **lucro por 10 mil de foco**. Esta última métrica permite comparar receitas diferentes pelo recurso mais escasso do jogador.

---

## 6. Pipeline de dados (ETL)

**Arquivos:** `scripts/gerar-itens.mjs`, `scripts/importar-receitas.mjs`, `scripts/montar-site.sh`

Os dados do jogo vêm de um repositório externo grande: só o arquivo de itens tem **~24 MB**, com nomes em todos os idiomas. Publicar isso no site seria lento.

**Solução: scripts Node.js que extraem e transformam só o necessário.**

- `gerar-itens.mjs` reduz os 24 MB a um JSON enxuto com código e nome em **PT-BR e EN-US**. O formato continua o mesmo do original, então o site lê os dois arquivos com o mesmo código. Cada item fica numa linha, para o **diff no Git ficar legível**.
- `importar-receitas.mjs` cruza quatro arquivos do dump (receitas, bônus por cidade, nomes e mapa do mundo) e gera **~7.000 receitas** categorizadas, além dos pesos dos itens.
  - **Idempotente e determinístico:** a saída é ordenada e o arquivo só é regravado se algo mudou.
  - **Relatório de importação** com totais por categoria, erros e avisos (`data/relatorio-importacao.txt`).
  - Opção `--checar-api` para conferir se os IDs gerados existem na API de preços.
  - Parâmetros de linha de comando (`--tier-min`, `--tier-max`).
- `montar-site.sh` monta a pasta `dist/` **só com o que o site publicado usa**, deixando de fora scripts, ferramentas e a imagem original da logo.

---

## 7. Experiência do usuário e robustez

- **Carregamento com plano B:** a página tenta ler os itens de 3 caminhos em ordem. Se o navegador bloquear (arquivo aberto com dois cliques, `file://`), ela mostra um seletor para o usuário escolher o arquivo manualmente, em vez de quebrar.
- **Estado na URL:** página e filtros ficam no endereço (`?p=produzir&cat=refino&cidade=Martlock`), atualizados com `history.replaceState`. Um link compartilhado abre a mesma análise. **Links de versões antigas** do site são traduzidos para o formato novo.
- **Migração de configurações:** na troca para o painel de configurações centralizado (`albion_config_v2`), as preferências antigas são aproveitadas automaticamente e as chaves antigas não são apagadas, então dá para voltar atrás.
- **Notificações do navegador** (Notification API) quando uma oportunidade passa da margem escolhida. Cada uma avisa só uma vez.
- **Assistente de primeira visita**, que não aparece para quem já usava o site.
- **Acessibilidade:** link "Pular para o conteúdo" e gráficos SVG com `role="img"` e descrição textual.
- **Progresso visível:** barra de progresso por lote e mensagens claras ("Consultando 2.400 itens em 9 requisições...").

---

## 8. Segurança

- Todo texto vindo de arquivos externos ou da API passa por uma função de **escape de HTML** antes de entrar na página, para evitar XSS:
  ```js
  function esc(t) {
    return String(t).replace(/[&<>"']/g, c => ({ "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;" }[c]));
  }
  ```
- Nenhuma chave de API ou segredo no código: a API é pública e o site é 100% estático.
- Arquivos pessoais e dados grandes ficam fora do repositório via `.gitignore`.

---

## 9. Deploy

- Hospedado no **Cloudflare** (Workers com arquivos estáticos), configurado em `wrangler.jsonc`.
- Build: `bash scripts/montar-site.sh` → pasta `dist/` → `npx wrangler deploy`.
- **Cache busting** nos scripts (`js/app.js?v=15`) para que os usuários recebam a versão nova depois de cada deploy. Os arquivos de dados são carregados com `cache: "no-cache"`, então são revalidados quando o importador roda.
- Fluxo com **Git e branches**: funcionalidades maiores, como o novo frontend, foram feitas numa branch separada e mescladas na `main`.

---

## 10. Como o projeto foi feito

O projeto foi desenvolvido com **apoio de ferramentas de IA** (Claude Code) no fluxo de trabalho. Minha parte foi:
- **definir o problema e as regras de negócio**: quais taxas valem em cada situação, o que conta como dado confiável, o que o usuário da guilda precisa ver;
- **dividir o trabalho** em módulos pequenos com responsabilidade clara (um arquivo por funcionalidade);
- **testar contra o jogo** e corrigir quando os números não batiam com a realidade;
- **diagnosticar problemas de desempenho e de limite da API** e decidir a solução (como o cache que travava a página);
- **documentar** as fórmulas e as suposições não confirmadas direto no código, para quem vier depois.

Também usei um grafo de conhecimento do código (`graphify-out/`) para a IA navegar o projeto com menos contexto e mais precisão.
