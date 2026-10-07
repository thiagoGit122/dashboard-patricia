# Design Spec — Dashboard de Inteligência Financeira Pessoal
**Data:** 2026-10-05  
**Proprietária:** Patricia de Oliveira Dantas  
**Status:** Aprovado — pronto para implementação

---

## 1. Visão e Objetivo

Transformar extratos bancários, faturas e dados de mercado em inteligência financeira prática. O sistema funciona como um CFO pessoal: lê os dados reais da proprietária, analisa padrões, identifica desperdícios, projeta cenários e emite recomendações objetivas diárias.

**Não é:** um dashboard de gráficos com dados fictícios.  
**É:** uma ferramenta de decisão financeira alimentada por dados reais.

---

## 2. Contexto Financeiro Inicial (dados reais fornecidos)

| Documento | Conteúdo | Período |
|-----------|---------|---------|
| Extrato Bradesco (conta 2795/92866-6) | Todas as movimentações da conta corrente | 01/05/2026 – 05/10/2026 |
| Fatura Carrefour Gold (530033******9888) | Gastos no cartão Carrefour | Out/2026 — R$ 1.716,47 |
| Fatura Bradesco (cartão) | Gastos no cartão Bradesco | Out/2026 — R$ 1.126,33 |
| Planner Financeiro Outubro 2026 | Fixos, variáveis, receitas, categorias | Out/2026 |

**Renda mensal identificada:** R$ 20.156,29  
**Dívidas ativas:** Realize Sociedade de Crédito (parcelas recorrentes)  
**Investimentos ativos:** nenhum — apenas metas de investimento a definir

---

## 3. Arquitetura do Sistema

### 3.1 Stack

| Camada | Tecnologia | Justificativa |
|--------|-----------|---------------|
| Framework | Next.js 14 (App Router) | SSR + API routes no mesmo projeto |
| Linguagem | TypeScript | Segurança de tipos em dados financeiros |
| Estilo | Tailwind CSS + shadcn/ui | Componentes prontos, profissionais |
| Gráficos | Recharts | Open-source, composable, responsivo |
| Ícones | Lucide React | Consistente com shadcn |
| ORM | Prisma | Type-safe, migrations, SQLite |
| Banco | SQLite | Local, privado, zero configuração |
| Parsing PDF | pdf-parse | Extração de texto de extratos |
| Parsing XLSX | xlsx (SheetJS) | Leitura de planilhas |
| Datas | date-fns | Manipulação de datas em pt-BR |
| Validação | Zod | Validação de dados importados |
| Cotações | yahoo-finance2 | Preços de ativos sem API key |
| HTTP client | node-fetch / built-in | Chamadas às APIs públicas |

### 3.2 Estrutura de Diretórios

```
financial-dashboard/
├── app/
│   ├── (dashboard)/
│   │   ├── layout.tsx              Sidebar + Header
│   │   ├── page.tsx                Visão Geral (home)
│   │   ├── hoje/page.tsx           Diagnóstico do Dia
│   │   ├── fluxo/page.tsx          Fluxo de Caixa + Gastos
│   │   ├── orcamento/page.tsx      Gestão de Orçamento
│   │   ├── metas/page.tsx          Objetivos Financeiros
│   │   ├── investimentos/page.tsx  Análise de Oportunidades
│   │   ├── dividas/page.tsx        Controle de Passivos
│   │   ├── alertas/page.tsx        Central de Alertas
│   │   ├── parecer/page.tsx        Parecer Financeiro Diário
│   │   ├── historico/page.tsx      Extrato Consolidado
│   │   └── configuracoes/page.tsx  Importação + Preferências
│   ├── api/
│   │   ├── transactions/route.ts
│   │   ├── import/route.ts
│   │   ├── budget/route.ts
│   │   ├── goals/route.ts
│   │   ├── market/route.ts
│   │   ├── alerts/route.ts
│   │   └── reports/route.ts
│   ├── globals.css
│   └── layout.tsx
├── components/
│   ├── ui/                         shadcn components
│   ├── charts/                     Recharts wrappers tipados
│   ├── cards/                      KPI cards, metric cards
│   ├── layout/                     Sidebar, Header, Navigation
│   └── modules/                    Dashboard, Hoje, Fluxo, etc.
├── lib/
│   ├── importers/
│   │   ├── bradesco-extrato.ts     Parser do extrato conta-corrente
│   │   ├── bradesco-fatura.ts      Parser da fatura cartão Bradesco
│   │   ├── carrefour-fatura.ts     Parser da fatura Carrefour
│   │   ├── planner.ts              Parser do planner XLSX/PDF
│   │   ├── normalizer.ts           Normalização de transações
│   │   └── index.ts                Auto-detector de tipo de arquivo
│   ├── engines/
│   │   ├── categorization.ts       Motor de categorização automática
│   │   ├── decision.ts             Financial Decision Engine
│   │   ├── budget.ts               Motor de orçamento dinâmico
│   │   ├── projections.ts          Projeções de fim de mês
│   │   ├── waste-detector.ts       Detector de desperdícios
│   │   └── history.ts              Janela móvel de 6 meses
│   ├── market/
│   │   ├── bcb.ts                  BCB: SELIC, CDI, IPCA, PTAX
│   │   ├── tesouro.ts              Tesouro Direto: títulos e taxas
│   │   ├── cvm.ts                  CVM: fundos e cotas
│   │   ├── yahoo.ts                Yahoo Finance: ações, FIIs, ETFs
│   │   └── aggregator.ts           Agregador + resolução de divergências
│   ├── db/
│   │   └── client.ts               Prisma client singleton
│   └── utils/
│       ├── currency.ts             Formatação BRL
│       ├── date.ts                 Helpers de data pt-BR
│       └── masks.ts                Mascaramento de dados sensíveis
├── prisma/
│   ├── schema.prisma
│   └── seed.ts                     Dados DEMO para desenvolvimento
├── types/
│   └── financial.ts                Tipos globais do domínio
├── .env.example
└── .env                            (gitignored)
```

