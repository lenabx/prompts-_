# Prompts de Claude Code — `formatar` do D-1101 (Balancete Mensal)

Sequência para ajustar `cenarios/dere_balancete_mensal/parametrizacoes/formatar.json`,
seguindo a convenção de `dere_cadastro_empresa/parametrizacoes/04-d1001.json`.

Rode **um prompt por vez** e confira o resultado antes de seguir. A ordem importa:
entender → mapear → gerar → validar.

---

## Prompt 0 — Reconhecimento (não gera nada ainda)

> Estou trabalhando no repositório `itau-ue4-infra-parametrizacao-cenario-dere`.
> Preciso que você entenda a convenção antes de qualquer coisa. Leia estes três arquivos:
>
> 1. `cenarios/dere_cadastro_empresa-dere_cadastro_empresa/parametrizacoes/04-d1001.json`
> 2. `cenarios/dere_cadastro_empresa-dere_cadastro_empresa/dominios/04-d1001.xsd`
> 3. `cenarios/dere_balancete_mensal/dominios/06-d1101.xsd`
>
> Depois me explique, em texto (sem escrever código ainda):
> - Como o `04-d1001.json` mapeia a árvore de elementos do `04-d1001.xsd`: a relação entre
>   `xml_blocks` / `children` e a hierarquia do XSD, e o papel de `mappings`, `source_type`,
>   `source_value`, `repeating`, `colunas_obrigatorias` e `coluna_agrupamento`.
> - Quais elementos do `06-d1101.xsd` têm `maxOccurs="unbounded"` (ou > 1) e `minOccurs`,
>   ou seja, onde o D-1101 é repetitivo. Liste a árvore de elementos do d1101 de forma indentada.
> - O `targetNamespace` declarado no `06-d1101.xsd`.
>
> Não altere nenhum arquivo. Só quero seu entendimento para validar antes de gerarmos o JSON.

**Por que:** força o Claude Code a ancorar no XSD real e a te devolver a árvore, que você
confere contra o MOD. Aqui você pega erros de leitura de schema antes de propagar.

---

## Prompt 1 — Esqueleto (estrutura sem mappings de negócio)

> Com base na árvore de elementos do `06-d1101.xsd` que você levantou, gere o esqueleto do
> `formatar.json` do D-1101, seguindo EXATAMENTE a mesma estrutura e estilo do `04-d1001.json`
> (mesmos nomes de chave, mesma indentação, mesmo formato de `output` / `xml_blocks` / `children`).
>
> Regras:
> - Cabeçalho: `id`, `name`, `version`, `status`, `description`, datas — adaptados para o D-1101.
> - `output.namespaces`: use o `targetNamespace` real do `06-d1101.xsd`.
> - Um bloco por elemento do XSD, respeitando a hierarquia via `children`.
> - `repeating: true` em todo elemento com `maxOccurs > 1`; caso contrário `false`.
> - Em cada bloco, deixe `mappings` como lista VAZIA por enquanto (`"mappings": []`).
> - Ainda NÃO preencha `colunas_obrigatorias` nem `coluna_agrupamento` — deixe como
>   `[]` e `null` provisoriamente.
>
> Escreva o resultado em `cenarios/dere_balancete_mensal/parametrizacoes/formatar.json`.
> Depois me mostre a árvore de `block_id` gerada, indentada, para eu conferir a hierarquia.

**Por que:** separa a parte estrutural (que sai do XSD, objetiva) da parte de negócio (o
de-para de colunas, que exige o MOD). Assim você valida a forma antes do conteúdo.

---

## Prompt 2 — Mappings (o de-para com o dado interno)

