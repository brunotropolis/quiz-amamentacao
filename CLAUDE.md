# Quiz de Amamentação — "Será que meu bebê está mamando bem?"
## Manual do Recém-Nascido / Dayane dos Anjos

Quiz de avaliação personalizada de amamentação. A pessoa responde, e no fim é levada (tela de carregamento + redirect, estilo `perfis.manualdorecemnascido.com.br`) para uma **página de resposta que é uma página de vendas modular disfarçada de resultado**: ela se monta com as faixas (sinais) que a mãe marcou, explica cada uma e conduz pro curso. Checkout na Greenn.

## Links e infra
- **URL:** https://quizama.manualdorecemnascido.com.br (GitHub Pages + Cloudflare proxied, `noindex` não setado ainda)
- **Repo:** `brunotropolis/quiz-amamentacao` (público)
- **Pasta local:** `D:\CLAUDE\quiz-amamentacao\`
- **DNS:** CNAME `quizama` → `brunotropolis.github.io` (proxied) na zona **manualdorecemnascido** (`42a253a5f2baf2b06c822e3ce9d8389d`). ⚠️ O `CLOUDFLARE_DNS_TOKEN` NÃO cobre essa zona — o registro foi criado pela **API interna do dash via Chrome logado** (`GET dash.cloudflare.com/api/v4/system/bootstrap` → `result.data.atok` → `POST .../zones/{zone}/dns_records` com headers `x-atok` + `x-cross-site-security: dash`; o wrapper do Chrome MCP zera output de fetch-com-credencial, contornar gravando em `window.__x` e lendo na 2ª chamada).
- **Fonte da verdade da copy + lógica:** `C:\Users\bruno\Downloads\QUIZ-AMAMENTAÇÃO.md` (briefing consolidado da Day). Onde houver conflito, vale o doc dela.

## Arquivos
| Arquivo | Função |
|---|---|
| `index.html` | O quiz (engine data-driven + 4 fluxos + roteamento + loading + redirect) |
| `resultado.html` | Página de resposta modular (formato página de vendas) — hoje só Fluxo 2 (Cat 2/Cat 3) |
| `server.js` | Servidor estático local só pra preview (gitignored, NÃO vai pro Pages) |
| `assets/hero.webp` | Foto do bebê mamando (reaproveitada de `tdapocket.manualdorecemnascido.com.br/assets/hero.webp`) |

## Identidade visual
Base na referência "CLUB SLEEP" (e-commerce de sono, pastel). Descritivo travado com o Bruno:
- **Paleta:** blush `#F6D7DD` (cor-mãe), rosa `#EE8DA8`, **verde pinho `#2F4A3D` nos CTAs** (contraste que faz o botão saltar), azul `#BFE1EC`, coral `#F5A38C`, lavanda `#ECE0EF`. Aprovados: verde + rosa blush.
- **Tipografia:** Playfair Display (títulos serif), Nunito (corpo/UI), Dancing Script (toques afetivos).
- **Elementos:** nuvens brancas, corações de papel flutuando (rosa/azul/coral), cantos arredondados, recorte orgânico (blob) nas fotos, botões pílula.

## Arquitetura do fluxo
```
QUIZ (index.html)
  P1 idade → roteia 4 fluxos: gestante / rn (0-30d) / m1a5 / m6a12
  perguntas condicionais (única/múltipla/foto sim-não)
  → calcula DESFECHO + (Fluxo 2) a "receita" de faixas
  ↓
TELA DE CARREGAMENTO (~3.3s, coração pulsando)   [só Fluxo 2 Cat 2/Cat 3 hoje]
  ↓
REDIRECT → resultado.html?cat=..&oferta=..&enfase=..&faixas=a,b,c
  ↓
PÁGINA DE VENDAS que se monta pela receita (quiz = cérebro, página = renderizador burro)
```
Demais desfechos (Cat 1/sono, Entre Sonecas, fluxos 1/3/4) ainda caem num **cartão de hand-off provisório** no próprio quiz (mostra o desfecho + `window.__quizResult`). Falta construir as páginas deles.

## Engine do quiz (`index.html`)
- Data-driven: `IDADE` (P1) + `FLOWS = {rn, m1a5, m6a12, gestante}`. Cada pergunta: `{id, type:'single'|'multi'|'foto', text, hint, options[], showIf?}`.
- `visibleQuestions()` respeita `showIf` (condicionais: P2b/P2c/P8b/P10).
- Respostas em `state.answers` (single=valor, multi=array, foto='sim'/'nao').
- `roteia(flow, answers)` → bucket de desfecho. `finish()` decide loading+redirect (Fluxo 2 Cat2/3) ou `renderEnd()` (resto).

### Roteamento — Fluxo 2 (rn, carro-chefe)
`categoriaRN()`: peito+fórmula 5-8/8-10 → **cat3**; só fórmula + quer voltar → **cat3**; só fórmula + não quer → **cat1** (sono); só peito → `bebeBemRN()` ? **cat1** : **cat2**.
Buckets: `rn_cat2_guia`, `rn_cat2_silicone` (Guia+Pesque Baby), `rn_cat3_acomp` (Guia+Acompanhamento), `rn_cat1_sono`, `rn_cat1_dormebem` (Entre Sonecas), `rn_formula_sono`.