---

## 4. Modelo de Dados (Prisma + SQLite)

```prisma
model Transaction {
  id                      String   @id @default(cuid())
  date                    DateTime
  description             String              // original do extrato
  normalizedDescription   String              // após normalização
  amount                  Float
  type                    TransactionType     // CREDIT | DEBIT | TRANSFER
  category                String
  subcategory             String?
  classification          Classification      // ESSENTIAL | IMPORTANT | DISCRETIONARY | WASTE | INVESTMENT | TRANSFER | DEBT
  recurrence              RecurrenceType      // ONE_TIME | RECURRING | INSTALLMENT
  installmentNumber       Int?
  installmentTotal        Int?
  establishment           String?
  paymentMethod           PaymentMethod       // PIX | CREDIT_CARD | DEBIT | TED | BOLETO | OTHER
  bank                    String
  account                 String?
  card                    String?
  sourceFile              String              // nome do arquivo origem
  importedAt              DateTime            @default(now())
  classificationConfidence Float             // 0.0 a 1.0
  isDemo                  Boolean             @default(false)
  notes                   String?
}

model BudgetRule {
  id          String   @id @default(cuid())
  category    String   @unique
  monthlyLimit Float
  updatedAt   DateTime @updatedAt
}

model Goal {
  id                  String     @id @default(cuid())
  name                String
  type                GoalType   // EMERGENCY_FUND | TRAVEL | CAR | HOME | INVESTMENT | DEBT_PAYOFF | CUSTOM
  targetAmount        Float
  currentAmount       Float      @default(0)
  deadline            DateTime?
  monthlyContribution Float?
  createdAt           DateTime   @default(now())
  updatedAt           DateTime   @updatedAt
  isActive            Boolean    @default(true)
}

model Alert {
  id        String      @id @default(cuid())
  type      AlertType
  severity  Severity    // CRITICAL | WARNING | INFO | OPPORTUNITY
  title     String
  message   String
  data      String?     // JSON com contexto
  readAt    DateTime?
  createdAt DateTime    @default(now())
}

model MarketSnapshot {
  id        String   @id @default(cuid())
  source    String   // BCB | TESOURO | CVM | YAHOO | B3
  asset     String   // SELIC | CDI | IPCA | USDBRL | IBOV | etc.
  value     Float
  metadata  String?  // JSON
  fetchedAt DateTime @default(now())
}

model ImportLog {
  id               String   @id @default(cuid())
  filename         String
  detectedType     String
  status           String   // SUCCESS | PARTIAL | ERROR
  transactionCount Int      @default(0)
  errors           String?
  importedAt       DateTime @default(now())
}

model MonthlyReport {
  id          String   @id @default(cuid())
  month       Int
  year        Int
  data        String   // JSON serializado
  generatedAt DateTime @default(now())
  @@unique([month, year])
}

enum TransactionType    { CREDIT DEBIT TRANSFER }
enum Classification     { ESSENTIAL IMPORTANT DISCRETIONARY WASTE INVESTMENT TRANSFER DEBT }
enum RecurrenceType     { ONE_TIME RECURRING INSTALLMENT }
enum PaymentMethod      { PIX CREDIT_CARD DEBIT TED BOLETO OTHER }
enum GoalType           { EMERGENCY_FUND TRAVEL CAR HOME INVESTMENT DEBT_PAYOFF CUSTOM }
enum AlertType          { ABNORMAL_EXPENSE BUDGET_EXCEEDED INCOME_DROP DEBT_INCREASE HIGH_INTEREST OPPORTUNITY GOAL_REACHED RECURRING_DETECTED }
enum Severity           { CRITICAL WARNING INFO OPPORTUNITY }
```

