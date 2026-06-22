# Plano do Projeto — Bancada de Teste de Fim de Linha (Fogões)

**Plataforma:** Siemens S7-1212C (CLP) + KTP900 Basic (IHM) + TIA Portal
**Tipo:** Bancada de teste funcional (EOL) totalmente simulada dentro do CLP
**Idioma do projeto:** pt-BR (todas as tags de CLP e IHM em português)

---

## 1. Visão geral e objetivo

A bancada simula o teste de fim de linha de um fogão. Uma **sequência mestre de passos** roda um a um; cada passo faz uma medição **simulada dentro do CLP**, compara com uma faixa definida pela **receita ativa** e marca o passo como **Aprovado** ou **Reprovado**.

Nada físico é conectado: todos os valores (temperatura, vazão, pressão, grandezas elétricas, faísca) são gerados internamente por modelos no próprio S7-1212C. A IHM apenas lê e escreve tags em blocos de dados.

Objetivos de aprendizado embutidos:
- Estrutura de dados com UDTs e arrays de UDT.
- Receitas armazenadas no CLP, selecionadas por índice.
- Construção dinâmica de um plano de execução (loops por queimador e por forno).
- Máquina de estados (sequenciador) em SCL.
- Modelos de simulação de processo (térmico, decaimento de pressão, etc.) em interrupção cíclica com `dt` fixo.
- Lógica de aprovação/reprovação por faixa e relatório final.
- Telas de IHM em painel Basic (animação por barra/medidor, cor por aparência dinâmica, visibilidade dinâmica — sem script).

---

## 2. Conceito de funcionamento

1. O operador seleciona o **tipo de produto** (índice da receita) e, opcionalmente, digita o **número de série**.
2. Ao iniciar, o CLP lê a receita ativa e **constrói o plano de execução**: expande a sequência mestre nos N queimadores e M fornos daquele modelo, na ordem de segurança, gerando uma lista plana de passos.
3. O **sequenciador** percorre essa lista. Para cada passo: configura o modelo de simulação, aguarda a duração/condição do passo, lê o(s) valor(es) medido(s), compara com os limites e grava o resultado.
4. **Roda todos os passos até o fim** (não aborta na primeira falha).
5. A **tela final** mostra o detalhamento completo (passo a passo) e destaca os **reprovados para retrabalho**.

A "lista mestre" é única: a **ordem é fixa** (definida pela sequência de segurança no construtor de plano). A **receita** só fornece parâmetros: quantidade de queimadores/fornos, tipo de cada forno, limites, durações e flags de habilitação de cada tipo de teste. Isso permite, no futuro, produtos que pulam etapas (ex.: cooktop 100% elétrico pula estanqueidade, faísca e vazão) apenas mudando as flags.

---

## 3. Hardware e restrições

| Item | Detalhe | Impacto no projeto |
|---|---|---|
| CPU S7-1212C | Pouca E/S física, mas CPU suficiente | A simulação roda em OB de interrupção cíclica sem problema. E/S física não limita, pois tudo é interno. |
| IHM KTP900 Basic | 9", 800×480, **sem scripts** | "Animação" = valor de barra/medidor, cor por aparência dinâmica, mostrar/ocultar objetos. Suficiente. |
| Receitas | Ficam **no CLP** (em DB) | **Não** usaremos a função de Receita do painel. Seleção por índice. |
| Registro de resultados | Logging limitado no Basic | Resultados ficam em DB durante a execução; persistência é limitada (ok para treino). |

`dt` da simulação: **OB30 (interrupção cíclica) a cada 100 ms**.

---

## 4. Arquitetura de software

### 4.1 Blocos de organização (OB)

| OB | Função |
|---|---|
| **OB1 (Main)** | Chama o sequenciador a cada ciclo; trata comandos da IHM (seleção, iniciar, abortar, resetar, OK/NOK visual). |
| **OB30 (Interrupção cíclica, 100 ms)** | Chama a simulação (`FB_Simulacao`): integra os modelos físicos com `dt` fixo. |
| **OB100 (Startup)** | Inicializa sementes do ruído, zera contadores se necessário. |

### 4.2 Blocos de função e tipos

| Bloco | Tipo | Papel |
|---|---|---|
| `FB_Sequenciador` | FB | Máquina de estados: Ocioso → Construir → Executar → Finalizado. |
| `FC_ConstruirPlano` | FC | Expande a receita ativa em `PlanoExecucao[]` na ordem de segurança. |
| `FB_ExecutarPasso` | FB | Lógica do passo atual (CASE por tipo): configura simulação, mede tempo, avalia, grava resultado. |
| `FB_Simulacao` | FB | Modelos físicos (térmico, pressão, vazão, elétricos, faísca) + gerador de ruído. |
| `FC_GerarRuido` | FC | LCG simples para ruído pseudoaleatório. |

