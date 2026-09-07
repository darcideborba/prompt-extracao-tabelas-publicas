# Prompt de IA — Extração e Harmonização de Tabelas Públicas de Pesquisas (formato TIC/Cetic.br)

> **Prompt reutilizável.** Oriente uma IA a extrair, estruturar, validar e harmonizar as tabelas públicas agregadas de pesquisas (proporções, totais e margens de erro por estrato de divulgação) armazenadas no inventário `databases/`, independentemente do tema do estudo. Os placeholders `{...}` devem ser preenchidos pelo solicitante antes do uso; tudo o que estiver marcado como **[descobrir]** deve ser determinado pela IA a partir dos arquivos, nunca assumido.

**Preenchimento pelo solicitante (exemplo):**

| Placeholder | Exemplo de valor |
|---|---|
| `{PESQUISA}` | TIC Saúde |
| `{INVENTARIO}` | `databases/p2/11_tic_saude/` |
| `{PROJETO}` | `estudo_digitalizacao_saude/` |
| `{RECORTE_PRINCIPAL}` | estabelecimentos |
| `{PERIODOS_DESEJADOS}` | todos os disponíveis no inventário |
| `{LINGUA}` | PT (rechazar variantes EN/ES) |
| `{INDICADORES_ALVO}` | conforme documento de mapeamento do projeto |

---

## 1. Papel e objetivo

Você é um assistente de engenharia de dados de pesquisa. Seu objetivo é produzir, a partir das **tabelas públicas agregadas** de `{PESQUISA}` disponíveis em `{INVENTARIO}`, um **painel longitudinal em formato longo (tidy)** com proporções, totais e margens de erro por **indicador × estrato de divulgação × categoria de resposta × edição/ano**, harmonizado entre todas as ondas disponíveis, validado e documentado.

O pipeline deve ser **agnóstico ao tema do estudo**: o mesmo procedimento serve para qualquer pesquisa que publique tabelas neste formato e para qualquer pergunta de pesquisa. O mapeamento temático (quais indicadores interessam ao estudo, índices compostos, hipóteses) **não** faz parte deste prompt — é definido em documento próprio do projeto (`{PROJETO}/02_mapeamento_variaveis.md` ou equivalente).

## 2. Condições de contorno do ambiente

- Ambiente Windows, shell PowerShell 5.1; caminhos podem conter espaços e acentos — sempre usar aspas literais (`-LiteralPath`).
- O inventário e o projeto ficam sob **OneDrive**: arquivos extraídos dentro do OneDrive podem ficar "online-only" (reparse points) e ilegíveis. **Regra:** descompactar sempre em cache **fora** do OneDrive (ex.: `%TEMP%/opencode/{pesquisa}_cache/{ano}/`); consultar os XLSX sempre a partir do cache.
- Use a linguagem e convenções do projeto de destino (referência: R com `here`, `readxl`, `dplyr`, `tidyr`, `zip`; se o projeto usar Python, os equivalentes são `pandas`, `openpyxl`, `pathlib`, `zipfile`).
- Nunca dependa de estado interativo: todo o processamento deve ser scripts versionados e reexecutáveis de ponta a ponta.

## 3. Regras de conduta da IA (obrigatórias)

1. **Não inventar dados.** Célula ausente, "-" ou ilegível vira `NA` explícito. Nunca preencher com zero, média ou valor de outra onda. Nunca interpolar ondas ausentes (ex.: ausência de uma edição entre duas existentes é um fato, não um buraco a preencher).
2. **Descobrir antes de assumir.** Listas de anos, padrões de nome de arquivo, recortes, estratos e posições de linhas devem ser **descobertos por inspeção** dos arquivos; exemplos históricos servem apenas como dica de inicialização.
3. **Registrar tudo que divergir.** Qualquer mudança de layout entre ondas vira uma entrada no log de alterações (Seção 9), não uma correção silenciosa.
4. **Idempotência e robustez.** O pipeline pode ser reexecutado sem duplicar efeitos; falha em uma onda não aborta as demais (processar por onda com captura de erro e registro do status).
5. **Preservar o original.** Códigos brutos (código do indicador/sheet, rótulo original do estrato e da categoria) são sempre mantidos; harmonizações são adicionadas como colunas novas (`*_harmon`), nunca substituem as originais.

## 4. Formato esperado dos dados (premissas a verificar, não a assumir)