---

## 5. Módulo de Importação

### 5.1 Auto-detector
Ao receber um arquivo, o sistema identifica o tipo antes de parsear:
- Detecta "Bradesco Celular" + "Ag." + "Conta" → Extrato Bradesco
- Detecta "FATURA MENSAL CARTÃO CARREFOUR" → Fatura Carrefour
- Detecta "FATURA" + "BRADESCO" → Fatura Bradesco cartão
- Detecta "PLANNER FINANCEIRO" ou colunas XLSX conhecidas → Planner

### 5.2 Regras de integridade
- Valor original **nunca** é alterado após importação
- Cada transação carrega `sourceFile` e `importedAt`
- `classificationConfidence` indica certeza da categorização (0–1)
- Duplicatas detectadas por hash (data + valor + descrição)
- Dados DEMO marcados com `isDemo: true`, filtrados por padrão na produção

### 5.3 Normalização
Cada descrição passa por normalização:
- Remove ruído ("PIX QR CODE DINAMICO DES:", "PGTO ELETRON COBRANCA")
- Extrai estabelecimento limpo ("LIGHT SERVIÇOS DE ELE" → "Light Energia")
- Mantém `description` original intacta

---

## 6. Motor de Categorização

Categorias fixas do sistema:
`Moradia · Alimentação · Mercado · Delivery · Transporte · Combustível · Saúde · Educação · Trabalho · Lazer · Viagens · Assinaturas · Compras · Tecnologia · Família · Impostos · Dívidas · Investimentos · Outros`

Lógica de classificação por camadas:
1. **Regras exatas** — estabelecimentos conhecidos (Carrefour→Mercado, Light→Moradia)
2. **Regras por padrão** — palavras-chave na descrição normalizada
3. **Regras por valor+frequência** — valor recorrente mensal → Assinaturas
4. **Fallback** — `Outros` com `confidence: 0.4`

Classificações semânticas:
- `ESSENTIAL` → Moradia, Saúde, Alimentação básica, Educação
- `IMPORTANT` → Transporte, Trabalho, Família
- `DISCRETIONARY` → Lazer, Compras, Viagens
- `WASTE` → Delivery excessivo, juros, multas, duplicatas
- `DEBT` → Parcelas, financiamentos

---

## 7. Janela Móvel de 6 Meses

Motor executado diariamente (ou ao abrir o dashboard):
- Filtra transações dos últimos 180 dias
- Calcula média mensal por categoria
- Detecta tendência (crescente / estável / decrescente)
- Identifica gastos anormais (>1.5x a média da categoria)
- Identifica recorrências (mesmo valor ±5%, mesma origem, ≥3 meses)
- Identifica despesas novas (primeiro mês de ocorrência)
- Nunca considera ausência de dado como zero — marca como `DADOS INSUFICIENTES`

---

## 8. Financial Decision Engine

Analisa o estado financeiro atual e gera recomendações classificadas:

| Símbolo | Classe | Critério |
|---------|--------|---------|
| 🔴 | AÇÃO IMEDIATA | Risco real, impacto >10% da renda |
| 🟠 | REDUZIR | Gasto acima da média de 6 meses em >30% |
| 🟡 | MONITORAR | Tendência preocupante sem urgência |
| 🟢 | MANTER | Comportamento saudável |
| 🔵 | OPORTUNIDADE | Capacidade ociosa de investimento |
| ⚪ | SEM DADOS | Histórico insuficiente |

