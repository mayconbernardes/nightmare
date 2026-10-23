# NIGHTMARE — O Edifício dos Condenados

Jogo de terror first-person em um único HTML, feito com Three.js. O objetivo é explorar um edifício procedural, restaurar a energia e escapar — enquanto uma criatura e um diretor de terror controlam o ritmo dos sustos.

## Como rodar

O jogo carrega modelos e texturas locais, então **não abra apenas com `file://`**. Use um servidor HTTP na pasta do projeto:

```bash
python3 -m http.server 8000
```

Depois abra <http://localhost:8000>. Também funciona com `npx serve`, Live Server do VS Code etc.

Requisitos: navegador moderno com WebGL e Web Audio (Chrome/Edge/Firefox/Safari atuais; iOS Safari 13+). Three.js, os loaders OBJ/FBX, o `fflate` e os addons de pós-processamento (`EffectComposer`, passes e shaders) vêm por CDN, então a primeira execução precisa de internet. Sem os addons, o jogo ainda roda sem pós-processamento. Use fones de ouvido. O áudio só começa após o clique em **Iniciar** (política de autoplay dos navegadores). Em iPhone, o jogo entra em tela cheia quando possível; se o navegador bloquear, a interface continua utilizável.

## Controles

### PC
- **WASD / setas** — mover
- **Mouse** — olhar (clique para capturar o ponteiro)
- **Shift** — correr; dentro do armário, segurar a respiração
- **C / Ctrl** — agachar
- **F / clique esquerdo** — ligar/desligar a lanterna
- **E** — interagir (fusíveis, selos, armários, portas, garrafas, saída)
- **Q** — arremessar garrafa para distrair o monstro
- **M** — mostrar/ocultar mapa
- **Esc** — pausar (libera o ponteiro)

### Mobile
Os controles de toque aparecem automaticamente em telas com ponteiro grosso: direcional à esquerda, olhar à direita e botões de luz, agachar, correr, jogar garrafa, pegar e pausa.

## Objetivos

### Fase 1 — Edifício
1. Encontre os **3 fusíveis** (brilho laranja no mapa).
2. Leve-os ao painel do **Gerador** e restaure a energia.
3. Fuja pela **porta verde** até a saída.

### Fase 2 — Subsolo
1. Encontre os **4 selos violeta**.
2. Ative o **portal**.
3. Sobreviva à perseguição e escape.

## Sistemas principais

- **Pós-processamento**: pipeline com `EffectComposer` (Three r128) — RenderPass, `UnrealBloom` discreto nas luzes quentes e um `ShaderPass` final com *grade* (lift/gain/saturação), vinheta adaptativa (mais fechada no escuro e sob medo), grão fino e leve aberração cromática radial que cresce com o medo. Se os addons não carregarem da CDN, cai automaticamente para `renderer.render` puro.
- **Performance adaptativa (não altera o jogo)**: o *pixel ratio* é limitado a 1.5; o **passe de sombra só roda quando a lanterna está acesa** (sem lanterna/blackout ele é pulado inteiro); a poeira do feixe anima a ~30 Hz e só quando visível; e o FPS é medido a cada ~2 s para **baixar ou subir** a qualidade em degraus (bloom on/off, resolução). Em último caso, desliga a sombra da lanterna (que você pode religar no menu). No console, `?debug` expõe `__nm.fps`, `__nm.fxQuality`, `__nm.pixelRatio` e `__nm.setQuality('high'|'med'|'low')` para testar.
- **Lanterna**: bateria finita (~4:20), spot principal com sombra + spill de refletor, viewmodel compacto, feixe volumétrico com partículas finas, inércia de mão, clique do interruptor e queda de tensão/temperatura de cor quando a carga cai. Ligar/desligar dá um *kick* visível no viewmodel e um flash de tela; apontar a luz direto no monstro gera flash ciano, tremor e mensagem "Ele recuou da luz!" — isso funciona já na fase 1 (atordoa, faz recuar ou dispersa), não só na fase 2.
- **Áudio procedural**: ambiente em camadas, passos por superfície (concreto, metal, madeira, azulejo, molhado), respiração, batimentos, sussurros e gritos. Tudo é gerado via Web Audio, sem arquivos de som.
- **Diretor de terror**: alterna calma, sinal, silêncio, susto e alívio. A intensidade cresce com o progresso e o medo. Há limite de sustos visuais consecutivos, válvula de segurança e blackout não punitivo.
- **IA do monstro (justa, dá tempo de escapar)**: patrulha, investiga ruídos, persegue, procura o último local visto e recua quando recebe luz. A visão tem cone e distância; agachar no escuro ajuda. O monstro acorda sozinho após ~10 s (primeira patrulha) e ao pegar o primeiro fusível. A navegação usa **A\*** em grid (heurística octile, sem cortar quinas), **suavização de caminho** e *steering* que desliza por paredes; se travar, reconecta na célula aberta mais próxima. Na perseguição, **flanqueia** ao perder de vista por pouco tempo.
  - **Não ataca "colado"**: ao te ver, primeiro entra numa fase de aproximação (`hunt`, ~3 s) e só depois parte para cima. Nos **primeiros ~22 s** de jogo ele nunca lava direto — apenas ronda.
  - **Mais lento que você correndo**: perseguição a ~3.15 (fase 1) / 3.5 (fase 2) contra sua corrida a 5.2 — dá para escapar sprintando, mas andar (3.1) é arriscado.
  - **Ataque telegrafado**: o bote tem *wind-up* de ~0,6 s (ele se agacha e avança devagar); se você sair do alcance, ele erra, fica atordoado e recua antes de tentar de novo.