### 4.3 Fluxo geral

```
IHM (seleção + iniciar)
        │
        ▼
OB1 → FB_Sequenciador ──(estado CONSTRUIR)──► FC_ConstruirPlano ──► PlanoExecucao[]
        │
        └──(estado EXECUTAR)──► FB_ExecutarPasso ──► configura DB_Simulacao
                                       ▲                     │
                                       │                     ▼
OB30 (100 ms) ───────────────► FB_Simulacao (integra modelos) ──► valores medidos
                                       │
                                       ▼
                              compara com limites → ResultadosPasso[]
                                       │
                                       ▼
                          (estado FINALIZADO) → AprovadoGeral, contadores, tela final
```

---

## 5. Estrutura de dados (UDTs e DBs)

### 5.1 UDTs (tipos de dados do usuário)

**`udt_MedicaoFaixa`** — uma grandeza medida com seus limites:
```
Nome         : String[20]   // ex.: "Aterramento", "Vazao", "Temperatura"
Unidade      : String[8]    // ex.: "A", "Ohm", "W", "MOhm", "mbar", "°C", "L/h"
LimiteMin    : Real         // limite inferior (usar valor muito baixo p/ desativar)
LimiteMax    : Real         // limite superior (usar valor muito alto p/ desativar)
ValorNominal : Real         // centro da simulação
Usado        : Bool         // se esta sub-medição é avaliada
```
> Limite de um lado só: para "queda de pressão ≤ 1,0", deixe `LimiteMin = -1.0e9`. Para "isolamento ≥ 2,0", deixe `LimiteMax = 1.0e9`.

**`udt_DefinicaoPasso`** — um passo já expandido no plano:
```
TipoTeste   : Int                              // ver enum (seção 6)
Alvo        : Int                              // 0 = global; senão índice do queimador/forno
Descricao   : String[40]                       // ex.: "Vazao maxima - Queimador 3"
Duracao     : Time
NumMedicoes : Int                              // 1..4
Medicoes    : Array[1..4] of udt_MedicaoFaixa
```

**`udt_ResultadoPasso`** — resultado de um passo executado:
```
Concluido      : Bool
Aprovado       : Bool
TempoGasto     : Time
ValoresMedidos : Array[1..4] of Real
```

**`udt_Receita`** — parâmetros de um modelo de fogão:
```
Nome            : String[30]            // ex.: "Fogao A 5Q 2F"
NumQueimadores  : Int
NumFornos       : Int
FornoEletrico   : Array[1..2] of Bool   // true = elétrico, false = gás

// Flags de habilitação (permite produtos que pulam etapas)
HabEstanqueidade : Bool
HabSegEletrica   : Bool
HabEletSistema   : Bool
HabFaisca        : Bool
HabVazao         : Bool
HabVisual        : Bool

// Durações (variam por receita)
DurEstanqueidade : Time
DurSegEletrica   : Time
DurEletSistema   : Time
DurFaisca        : Time
DurVazaoMax      : Time
DurVazaoMin      : Time
DurTemperatura   : Time   // tempo LIMITE para atingir a faixa
DurEletForno     : Time

// Limites por teste/componente
Estanqueidade : udt_MedicaoFaixa                       // ΔP máx
SegEletrica   : Array[1..2] of udt_MedicaoFaixa         // isolamento, fuga
EletSistema   : Array[1..4] of udt_MedicaoFaixa         // aterramento, corrente, potência, resistência
VazaoMax      : Array[1..6] of udt_MedicaoFaixa         // por queimador
VazaoMin      : Array[1..6] of udt_MedicaoFaixa         // por queimador
Temperatura   : Array[1..2] of udt_MedicaoFaixa         // faixa-alvo por forno
EletForno     : Array[1..2, 1..3] of udt_MedicaoFaixa   // por forno elétrico: resistência, corrente, potência
```
> As durações são por receita e por tipo de teste (aplicadas a todos os componentes daquele tipo). Se quiser variar por componente no futuro, basta transformar em array.

### 5.2 `DB_Receitas` (catálogo no CLP)
```
Receitas : Array[1..3] of udt_Receita   // índice = tipo de produto (1=Fogão A, 2=Fogão B, 3=Fogão C)
```
Inicializado com os 3 modelos (valores na seção 9).