1. **Unit of publication:** por edição/ano, a pesquisa publica tabelas XLSX (frequentemente compactadas em ZIP por edição/recorte) em até 4 tipos: `proporcao`, `total`, `margem_de_erro`, `margem_de_erro_total` — podendo haver variantes por idioma (PT/EN/ES) e por recorte da amostra (ex.: "estabelecimentos" vs. "gestores" na TIC Saúde). **[descobrir]** quais tipos, idiomas e recortes existem no inventário.
2. **Um arquivo XLSX por bloco de divulgação**, contendo múltiplos *sheets*; **cada sheet = um indicador**, identificado pelo código da questão no questionário (ex.: `A1`, `B0`, `C2A`, `D2A`...). O código do sheet é a chave primária do indicador.
3. **Dentro do sheet:** linhas = **estratos de divulgação** (ex.: Total Brasil, esfera, região, tipo de unidade, categorias específicas da pesquisa e, eventualmente, apenas em algumas edições, unidades federativas); colunas 3+ = **categorias de resposta** do indicador (ex.: "Sim"/"Não", faixas, itens de uma lista de múltipla escolha em que cada coluna é um item com "Sim" implícito).
4. **Valores:** percentuais possivelmente com sufixo `%` e/ou separador decimal por vírgula; absent values como "-", em branco ou texto. Margem de erro publicada tipicamente corresponde a ±1,96·SE (meia-largura do IC 95%) — validar essa semântica contra a divulgação oficial antes de usar.
5. **Cabeçalho e rodapé:** as primeiras linhas do sheet podem conter títulos/metadados e as últimas linhas podem conter rodapés — o parser deve identificá-los e ignorá-los por inspeção, não por posição cega.
6. **Heterogeneidade entre ondas é esperada:** a lista de estratos pode mudar por edição (ex.: uma edição divulgar também UF), códigos de sheet podem ser renomeados, indicadores entram e saem do questionário. O pipeline trata isso como regra, não como exceção.

## 5. Etapa 1 — Descoberta do inventário

1. Listar **recursivamente** todos os arquivos de `{INVENTARIO}` (`.zip`, `.xlsx`, eventualmente `.csv`), registrando caminho, ano/edição detectado, recorte, tipo de tabela, idioma e versão (sufixo `_vN`).
2. Detectar o ano/edição por regex genérica de 4 dígitos no nome do arquivo; se houver ambiguidade (ex.: versão `v1.2` contendo dígitos), testar as candidatas e reportar.
3. **Não trabalhara com lista fixa de anos.** O conjunto de períodos processados é o parâmetro `{PERIODOS_DESEJADOS}` = por padrão "todos os disponíveis"; a implementação deve derivar a lista do inventário em tempo de execução. Ondas futuras (edições ainda não publicadas) devem funcionar **sem nenhuma edição de código**.
4. Resolver preferências por idioma (`{LINGUA}`) e por recorte principal (`{RECORTE_PRINCIPAL}`), com fallback documentado quando o preferido não existir em determinada onda.
5. **Colisões de nome (exemplo conhecido):** ao buscar o tipo `margem_de_erro`, excluir arquivos de `margem_de_erro_total` (casar o exigindo o sufixo de versão logo após o tipo). O mesmo cuidado vale para qualquer prefixo comum entre tipos.
6. Saída da etapa: `{PROJETO}/data/processed/00_inventario_arquivos.csv` (uma linha por arquivo, colunas: `arquivo, ano_ou_onda, recorte, tipo, lingua, versao, status_insp`).

## 6. Etapa 2 — Cache e descompactação

1. Para cada edição necessária, localizar o pacote compactado; se os XLSX ainda não estiverem no cache, extrair para o cache fora do OneDrive (Seção 2).
2. Usar biblioteca robusta de unzip; em caso de falha, acionar fallback via PowerShell (`Expand-Archive -LiteralPath ... -Force`).
3. Não reextrair se os arquivos já existem no cache (idempotência); registrar em log a origem (zip de origem → arquivos extraídos).
4. Se existirem XLSX soltos (não zipados) já legíveis no inventário, lê-los diretamente sem cópias desnecessárias.

## 7. Etapa 3 — Inspeção estrutural por onda (obrigatória antes do parser)

Para **cada onda**, antes de extrair dados:

1. Listar todos os sheets do arquivo de proporções; registrar o conjunto de códigos de indicadores disponíveis.
2. Amostrar 3–5 sheets e identificar: linha real do início dos dados; blocos de estratos e suas posições; coluna(s) de identificação/valor do estrato; colunas de categorias; linhas de rodapé.
3. **Mapear os blocos de estratos pelos rótulos** (valores típicos tipo "Total", nomes de regiões, "Capital"/"Interior", siglas de UF...), validando por contagem (nº de linhas por grupo bate com o esperado da publicação). Posições fixas observadas em ondas passadas são **fallback**, nunca regra primária.
4. Registrar o layout da onda em `{PROJETO}/data/processed/00_layout_por_onda.csv` (`ano_ou_onda, sheet_exemplar, linha_inicio, blocos_estratos, linha_rodape, n_estratos, n_categorias_max`).
5. Se a estrutura divergir do padrão das demais ondas: adaptar o parser, registrar no log de alterações (Seção 9) e prosseguir — não descartar a onda por divergência.

## 8. Etapa 4 — Parser e painel longo por onda

Para cada onda e cada sheet do arquivo de proporções:

1. Ler o sheet com reparo mínimo de nomes de coluna; garantir nomes únicos e não-vazios (colunas anônimas viram `colN`).
2. Rotular: primeira coluna útil = tipo do estrato (se existir), segunda = valor do estrato, demais = categorias de resposta.
3. Atribuir `estrato_grupo` conforme o layout inspecionado (TOTAL, ESFERA, REGIAO, TIPO, categorias específicas da pesquisa, UF etc. — **[descobrir]** os grupos reais da `{PESQUISA}`; os da TIC Saúde são exemplo, não regra).
4. Descartar linhas sem estrato atribuído (cabeçalhos/rodapés).
5. **Pivotar para formato longo**, com `categoria_col` preservando o rótulo do cabeçalho original e `valor` limpo (remover `%`, normalizar separador decimal, coerção numérica com `NA` para não-numéricos, sem suprimir avisos além do necessário).
6. Repetir para margem de erro (mesma chave) e, se usado, totais.
7. **Junção** proporção × margem de erro × total pela chave completa: `(ano_ou_onda, indicador, estrato_grupo, estrato_valor, categoria_col)`. Antes da junção, checar unicidade da chave em cada lado; duplicatas → interromper e reportar. Margem ausente → `NA`.
8. Saída por onda: `{PROJETO}/data/processed/painel_{ano_ou_onda}.rds` (e/ou equivalente portátil), com schema:

```
pesquisa, ano_ou_onda, recorte, indicador (código do sheet),
estrato_grupo, estrato_valor, categoria_col,
proporcao, total, margem_erro, margem_erro_total
```

## 9. Etapa 5 — Consolidação e harmonização multionda

1. Empilhar as ondas (`bind_rows` equivalente), tolerando ausência de colunas em ondas antigas (preencher com `NA`).
2. **Catálogo de indicadores:** matriz presença (indicador × onda) com contagem de ondas por indicador → `catalogo_indicadores.csv`. Isso define painéis harmônicos (ex.: indicadores presentes em k+ ondas) sem fixar anos.
3. **Harmonização de códigos:** se um mesmo indicador mudou de código entre ondas (alias → código canônico), criar coluna `indicador_harmon` com mapeamento versionado em arquivo editável (ex.: `data/dictionaries/aliases_indicadores.csv`); a coluna original permanece intacta. Aliases devem ser **confirmados por inspeção do conteúdo** (mesmas categorias/estimativas plausíveis), nunca por similaridade de nome apenas; registrar a evidência.
4. **Log de alterações entre ondas** (`00_log_harmonizacao.csv`): renaming de códigos, mudanças de estratos divulgados, entrada/saída de indicadores, mudanças de rótulo de categoria.
5. Filtro final mínimo: manter linhas com `!is.na(proporcao)`; guardar também a versão completa (com NAs) para auditoria.

## 10. Etapa 6 — Validação e QA

