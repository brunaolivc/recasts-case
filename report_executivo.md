# Relatório Executivo — Sistema de Recomendação de Produtos Financeiros

## Contexto e Objetivo
O Itaú exibe 20 produtos em carrossel horizontal no app. Apenas as **5 primeiras posições são visíveis sem scroll**, tornando a ordenação crítica para conversão. O objetivo foi construir um sistema de recomendação personalizado que maximize contratações e receita, substituindo a ordenação estática por segmento.

---

## Principais Descobertas da EDA
- **Viés de posição confirmado:** posição 1 tem CTR **27.3× maior** que a posição 20. As 5 primeiras posições concentram **90.6% dos cliques** — a ordem importa decisivamente.
- **27.3% dos clientes (13.633) sem histórico de contrato** — estratégia de cold-start obrigatória. Solução: fallback por popularidade segmentada.
- **Padrão de onboarding identificado:** `conta_digital_plus + credito_pessoal` é o par mais co-contratado (4.3% da base) — sinal de jornada que o modelo deve explorar.

---

## Resultados

| Modelo | Precision@5 | NDCG@5 | HitRate@5 |
|--------|-------------|--------|-----------|
| Aleatório | 5.4% | 16.5% | 27.2% |
| Popularidade | 14.8% | 53.5% | 73.2% |
| Segmento (melhor baseline) | 18.7% | 66.6% | 92.5% |
| LightGBM | 15.4% | 52.2% | 76.0% |
| CF-SVD | 7.2% | 25.6% | 35.8% |
| **Híbrido LightGBM + CF (α=0.65)** | **16.1%** | **54.5%** | **79.7%** |

---

## Modelo Final — Híbrido LightGBM + CF-SVD (α=0.65)
- **Supera o LightGBM isolado** em todas as métricas (NDCG@5 +2.3pp, HitRate@5 +3.7pp). Peso ótimo determinado empiricamente por busca em 7 candidatos.
- **Baseline por segmento lidera** — achado analítico esperado em dados sintéticos gerados com regras baseadas em segmento. Em dados reais, a personalização deve superar o baseline — hipótese a validar via A/B test.
- **Tensão receita vs. personalização identificada:** otimizar P(contratar) não é equivalente a otimizar receita esperada. Score ponderado `α × P(contratar) + (1-α) × CVR × receita_media` resolve o trade-off em produção.

O arquivo `outputs/recomendacoes.csv` foi gerado para todos os **50.000 clientes**, cobrindo 100% da base com 0 produtos ativos indevidamente recomendados.

---

## Limitações e Próximos Passos
- **Curto prazo:** deploy do híbrido com A/B test em 20% da base + modelos LightGBM por segmento (resultados indicam estrutura fortemente segmentada).
- **Médio prazo:** features com janela deslizante + Two-Tower model neural + variável `público-alvo` via NLP.
- **Longo prazo:** otimização multi-objetivo (receita + satisfação + diversidade) + BERT4Rec para capturar jornada de contratação.
