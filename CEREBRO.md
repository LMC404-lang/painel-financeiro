# CÉREBRO — Painel Financeiro

> Este é o arquivo de contexto vivo do projeto. Qualquer sessão de IA
> (Claude, outra conta, etc) deve ler este arquivo no início do trabalho
> e atualizá-lo no fim, sempre que houver commit/push de mudança no
> dashboard, decisão tomada, ou bug encontrado/corrigido.
>
> Última atualização: 21/08/2026

---

## Quem somos

- **Lucas Ceve** (lucas.ceve@levitsbpo.net) — responsável técnico 100% do
  projeto (dashboard + sync + VPS). Toda mudança passa por ele.
- **Camilli** — também usa a mesma ferramenta de IA neste projeto.
- **Edgar** — dono da empresa, decisor de negócio, não mexe em código.

## O que é o projeto

Dashboard financeiro multi-empresa para 3 empresas: **KMNO Sports**,
**RT Esportes e Eventos**, **NOVAH** (ex-Blue Line, rebranding concluído
20/07/2026 — usar sempre `NOVAH` no código, nunca `BLUELINE`).

- Front-end: `index.html` único (~18k+ linhas), vanilla JS + Chart.js,
  hospedado no GitHub Pages.
- Repo: **`Lceve/painel-financeiro`** (mudou de `LMC404-lang` em 21/08/2026 —
  usar sempre o novo nome).
- Backend: 4 projetos Supabase — KMNO (`enedbeguahicctwwhpmb`), Novah
  (`yppfzhptzcesmxiruaxk`), RT (`jdifejativsnghfxxeqe`), consolidado/dashboard
  (`ihekejwxdvipgldblskn`).
- Fonte de dados: Omie ERP via API → sync Node.js na VPS → Supabase.
- Jan–Mai/2026: dados congelados de Excel. Jun/2026+: só Omie/Supabase
  (fonte única da verdade).
- VPS: Hostinger Ubuntu 24.04, `srv1817879`, usuário `lucasadmin`, todo
  comando precisa de `sudo`.

## Regras de trabalho (sempre seguir)

1. Mudança que afeta mais de um local no `index.html` → pedir permissão
   antes, explicando o quê e onde.
2. Qualquer dúvida, por menor que seja → perguntar antes de agir, nunca
   assumir.
3. Sempre dizer o que vai fazer **antes** de fazer.
4. Lucas valida tudo — nada vai pra produção sem passar por ele.
5. Nunca sugerir F5/Ctrl+Shift+R, exceto último recurso.
6. Toda mudança no dashboard → entregar arquivo atualizado + resumo
   rápido e direto do que mudou.
7. Comandos de VPS sempre com `sudo`.
8. Ao final de qualquer sessão com commit/push, mudança de escopo, ou
   descoberta relevante → atualizar este arquivo antes de encerrar.

## Aprendizados críticos (nunca esquecer)

- **Saldo bancário KMNO**: o bloco "CAIXAS" da planilha/dashboard para
  KMNO inclui contas físicas nomeadas "RT" (Brasil RT, Sicredi RT) — são
  contas físicas da KMNO, **não** uma seleção separada da empresa RT.
  Nunca tratar isoladamente.