### 5.3 `DB_Execucao` (runtime / status — ponte com a IHM)
```
// Seleção e identificação
ProdutoSelecionado : Int            // índice da receita
NumeroSerie        : String[20]

// Estado do sequenciador
Estado             : Int            // 0=Ocioso 1=Construir 2=Executar 3=Finalizado
PassoAtual         : Int            // 1..TotalPassos
TotalPassos        : Int

// Espelho do passo atual (para a tela de execução)
TipoTesteAtual     : Int
AlvoTextoAtual     : String[20]     // ex.: "Queimador 3", "Forno 2", "Sistema"
NomePassoAtual     : String[40]
ValorMedidoAtual   : Real
SubMedidaAtual     : Array[1..4] of Real
LimiteMinAtual     : Real
LimiteMaxAtual     : Real
DuracaoPassoAtual  : Time
TempoDecorridoPasso: Time
ResultadoPassoAtual: Int            // 0=rodando 1=passou 2=falhou

// Resultado geral
AprovadoGeral      : Bool
ContadorAprovados  : Int
ContadorReprovados : Int
ListaReprovados    : String[254]    // texto montado p/ a tela final

// Comandos da IHM (Bool)
IniciarTeste       : Bool
AbortarTeste       : Bool
Resetar            : Bool
ForcarFalhaPassoAtual : Bool        // treino/demo
VisualOK           : Bool
VisualNOK          : Bool

// Plano e resultados
PlanoExecucao   : Array[1..30] of udt_DefinicaoPasso
ResultadosPasso : Array[1..30] of udt_ResultadoPasso
```
> Tamanho 30: máximo teórico = 4 globais + 6×2 vazão + 2×2 forno + 1 visual = 21. Folga suficiente.

### 5.4 `DB_Simulacao` (estados dos modelos)
```
ModeloAtivo     : Int           // geralmente = TipoTesteAtual
AlvoAtivo       : Int
Habilitar       : Bool          // medição em andamento

// Térmico
Temperatura     : Real
TempAmbiente    : Real          // ex.: 25.0
TempRegime      : Real          // alvo de regime (acima do limite p/ cruzar a faixa)
TauTermico      : Real          // constante de tempo (s)

// Pressão
Pressao         : Real
PressaoInicial  : Real          // P0, ex.: 30.0 mbar
DeltaPressao    : Real          // P0 - P_atual
TaxaVazamento   : Real          // mbar/s (maior = pior)

// Vazão / elétricos
Vazao           : Real
ValorEletrico   : Array[1..4] of Real

// Faísca
NumCentelhadores : Int          // N queimadores + nº de fornos a gás
FaiscaDetectada  : Array[1..8] of Bool

// Ruído
Semente         : DInt
AmplitudeRuido  : Real          // ex.: 0.01 (±1% do nominal)
```

---

## 6. Catálogo de tipos de teste e ordem de execução

### 6.1 Enum dos tipos (constantes do CLP)

| Constante | Valor | Teste | Escopo |
|---|---|---|---|
| `TIPO_ESTANQUEIDADE` | 1 | (D) Estanqueidade do sistema de gás | Global |
| `TIPO_SEG_ELETRICA` | 2 | (E) Segurança elétrica (isolamento/fuga) | Global |
| `TIPO_ELET_SISTEMA` | 3 | (G) Teste elétrico do sistema (aterramento, corrente, potência, resistência) | Global |
| `TIPO_FAISCA` | 4 | (F) Faísca/ignição — **verifica todos os centelhadores de uma vez** | Global |
| `TIPO_VAZAO_MAX` | 5 | (B) Vazão máxima do queimador | Por queimador |
| `TIPO_VAZAO_MIN` | 6 | (C) Vazão mínima do queimador (chama mantida) | Por queimador |
| `TIPO_TEMPERATURA` | 7 | (A) Temperatura do forno (termopar) | Por forno |
| `TIPO_ELET_FORNO` | 8 | (I) Teste elétrico do elemento (forno elétrico) | Por forno elétrico |
| `TIPO_VISUAL` | 9 | (H) Inspeção visual (OK/NOK do operador) | Global, **no fim** |

Estados do sequenciador:
```
EST_OCIOSO     := 0
EST_CONSTRUIR  := 1
EST_EXECUTAR   := 2
EST_FINALIZADO := 3
```

### 6.2 Ordem de execução (definida por segurança)

A ordem é **fixa** no construtor de plano:

1. **Estanqueidade (D)** — antes de acender qualquer coisa.
2. **Segurança elétrica (E)** — antes de energizar para os testes funcionais.
3. **Elétrico do sistema (G)** — aterramento + corrente + potência + resistência.
4. **Faísca (F)** — passo global único que verifica todos os centelhadores.
5. **Loop por queimador** `i = 1..N`: Vazão máx (B) → Vazão mín (C).
6. **Loop por forno** `j = 1..M`: Temperatura (A); se o forno for elétrico, em seguida Teste elétrico do elemento (I).
7. **Inspeção visual (H)** — no fim.

