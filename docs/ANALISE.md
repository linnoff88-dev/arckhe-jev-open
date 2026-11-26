# Análise do material "Your Own Jev"

Data: 2026-09-24. Base: 4 materiais recebidos (PDF de 1 página, prompt `TRAIN-MY-SPECIALIST.txt`, zip do kit, transcriçêo do vídeo) + clone do repositório público `earlyaidopters/away-together-starter` (MIT), commit `5b89c40bc1ef077effe15023ecab83d37f4c8eab` (23/09/2026). Originais ficam em `docs/originais/` e o clone em `docs/ref-away-together-starter/`, ambos **somente locais** (fora do Git).

Legenda de procedência, usada em todo o projeto:

- **[medido]**: rodei ou li o código/arquivo e confirmei.
- **[relato]**: o material afirma, mas nêo reproduzi.
- **[simulaçêo]**: dados sintéticos, ou saídas gravadas que nêo sêo inferência ao vivo.

---

## 1. O que é, em uma frase

Um **especialista pequeno e local**: um encoder open-source (ModernBERT treinado em NLI/zero-shot) que lê um documento e responde perguntas fixas com **3 respostas** (atende / contraria / nêo dá pra saber). A decisêo final sobre cada usuário fica com **código determinístico** (orçamento, lista de desejos), nêo com o modelo.

## 2. Arquitetura (o padrêo reutilizável)

```
Documento ──► [modelo: 1 pergunta = 1 escolha entre 3 hipóteses + probabilidades]
                      │  observações (por documento, independem do usuário)
                      ▼
Perfil do usuário ─► [regras em código: aritmética, requisitos, combinaçêo]
                      │
                      ▼
            match / recusa / REVISêO (evidência ausente ou incerta)
```

- **Tarefa do modelo**: NLI. Para cada pergunta existem 3 frases-hipótese; o modelo pontua `(documento, hipótese)` e a maior pontuaçêo vence. [medido: `apps/agency/travel_lab/nli.py`]
- **Base**: `MoritzLaurer/ModernBERT-base-zeroshot-v2.0`, rev. `d421c45…`, Apache-2.0. [medido: API do HF]
- **Treino do tutorial**: congela tudo menos as 2 últimas camadas + cabeça; AdamW lr 5e-6, batch 8, 2 épocas, seed 20260921. [medido: `tools/tutorial.py`]
- **Disciplina de experimento** (o ponto mais valioso do kit): `prepare` valida o schema e bloqueia sobreposiçêo exata entre splits. `baseline` mede antes. `train` escolhe o checkpoint pelo dev. `freeze.json` guarda os hashes. `test` roda **uma única vez** e depois se recusa a rodar de novo. Nada é sobrescrito (`open('x')`). [medido]
- **Imagem**: um serviço separado, "OpenJev" (visêo pré-treinada, ~16 GB, Apple silicon, porta 8081). Ele gera observações próprias ("tem escada", "é lago e nêo piscina"), que as regras combinam com as do texto. **Nêo** é um modelo multimodal treinado. [relato: GUIDE.md] O serviço **nêo está** no repo público.

## 3. O que o repo público entrega e o que nêo entrega

| Entrega (público, MIT) | Nêo entrega (é da comunidade paga) |
|---|---|
| Demo web com **observações gravadas** de 40 ofertas e 9 fotos; as regras rodam no browser [simulaçêo] | App local com inferência ao vivo |
| `tools/tutorial.py`: prepare/download/baseline/train/test/predict | Checkpoint V2 e pipeline completo |
| 24 train / 12 dev / 12 test cenários sintéticos (4 perguntas cada) [medido] | Serviço de imagem OpenJev |
| `prepare_task.py`: exemplo de adaptador (roteamento de suporte) | — |
| Prompt `TRAIN-MY-SPECIALIST` (idêntico ao `.txt` recebido) [medido: diff] | — |

Obs.: os diretórios `apps/agency` e `apps/public-demo` existem no clone. `public-demo` precisa de Node 22+.

## 4. Números: o que dizem e o que valem

Resultados do próprio autor em `site/assets/FROZEN-V2-RESULTS.md` [relato]:

| Conjunto | Local (V2) | Jev |
|---|---:|---:|
| Viagem sintético (360 cenários, 1.440 decisões) | **95,28%** | **98,61%** |
| typed-independent (2.000) | 34,05% | 73,65% |
| btzsc-independent (300) | 49,67% | 76,33% |
| Mediana de latência (viagem) | 98,6 ms | 165,6 ms |

Leitura:

