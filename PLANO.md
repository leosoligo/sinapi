# Plano — Sistema aberto de consulta de composições de preços (SINAPI e similares)

## 1. Objetivo

Aplicação web, aberta e escalável, para consultar **insumos e composições de custo** (SINAPI hoje; SETOP, SBC e outras depois), filtrando por **UF, mês/ano de referência e regime (desonerado / não desonerado / sem desoneração)**, com **navegação por hiperlinks entre composições e composições auxiliares** (estilo Orçafascio). Escopo inicial: **RR e MG**.

Requisitos-chave:
1. Ingestão automática dos arquivos mensais da CAIXA **ou** upload manual de ZIPs.
2. Suportar o formato antigo (≤ 2024) e o novo (≥ 2025) das planilhas, normalizando para um único modelo.
3. Arquitetura multi-fonte: adicionar SETOP/SBC = escrever um novo "adaptador", sem mexer no resto.
4. Uso diário prático: busca rápida, árvore de composição, custo calculado, exportação.

## 2. Stack proposta (TypeScript ponta a ponta)

| Camada | Escolha | Motivo |
|---|---|---|
| Monorepo | pnpm workspaces + Turborepo | compartilha tipos entre web/api/ingestão |
| Frontend | Next.js (App Router) + React + Tailwind + shadcn/ui + TanStack Table/Query | UI rápida, tabelas grandes, SSR para links compartilháveis |
| API | Fastify (ou rotas do Next no MVP) + Zod | tipagem e validação |
| Banco | **PostgreSQL** + Drizzle ORM | CTE recursivas (árvore de composições), `pg_trgm` + full-text em português |
| Fila/jobs | pg-boss (usa o próprio Postgres) | evita Redis no começo; agenda coleta mensal |
| Parsing de planilhas | `exceljs` / SheetJS (xlsx), `csv-parse`, `unzipper` | |
| Coleta web | `undici` + `cheerio` (fallback Playwright se o site exigir JS) | |
| Armazenamento bruto | Disco local / S3-compatível (MinIO) | guarda ZIPs originais p/ reprocessar |
| Deploy | Docker Compose (dev/VPS) → depois Fly.io/Railway/K8s | |
| Qualidade | Vitest, Playwright (e2e), ESLint, GitHub Actions | |

## 3. Estrutura do repositório

```
apps/
  web/            # Next.js
  api/            # Fastify (REST) — pode nascer dentro do web
  worker/         # jobs: coleta + ingestão
packages/
  core/           # modelo de domínio, tipos, cálculo de custo
  db/             # schema Drizzle + migrations
  ingest/         # pipeline genérico + adaptadores
    sources/
      sinapi/
        fetcher.ts          # descobre/baixa ZIPs no site da CAIXA
        parsers/legacy.ts   # layout ≤ 2024
        parsers/v2025.ts    # layout ≥ 2025
        detect.ts           # detecta layout pela estrutura da planilha
      setop/ sbc/           # futuros
fixtures/         # planilhas amostra (pequenas) p/ testes
docs/
```

## 4. Modelo de dados (normalizado e multi-fonte)

- `source` (SINAPI, SETOP, SBC…)
- `reference` — (source, uf, ano, mês, regime) + `import_id` → permite histórico e comparação mensal
- `item` — identidade estável: (source, `codigo`, `tipo` = INSUMO | COMPOSICAO), descrição, unidade, classe/grupo, origem de preço
- `item_price` — (item, reference, preço, situação). Insumo: preço mediano; composição: custo total/unitário, e parcelas MO/material/equip.
- `composition_item` — (composição, reference, item filho, coeficiente, tipo do filho, preço unitário e total no momento) → **é a aresta do grafo**; filho que é composição = link para a página dela
- `import` — arquivo, hash SHA-256, URL de origem, layout detectado, status, log de erros, contagem de linhas (idempotência e auditoria)
- `import_issue` — linhas rejeitadas/avisos (ex.: filho sem preço)
- Índices: `pg_trgm` na descrição, `(source,codigo)`, `(reference_id,item_id)`.

Árvore completa de uma composição = `WITH RECURSIVE` sobre `composition_item`, com detecção de ciclos e custo acumulado por nível.

## 5. Ingestão

### 5.1 Coleta automática
- Worker agendado (cron mensal + retentativa diária): lê a página de downloads da CAIXA, extrai links de "SINAPI – Referências de preços e custos" por mês/ano, compara com `import` já feitos (hash/URL) e baixa só o que é novo.
- Tolerante a mudança do site: seletor configurável, logs claros, alerta quando nada é encontrado.
- Respeito ao servidor: User-Agent identificado, 1 requisição por vez, cache.
- **Risco:** o site da CAIXA pode bloquear robôs/exigir JS; mitigação → fallback Playwright e, sempre, **upload manual**.