- **Animação do monstro**: o FBX do corpo não tem clipes, então as poses são geradas por código como "clipes" (idle/walk/run/attack/crouch) e misturadas por peso com crossfade exponencial em cada bone, sem "pop". Por cima, um **movimento de dança retargetado** (`Rumba Dancing.fbx`, esqueleto Mixamo) dirige tronco, quadris, braços e pernas: o clipe é amostrado por um `AnimationMixer` e a *delta* de cada osso Mixamo é aplicada sobre o repouso do osso correspondente do corpo, preservando as proporções.
- **Áudio espacial com oclusão**: além do panning HRTF, os sons contam paredes entre a fonte e o jogador e são abafados por um lowpass conforme os obstáculos; o *wet* do reverb cresce com paredes e com a abertura da sala, dando a sensação de som vindo de outro cômodo. Uma camada de perseguição (cordas dissonantes + pulso) entra rápido durante a caçada e sai devagar.
- **Sangue e pegadas visíveis**: poças maiores e mais brilhantes, com pelo menos 5 manchas garantidas perto do ponto de partida e trilhas de pegadas cruzando o início.
- **Ruído e garrafas**: correr, andar, abrir portas e arremessar garrafas geram ruído espacial. O monstro investiga o ponto de impacto e pode ser desviado durante uma perseguição.
- **Portas**: alguns corredores têm portas fechadas. O jogador abre com **E**; o monstro também as abre ao passar. Elas bloqueiam o jogador até serem abertas.
- **Sangue e pegadas**: poças secas e trilhas de pegadas antigas espalham-se pelo prédio; o monstro deixa pegadas novas enquanto se move.
- **Armários**: esconder-se reduz o risco, mas a lanterna acesa e a respiração presa por tempo demais podem denunciar você. Se ele viu você entrar, continuará procurando o armário e pode abri-lo.
- **Medo compartilhado**: alimenta respiração, batimentos, tremor de câmera e o próprio diretor.

## Mixer e acessibilidade

No menu:

- Volume geral, Efeitos (SFX), Ambiente e Interface (buses separados)
- Sensibilidade do mouse
- Tremor de câmera e efeito bodycam
- Volume da respiração
- Sombras (qualidade) e visibilidade do mapa

## Assets opcionais

O jogo já funciona sem arquivos de áudio externos. Se quiser trocar o som procedural por gravações reais, coloque os arquivos na pasta e adapte o carregamento:

- `ambient_loop.ogg` — drone/ambiente contínuo
- `monster_breath.ogg` — respiração da criatura
- `monster_step_*.ogg` — passos por superfície
- `whisper_*.ogg` — sussurros
- `scream.ogg` — grito
- `heartbeat.ogg` — batimentos

Formatos recomendados: `.ogg` ou `.mp3`, mono para sons espaciais. Enquanto não houver arquivos, o áudio procedural cobre todos os eventos.

## Assets do monstro

- `Meshs/Zombie_Schoolgirl_01.fbx` — modelo rigado (198 bones, 2 SkinnedMeshes), usado como padrão
- `Zombie_Schoolgirl_01.obj` / `zombie.obj` — fallback estático se o FBX falhar
- Texturas `zombie_base.jpg`, `zombie_normal.jpg`, `zombie_rough.jpg`, `zombie_hair*.jpg`
- `Meshs/` também contém OBJ/MTL originais
- `Blender/Custom Rigg/Zombie_Schoolgirl_01.blend` contém o rig original

O FBX não traz clipes de animação. O jogo reconstrói a hierarquia dos bones por nome e aplica animação procedural de caminhada/corrida, tronco, cabeça, mandíbula e braços. Se o FBX ou o loader falharem, o OBJ é carregado e o monstro volta ao sway/lean/bob antigo.

No Three.js r128 `Texture` não possui `userData`; as texturas UDIM do FBX guardam os clones em um `WeakMap` próprio para aplicar o atlas 2x2 sem quebrar o rig. Se a tela mostrar o modelo como OBJ (sem animação), verifique o console: deve aparecer `Zombie FBX rig OK — 198 bones`.

## Desenvolvimento

Abra com `?debug` para expor `window.__nm` no console (estado do diretor, monstro, player, IA, depuração de render e funções de teste). Disponível para validação sem esperar o ciclo completo: `__nm.rigWeights` (pesos de blend da animação), `__nm.aStar/smoothPath/follow/nearestOpen`, `__nm.bloodPools/footprints`, `__nm.reactLight`, `__nm.audio`, além das funções de estado (`startPatrol`, `startChase`, `beginLunge`, `restorePower`, `advanceToPhase2`, `win`, `caught`) e dos parâmetros de IA (`__nm.AI`: velocidades, *grace* e *wind-up*). Rótulo `BUILD 7 · PERFORMANCE` no HUD confirma qual versão o navegador carregou (se não aparecer, faça *hard refresh*).

## Limitações conhecidas

- O FBX do monstro não possui clipes; a animação é um sistema de poses compostas com crossfade (não usa `AnimationMixer`). Em caso de falha, o fallback OBJ fica estático com sway/lean.
- Sem modelos `.glb`/asset de props nem SFX externos: gráficos e áudio continuam procedurais, agora com pós-processamento e áudio espacial ocluído.
- iOS antigo (antes do iOS 13) pode não suportar alguns efeitos de pós-processamento com boa performance; a qualidade adaptativa reduz efeitos automaticamente.
- Em iOS antigo (antes do iOS 13), não há fallback completo de Pointer Events para o pointer lock; o alvo suportado é iOS 13+.
- O áudio é 100% procedural até serem adicionados arquivos reais.
- O labirinto é gerado a cada partida; a progressão é a mesma, mas o layout muda.