- **Categorias dinâmicas**: nomes de categoria (ex: "Transferências entre
  contas") vêm do Supabase em tempo de execução, não são hardcoded —
  filtrar por `cat.descricao` no JS, nunca buscar no arquivo estático.
- **`buildArvoreNativaDre`**: valida em ~0,1% de precisão desde julho — é
  a fonte primária pra DRE por Competência.
- **`sudo cd` falha silenciosamente** na VPS — sempre usar
  `sudo bash -c "cd /path && comando"`.
- **Escrever arquivos na VPS**: usar `sudo tee /path/arquivo > /dev/null`
  com heredoc — `sudo` com `>` de redirecionamento falha (redirect roda
  como não-root).

## Links úteis

- Repo: https://github.com/Lceve/painel-financeiro
- Dashboard: https://lceve.github.io/painel-financeiro/

---

## PENDÊNCIAS ABERTAS

### 🔴 Divergência de Impostos — DRE por Competência KMNO

Card diverge do relatório nativo Omie em 5 pontos por `codigo_dre`.

**1. Impostos (`1.01.02`)** — o mais investigado:
- Excel nativo (por emissão e por registro, idênticos): Jun/2026 =
  R$ 234.358,999991
- Dashboard pós-fix de NFe cancelada: Jun/2026 = R$ 307.127
- Diferença: ~R$ 72.768, **não muda antes/depois do fix**

Fix aplicado 21/08/2026 (commit `d23a589`): exclui do somatório de
imposto embutido na NFe (`buildArvoreNativaDre`, loop `nfeRows`) notas
com `dt_cancelamento`, `denegada='S'`, ou `dt_inutilizacao` preenchidos.
Validado: só 7 notas no período, R$1.342,99 de imposto excluído — fix
correto, mas não é a causa principal da divergência.

RT não tem colunas `dt_cancelamento`/`denegada`/`dt_inutilizacao` em
`omie_nfe` (só KMNO e Novah têm) — erro não bloqueante no sync, decisão
de estender pra RT pendente.

Pista não testada a fundo: "documents with exactly 3 lines per NFe number
in Omie native export = false positive in DRE gap analysis" — parte do
gap pode ser falso positivo de metodologia de comparação, não bug real.

Categorias reais mapeadas pra `1.01.02` (KMNO): `2.06.03` (PIS/COFINS),
`2.06.04` (Fundo Social FUMDES), `2.06.07` (ICMS — tem exclusão especial
a partir de abr/2026, ver `KMNO_ICMS_RESIDUO_CATEGORIA` no código),
`2.06.01` (DIFAL), `2.14.98` (Simples Nacional Parcelado).

Query exploratória (não conclusiva, não considerou exclusão de ICMS
resíduo nem gateway `1.04.96`): NFe com categoria = R$213.574,69;
movements = -R$88.004,09; soma = R$125.570,60 — não bateu com nenhum dos
dois números de referência.

**Próximo passo**: testar com a lógica COMPLETA do dashboard
(`fmCategFallback`, `extratoSemTitulo`, `isTituloExcluidoManualmenteKmno`,
exclusão de ICMS resíduo, exclusão de gateway `1.04.96`), não soma
simples de títulos.

**2. Outras Receitas (`1.11.01`)**: zerado em query simples, Omie mostra
R$39k. Suspeita: vem de `extrato-sem-título`. Não testado.

**3. Custo Serviços (`1.21.02`)**: gap ~R$105k. Causa desconhecida.

**4. Despesas Variáveis (`2.01.01`)**: gap ~R$11k. Bug pequeno confirmado
não corrigido: em `buildResumoCompetenciaViaArvoreNativa` (~linha 6194),
`despesasVariaveis = -soma('2.01.01') + soma('2.01.02')` trata
Recuperação de Despesas Variáveis com sinal errado (~R$140/mês, efeito
pequeno, não é a causa principal do gap).

**5. Despesas Pessoal/Administrativas (`2.11.01`/`2.11.02`)**: nosso
valor mais NEGATIVO que o Omie — direção oposta, sugere miscategorização
ou fallback de categoria faltando.

Contexto adicional: Jun/2026 nosso ≈ -R$197k vs Omie ≈ -R$158k (gap
~R$40k); Jul/2026 diverge ~R$320k incluindo inversão de sinal (nós
mostramos prejuízo, Omie mostra lucro).

---

### 🟡 DRE por CMC (novo card)

Tela separada no menu principal, replica a cascata do "Resumo (Grupo,
R$)" trocando CMV por CMC (custo real de estoque: quantidade vendida ×
custo unitário do produto).

**Decisões fechadas**: tela própria; estrutura pronta pras 3 empresas mas
só populando KMNO agora; Supabase com resumo agregado mensal (sem
detalhe por produto/drill-down); sync dentro de
`/root/omie-sync/omie-supabase-sync-KMNO` na VPS, diário; CMC usa data de
cada venda individual (não fixo de fim de mês).

**Reaproveitamento de código**: usar `buildResumoCategoriasMes`, pegar o
objeto de retorno inteiro, substituir só `cmv` pelo `cmc` novo,
recalcular cascata abaixo. Alternativa: rodar `buildResumo____` normal e
sobrescrever a linha de CMV antes de `extrairMetricasResumo(rows)`.

**API Omie validada**:
- Custo por produto: `ListarPosEstoque`
  (`https://app.omie.com.br/api/v1/estoque/consulta/`), retorna `nCMC` e
  `nSaldo`, paginado, filtro `codigo_local_estoque=10334036177` +
  `cExibeTodos="S"`. ~9.602 produtos / ~193 páginas de 50.
