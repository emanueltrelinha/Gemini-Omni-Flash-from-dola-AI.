---
name: gemini-omni-flash-video
description: Use when the user wants to generate, edit, extend, or iterate on videos with Google's Gemini Omni Flash (gemini-omni-1.1-flash) via the Interactions API — including text-to-video, image-to-video, first/last-frame interpolation, reference-to-video, conversational multi-turn editing, scene extension up to 40s, 360p drafting, and 4K upscaling. Covers prompt craft, media-role tags, API parameters, pricing, limitations, and production workflows. Do NOT use for Veo 3.1 (native 4K / spatial audio, single-shot) or other video engines.
---

# Gemini Omni Flash — Skill de Geração & Edição de Vídeo

Habilidade (Skill) para extrair o máximo do **Gemini Omni Flash** (`gemini-omni-1.1-flash`), o modelo multimodal de vídeo do Google, via **Interactions API**. Conteúdo em português; templates de prompt em inglês (idioma com suporte completo e resultados mais estáveis — veja Limitações).

> Fontes oficiais: [Google AI Docs — Omni](https://ai.google.dev/gemini-api/docs/omni), [Blog Google — Omni 1.1 Flash](https://blog.google/innovation-and-ai/technology/developers-tools/build-with-gemini-omni-1-1-flash/), [Model Card](https://storage.googleapis.com/deepmind-media/Model-Cards/Gemini-Omni-Flash-Model-Card.pdf).

---

## 1. Quando usar esta Skill (e quando NÃO usar)

**Use Omni Flash quando o brief envolver:**
- Edição iterativa em linguagem natural ("mude o céu para pôr do sol", "troque o carro") dentro de uma mesma conversa — é o seu diferencial principal.
- Input "any-to-any": texto + imagem + referência de vídeo num único prompt.
- Cenas curtas (até 10s) para redes sociais, reels, shorts, protótipos.
- Interpolação entre dois quadros (first/last frame) para transições cinematográficas ou loops perfeitos.
- Extensão de cena em incrementos de 10s até 40s totais.
- Cadeia com imagem: gere/refine um still com **Nano Banana 2 Lite** e anime no mesmo thread.

**Não use Omni Flash (prefira Veo 3.1 ou outro) quando:**
- Precisar de **4K nativo** e áudio espacial limpo para TVC/brand film (Veo 3.1). Omni faz 4K por *upscaling*.
- O trabalho for single-shot, sem iteração, e priorizar polimento cinematográfico (dolly, rack focus) — Veo 3.1 interpreta vocabulário de DP com mais restrição.
- Precisar de clipes >40s ou editar voz/fala existente (não suportado).

---

## 2. Ficha técnica (sem chutar)

| Item | Valor |
|---|---|
| Model ID (produção) | `gemini-omni-1.1-flash` |
| Model ID (preview legado) | `gemini-omni-flash-preview` |
| API | Interactions API — `POST https://generativelanguage.googleapis.com/v1beta/interactions` |
| Duração / geração | **10s** (cap rígido) |
| Extensão de cena | +10s por chamada, até **40s acumulados**; só apenda no final; analisa até 10s de contexto anterior |
| Resolução | `360p` (rascunho) · `720p` (padrão) · `1080p` (upscale) · `4k` (upscale) |
| Proporção | `16:9` (padrão) · `9:16` (vertical) |
| Saída | `video/mp4` **com áudio** sincronizado ao prompt; Base64 ou URI |
| Preço (API) | ~US$ 0,10/seg de vídeo gerado; `360p` custa **1/3** de `720p` e é até **60% mais rápido** |
| Edições multi-turno | até **3 edições sequenciais** preservando sessão (`previous_interaction_id`) |
| Refs de vídeo | máx. **3 clipes × 3s** cada; áudio das refs é ignorado |
| Marca d'água | **SynthID** em toda saída (invisível, detectável para proveniência) |
| Idioma | inglês com suporte completo; outros idiomas podem funcionar mas variam |

---

## 3. API — o essencial

### 3.1 Chamada mínima (REST)
```bash
curl -X POST "https://generativelanguage.googleapis.com/v1beta/interactions?key=$GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemini-omni-1.1-flash",
    "input": "A cinematic drone shot of a mountain landscape at golden hour, slow forward movement, volumetric light.",
    "response_format": { "type": "video", "aspect_ratio": "16:9", "resolution": "720p" }
  }'
```

### 3.2 `response_format` — parâmetros de saída
- `type`: `"video"` (opcional, ajuda o modelo a entender o destino).
- `aspect_ratio`: `"16:9"` | `"9:16"`.
- `resolution`: `"360p"` | `"720p"` | `"1080p"` | `"4k"`.
- `delivery`: use `"uri"` para vídeos >4MB (ex.: 1080p/4K) para evitar limite de payload.

### 3.3 `generation_config.video_config.task` — só quando o prompt não bastar
Valores: `text_to_video` · `image_to_video` · `reference_to_video` · `edit` · `extend`.
> Regra: **prefira resolver com prompt**. Use `task` apenas se o modelo insistir em interpretar errado a intenção — o campo adiciona restrições.

### 3.4 Flags de performance
Para geração síncrona rápida: `background=false`, `store=false`, `stream=false`.
> ⚠️ `store=false` significa que o vídeo **não poderá ser editado/estendido** depois (sem `previous_interaction_id`). Se planeja iterar, mantenha `store=true` (ou omita).

### 3.5 Multi-turno (edição conversacional / extensão)
Passe `previous_interaction_id` da geração anterior e descreva a mudança em `input`. O modelo preserva o que não foi tocado.
```python
interaction = client.interactions.create(
    model="gemini-omni-1.1-flash",
    previous_interaction_id=video_anterior.id,
    input=[{"type": "text", "text": "Continue the scene. Camera slowly pulls back, dramatic music score."}],
    response_format={"resolution": "360p"},
)
```

---

## 4. Engenharia de Prompt — o coração da Skill

### 4.1 Anatomia de um prompt forte (7 eixos)
Ordene: **sujeito → ação → câmera → iluminação → ambiente → estilo/estética → áudio**.
Exemplo:
> *"A female barista in a beige apron [sujeito] pours steamed milk into a latte, slow motion [ação]. Tight close-up, shallow depth of field, slight handheld drift [câmera]. Warm morning side light through a window, soft shadows [iluminação]. Cozy minimalist café, steam rising [ambiente]. Photorealistic, 35mm film grain, color grade warm teal-orange [estilo]. Ambient café hum, gentle milk-pour sound [áudio]."*

### 4.2 Tags de papel de mídia (obrigatório aprender)
Vinculam cada imagem/vídeo enviado a um papel específico.

**Tags simples (recomendado):**
- `<FIRST_FRAME>` — usa a imagem como quadro inicial.
- `<LAST_FRAME>` — usa como quadro final (sempre junto de `<FIRST_FRAME>`).
- `<IMAGE_REF_N>` — referência (0-indexado): estilo, personagem, objeto. Ex.: `in the style of <IMAGE_REF_0>, the woman <IMAGE_REF_1> is walking`.
- `<VIDEO_REF_N>` — referência de personagem/objeto/ movimento: `the person in <VIDEO_REF_0> is playing the violin`.

**Declarações explícitas (para múltiplas mídias/papéis):** no início do prompt:
- `[# Sources <FIRST_FRAME>@Image1]` — Image1 = quadro inicial.
- `[# Sources <FIRST_FRAME>@Image1 <LAST_FRAME>@Image2]` — interpolação Image1→Image2.
- `[# Sources <FIRST_FRAME>@Image1 <LAST_FRAME>@Image1]` — **loop perfeito** (mesma imagem início/fim).
- `[# Sources <FIRST_FRAME>@Image1] [# References <IMAGE_REF_0>@Image2]` — início + referência.
- `[# Sources <VIDEO_0>@Video1]` — vídeo a ser editado.
- `[# Sources <PREVIOUS_VIDEO>@Video1]` — estender vídeo do turno anterior.
- `[# References <IMAGE_REF_0>@Image1 <VIDEO_REF_0>@Video1]` — refs mistas.

No final, adicione instruções-guia: *"Use Image1 as the starting frame. Use Image2 as a reference, not as a literal initial frame."*

### 4.3 Prompt segmentado no tempo
Para cenas com múltiplas batidas dentro de 10s:
```
[0-3s] A studio fashion sequence. The woman <IMAGE_REF_0> holds <IMAGE_REF_1>.
[3-6s] Whip-pan reveals the man <IMAGE_REF_2> holding <IMAGE_REF_3>.
[6-10s] Another woman <IMAGE_REF_4> holds <IMAGE_REF_5> while walking toward camera.
One continuous shot, no jump cuts.
```

### 4.4 Instruções negativas — no próprio prompt
Não existe parâmetro de negative prompt. Escreva proibições como texto:
> *"Do not show the reference drawing in the final video. No text overlays. No extra characters."*

### 4.5 Image-to-video — dica crítica
Imagens em alta resolução + descrição de movimento **específica**. *"Make it move"* dá resultado pobre; *"the character turns her head left and smiles, camera orbits 30 degrees, wind moves her hair"* dá resultado bom.

### 4.6 Consistência de personagem (ponto fraco conhecido — mitigue)
- Use 2–3 fotos de referência do mesmo personagem como `<IMAGE_REF_N>` em ângulos diferentes.
- Para movimento/coreografia, use `<VIDEO_REF_N>` (até 3 clipes × 3s) e ordene: *"The dog <IMAGE_REF_0> performs the classical dance from <VIDEO_REF_0>."*
- Evite trocar de cena/cenário no mesmo clipe se a consistência for crítica — o modelo enfraquece em mudanças de cena.

---

## 5. Playbooks de Produção (receitas passo a passo)

### Playbook A — Social clip iterativo (barato)
1. Gere rascunho em `360p` (1/3 do custo, 60% mais rápido).
2. No mesmo thread (`previous_interaction_id`), peça ajustes: *"Move subject left, warmer color grade, add rain."* (até 3 edições).
3. Finalize regerando em `1080p` ou `4k` com `delivery="uri"`.

### Playbook B — Foto → vídeo de produto ("Omni Product Studio")
1. (Opcional) Refina a still com **Nano Banana 2 Lite** (`gemini-3.1-flash-lite-image`) no mesmo thread.
2. Passe a foto como `<FIRST_FRAME>` + 1–2 refs de produto como `<IMAGE_REF_N>`.
3. Prompt: *"The product <IMAGE_REF_0> slowly rotates 360° on a marble surface, overhead key light raking left, macro close-up, luxury e-commerce hero shot, no text."*

### Playbook C — Transição cinematográfica / loop (first/last frame)
1. Prepare duas stills (início e fim) — ou a mesma para loop.
2. `[# Sources <FIRST_FRAME>@Image1 <LAST_FRAME>@Image2]` + prompt descrevendo movimento de câmera: *"One continuous optical dolly-zoom, no jump cuts, smooth morph from forest sunrise to snowy starry night."*

### Playbook D — Extensão até 40s
1. Gere clipe base de 10s (`store=true`).
2. A cada chamada: `previous_interaction_id` + *"Continue the scene. [nova batida + movimento de câmera + áudio]."*
3. Repita até 40s acumulados. Rascunhe em 360p, finalize em alta resolução.
> Restrições: só apenda no final (não prepend/meio). Vídeo enviado (upload) com pessoa falando não pode ser extendido com diálogo (personagem em silêncio ok; extensão de fala só em vídeo gerado via multi-turno).

### Playbook E — Mashup any-to-any
Envie no mesmo `input`: imagem de moodboard + referência de vídeo (movimento) + texto. O modelo funde composição (img), movimento (vídeo ref) e ação (texto) sem dividir em ferramentas separadas.

---

## 6. Limitações & Workarounds (leia antes de prometer)

| Limitação | Workaround |
|---|---|
| Cap de 10s por geração | Estenda em blocos de 10s até 40s (Playbook D) |
| Sem upload de referência de áudio | Descreva o áudio no texto ("ambient rain, orchestral score"); áudio é gerado sincronizado |
| Sem edição de voz/fala | Não prometa dublagem; gere fala nova apenas em extensão multi-turno de vídeo gerado |
| Refs de vídeo: máx 3×3s, áudio ignorado | Corte refs em clipes curtos antes; não dependa do áudio delas |
| Sem raciocínio entre múltiplos vídeos | Não mande mais de 3 refs de vídeo; degrada performance |
| EEA / Suíça / Reino Unido: sem upload/edição de imagens com menores ou pessoas reconhecíveis; sem editar/estender vídeos enviados (vídeos gerados pelo modelo ok) | Use geração do zero (text-to-video) ou gere em região suportada |
| Sem `temperature`, `top_p`, stop sequences, negative prompt | Coloque restrições no texto do prompt (§4.4) |
| Sem YouTube como fonte | Baixe/extraia frames antes se necessário |
| Consistência de personagem fraca em troca de cena | Mantenha mesmo cenário por clipe; use múltiplas `<IMAGE_REF>` + `<VIDEO_REF>` |
| Provisioned throughput não suportado | Planeje filas/retries para carga alta |
| Vídeos >4MB estouram payload Base64 | Use `delivery="uri"` |

---

## 7. Checklist de qualidade antes de entregar

- [ ] Prompt cobre os 7 eixos (sujeito, ação, câmera, luz, ambiente, estilo, áudio).
- [ ] Proporção correta para a plataforma (9:16 para Reels/Shorts/TikTok).
- [ ] Rascunhou em 360p antes de pagar por 1080p/4K.
- [ ] Mídias enviadas têm papel declarado (`<FIRST_FRAME>` / `<IMAGE_REF_N>` / `<VIDEO_REF_N>`).
- [ ] Se for iterar: `store=true` e `previous_interaction_id` corretos.
- [ ] Negativas escritas no prompt (não existe negative prompt).
- [ ] Vídeo final >4MB? Usou `delivery="uri"`.
- [ ] Ciente do SynthID (não prometa "sem marca d'água").
- [ ] Não prometeu fala/edit de voz nem clipes >40s.

---

## 8. Arquivos de referência

- `references/prompt-templates.md` — templates prontos (copiar-colar) para: produto, paisagem cinematográfica, reel vertical, loop, talking head, interpolação first/last frame, multi-personagem com vídeo-ref.

---

*Atualizado: out/2026 — baseado em `gemini-omni-1.1-flash` (Omni 1.1). Omni Pro (tier superior) confirmado pela Google mas sem data; planeje em torno do que já está em produção.*