Cada recomendação inclui obrigatoriamente:
- Valor gasto no período
- Média dos 6 meses anteriores
- Variação percentual
- Impacto anual projetado
- Impacto na meta financeira principal

Exemplo de saída:
```
🟠 REDUZIR DELIVERY — R$ 312/mês
Média 6 meses: R$ 198/mês · Variação: +57,5%
Impacto anual: R$ 3.744 · Economia potencial: R$ 114/mês
Impacto na Reserva de Emergência: -2,1 meses de prazo
```

---

## 9. Orçamento Dinâmico

Não usa valor fixo. Calcula o limite por categoria baseado em:
- Média histórica dos últimos 6 meses (peso 50%)
- Renda disponível após fixos (peso 30%)
- Metas ativas (peso 20%)

Exibição por categoria:
```
Delivery
Orçamento: R$ 220  |  Gasto: R$ 312  |  142% utilizado
Ritmo: R$ 10,4/dia  |  Projeção fim do mês: R$ 380
Diferença: R$ 160 acima do orçamento
```

---

## 10. Detector de Desperdícios

Detecta automaticamente:
- Assinaturas recorrentes (valor ≤R$ 200, frequência mensal, estabelecimento de streaming/software)
- Juros e encargos (palavras-chave: "juros", "encargo", "multa", "mora")
- Parcelas de crédito rotativo
- Delivery acima de 15% da categoria Alimentação
- Taxas bancárias recorrentes
- Gastos < R$ 50 com frequência >5x/mês no mesmo estabelecimento

Para cada item detectado:
```
Netflix
Custo mensal: R$ 44,90
Custo anual: R$ 538,80
Classificação: ASSINATURA — MONITORAR
```

---

## 11. Projeção de Fim de Mês

Metodologia baseada em ritmo de consumo:
- `Ritmo atual` = total gasto até hoje ÷ dias do mês passados
- `Sobra esperada` = renda projetada − (fixos confirmados + variáveis projetados)
- `Sobra mínima` = cenário onde variáveis mantêm ritmo acelerado
- `Sobra provável` = regressão à média dos 6 meses
- `Sobra otimista` = se cortar gastos discricionários já identificados

Todas as projeções exibem metodologia resumida.

---

## 12. Fontes de Mercado

| Fonte | Dados | Endpoint | Chave? |
|-------|-------|---------|--------|
| BCB | SELIC, CDI, IPCA, PTAX | api.bcb.gov.br/dados/serie | Não |
| IBGE | IPCA histórico, INPC | servicodados.ibge.gov.br | Não |
| Tesouro Direto | Títulos, taxas, preços | tesourodireto.com.br/api/1/products | Não |
| CVM | Fundos, cotas, patrimônio | dados.cvm.gov.br | Não |
| Yahoo Finance | Ações, FIIs, ETFs, índices | via yahoo-finance2 npm | Não |
| B3 | IBOV, IFIX, SMLL | endpoints públicos | Não |

Regras:
- Cache local de 15 minutos para cotações
- Registro de fonte + data em todo dado de mercado
- Divergência entre fontes → mostrar ambas + indicar a mais confiável
- Nunca inventar cotação, taxa ou rentabilidade

---

## 13. Design Visual

### Identidade
- Tema: dark mode por padrão, toggle dark/light
- Fonte: Inter (padrão Next.js)
- Paleta: tons de cinza escuro para fundo, verde/emerald para positivo, vermelho/rose para negativo, azul/indigo para neutro/informação
- Estética: produto financeiro premium (inspiração: Nubank insights, Notion finance)

### Layout principal
```
┌─────────────────────────────────────────────────────────┐
│ SIDEBAR (240px)     │  CONTEÚDO PRINCIPAL                │
│                     │                                    │
│  ◉ Visão Geral      │  [KPI cards — linha superior]      │
│  ○ Diagnóstico      │  [Gráfico principal — centro]      │
│  ○ Fluxo de Caixa   │  [Cards secundários — grid]        │
│  ○ Orçamento        │  [Alertas + Recomendações]         │
│  ○ Objetivos        │                                    │
│  ○ Oportunidades    │                                    │
│  ○ Passivos         │                                    │
│  ○ Alertas          │                                    │
│  ○ Parecer          │                                    │
│  ○ Extrato          │                                    │
│  ○ Configurações    │                                    │
└─────────────────────────────────────────────────────────┘
```

