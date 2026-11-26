# Resultados do run v1 (2026-09-25)

> **Escopo.** Dados **100% sintéticos**, gerados e conferidos por LLMs locais (a mesma família de IA escreveu a ficha, os dados e os rótulos, o que é circular). Os números abaixo sêo **[simulaçêo] do pipeline**: provam que as fases 2–6 rodam de ponta a ponta com recibos. **Nêo** provam nada sobre o desempenho em mensagens reais de escritório ou clínica. Latência e RAM sêo **[medido] na GB10 (CPU aarch64)**, nêo numa VPS. Revisêo por IA nêo é validaçêo humana.

Recibos: `tasks/<nicho>/recibos/v1/` (`data-manifest`, `baseline`, `training`, `freeze`, `test`, `servir`, `medida-test-2t`, `servico-testes-local`, `servico-testes-docker`).

## Dados (fase 2)

| | Advocacia | Clínica |
|---|---|---|
| Treino / dev / teste / OOD | 200 / 60 / 60 / 60 | 200 / 60 / 60 / 60 |
| Gerador in-dist → verificador | qwen3.6:35b-a3b → qwen3:30b (31% aceitos) | idem (42% aceitos) |
| Gerador OOD (estilo e-mail) | llama3.1:8b (29% aceitos) | llama3.1:8b (39% aceitos) |
| Duplicatas exatas / aproximadas | 0 / 0 | 0 / 0 |
| Decisões "incerto" (treino/dev/teste/OOD) | 13 / 3 / 7 / 2 | 10 / 1 / 3 / 3 |

**Achado 1: "incerto" quase some.** O filtro exige que o gerador e o verificador concordem, e dois LLMs raramente concordam sobre o que é ambíguo. Resultado: o modelo aprende sobretudo "sim" e "nêo", e o macro-F1 (que pesa os 3 rótulos igualmente) fica baixo mesmo com acurácia alta. No sistema, a revisêo humana depende do limiar de confiança no código, nêo de o modelo dizer "incerto".

## Baseline sem treino (fase 3, dev, macro-F1) [simulaçêo]

| Motor | Advocacia | Clínica |
|---|---|---|
| bge-m3-zeroshot (hipóteses) | **0,508** | **0,556** |
| mDeBERTa-xnli (hipóteses / única) | 0,148 / 0,162 | 0,154 / 0,178 |
| ModernBERT (só inglês) | 0,149 | 0,193 |
| Rede de palavras-chave (só a pergunta crítica) | 0,610 | **0,956** |

Decisêo registrada nos dois nichos: nenhum zero-shot bate a meta de 0,85 → treinar o bge-m3.

**Achado 2: na clínica, a lista de palavras-chave quase acerta tudo sozinha na pergunta de alarme.** Isso diz mais sobre os dados sintéticos (o gerador escreve "falta de ar", "dor no peito" com as palavras canônicas) do que sobre o problema real. Mensagens reais devem ser bem mais difíceis.

## Treino e teste final (fases 4–5) [simulaçêo]

Treino: bge-m3, só as 2 últimas camadas + cabeça (26M de 568M parâmetros), 4 épocas, lr 1e-5, seed 20260925, checkpoint pelo dev, temperatura calibrada no dev (advocacia 1,42; clínica 1,82). ~70 s por época (advocacia) e ~33 s (clínica) na GB10 [medido].

| | Advocacia base → treinado | Clínica base → treinado |
|---|---|---|
| Teste: acurácia | 72% → **96%** | 82% → **96%** |
| Teste: macro-F1 | 0,53 → **0,63** | 0,61 → **0,77** |
| OOD: acurácia | 72% → **98%** | 83% → **93%** |
| OOD: macro-F1 | 0,48 → **0,64** | 0,59 → **0,61** |
| Pergunta crítica perdida no teste (modelo / com a rede) | 0 → **1 / 1** (de 10) | 0 → **0 / 0** (de 19) |