> Agora vamos preencher os `mappings`. Para cada elemento-folha do D-1101 (campos que recebem
> valor, não os elementos de agrupamento), adicione um objeto em `mappings` no formato:
>
>     { "xpath": "<nome do elemento no XSD>",
>       "source_type": "column",        // ou "column_list" se o elemento for repetitivo a partir de uma lista
>       "source_value": "<COLUNA_INTERNA>",
>       "required": <true|false conforme minOccurs do XSD> }
>
> Onde eu ainda não sei o nome exato da coluna interna, use um placeholder no formato
> `TODO_<nome_do_elemento>` como `source_value`, para eu preencher depois. NÃO invente nomes
> de coluna — só use placeholder quando não houver correspondência óbvia.
>
> Regras herdadas do d1001:
> - `required` vem do `minOccurs` do XSD (1 → true, 0 → false).
> - Elementos repetitivos que vêm de lista usam `source_type: "column_list"`.
>
> Ao final, liste TODOS os `source_value` que ficaram como `TODO_...` para eu resolver com o MOD.

**Por que:** o Claude Code não conhece o modelo de dados interno de vocês — então em vez de
alucinar nomes de coluna, ele te devolve uma lista de lacunas explícita. Você resolve essas
lacunas com o MOD do D-1101 e o schema da tabela interna.

---

## Prompt 3 — Fechamento do contrato (colunas_obrigatorias + agrupamento)

> Agora finalize o arquivo:
> - Preencha `colunas_obrigatorias` com a lista de TODAS as colunas internas referenciadas em
>   qualquer `source_value` do arquivo (sem duplicatas, incluindo as que eu já resolvi e
>   deixando de fora os placeholders `TODO_` ainda não resolvidos — me avise se houver algum).
> - Defina `coluna_agrupamento` seguindo a mesma lógica do `04-d1001.json`: a coluna que
>   identifica o agrupamento do evento. No d1001 era `numero_inscricao_empresa`; me diga qual
>   você inferiu para o balancete a partir do XSD e por quê, e deixe eu confirmar.
>
> Não altere mais nada da estrutura.

**Por que:** `colunas_obrigatorias` é o contrato de entrada do passo — se faltar coluna, o
formatador quebra em runtime. Melhor derivá-la automaticamente do que manter à mão.

---

## Prompt 4 — Validação contra o XSD

> Escreva um teste rápido (no padrão da pasta `tests/` do repo, se houver; senão um script
> Python standalone) que:
> 1. Carregue o `formatar.json` do D-1101.
> 2. Gere um XML de exemplo a partir dele, usando dados fictícios para cada coluna.
> 3. Valide esse XML contra `cenarios/dere_balancete_mensal/dominios/06-d1101.xsd`
>    (use `lxml.etree.XMLSchema`).
> 4. Falhe com mensagem clara se algum elemento obrigatório do XSD não for produzido, ou se a
>    ordem/estrutura divergir do schema.
>
> Rode o teste e me mostre o resultado. Se falhar, aponte qual regra do XSD não foi satisfeita
> — não "conserte" o JSON sem antes me explicar a divergência.

**Por que:** fecha o loop. O XSD é a fonte da verdade; se o XML gerado valida contra ele,
sua parametrização está estruturalmente correta. O "explique antes de consertar" evita que o
Claude Code mascare um erro de mapeamento com um ajuste cosmético.

---

## Depois do D-1101

Para o **D-1106**, o fluxo é o mesmo, com duas diferenças:
1. É cenário novo → antes do Prompt 0, peça para criar a estrutura de pastas
   (`cenarios/dere_aplicacoes_financeiras/dominios/` e `.../parametrizacoes/`) e colocar o XSD.
2. Confirme com o time a **obrigatoriedade do D-1106 para serviços financeiros** e se a
   **reconciliação de saldos com o D-1101** entra no `formatar` ou fica no `criticar`.

## Lembretes de convenção
- Nunca cole conteúdo de arquivo no prompt: mande o Claude Code LER o arquivo do repo.
- Rode um prompt por vez; confira a saída antes do próximo.
- O `04-d1001.json` é o molde de ouro — quando em dúvida de formato, o Claude Code deve
  espelhar ele.
- A nomenclatura de pastas do repo está repetitiva (já foi criticada); siga o padrão atual
  para não misturar mudança de convenção com entrega funcional.