### 5.2 Upload manual
- Tela "Importar" (drag-and-drop de ZIP). Aceita ZIP do pacote mensal completo ou planilhas isoladas. Mostra pré-visualização (layout detectado, UF, mês, nº de linhas) antes de confirmar.

### 5.3 Pipeline (idempotente)
`ZIP → extrair → classificar arquivos (insumos / composições analíticas / sintéticas / custos) → detectar layout → parser → normalizar → filtrar UFs habilitadas → upsert em transação → validar → publicar referência`

Validações: totais por composição recalculados vs. valor da planilha (tolerância de centavos), filhos órfãos, unidades inconsistentes, coeficientes ≤ 0.

### 5.4 Os dois layouts
Não tenho amostras reais ainda (o ambiente aqui não alcançou o site da CAIXA). **Primeiro passo prático: você fornece 1 ZIP de 2024 e 1 de 2025 (RR ou MG)** — eu monto as fixtures e escrevo os parsers/`detect.ts` com base na estrutura real, em vez de adivinhar colunas.

## 6. Funcionalidades (MVP → evolução)

**MVP**
1. Seletor global: fonte, UF, mês/ano, regime (persistido na URL).
2. Busca unificada por código ou texto (tolerante a acento/erro de digitação), filtros por tipo/classe/unidade.
3. **Página da composição**: cabeçalho (código, descrição, unidade, custo), tabela de itens (insumos e composições auxiliares com **hiperlink**), coeficiente, preço unitário, total, % de participação, grupos MO/Material/Equipamento; botão expandir árvore inteira; breadcrumb de navegação.
4. **Página do insumo**: preço, histórico e "usado em" (composições que o utilizam).
5. Comparar o mesmo item entre UFs (RR × MG) e ao longo dos meses (gráfico de variação).
6. Exportar para Excel/CSV/PDF.
7. Tela de importações (status, erros, reprocessar).

**Fase 2 — dia a dia de orçamentista**
- Orçamentos próprios: projetos com itens, quantidades, BDI, encargos, curva ABC, planilha orçamentária exportável.
- Composições próprias (copiar/editar uma composição oficial) e base própria de insumos.
- Favoritos, histórico de consultas, links compartilháveis.
- Reajuste/atualização de orçamento para outro mês de referência.

**Fase 3 — escala**
- Novas fontes (SETOP-MG, SBC, etc.) via adaptadores; demais UFs só mudando configuração.
- Contas/usuários (Auth.js), times, API pública documentada (OpenAPI).
- Cache (Redis), CDN, read-replicas se necessário.

## 7. Roteiro por marcos

| Marco | Entrega | Critério de pronto |
|---|---|---|
| M0 | Monorepo, Docker Compose (Postgres), CI, lint/test | `pnpm dev` sobe tudo |
| M1 | Schema + migrations + parser SINAPI (2025+) com fixtures | Importa RR e MG; testes de parser verdes |
| M2 | Parser legado (≤ 2024) + detecção automática de layout | Mesma consulta funciona p/ 2023 e 2025 |
| M3 | API de busca + página de composição com hiperlinks e árvore | Navegar composição → auxiliar → insumo |
| M4 | Upload manual + fetcher automático + agendamento | Mês novo entra sem intervenção; ZIP manual funciona |
| M5 | Comparativos, exportações, tela de importações | Uso diário viável |
| M6 | Orçamentos e composições próprias (Fase 2) | |

## 8. Decisões que preciso de você

1. **Amostras reais**: pode subir 1 ZIP SINAPI de 2024 e 1 de 2025 (podem ser só RR)? Isso destrava M1–M2.
2. **Regimes**: precisa de desonerado e não desonerado desde já, ou só um?
3. **Uso**: só você (local/VPS simples) ou multiusuário/público desde o início? Define autenticação e hospedagem.
4. **Histórico**: quantos anos para trás carregar (ex.: 2020 em diante)?
5. **Stack**: confirma Next.js + Fastify + PostgreSQL? (Alternativa mais simples para uso individual: SQLite + um único app Next.js.)

## 9. Riscos

- Mudanças de layout da CAIXA (já ocorreu; haverá novas) → parsers versionados + detecção + testes com fixtures de cada formato.
- Bloqueio/instabilidade do site → upload manual sempre disponível; guardamos os ZIPs originais.
- Volume: SINAPI tem ~6 mil insumos e ~10 mil composições por UF/mês; com índices adequados, 2 UFs × 5 anos × 12 meses × 2 regimes cabe com folga em um Postgres pequeno.
- Aspectos de licença/uso dos dados: os dados são públicos, mas cite sempre a fonte e a referência (mês/UF) na interface.