---

## 7. Sequenciador e construtor de plano

### 7.1 Construtor de plano (`FC_ConstruirPlano`) — pseudocódigo

```
idx := 0;
r := DB_Receitas.Receitas[ProdutoSelecionado];

// 1) Globais, em ordem de segurança
SE r.HabEstanqueidade ENTÃO idx++; Plano[idx] := PassoGlobal(TIPO_ESTANQUEIDADE, r.DurEstanqueidade, [r.Estanqueidade]);
SE r.HabSegEletrica   ENTÃO idx++; Plano[idx] := PassoGlobal(TIPO_SEG_ELETRICA,  r.DurSegEletrica,   r.SegEletrica[1..2]);
SE r.HabEletSistema   ENTÃO idx++; Plano[idx] := PassoGlobal(TIPO_ELET_SISTEMA,  r.DurEletSistema,   r.EletSistema[1..4]);
SE r.HabFaisca        ENTÃO idx++; Plano[idx] := PassoGlobal(TIPO_FAISCA,        r.DurFaisca,        [faisca]);

// 2) Queimadores (individual e em ordem)
SE r.HabVazao ENTÃO
  PARA i := 1 ATÉ r.NumQueimadores FAÇA
    idx++; Plano[idx] := PassoAlvo(TIPO_VAZAO_MAX, i, r.DurVazaoMax, [r.VazaoMax[i]]);
    idx++; Plano[idx] := PassoAlvo(TIPO_VAZAO_MIN, i, r.DurVazaoMin, [r.VazaoMin[i]]);
  FIM;

// 3) Fornos (individual e em ordem)
PARA j := 1 ATÉ r.NumFornos FAÇA
  idx++; Plano[idx] := PassoAlvo(TIPO_TEMPERATURA, j, r.DurTemperatura, [r.Temperatura[j]]);
  SE r.FornoEletrico[j] ENTÃO
    idx++; Plano[idx] := PassoAlvo(TIPO_ELET_FORNO, j, r.DurEletForno, r.EletForno[j,1..3]);
  FIM;

// 4) Visual no fim
SE r.HabVisual ENTÃO idx++; Plano[idx] := PassoGlobal(TIPO_VISUAL, T#0s, []);

TotalPassos := idx;
```

### 7.2 Sequenciador (`FB_Sequenciador`) — estados

- **EST_OCIOSO**: aguarda `IniciarTeste`. Ao receber, zera `ResultadosPasso[]`, `PassoAtual := 0` e vai para EST_CONSTRUIR.
- **EST_CONSTRUIR**: chama `FC_ConstruirPlano`; `PassoAtual := 1`; vai para EST_EXECUTAR.
- **EST_EXECUTAR**: chama `FB_ExecutarPasso` para o `PassoAtual`. Quando o passo conclui (grava `ResultadosPasso[PassoAtual]`):
  - se `PassoAtual < TotalPassos` → `PassoAtual++`;
  - senão → EST_FINALIZADO.
  - `AbortarTeste` a qualquer momento → marca abortado e volta a EST_OCIOSO.
- **EST_FINALIZADO**: `AprovadoGeral := E lógico de todos os ResultadosPasso[].Aprovado`; incrementa `ContadorAprovados`/`ContadorReprovados`; monta `ListaReprovados`. Aguarda `Resetar`.

### 7.3 Executor de passo (`FB_ExecutarPasso`) — lógica por tipo

Estrutura `CASE Plano[PassoAtual].TipoTeste OF`. Padrão de cada passo:

1. **Entrada do passo**: copia descrição/limites para o espelho (`DB_Execucao`), configura `DB_Simulacao` (modelo, alvo, parâmetros), `ResultadoPassoAtual := 0`, arma o temporizador IEC TON com a duração do passo.
2. **Durante**: `FB_Simulacao` (em OB30) atualiza o valor; o executor copia para `ValorMedidoAtual`/`SubMedidaAtual` e atualiza `TempoDecorridoPasso`.
3. **Condição de fim** (varia por tipo — ver abaixo).
4. **Avaliação**: para cada medição usada, verifica `LimiteMin ≤ valor ≤ LimiteMax`. O passo é **Aprovado** se **todas** as medições usadas passam.
5. **Grava** `ResultadosPasso[PassoAtual]`, `ResultadoPassoAtual := 1 ou 2`, `Concluido := true`.

Condições de fim por tipo:

| Tipo | Condição de fim | Critério de aprovação |
|---|---|---|
| Estanqueidade (D) | tempo de espera (Duração) expira | ΔP ≤ LimiteMax |
| Seg. elétrica (E) | duração curta expira | todas as sub-medições na faixa |
| Elét. sistema (G) | duração curta expira | todas as sub-medições na faixa |
| Faísca (F) | duração curta expira | **todos** os centelhadores detectados |
| Vazão máx (B) | duração expira | vazão na faixa |
| Vazão mín (C) | duração expira | vazão na faixa **e** chama mantida |
| Temperatura (A) | **temperatura entra na faixa** (aprova) **ou** tempo limite expira (reprova) | atingiu a faixa dentro do tempo |
| Elét. forno (I) | duração curta expira | todas as sub-medições na faixa |
| Visual (H) | operador pressiona **OK** ou **NOK** | OK = aprovado, NOK = reprovado |

> A temperatura é o único passo "corrida contra o tempo"; o visual é o único sem temporizador (aguarda o operador). Boa variedade de lógica para treino.

---

## 8. Modelos de simulação (`FB_Simulacao`, OB30, dt = 0,1 s)

Todos os modelos só evoluem de forma significativa quando `Habilitar = true` e `ModeloAtivo` corresponde. Caso contrário, ficam em repouso (ou esfriam, no caso térmico).

### 8.1 Térmico (forno) — 1ª ordem
```
SE aquecendo:   Temperatura := Temperatura + (TempRegime  - Temperatura) * (dt / TauTermico);
SENÃO:          Temperatura := Temperatura + (TempAmbiente - Temperatura) * (dt / TauResfria);
```
- `TempRegime` deve ficar **acima** de `LimiteMax` (ex.: alvo 250 °C, faixa 240–260, `TempRegime = 280`) para garantir que a temperatura **cruza** a faixa.
- Forno a gás vs elétrico: muda apenas `TauTermico` e `TempRegime`.
- **Forçar falha**: reduzir `TempRegime` abaixo de `LimiteMin` (nunca alcança) ou aumentar `TauTermico` (não chega no tempo).
- **Ótimo para animação**: medidor 0–300 °C subindo ao vivo + barra de tempo.

### 8.2 Decaimento de pressão (estanqueidade)
```
Fase 1 (pressurização): Pressao sobe rápido até PressaoInicial (P0).
Fase 2 (medição, dura Duração):
    Pressao := Pressao - TaxaVazamento * dt;
    DeltaPressao := PressaoInicial - Pressao;
```
- Produto bom: `TaxaVazamento` pequena (ex.: 0,02 mbar/s → ΔP ≈ 0,3 mbar em 15 s, abaixo do limite 1,0).
- **Forçar falha**: aumentar `TaxaVazamento` (ex.: 0,1 mbar/s → ΔP ≈ 1,5 mbar > 1,0 → reprova).
- **Ótimo para animação**: barra de pressão caindo + cronômetro + valor de ΔP.

### 8.3 Vazão (queimador)
```
Vazao := ValorNominal + ruido;   // estabiliza rápido; rampa opcional de 1–2 s no início
```
- Vazão mínima: condição extra de **chama mantida** (flag `ChamaAtiva`). Se cair, reprova.
- **Forçar falha**: deslocar o valor para fora da faixa.

### 8.4 Elétricos (E, G, I)
```
PARA cada sub-medição k usada:
    ValorEletrico[k] := Medicao[k].ValorNominal + ruido;
```
- Passo aprova se todas as sub-medições usadas estão na faixa.
- **Forçar falha**: deslocar uma das sub-medições para fora.

### 8.5 Faísca/ignição (global)
```
PARA k := 1 ATÉ NumCentelhadores:
    após um curto atraso, FaiscaDetectada[k] := true;
```
- Aprova se todos os centelhadores habilitados detectaram faísca.
- **Forçar falha**: deixar um `FaiscaDetectada[k] := false`.

### 8.6 Gerador de ruído (`FC_GerarRuido`) — LCG simples (SCL)
```pascal
// LCG: o overflow de DInt no S7-1200 "dá a volta" sozinho — é o desejado
#Semente := #Semente * 1103515245 + 12345;
#u := DINT_TO_REAL(#Semente AND 16#7FFFFFFF) / 2147483647.0;   // 0..1
#ruido := (#u - 0.5) * 2.0 * (#ValorNominal * #AmplitudeRuido); // ±(amplitude % do nominal)
```
> Pseudoaleatório suficiente para treino. Semeie a partir do relógio do sistema em OB100 para variar entre execuções.