### Nomenclatura profissional (UI vs arquivo)
| Arquivo/dado | Nome na UI |
|-------------|-----------|
| "Planner_Financeiro_2026" | Planejamento Mensal |
| "extrato" | Extrato Consolidado |
| "fatura" | Passivo do Cartão |
| "gastos" | Fluxo de Saídas |
| "sobra" | Resultado Líquido |
| "dívidas" | Gestão de Passivos |
| "investimentos" | Alocação de Capital |
| "metas" | Objetivos Financeiros |
| "desperdícios" | Oportunidades de Redução |
| "saldo" | Posição de Caixa |
| "receita" | Receita Bruta |

---

## 14. Página "Diagnóstico do Dia"

Exibe diariamente:
- Posição de Caixa atual
- Resultado parcial do mês (receita − despesas até hoje)
- Resultado Líquido projetado
- Taxa de poupança corrente
- Quanto resta de orçamento por categoria crítica
- Alertas ativos
- **Decisão do Dia** — uma recomendação objetiva e acionável

---

## 15. Página "Parecer Financeiro"

Gerado diariamente, estrutura fixa:

1. Situação atual — síntese do momento
2. Evolução positiva — o que melhorou
3. Pontos de atenção — o que piorou
4. Principais concentrações de gasto
5. Reduções recomendadas
6. O que manter
7. Capacidade de poupança
8. Capacidade de investimento
9. Riscos identificados
10. Oportunidades
11. Prioridades da semana
12. **Decisão do Dia**

---

## 16. Segurança e Privacidade

- `.env` para qualquer variável de ambiente
- `.gitignore` cobre `.env`, `*.db`, arquivos importados
- Logs sem valores monetários em texto claro (mascarados: `R$ ***`)
- Arquivos importados processados em memória, não persistidos em disco além do SQLite
- Sem autenticação (uso local, single-user — pode ser adicionado na Fase 4)
- Dados de mercado: apenas busca de preços públicos, nenhum dado pessoal enviado

---

## 17. Fases (Roadmap Completo)

### Fase 1 — Base + Visibilidade (implementação atual)
- [x] Scaffold Next.js 14 + TS + Tailwind + shadcn/ui
- [x] SQLite + Prisma (schema completo)
- [x] Seed com dados DEMO identificados
- [x] Importadores: extrato Bradesco, faturas, planner
- [x] Motor de categorização
- [x] Janela móvel 6 meses
- [x] Dashboard home
- [x] Página Diagnóstico do Dia
- [x] Página Fluxo de Caixa
- [x] Orçamento dinâmico básico
- [x] Indicadores BCB em tempo real
- [x] Sistema de alertas básico
- [x] Testes unitários dos engines

### Fase 2 — Inteligência Financeira
- Financial Decision Engine completo
- Detector de desperdícios
- Projeção de fim de mês (4 cenários)
- Parecer financeiro diário completo
- Relatório mensal automático

### Fase 3 — Análise de Investimentos
- Investment Intelligence Engine
- Integração Tesouro Direto + CVM + Yahoo Finance
- Score Opportunity / Risk / Confidence
- Análise por tipo de ativo (Tesouro, CDB, FII, ETF, Ação)
- Cenários pessimista / base / otimista

### Fase 4 — Planejamento Avançado
- Módulo de Objetivos Financeiros com projeção
- Exportação de relatórios PDF
- Modo multi-mês com comparações históricas longas
- Autenticação opcional

---

## 18. Limitações Conhecidas (Fase 1)

- Cotações B3 via Yahoo Finance têm delay de ~15 minutos (não são tick-a-tick)
- PDF de extrato Bradesco com encoding não-UTF8 requer normalização de caracteres
- Fatura Bradesco (arquivo com nome em PT com acentos) pode ter problema de leitura no Windows — contingência: mover arquivo ou renomear antes de importar
- Dados de investimentos: apenas oportunidades de mercado, não carteira real (usuária não possui investimentos ainda)
- Histórico: 6 meses disponíveis desde o primeiro import — até então, cálculos de média usam o que há disponível

---

## 19. Testes

- Unitários: cada importer, categorization engine, budget engine, projections
- Integração: fluxo completo de import → normalização → categorização → storage
- E2E: não incluído na Fase 1
- Framework: Jest + ts-jest
- Cobertura mínima: 80% nas funções de engine

---

*Spec revisada e aprovada em 2026-10-05. Próximo passo: writing-plans → implementação.*