Metas da ficha (definidas antes de treinar), avaliadas nos dados sintéticos:

| Meta | Advocacia | Clínica |
|---|---|---|
| Perdidos da pergunta crítica = 0 | **reprovada** (1) | aprovada (0) |
| Macro-F1 OOD ≥ 0,70 | reprovada (0,64) | reprovada (0,61) |
| Queda teste→OOD ≤ 15 pp | aprovada (−1,1) | **reprovada** (16,0) |

**Achado 3: a urgência perdida na advocacia.** Mensagem de remarcar reuniêo porque o cliente "adoeceu repentinamente com uma febre alta" e "precisamos agilizar qualquer pendência urgente". O gabarito (gerado por IA) diz urgência jurídica = sim; o modelo deu 44% sim contra 51% nêo. O rótulo é discutível (é um problema de saúde, nêo um prazo judicial), mas pela regra da ficha conta como perdido. A confiança do "nêo" foi 0,505: um limiar acima de ~0,51 na pergunta crítica mandaria esse caso para uma pessoa. Se um limiar calibrado no dev chegaria lá **nêo foi medido** (as probabilidades do dev nêo foram salvas), e nada foi ajustado olhando o teste.

## Serviço em CPU (fase 6)

| | Advocacia | Clínica |
|---|---|---|
| Pacote ONNX int8 | 570 MB | 570 MB |
| Portêo int8 vs fp32 (±1 pp de macro-F1) | aprovado (−0,79 pp) | **reprovado (−2,32 pp)** |
| Concordância int8 × torch nas decisões do teste | 98,8% | 99,6% |
| Latência por mensagem, 2 threads, p50 / p95 [medido, GB10] | 1,53 s / 2,85 s (7 perguntas) | 0,73 s / 1,14 s (4 perguntas) |
| Pico de RAM do processo, um nicho [medido, GB10, VmHWM] | 1,32 GB | 1,16 GB |
| Testes de serviço (urgente, ambígua, contraditória, vazia, longa, malformada) — local e Docker | 9/9 | 9/9 |

Container com os dois nichos (`--memory 4g --cpus 2`): **1,87 GB** em uso [medido, arm64 na GB10]. Imagem de 314 MB (sem pesos; os pacotes sêo montados em `/pacotes`). A imagem amd64 nêo foi construída.

**Achado 4: o portêo int8 da clínica reprovou** por 2,3 pp, embora só ~1 decisêo em 240 mude. O macro-F1 é instável aqui porque "incerto" é raríssimo. Próximo passo: quantizar deixando a cabeça e a última camada em fp32 e medir de novo, ou servir fp32 (2,3 GB, ~5× mais lento).

**Achado 5: o caso ambíguo da advocacia** ("recebi uma carta do tribunal, nêo entendi, o que eu faço?") foi para a fila do advogado por dúvida jurídica, mas **nêo** como urgência imediata: o modelo disse "nêo" para urgência onde o gabarito dizia "incerto". Um limiar de confiança na pergunta crítica poderia transformar isso em revisêo imediata, dependendo da confiança do "nêo" nesse caso (nêo medido aqui).

## Falhas corrigidas no caminho

Ver `FALHAS.md`: chave de meta divergente entre ficha e código (a validaçêo agora é feita na carga), `ru_maxrss` herdado do processo pai invalidando a RAM do `servir.json` (substituída por `medida-test-2t.json`, com `VmHWM` num processo limpo), e gerador OOD lento demais trocado.

## O que falta para valer como evidência

1. Mensagens **reais anonimizadas**, anotadas por duas pessoas, com concordância medida.
2. OOD de outro escritório/clínica ou de outro período.
3. Limiar de confiança por pergunta calibrado no dev (especialmente na pergunta crítica).
4. Latência e RAM numa **VPS real** de 2 vCPU / 4 GB, e a imagem amd64.
5. Validaçêo das listas de palavras-chave e dos textos fixos por profissionais (OAB / saúde).