### 8.7 Como "forçar falha" age
Uma flag global `ForcarFalhaPassoAtual` (botão na IHM) é lida pela simulação. Quando ativa para o passo atual, o modelo desloca o valor (ou desativa um centelhador / mantém ΔP alto / não atinge a temperatura), de forma a reprovar o passo de propósito — ideal para demonstração.

---

## 9. Receitas de exemplo

### 9.1 Visão geral dos modelos

| Modelo | Índice | Queimadores | Fornos | Forno 1 | Forno 2 | Passos totais |
|---|---|---|---|---|---|---|
| Fogão A | 1 | 5 | 2 | Gás | Elétrico | 18 |
| Fogão B | 2 | 4 | 1 | Gás | — | 14 |
| Fogão C | 3 | 4 | 1 | Elétrico | — | 15 |

Sequência expandida do **Fogão A** (exemplo):
```
1  D  Estanqueidade            (sistema)
2  E  Seguranca eletrica       (sistema)
3  G  Eletrico do sistema      (sistema)
4  F  Faisca - todos           (sistema)
5  B  Vazao maxima             (Queimador 1)
6  C  Vazao minima             (Queimador 1)
7  B  Vazao maxima             (Queimador 2)
8  C  Vazao minima             (Queimador 2)
9  B  Vazao maxima             (Queimador 3)
10 C  Vazao minima             (Queimador 3)
11 B  Vazao maxima             (Queimador 4)
12 C  Vazao minima             (Queimador 4)
13 B  Vazao maxima             (Queimador 5)
14 C  Vazao minima             (Queimador 5)
15 A  Temperatura              (Forno 1 - gas)
16 A  Temperatura              (Forno 2 - eletrico)
17 I  Eletrico do elemento     (Forno 2 - eletrico)
18 H  Inspecao visual          (sistema)
```

### 9.2 Limites e durações de exemplo (calibrar depois)

> Valores ilustrativos para começar. `Min "—"` = sem limite inferior (use −1e9); `Max "—"` = sem limite superior (use +1e9).

| Teste | Grandeza | Unid. | Min | Max | Nominal | Duração (exemplo) |
|---|---|---|---|---|---|---|
| Estanqueidade (D) | Queda de pressão ΔP | mbar | — | 1,0 | 0,3 | 15 s (P0 = 30 mbar) |
| Seg. elétrica (E) | Resist. isolamento | MΩ | 2,0 | — | 5,0 | 5 s |
| Seg. elétrica (E) | Corrente de fuga | mA | — | 0,5 | 0,2 | (mesmo passo) |
| Elét. sistema (G) | Resist. aterramento | Ω | — | 0,1 | 0,05 | 5 s |
| Elét. sistema (G) | Corrente de operação | A | 8,0 | 12,0 | 10,0 | (mesmo passo) |
| Elét. sistema (G) | Potência | W | 1800 | 2200 | 2000 | (mesmo passo) |
| Elét. sistema (G) | Resistência | Ω | 18 | 26 | 22 | (mesmo passo) |
| Faísca (F) | Centelhadores OK | — | todos | — | — | 3 s |
| Vazão máx (B) | Vazão | L/h | 180 | 240 | 210 | 4 s por queimador |
| Vazão mín (C) | Vazão | L/h | 25 | 55 | 40 | 4 s por queimador |
| Temperatura (A) | Temp. atingida | °C | 240 | 260 | alvo 250 | até 120 s |
| Elét. forno (I) | Resist. elemento | Ω | 18 | 24 | 21 | 5 s |
| Elét. forno (I) | Corrente | A | 9 | 11 | 10 | (mesmo passo) |
| Elét. forno (I) | Potência | W | 2000 | 2400 | 2200 | (mesmo passo) |

> "Durações variam por receita": o Fogão A pode, por exemplo, ter forno maior com `DurTemperatura = 150 s`, enquanto o Fogão C usa 120 s.

---

## 10. Telas da IHM (KTP900 Basic)

### Tela 1 — Início / Seleção de Produto
- **Seletor de produto** (campo de E/S simbólico → `ProdutoSelecionado`: 1/2/3 mapeado para o nome).
- Exibe: `NomeReceita`, `NumQueimadores`, `NumFornos`, tipo de cada forno.
- Campo de entrada: `NumeroSerie`.
- Botão **Iniciar Teste** (`IniciarTeste`).
- Indicadores: `ContadorAprovados`, `ContadorReprovados`.

### Tela 2 — Execução do Passo Atual
- Cabeçalho: **"Passo X de Y"** (`PassoAtual` / `TotalPassos`), `NomePassoAtual`, `AlvoTextoAtual`.
- Área de medição:
  - **Medidor/barra** do valor principal (`ValorMedidoAtual`) com escala (ex.: temperatura 0–300 °C; pressão como barra + ΔP).
  - Campos das sub-medições (`SubMedidaAtual[1..4]`) para os passos elétricos.
