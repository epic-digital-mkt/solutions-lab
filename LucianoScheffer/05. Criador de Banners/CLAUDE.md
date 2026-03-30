# CLAUDE.md — Criador de Banners

## O que é este projeto

Ferramenta web single-file para gerar banners a partir de designs do Figma.
O usuário seleciona um template do Figma, edita textos e foto, e exporta o banner final como PNG.

---

## Arquivos principais

| Arquivo | Descrição |
|---------|-----------|
| `gerador-banners-dot-v32.html` | **Arquivo ativo** — versão em desenvolvimento |
| `gerador-banners-dot-v31.html` | **Ponto de retorno seguro** — não alterar |
| `index.html` | Redirecionamento para o arquivo ativo (GitHub Pages) |
| `server.js` | Servidor Node.js porta 3333 (uso local) |

**Figma fileKey ativo (DOT Digital Group):** `yLEcNJEiQNSjNPsFRwfzfj`

---

## Servidor

```bash
# Subir (pasta correta — server.js está na subpasta desde o git mv)
cd "c:/_Luciano Diversos/_estudos IA/05. Criador de Banners/LucianoScheffer/05. Criador de Banners" && node server.js

# Verificar se está rodando (deve retornar ~182000+ bytes)
curl -s http://localhost:3333/ | wc -c

# Ver log remoto de debug
curl http://localhost:3333/log

# Encerrar
powershell -Command "Stop-Process -Id <PID> -Force"
# ou matar todos os node do projeto:
powershell -Command "Get-WmiObject Win32_Process -Filter \"name='node.exe' AND CommandLine LIKE '%server.js%'\" | ForEach-Object { $_.Terminate() }"
```

O servidor:
- Serve o HTML mais recente em `/` (seleciona automaticamente por regex de nome)
- Proxy da API Figma: `/figma-api/...`
- Proxy de imagens (evita CORS no canvas): `/figma-img?url=...`
- Endpoint de log remoto: GET `/log` retorna buffer JSON; POST `/log` adiciona entrada ou `{clear:true}` limpa
- Claude lê o log via `curl http://localhost:3333/log` — não precisa que o usuário copie e cole

---

## Regras fundamentais

### Sempre fiel ao Figma
- Todos os valores visuais (cores, tamanhos, posições, fontes) vêm da API Figma
- Nunca hardcodar valores que estão no Figma
- Nunca substituir texto do Figma por placeholder genérico — o texto do Figma é a fonte da verdade
- O usuário edita *a partir* do texto do Figma, nunca de um placeholder

### Comentários em português
- Todos os comentários de código devem ser escritos em português

### Não escalar além do necessário
- Não adicionar features, refatorações ou melhorias além do que foi pedido
- Uma correção de bug não precisa limpar o código ao redor

---

## Regras técnicas aprendidas

- **fillOpacity:** `Math.round((color.a ?? 1) * (fill.opacity ?? 1) * 100) / 100` — multiplicar os dois campos
- **blurRadius:** API Figma retorna `BACKGROUND_BLUR.radius` como 2× o valor CSS → dividir por 2
- **Glass fill:** usar fill com `blendMode === 'NORMAL'`; ignorar fills com blend modes especiais (PLUS_LIGHTER etc.)
- **blocoGlass / blocoTX / blocoTag / field:olhos:** NÃO exportam PNG — renderizados programaticamente; marcar `renderedDynamic: true` em `buildLayerDefs`, skip em `loadLayerImages`, skip em `layersReady`
- **effectiveTextWidth:** sempre usar para textos dentro de containers (cap ao glassBox ou blocoTX, tolerância ±20px)
- **pillBaseY fallback:** `fTag.y` (não `pf.y`) — para que `pillDY = 0` e a pill fique na posição correta
- **walkNodes roda 2×** (discoverLayouts + extractLayoutMeta): código que deve rodar apenas na segunda vez deve estar FORA do guard `if (!meta.fields[key])`

---

## Gestos touch — versão mobile

| Gesto | Comportamento |
|-------|--------------|
| 1 dedo (modo arrastar desativado) | Scroll vertical normal da página |
| 1 dedo (modo arrastar ativo) | Arrasta a foto no canvas |
| 2 dedos (pinch) | Zoom in/out do canvas |

> **TODO:** organizar em documentação formal junto com o guia "Como preparar o Figma" — incluir convenção de layers, gestos mobile, e boas práticas de template.

---

## Convenção de nomes de layers no Figma

| Nome da layer | Tipo Figma | Função |
|---------------|-----------|--------|
| `field:titulo` | TEXT | Título editável |
| `field:texto` | TEXT | Subtítulo/corpo editável |
| `field:tag` | TEXT | Tag/categoria editável |
| `field:cta` | FRAME | Container do botão CTA |
| `field:ctatx` | TEXT | Texto interno do CTA — **invisível no Figma** (opacity 0) |
| `field:foto` | FRAME | Área da foto do usuário |
| `field:olhos` | qualquer shape | Marca onde os olhos devem aparecer (cx/cy = alvo, w = IOD) — **invisível no Figma** |
| `maskFoto` | FRAME | Máscara de recorte da foto |
| `asset/[nome]` | qualquer | Asset estático exportado (logo, ícone, fundo, etc.) |
| `blocoGlass` | FRAME | Container com efeito vidro fosco (blur + fill) |
| `blocoTX` | FRAME | Container de texto com fundo sólido |
| `blocoTag` | FRAME | Container visual da pill de tag |

### Onde posicionar `field:olhos` conforme o tipo de layout

| Estrutura da foto | Onde colocar `field:olhos` |
|-------------------|---------------------------|
| `maskFoto/field:foto` | Dentro do frame `maskFoto`, na posição dos olhos na área recortada |
| `field:foto` solto (crop-fill, com ou sem quadrado) | No frame do layout, na posição absoluta desejada no canvas |

---

## U2 — Posicionamento automático por olhos (em desenvolvimento)

**Status:** funcional em v24_olhos. Detecção L1/L2/L3/L4/L5 com face-api.js.

**CDNs:**
```javascript
const FACE_API_CDN    = 'https://cdn.jsdelivr.net/npm/face-api.js@0.22.2/dist/face-api.min.js';
const FACE_API_MODELS = 'https://cdn.jsdelivr.net/gh/justadudewhohacks/face-api.js@0.22.2/weights/';
// IMPORTANTE: usar CDN GitHub (cdn.jsdelivr.net/gh/) para os modelos — o CDN npm não tem pasta weights/
```

**Atenção:** o código do U2 está no segundo `<script>` do HTML. Deve ser `<script>` regular, **NUNCA `<script type="module">`** — funções em módulo ficam em escopo isolado e não são visíveis para o primeiro script.

---

## Deploy

A ferramenta funciona em dois ambientes via `IS_LOCAL` (detectado automaticamente):
- **Local:** `node server.js` → `http://localhost:3333` — usa proxy do servidor
- **GitHub Pages:** `epic-digital-mkt/solutions-lab` → `LucianoScheffer/05. Criador de Banners/` — chama API Figma e CDN direto via CORS

A cada nova versão: atualizar `index.html` com o novo nome de arquivo.

## Pendências atuais

- **U2 translação:** verificar se ox/oy estão posicionando corretamente (em teste)
- **Feature A:** input de nome do arquivo ao baixar PNG

---

## Visão de longo prazo (4 fases)

1. Editor manual perfeito — Figma → peça *(atual)*
2. Agente de texto — IA gera copy no tom de cada cliente
3. Geração autônoma — busca template + imagem + texto por tags e gera
4. Publicação autônoma — agendamento + postagem (Instagram Graph API, HubSpot Social)