- Quantidade vendida: `ListarPedidos`
  (`https://app.omie.com.br/api/v1/produtos/pedido/`), filtrar por
  período, usar `produto.codigo` + `produto.quantidade`, confirmar
  `infoCadastro.faturado="S"`. Validado Jun/2026: 489 pedidos, 456
  faturados, 536 produtos únicos.
- Descartado: `ObterResumoProdutos` (só resumo executivo, sem detalhe por
  produto).
- CORS: API só via backend/VPS, nunca do `index.html` direto.
- Fórmula: `CMC do mês = Σ (quantidade vendida do produto × nCMC na data
  de posição de estoque usada)`.

**Exceção de produto**: códigos `307209-0-UNICO` e `307210-0-UNICO`
("PATCH KIMONO BORDADO") vendidos por ponto de bordado (unidade
"milheiro"), não por peça — excluir do cálculo de CMC.

**Credenciais**: `/root/omie-sync/omie-supabase-sync-KMNO/.env` tem
`OMIE_APP_KEY`/`OMIE_APP_SECRET` (sem sufixo `_KMNO`). Padrão de chamada:
```bash
sudo bash -c 'set -a; source /root/omie-sync/omie-supabase-sync-KMNO/.env; set +a; <comando>'
```

**Pendente antes de escrever código**: desenhar tabela Supabase; escrever
script de sync (paginar `ListarPosEstoque` e `ListarPedidos`, cruzar por
`nCodProd`/`produto.codigo`, agregar); cruzar quantidade-por-produto de
Jun/2026 contra `PosicaoEstoque` pra validar contra referência esperada
(não feito ainda); construir tela nova seguindo padrão visual do "Resumo
(Grupo, R$)".

---

### 🟡 Faturador KMNO (projeto separado, não é o dashboard)

Roda em `C:\omie-sync\omie-sync-novah`, máquina Windows do Edgar
(separado da VPS/dashboard).

**Estado**: 15 pedidos KMNO pré-01/07/2026 cancelados no Omie com nota
(exceto nº2733/2271805680, bloqueado pós-cancelamento, deixar como está).
Scripts (`categorizador-final.js`, `megaconsolidacao-coletar.js`,
`criar-consolidado-2.js`) foram perdidos do disco e reconstruídos com
preços validados:

| Item | Preço |
|---|---|
| Gandola Adulto / Infantil | R$18 / R$13,50 |
| Calça Adulto / Infantil | R$12 (PRO/STUCK R$18) / R$9 |
| Faixa Lisa / com Friso | R$2,50 / R$3,50 |
| Bermuda / Rash Guard (incl. Mormaii) | R$7,50 |
| Kimono Infantil Sarja / Mormaii Kimono | R$20 / R$30 |
| Patch Composição / Sublimado P-M-G | R$1,25 / R$3,50-R$5-R$6 |
| Resto | CFOP 5125 |

Excluídos por falta de preço/cadastro inativo: "Patch Gola em Z", "Patch
Sublimado Embutido", família "Patch Kimono Bordado".

**Suspeita não investigada**: `CancelarPedidoVenda` não atualiza `dAlt`,
sync incremental pode perder cancelamentos puros.

**Universo atual (muda diariamente, sempre revalidar)**: ~110 pedidos
candidatos (etapa 50, cliente KMNO, ≥01/07, sem CFOP 6xxx) → ~329 itens
únicos → 3 lotes de até 110 itens. 97 pedidos abertos em etapa=50 (Novah,
cliente KMNO) — não confirmado se são os 77 originais dos consolidados
cancelados nº3835/3836 ou incluem novos.

**Nada criado ainda**: script de consolidação real, decisão de
cancelar+faturar originais, cron real na VPS com log diário.

**Agendamento planejado**: Segunda–Quinta, teto ~90-110 itens/lote (não
por pedido), Quinta libera o resto da semana, meta ~30-31
consolidados/mês.

**Limites técnicos Omie**: `IncluirPedido` com 100+ itens precisa
`AbortController` 240s timeout (servidor pode completar após abort —
sempre checar com `ListarPedidos`); lote confiável máx ~110 itens;
`AlterarPedidoVenda` exige `quantidade` mesmo sem mudança.

---

### ⚪ Outras pendências menores

- `migration_metas.sql` — status de execução não confirmado.
- `index (teste).html` órfão no repo — decisão de remover ou restaurar
  pendente.