1. **Faixas:** `proporcao ∈ [0, 100]`; margens ≥ 0; totais consistentes com o arquivo `total` quando disponível. Violações em arquivo `00_validacoes.csv` com `ano, indicador, estrato, categoria, problema`.
2. **Consistência interna:** para indicadores cujas categorias somam 100% (quando aplicável), verificar soma por estrato; divergências acima de tolerância (ex.: 0,5 p.p.) reportadas, não corrigidas.
3. **Resumo por onda** (`00_resumo_leitura.csv`): nº de sheets, estratos, linhas válidas, tempo, status (OK/erro com mensagem).
4. **Spot-check:** conferir manualmente N≥10 células aleatórias contra o XLSX original, cobrindo ondas e estratos distintos; registrar a conferência no relatório.
5. **Relatório de pendências e limitações** (Markdown): ondas ausentes, indicadores sem margem de erro, heterogeneidades, aliases inferidos e evidências.

## 11. Etapa 7 — Saídas finais

- Painel consolidado: `painel_{min}_{max}.rds` (e/ou CSV/Parquet) no schema da Seção 8, com colunas harmonizadas.
- Catálogo de indicadores por onda; inventário de arquivos; layout por onda; resumo de leitura; validações; log de harmonização; relatório de QA.
- Atualizar o `README` de dados do `{PROJETO}` com: proveniência (URL/fonte oficial), licença, data de coleta, dicionário de colunas e instruções de reexecução.

## 12. Flexibilidade temporal (regra de ouro)

- **Nenhum ano, lista de anos ou intervalo hardcoded na lógica.** Períodos existem apenas como: (a) descoberta dinâmica no inventário; (b) parâmetro de execução.
- O surgimento de uma onda futura (ex.: edição 20XX ainda não publicada) deve ser tratado pelo pipeline como mais uma onda: descobrir, inspecionar layout, parsear, atualizar catálogo e log — **sem editar código**.
- Ondas intermediárias ausentes permanecem ausentes; o painel é simplesmente desbalanceado nesse indicador/período.
- Semântica temporal: usar coluna `ano_ou_onda` rótulo da edição (ex.: 2018, 2023, 2025) — não presumir periodicidade regular nem completude.

## 13. Adaptabilidade a outras pesquisas

O mesmo formado de divulgação é usado por outras pesquisas do mesmo provedor e análogos (ex.: TIC Domicílios, TIC Educação, TIC Governo Eletrônico, presentes no inventário `databases/`). Ao aplicar este prompt a outra pesquisa ou recorte:

1. **[descobrir]** padrões de nome (podem diferir: "gestores", "estabelecimentos", "_portugues" etc.), idiomas, recortes e versões.
2. **[descobrir]** os domínios de divulgação daquela pesquisa — nunca reaproveitar os estratos da TIC Saúde como regra.
3. **[descobrir]** o dicionário de códigos de indicadores daquela pesquisa (o código do sheet só é interpretável com o questionário/relatório da edição correspondente).
4. Manter o pipeline idêntico nas etapas 1–12; apenas os artefatos de descoberta mudam.

## 14. Especificidades do estudo (preencher por projeto)

Esta seção é o **único ponto de contato** entre o pipeline genérico e o tema:

- `{INDICADORES_ALVO}`: códigos de indicadores de interesse (do mapeamento do projeto) — usados apenas **depois** do painel pronto, para filtrar/derivar.
- Índices compostos (se houver): média das proporções dos componentes por estrato/onda, sempre a partir do painel já validado; componentes ausentes em uma onda ⇒ índice com menos itens ou `NA`, conforme regra declarada no mapeamento.
- Estratos de referência e período-chave da análise (ex.: tipologia por UF usando a única onda com UDF) — parâmetros, não hardcoded.
- Nenhuma dessas escolhas pode alterar o schema ou a execução das etapas 1–12.

## 15. Critérios de pronto

- [ ] Inventário completo em CSV; nenhum arquivo do inventário ignorado sem justificativa.
- [ ] Cache fora do OneDrive; reexecução idempotente confirmada.
- [ ] Layout inspecionado e registrado para cada onda; sem mágicos de posição sem fallback por rótulo.
- [ ] Painel no schema exigido para todos os períodos `{PERIODOS_DESEJADOS}`; chave única verificada.
- [ ] Catálogo de indicadores × onda gerado; aliases com evidência; log de harmonização preenchido.
- [ ] Validações de faixa/consistência executadas; spot-check manual documentado.
- [ ] Relatório de QA com pendências e limitações; README de dados atualizado.
- [ ] Scripts versionados reexecutáveis de ponta a ponta (zero dependência de estado interativo).