- Limites visíveis: `LimiteMinAtual`, `LimiteMaxAtual`.
- **Barra de progresso de tempo**: `TempoDecorridoPasso` / `DuracaoPassoAtual`.
- **Lâmpada de resultado** (cor por aparência dinâmica via `ResultadoPassoAtual`: cinza=rodando, verde=passou, vermelho=falhou).
- Botão **Forçar Falha** (`ForcarFalhaPassoAtual`) — treino/demo.
- Botões **OK** / **NOK** — **só aparecem** no passo visual (visibilidade dinâmica por `TipoTesteAtual = TIPO_VISUAL`).
- Botão **Abortar** (`AbortarTeste`).

### Tela 3 — Resultado Final / Detalhamento
- **Resultado geral** grande: APROVADO/REPROVADO (`AprovadoGeral`, cor dinâmica).
- `NumeroSerie` e `NomeReceita`.
- **Detalhamento passo a passo**: linhas com Nº, Tipo, Alvo, Valor medido, Faixa, P/F, ligadas a `ResultadosPasso[1..n]` (célula P/F colorida).
- **Reprovados em destaque** (`ListaReprovados`) — para retrabalho.
- Contadores totais.
- Botão **Próxima Unidade** (`Resetar`).

> **Limitação de tabela no Basic:** não há tabela rolável nativa. Abordagem prática: (a) montar `ListaReprovados` como texto no CLP e exibir em um campo de texto multilinha (resumo para retrabalho) e (b) para o detalhamento completo, usar linhas fixas de campos de E/S (ex.: ~20 linhas) ligadas ao array `ResultadosPasso[]`, distribuídas em uma ou duas sub-telas.

### Tela 4 (opcional) — Modo Treino / Diagnóstico
- Ajuste de `TauTermico`, `TaxaVazamento`, `AmplitudeRuido` (para o instrutor variar o comportamento).
- Reset de contadores.
- (Opcional) toggles de forçar falha por tipo de passo.

### Alarmes (opcional)
- Alarmes discretos por categoria de reprovação. O painel Basic suporta alarmes básicos.

---

## 11. Lista de tags da IHM

| Tag IHM | Tag CLP (DB.membro) | Tipo | Acesso | Uso |
|---|---|---|---|---|
| ProdutoSelecionado | DB_Execucao.ProdutoSelecionado | Int | RW | Seleção de modelo |
| NomeReceita | DB_Receitas.Receitas[x].Nome | String | R | Nome do modelo |
| NumQueimadores | DB_Receitas.Receitas[x].NumQueimadores | Int | R | Info |
| NumFornos | DB_Receitas.Receitas[x].NumFornos | Int | R | Info |
| NumeroSerie | DB_Execucao.NumeroSerie | String | RW | Identificação |
| IniciarTeste | DB_Execucao.IniciarTeste | Bool | W | Botão iniciar |
| AbortarTeste | DB_Execucao.AbortarTeste | Bool | W | Botão abortar |
| Resetar | DB_Execucao.Resetar | Bool | W | Próxima unidade |
| PassoAtual | DB_Execucao.PassoAtual | Int | R | "Passo X" |
| TotalPassos | DB_Execucao.TotalPassos | Int | R | "de Y" |
| NomePassoAtual | DB_Execucao.NomePassoAtual | String | R | Descrição do passo |
| AlvoTextoAtual | DB_Execucao.AlvoTextoAtual | String | R | Componente sob teste |
| TipoTesteAtual | DB_Execucao.TipoTesteAtual | Int | R | Controla visibilidade |
| ValorMedidoAtual | DB_Execucao.ValorMedidoAtual | Real | R | Medidor/barra |
| SubMedidaAtual[1..4] | DB_Execucao.SubMedidaAtual | Real | R | Sub-medições elétricas |
| LimiteMinAtual | DB_Execucao.LimiteMinAtual | Real | R | Limite inferior |
| LimiteMaxAtual | DB_Execucao.LimiteMaxAtual | Real | R | Limite superior |
| TempoDecorridoPasso | DB_Execucao.TempoDecorridoPasso | Time | R | Barra de tempo |
| DuracaoPassoAtual | DB_Execucao.DuracaoPassoAtual | Time | R | Barra de tempo |
| ResultadoPassoAtual | DB_Execucao.ResultadoPassoAtual | Int | R | Cor da lâmpada |
| ForcarFalhaPassoAtual | DB_Execucao.ForcarFalhaPassoAtual | Bool | RW | Treino/demo |
| VisualOK | DB_Execucao.VisualOK | Bool | W | Inspeção visual |
| VisualNOK | DB_Execucao.VisualNOK | Bool | W | Inspeção visual |
| AprovadoGeral | DB_Execucao.AprovadoGeral | Bool | R | Resultado final |
| ContadorAprovados | DB_Execucao.ContadorAprovados | Int | R | Estatística |
| ContadorReprovados | DB_Execucao.ContadorReprovados | Int | R | Estatística |
| ListaReprovados | DB_Execucao.ListaReprovados | String | R | Retrabalho |
| ResultadosPasso[i].* | DB_Execucao.ResultadosPasso | UDT | R | Detalhamento final |

