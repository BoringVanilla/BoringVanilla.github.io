# PLANO — Calculadora de VRAM para LLMs (site interativo em um único `index.html`)

## Objetivo

Substituir a landing page atual (Midas) por um **super site interativo**: uma calculadora
que responde *"N bilhões de parâmetros dá em quanto de VRAM?"* — cobrindo inferência
com quantização, KV cache e também treinamento (AdamW, 8-bit, QLoRA).

## Decisões já validadas com o usuário

| Decisão | Escolha |
|---|---|
| Landing page atual do Midas | **Substituir totalmente** (nenhum link mantido) |
| Escopo do cálculo | **Inferência + Treinamento** |
| GPUs no comparador | **Consumidor + Datacenter** |

## Estrutura da página (pt-BR, tema dark atual: `#0b1020` / `#4f9bff`, fonte Inter)

1. **Header** — logo "VRAM Calc", sticky com blur.
2. **Hero** — título com gradiente + subtítulo.
3. **Grid da calculadora** (2 colunas; empilha no mobile):
   - **Painel de controles (esquerda):**
     - Chips de presets: Llama 3.1 8B/70B/405B, Mistral 7B, Phi-4 14B, Gemma 2 27B,
       Qwen 2.5 72B, DeepSeek R1 671B (MoE, KV estilo MLA aproximado).
     - Parâmetros: input numérico + slider logarítmico (0,1B → ~2.000B), sincronizados.
     - Tabs: **⚡ Inferência** / **🔥 Treinamento**.
     - Inferência: precisão dos pesos (FP32 4B, FP16/BF16 2B, FP8/INT8 1B, INT4 0,53B,
       Q4_K_M 0,60B) + precisão do KV cache (FP16 2B, Q8_0 ~1B, Q4 ~0,56B).
     - Treinamento: otimizador (AdamW MP 16B/param, AdamW FP32 16B, AdamW 8-bit 6B,
       Adafactor ~4,5B, QLoRA base 4-bit + adapter) + toggle gradient checkpointing.
     - QLoRA: slider de rank (4–256, default 32) + checkbox "LoRA também nas MLP".
     - Comum: contexto (512 → 1M tokens em passos de 2×), batch (1–128),
       overhead do runtime (0–4 GB).
     - `<details>` "Arquitetura (avançado)": camadas, KV heads (GQA), head_dim, d_model, d_ffn — auto-preenchido pelos presets.
   - **Painel de resultado (direita, sticky no desktop):**
     - Número grande animado (GB total estimado) + resumo do modo.
     - **Donut chart em SVG puro** (segmentos animados via `stroke-dasharray`):
       pesos / KV cache / overhead (inferência) ou pesos / gradientes / otimizador /
       ativações / overhead (treinamento) ou base 4-bit / adapter / grads / otim / ativações (QLoRA).
     - Centro do donut: % usada da GPU selecionada (verde/âmbar/vermelho).
     - Legenda colorida com valores + **bloco de fórmulas** mostrando a conta passo a passo.
     - Botões: **Copiar link do cálculo** (estado via query string) e **Resetar**.
4. **Comparador de GPUs** — grade de cards clicáveis (a selecionada fixa no donut):
   - Consumidor: RTX 3050 8GB, RTX 3060 12GB, RTX 4060 Ti 16GB, RX 7900 XTX 24GB,
     RTX 3090 24GB, RTX 4090 24GB, RTX 5090 32GB, Mac Studio (unified) 64GB.
   - Datacenter: A100 40/80GB, H100 80GB, H200 141GB, MI300X 192GB, B200 192GB.
   - Cada card: VRAM ×quantidade (seletor ×1–×8), barra de uso, badge "✓ Cabe" / "✗ Não cabe",
     "sobram X GB" ou "faltam Y GB · precisa ~N× dessa GPU".
5. **Calculadora reversa** — input de GB → tabela "cabem ~X B parâmetros" por cenário
   (FP32, FP16, INT8, INT4, Q4_K_M, NF4, treino AdamW MP, treino 8-bit).
6. **Cola de fórmulas (cheatsheet)** — tabelas de bytes/param (com exemplo para 8B),
   custo de treinamento por parâmetro e as fórmulas usadas + nota GB×GiB.
7. **Footer** — créditos, nota de estimativas, "tudo em um único index.html".

## Fórmulas

- **Pesos** = `parâmetros × bytes_por_param`
- **KV cache** = `2 × camadas × kv_heads × head_dim × contexto × batch × bytes_elem`
- **Treinamento** = pesos + gradientes + estados do otimizador (m+v), por param:
  - AdamW FP32: 4+4+8 = **16 B** · AdamW mixed precision: 2+4(master)+2+8 = **16 B**
  - AdamW 8-bit: 2+2+2 = **6 B** · Adafactor: 2+2+~0,5 = **~4,5 B**
- **QLoRA** = base NF4 (0,54 B/param) + adapter fp16 + grads + otimizador 8-bit,
  onde `adapter_params = camadas × rank × [2·d + 2·(d + kv_heads·head_dim)] (+ 3·rank·(d+d_ffn) se MLP)`
- **Ativações (treino)** ≈ `contexto × batch × camadas × d_model × k`, com `k=34` sem
  checkpointing e `k=2` com checkpointing (fp16, estimativa).
- **Overhead** = ajustável, default 1 GB (contexto CUDA, buffers, ativações de inferência).
- Unidade interna: **GiB** (`2³⁰`), exibida como "GB" (padrão de mercado para VRAM).

## Técnica

- **Vanilla JS + CSS inline**, zero dependências externas (só Google Fonts, já usada hoje).
- Estado completo na URL (`?mode=inf&p=8.03&q=fp16&ctx=8192&gpu=rtx4090&qty=1...`)
  via `history.replaceState`; botão copiar link (`navigator.clipboard` + fallback).
- Donut em SVG (círculos com dasharray animado por transição CSS), contador de GB com
  `requestAnimationFrame` (ease-out), tooltips via CSS `[data-tip]`.
- Acessibilidade: tabs com `role`/`aria-selected`, labels em todos os inputs, `focus-visible`,
  respeito a `prefers-reduced-motion`.
- Responsivo: grid empilha abaixo de ~920px; cards de GPU em `auto-fill minmax(230px, 1fr)`.

## Status

- [x] Plano validado com o usuário
- [ ] Implementação do `index.html` (substituir a landing Midas)
- [ ] Verificação de sintaxe do JS (`node --check`) e revisão dos IDs HTML↔JS