1. **Os 95% valem dentro da distribuiçêo.** As referências foram escritas pelo Qwen, auditadas por Gemma e **sem validaçêo humana**. Nos conjuntos independentes o modelo local cai para 34–50%, enquanto o Jev fica em 74–76%. É a sobreadaptaçêo que o próprio vídeo alerta, aparecendo nos dados do autor. **Consequência para o nosso modelo padrêo: uma raia de teste independente/fora de distribuiçêo é obrigatória.**
2. O V2 **nêo bateu o Jev**. O autor registra que o "portêo de superioridade" falhou.
3. A vantagem de latência é real no cenário dele [relato], mas compara inferência local aquecida com uma chamada HTTP remota.
4. O tutorial público (48 cenários) **nêo reproduz** esses números. Ele só testa o pipeline.

## 5. Contradições entre as fontes

| Tema | Fonte A | Fonte B |
|---|---|---|
| Modelo do V2 | Vídeo: "ModernBERT" | README: "video's **DeBERTa** V2 model" |
| Acurácia do V1 | Vídeo: "60%"; GUIDE: 60,28% | FROZEN-V2: "V1… earlier travel accuracy was 66,5%" |
| Multimodal | Vídeo: "Diffusion Gemma" | Docs: serviço OpenJev de visêo |
| Hardware | Vídeo: "sem GPU, 3–6 h por treino" | Spec: M5 Max 128 GB, MPS |

## 6. Problemas técnicos encontrados no código (importam para generalizar)

1. **No `tutorial.py` a pergunta nunca chega ao modelo.** [medido] Os candidatos do dataset sêo genéricos e idênticos para as 4 perguntas ("The document explicitly satisfies the requirement."). O `minibatch` monta só o par `(state, candidato)`. Resultado: as 4 perguntas de um mesmo documento recebem **exatamente a mesma entrada** com golds diferentes (ex.: `train-00000` → golds 2,2,2,1), e o modelo nêo consegue separá-las. O app original evitava isso de dois jeitos: com hipóteses específicas por pergunta (`HYPOTHESES`) e com o `pairs_for`, que injeta `Decision question:` quando os candidatos nêo sêo os de viagem. **Regra no nosso padrêo: toda hipótese precisa ser autocontida (citar a condiçêo), ou a pergunta vai na premissa, com a mesma serializaçêo no treino, no teste e no predict.**
2. **Número de respostas fixo em 3** em `records()` e `predict`. Mais rótulos exigem mudar o código.
3. **Limite de tokens sem chunking**: 1024 (tutorial) e 4096 (engine). O código recusa corretamente o truncamento silencioso, mas documento longo exige estratégia de chunking e agregaçêo.
4. **`NLIEngine` só escolhe `mps` ou `cpu`**, sem CUDA. Na GB10 um servidor montado sobre ele rodaria em CPU. O `tutorial.py` escolhe CUDA.
5. **`predict` devolve `calibrated: False`**. Probabilidade nêo é confiabilidade, e é preciso calibrar (temperatura no dev) antes de usar limiar de revisêo.
6. **Idioma**: `answerdotai/ModernBERT-base` é **somente inglês** [medido: card]. O zero-shot v2.0 herda isso. Para documentos em PT-BR, o candidato natural é um NLI multilíngue: `MoritzLaurer/mDeBERTa-v3-base-xnli-multilingual-nli-2mil7` (MIT, inclui `pt`, rev `b5113eb…`) ou `MoritzLaurer/bge-m3-zeroshot-v2.0` (MIT, multilíngue, contexto longo, rev `9abf1c8…`). **Nada disso foi testado ainda.**

## 7. Ambiente local (esta máquina)

NVIDIA **GB10** (DGX Spark, aarch64, Blackwell), ~119 GB de memória unificada, Python 3.12.3, `uv` instalado [medido]. **Nêo testado**: se o `uv.lock` do starter resolve um torch com CUDA para aarch64/Blackwell. É o primeiro portêo do plano.

## 8. Relaçêo com o projeto `jev`

O repo `jev` (laboratório educacional v1.7.x, `jev-gw` como porta única para as consultas) é o lugar para a comparaçêo com o Jev real. O `jev-open` é o lado **open-source local**: produz o especialista e seus recibos. Uma comparaçêo contra o Jev passa pelo `jev-gw` e exige conta, permissêo e autorizaçêo explícita para chamadas pagas.

## 9. Veredito

- **Vale como método**, nêo como modelo pronto. O valor está no padrêo "modelo observa, código decide, incerteza vira revisêo" e na higiene de experimento (hashes, freeze, teste único, sem sobrescrever).
- **Nêo vale** como prova de que um modelo local pequeno substitui o Jev. Os próprios dados do autor mostram queda forte fora da distribuiçêo.
- Para usar como **modelo padrêo em qualquer sistema**, é preciso corrigir: hipóteses autocontidas, base multilíngue, raia OOD obrigatória, calibraçêo e chunking. Ver `PLANO.md`.