---

## 12. Convenções de nomenclatura (pt-BR)

- **Blocos de dados:** `DB_Receitas`, `DB_Execucao`, `DB_Simulacao`.
- **Blocos de função:** `FB_Sequenciador`, `FB_ExecutarPasso`, `FB_Simulacao`.
- **Funções:** `FC_ConstruirPlano`, `FC_GerarRuido`.
- **UDTs:** prefixo `udt_` (`udt_MedicaoFaixa`, `udt_DefinicaoPasso`, `udt_ResultadoPasso`, `udt_Receita`).
- **Constantes (enum):** `TIPO_*`, `EST_*` (PLC user constants).
- **Tags/membros:** PascalCase em português, nomes descritivos (`PassoAtual`, `TempoDecorridoPasso`, `ForcarFalhaPassoAtual`).
- **OBs:** OB1 (Main), OB30 (Cyclic 100 ms), OB100 (Startup).

---

## 13. Roteiro de construção em fases

| Fase | Entrega | O que validar |
|---|---|---|
| **1 — Estrutura de dados** | UDTs + `DB_Receitas` (só **Fogão B**, o mais simples) + `DB_Execucao` + `DB_Simulacao`. | Tipos compilam; receita preenchida. |
| **2 — Sequenciador básico** | Máquina de estados percorrendo um plano fixo, resultados forçados (tudo passa). Telas 1 e 2 básicas. | Fluxo Ocioso→Executar→Finalizado; "Passo X de Y" anda. |
| **3 — Construtor de plano** | `FC_ConstruirPlano` expandindo a receita (loops de queimador/forno). | `TotalPassos` e a sequência batem com a tabela do Fogão B. |
| **4 — Simulação** | OB30 + modelos (começar pelo **térmico** e **pressão**, que são visuais) + ruído + comparação com limites reais. | Medidor/barra animam; aprovação correta. |
| **5 — Falha + visual + final** | Botão **Forçar Falha**, passo **visual** (OK/NOK), **Tela 3** com detalhamento. | Reprovação sob demanda; lista de reprovados; contadores. |
| **6 — Demais receitas + acabamento** | Fogão A e C; calibrar valores; cores/animações; alarmes (opcional). | Troca de modelo muda a sequência; visual consistente. |
| **7 — Extensões** | Função de Receita do painel, log de resultados, estatísticas, OEE. | (Aprendizado avançado.) |

---

## 14. Pontos de aprendizado por parte

- **UDTs e arrays de UDT:** modelagem de dados limpa e reutilizável.
- **Construtor de plano:** loops `FOR`, indexação, "compilar" uma receita em uma lista de passos.
- **Máquina de estados (SCL):** padrão de sequenciador industrial.
- **Executor por tipo (CASE):** múltiplas lógicas de passo (corrida contra o tempo, espera fixa, entrada do operador).
- **Interrupção cíclica + `dt` fixo:** integração de modelos físicos (1ª ordem, decaimento).
- **Lógica de faixa e relatório:** aprovação multi-critério e detalhamento final.
- **IHM Basic sem script:** animação por valor, aparência dinâmica (cor) e visibilidade dinâmica.

---

## 15. Próximos passos / extensões possíveis

- Migrar as receitas para a **função de Receita do painel** (deliberadamente, como próximo nível).
- **Log de resultados** com número de série, data/hora e P/F por unidade.
- **Estatísticas/OEE**: taxa de aprovação, falhas mais comuns por categoria.
- Produtos sem gás (cooktop elétrico): exercitar as **flags de habilitação** para pular estanqueidade/faísca/vazão.
- Tempo de ciclo por modelo e gráfico de tendência (temperatura ao vivo) na IHM.

---

*Confirme/ajuste os termos em pt-BR e os valores de exemplo. Em seguida, podemos detalhar a Fase 1: a declaração completa das UDTs e o preenchimento do `DB_Receitas` para o Fogão B, já prontos para digitar no TIA Portal.*
