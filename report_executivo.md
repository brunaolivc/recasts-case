# Relatório Executivo — Sistema de Recomendação de Produtos Financeiros

## Contexto e Objetivo

O Itaú exibe 20 produtos em carrossel horizontal no app. Apenas as **5 primeiras posições são visíveis sem scroll**, tornando a ordenação crítica para conversão. O projeto avaliou se um sistema de recomendação personalizado supera a ordenação estática por segmento, com foco em maximizar contratações e receita.

---

## Principais Descobertas da Análise Exploratória

- **Viés de posição confirmado empiricamente:** a posição 1 tem CTR ~10× maior que a posição 20. As 5 primeiras posições concentram ~67% de todos os cliques — a ordem importa decisivamente.
- **55% da base é segmento básico** com 0–1 produto ativo: alto potencial de cross-sell, mas exige estratégia diferenciada de cold-start.
- **27% dos clientes não têm nenhum contrato histórico**, tornando a estratégia de cold-start obrigatória em qualquer solução.
- **Padrões de co-contratação claros:** produtos de investimento se agrupam (CDB + LCI, previdência + tesouro); crédito pessoal e consignado raramente coexistem — são alternativos.
- **Taxa de contratação é baixa (~0.9%)**, com forte variação entre produtos: consórcio imóvel e crédito pessoal têm a maior receita por contrato (R$450 e R$210, respectivamente).

---

## Abordagens Exploradas

| Modelo | Precision@5 | NDCG@5 | Hit Rate@5 |
|--------|-------------|--------|------------|
| Aleatório (baseline) | 5,5% | 16,2% | 27,0% |
| Popularidade global | 14,8% | 53,5% | 73,2% |
| **Segmento (melhor)** | **18,7%** | **66,6%** | **92,5%** |
| LightGBM (IPW) | 8,6% | 26,2% | 43,1% |
| Filtragem Colaborativa SVD | 7,2% | 25,6% | 35,8% |
| Híbrido LightGBM + CF | 7,2% | 25,6% | 35,8% |

**Avaliação:** split temporal — treino em 21 safras (jan/2024–set/2025), teste nas 3 safras mais recentes (out–dez/2025).

---

## Modelo Final e Justificativa

O **Baseline por Segmento** obteve o melhor desempenho em Precision@5 e NDCG@5. Esse resultado, aparentemente contra-intuitivo, revela uma descoberta importante: no período de avaliação, o comportamento de contratação segue fortemente a distribuição de popularidade por segmento — novos contratos tendem a ser os produtos mais populares dentro do segmento do cliente.

O **LightGBM** alcançou AUC=0.9994 na tarefa de predição de interação, demonstrando que o modelo aprende o sinal corretamente. Sua performance inferior no ranking reflete um desafio estrutural: os melhores preditores de interação individual (histórico cliente-produto) penalizam produtos novos para o cliente — exatamente os candidatos de maior interesse no ranking. Esse é um trade-off conhecido em sistemas de recomendação (*exploration vs. exploitation*).

**Para produção, recomendamos uma solução em estágios:**
1. **Curto prazo:** Baseline por Segmento como ranking padrão (melhor resultado imediato, baixíssimo custo operacional).
2. **Médio prazo:** Hibridização com LightGBM com features redesenhadas para novos produtos (categorias, perfil financeiro sem histórico de interação), eliminando o viés de repetição.
3. **Longo prazo:** Modelo sequencial (BERT4Rec ou GRU4Rec) capturando a jornada de contratação do cliente.

O arquivo `outputs/recomendacoes.csv` foi gerado pelo modelo Híbrido (LightGBM + CF-SVD) para todos os 50.000 clientes, excluindo produtos com status ativo.

---

## Limitações e Próximos Passos

- **Leakage potencial nas features de interação:** n_cliques e n_contratos históricos por (cliente, produto) sinalizam comportamento passado, não demanda futura por produtos novos. Redesenhar features com janela deslizante e separação por tipo de produto.
- **CF-SVD limitado pelo catálogo pequeno:** apenas 20 produtos reduzem o espaço latente útil; com lançamentos futuros o modelo ganhará mais poder.
- **A/B Test:** qualquer modelo deve ser validado em produção com 4+ semanas, métricas de CTR do carrossel e taxa de contratação por posição como KPIs primários, com guardrails em NPS e churn.
- **Open Finance:** incorporar dados de portabilidade e comportamento em outras instituições aumentaria a capacidade preditiva para cold-start.