### Motor de faixas (`montaFaixas`, seção 4 do doc)
4 classes, 1 faixa por classe (a de maior peso marcada), reforço preenche até ~4, exibição na ordem das classes. **Validado contra os 4 exemplos da Day (bate 100%).**
```
C1 extração : pouco_leite > dor > empedrado
C2 esforço  : calo > pouca_fralda
C3 bico     : silicone > raynaud > batom
C4 peso     : peso_abaixo   (só quando peso abaixo — ver pendência)
```
Gatilhos: dor←P6 doendo/rachadura/sangrando · empedrado←P6 empedrados · calo←P7=sim · pouca_fralda←P4<6 · silicone←P6 silicone · raynaud←P6 raynaud · batom←P8=sim · peso_abaixo←P5 abaixo.
⚠️ **pouco_leite (1A) não tem caixinha própria** — gatilho provisório: P9=produzir_leite OU P3=horas_peito/larga_quer. **A validar com a Day.**

## Página de resposta (`resultado.html`) — formato página de vendas
Lê a receita da URL e monta seções full-width (fundos alternados, reveal-on-scroll, barra CTA fixa):
```
HERO (foto Day + diagnóstico) → FAIXAS modulares (1 seção cada, cores alternadas)
→ VIRADA (seção verde) → MÉTODO (3 cards) → AUTORIDADE (Day+stats)
→ O QUE VAI APRENDER (checklist) → OFERTA (preço+CTA) → GARANTIA → FAQ → CTA FINAL
```
Params: `cat` (cat2|cat3), `oferta` (guia|guia_pesque|guia_acomp), `enfase` (P9), `faixas` (lista por vírgula).

**Copy real da Day** (do doc): aberturas, todas as faixas, virada, junção, textos dos CTAs.
**Rascunho meu a validar com a Day** (a estrutura de vendas pediu conteúdo fora do doc): headline do hero, seção Método, autoridade+stats ("+600 mil" veio do IG; resto genérico), checklist "o que vai aprender", garantia 7 dias, FAQ.

## Deploy (GitHub Pages)
```bash
cd /d/CLAUDE/quiz-amamentacao && source /d/CLAUDE/.env.meta
git add -A && git -c user.email="contato@brunotropolis.com.br" -c user.name="brunotropolis" commit -q -m "..." && git push -q origin main
# purge cache Cloudflare (zona manualdo):
curl -s -X POST "https://api.cloudflare.com/client/v4/zones/$CLOUDFLARE_ZONE_MANUALDORECEMNASCIDO/purge_cache" \
  -H "Authorization: Bearer $CLOUDFLARE_CACHE_TOKEN_MANUALDO" -H "Content-Type: application/json" --data '{"purge_everything":true}'
```
- ⚠️ **Build do Pages é LENTO nesse repo** (~7-10 min preso em "building" na fila). Destrava com um **commit vazio de nudge** (`git commit --allow-empty -m "nudge"; git push`) + `POST /repos/.../pages/builds`.
- Validar publicação batendo no **origin do GitHub over HTTP** (bypassa Cloudflare/TLS): `curl --resolve quizama.manualdorecemnascido.com.br:80:185.199.108.153 http://quizama.manualdorecemnascido.com.br/...`.
- Sempre **purgar o cache** do Cloudflare depois, senão o domínio serve a versão antiga.

## Preview local
```
preview_start "quiz-amamentacao"   →   http://localhost:3355
```
⚠️ Nessa máquina o alias `python` é o fake do WindowsApps (abre a Store). O preview usa **`node server.js`** (`C:/Program Files/nodejs/node.exe`), configurado em `D:\CLAUDE\.claude\launch.json`. `screenshot` do preview às vezes trava — o `snapshot`/`eval` continuam funcionando; reiniciar o server destrava.

## Pendências
```
[ ] Validar com a Day: gatilho da faixa "pouco leite" (1A); faixa "bom ganho + dificuldades"
    (peso_bom_dif) está DESLIGADA por ora (senão apareceria em quase todo Cat 2, quebrando os
    exemplos dela) — copy pronta em resultado.html, falta o gatilho certo.
[ ] Passar as faixas do Fluxo 2 pra 3ª pessoa geral (regra 3 do doc) — feito COM a Day (copy dela).
[ ] Revisar scaffolding de vendas com a Day (headline hero, autoridade/stats, garantia, FAQ).
[ ] Checkout real da Greenn nos CTAs (hoje é placeholder/alert).
[ ] Preço do Pesque Baby (doc: "a definir") — no combo silicone hoje mostra só "R$67".
[ ] Imagens reais: ilustrações das faixas (calo/batom são foto), foto da Day na autoridade.
[ ] Construir as páginas dos outros desfechos: "é sono" (compartilhada fluxos 2/3/4),
    "tudo ótimo/Entre Sonecas", e replicar o modelo pros fluxos 3, 4 e 1.
[ ] Pixel Meta / CAPI + UTM passthrough quiz→checkout (quando entrar tráfego pago).
```

## Decisões do Bruno (travadas)
- Quiz **100% self-contained** — NÃO encosta no quiz de sono/perfis (esse é de amamentação, lógica e páginas próprias). Onde o doc diz "redireciona pro quiz de sono", vira página própria deste quiz.
- P1 idade = as 4 opções (gestante / 0-30d / 1-5m / 6-12m).
- Resposta = **página de vendas de verdade** (formato perfis), modular, alcançada por loading+redirect — não boxes dentro do quiz.
- Verde pinho + rosa blush confirmados.
