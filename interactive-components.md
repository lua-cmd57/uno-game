# Componentes Interativos — DotField, StaggeredMenu, TargetCursor

Construídos e testados em sessão. Prontos para uso em React + Vite + TS ou vanilla.
Paleta: segue as regras do SKILL.md. Cores nos exemplos são customizáveis via props/CSS vars.

---

## 01 — DotField (Canvas Interativo com bulge/repulsão)

Fundo animado de pontos em canvas que reage ao movimento do cursor.
Dois modos: `bulgeOnly` (pontos fogem do cursor) ou repulsão física com velocidade.

### Props

| Prop           | Default                        | Descrição                                  |
|----------------|--------------------------------|--------------------------------------------|
| dotRadius      | 1.5                            | Raio de cada ponto (px)                    |
| dotSpacing     | 14                             | Espaçamento entre pontos (px)              |
| cursorRadius   | 500                            | Raio de influência do cursor (px)          |
| cursorForce    | 0.1                            | Força de repulsão (modo não-bulge)         |
| bulgeOnly      | true                           | true = bulge suave, false = repulsão física|
| bulgeStrength  | 67                             | Intensidade do bulge                       |
| sparkle        | false                          | Pontos piscam aleatoriamente               |
| waveAmplitude  | 0                              | Amplitude da onda passiva (0 = desligado)  |
| gradientFrom   | 'rgba(168,85,247,0.35)'        | Cor início do gradiente dos pontos         |
| gradientTo     | 'rgba(180,151,207,0.25)'       | Cor fim do gradiente dos pontos            |

### Uso no B&W preset

```jsx
// Versão editorial B&W
<div style={{ position: 'relative', width: '100%', height: '600px' }}>
  <DotField
    dotRadius={1.5}
    dotSpacing={14}
    bulgeStrength={67}
    bulgeOnly
    gradientFrom="rgba(10,10,10,0.5)"
    gradientTo="rgba(10,10,10,0.15)"
  />
</div>
```

### DotField.jsx (self-contained, sem CSS externo)

```jsx
import { useEffect, useRef, memo } from 'react';
const TWO_PI = Math.PI * 2;

const DotField = memo(({
  dotRadius = 1.5, dotSpacing = 14, cursorRadius = 500, cursorForce = 0.1,
  bulgeOnly = true, bulgeStrength = 67, sparkle = false, waveAmplitude = 0,
  gradientFrom = 'rgba(168,85,247,0.35)', gradientTo = 'rgba(180,151,207,0.25)',
  ...rest
}) => {
  const canvasRef = useRef(null);
  const dotsRef   = useRef([]);
  const mouseRef  = useRef({ x:-9999, y:-9999, prevX:-9999, prevY:-9999, speed:0 });
  const rafRef    = useRef(null);
  const sizeRef   = useRef({ w:0, h:0, offsetX:0, offsetY:0 });
  const engagement = useRef(0);
  const propsRef  = useRef({});
  propsRef.current = { dotRadius,dotSpacing,cursorRadius,cursorForce,bulgeOnly,bulgeStrength,sparkle,waveAmplitude,gradientFrom,gradientTo };
  const rebuildRef = useRef(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d', { alpha: true });
    const dpr = Math.min(window.devicePixelRatio || 1, 2);
    let resizeTimer;

    function doResize() {
      if (!canvas.parentElement) return;
      const rect = canvas.parentElement.getBoundingClientRect();
      const w = rect.width, h = rect.height;
      canvas.width = w*dpr; canvas.height = h*dpr;
      canvas.style.width = `${w}px`; canvas.style.height = `${h}px`;
      ctx.setTransform(dpr,0,0,dpr,0,0);
      sizeRef.current = { w, h, offsetX: rect.left+window.scrollX, offsetY: rect.top+window.scrollY };
      buildDots(w, h);
    }

    function buildDots(w, h) {
      const p = propsRef.current;
      const step = p.dotRadius + p.dotSpacing;
      const cols = Math.floor(w/step), rows = Math.floor(h/step);
      const padX = (w%step)/2, padY = (h%step)/2;
      const dots = new Array(rows*cols);
      let idx = 0;
      for (let row=0; row<rows; row++)
        for (let col=0; col<cols; col++) {
          const ax = padX+col*step+step/2, ay = padY+row*step+step/2;
          dots[idx++] = { ax, ay, sx:ax, sy:ay, vx:0, vy:0, x:ax, y:ay };
        }
      dotsRef.current = dots;
    }

    function onMouseMove(e) {
      const s = sizeRef.current;
      mouseRef.current.x = e.pageX - s.offsetX;
      mouseRef.current.y = e.pageY - s.offsetY;
    }

    const speedInterval = setInterval(() => {
      const m = mouseRef.current;
      const dist = Math.hypot(m.prevX-m.x, m.prevY-m.y);
      m.speed += (dist - m.speed) * 0.5;
      if (m.speed < 0.001) m.speed = 0;
      m.prevX = m.x; m.prevY = m.y;
    }, 20);

    let frameCount = 0;
    function tick() {
      frameCount++;
      const dots = dotsRef.current, m = mouseRef.current;
      const { w, h } = sizeRef.current, p = propsRef.current;
      const len = dots.length, t = frameCount * 0.02;
      const targetEng = Math.min(m.speed/5, 1);
      engagement.current += (targetEng - engagement.current) * 0.06;
      if (engagement.current < 0.001) engagement.current = 0;
      const eng = engagement.current;

      ctx.clearRect(0,0,w,h);
      const grad = ctx.createLinearGradient(0,0,w,h);
      grad.addColorStop(0, p.gradientFrom); grad.addColorStop(1, p.gradientTo);
      ctx.fillStyle = grad;

      const crSq = p.cursorRadius**2, rad = p.dotRadius/2, isBulge = p.bulgeOnly;
      ctx.beginPath();
      for (let i=0; i<len; i++) {
        const d = dots[i];
        const dx = m.x-d.ax, dy = m.y-d.ay, distSq = dx*dx+dy*dy;
        if (distSq < crSq && eng > 0.01) {
          const dist = Math.sqrt(distSq);
          if (isBulge) {
            const tv = 1-dist/p.cursorRadius;
            const push = tv*tv*p.bulgeStrength*eng;
            const angle = Math.atan2(dy,dx);
            d.sx += (d.ax - Math.cos(angle)*push - d.sx)*0.15;
            d.sy += (d.ay - Math.sin(angle)*push - d.sy)*0.15;
          } else {
            const angle = Math.atan2(dy,dx);
            const move = (500/dist)*(m.speed*p.cursorForce);
            d.vx += Math.cos(angle)*-move; d.vy += Math.sin(angle)*-move;
          }
        } else if (isBulge) {
          d.sx += (d.ax-d.sx)*0.1; d.sy += (d.ay-d.sy)*0.1;
        }
        if (!isBulge) {
          d.vx*=0.9; d.vy*=0.9;
          d.x=d.ax+d.vx; d.y=d.ay+d.vy;
          d.sx+=(d.x-d.sx)*0.1; d.sy+=(d.y-d.sy)*0.1;
        }
        let drawX=d.sx, drawY=d.sy;
        if (p.waveAmplitude>0) {
          drawY += Math.sin(d.ax*0.03+t)*p.waveAmplitude;
          drawX += Math.cos(d.ay*0.03+t*0.7)*p.waveAmplitude*0.5;
        }
        if (p.sparkle) {
          const hash = ((i*2654435761)^(frameCount>>3))>>>0;
          const r2 = (hash%100)<3 ? rad*1.8 : rad;
          ctx.moveTo(drawX+r2,drawY); ctx.arc(drawX,drawY,r2,0,TWO_PI);
        } else {
          ctx.moveTo(drawX+rad,drawY); ctx.arc(drawX,drawY,rad,0,TWO_PI);
        }
      }
      ctx.fill();
      rafRef.current = requestAnimationFrame(tick);
    }

    setTimeout(doResize, 0);
    window.addEventListener('resize', () => { clearTimeout(resizeTimer); resizeTimer=setTimeout(doResize,100); });
    window.addEventListener('mousemove', onMouseMove, { passive:true });
    rafRef.current = requestAnimationFrame(tick);
    rebuildRef.current = () => { const {w,h}=sizeRef.current; if(w>0&&h>0) buildDots(w,h); };

    return () => {
      cancelAnimationFrame(rafRef.current);
      clearInterval(speedInterval);
      window.removeEventListener('mousemove', onMouseMove);
    };
  }, []);

  useEffect(() => { rebuildRef.current?.(); }, [dotRadius, dotSpacing]);

  return (
    <div style={{ position:'absolute', top:0, left:0, width:'100%', height:'100%', overflow:'hidden', zIndex:0 }} {...rest}>
      <canvas ref={canvasRef} style={{ position:'absolute', inset:0, width:'100%', height:'100%' }} />
    </div>
  );
});

DotField.displayName = 'DotField';
export default DotField;
```

### Variações de preset

```js
// Pontos brancos sobre fundo escuro (B&W)
gradientFrom="rgba(250,250,250,0.55)"
gradientTo="rgba(250,250,250,0.12)"

// Onda passiva suave (sem cursor)
waveAmplitude={3} bulgeStrength={0}

// Repulsão física (não-bulge)
bulgeOnly={false} cursorForce={0.15} cursorRadius={300}

// Sparkle mode
sparkle={true} dotRadius={1.2} dotSpacing={12}
```

---

## 02 — StaggeredMenu (Menu fullscreen com GSAP stagger)

Menu lateral com pre-layers animados, items com stagger reveal e social links.
Requer: `gsap` instalado (`npm i gsap`).

### Props

| Prop                  | Default          | Descrição                                       |
|-----------------------|------------------|-------------------------------------------------|
| position              | 'right'          | 'left' ou 'right'                               |
| colors                | ['#B497CF','#5227FF'] | Array de cores dos pre-layers              |
| items                 | []               | `[{ label, link, ariaLabel }]`                  |
| socialItems           | []               | `[{ label, link }]`                             |
| displaySocials        | true             | Mostra seção de socials                         |
| displayItemNumbering  | true             | Numeração 01, 02... nos items                   |
| logoUrl               | '/logo.svg'      | URL do logo no header                           |
| menuButtonColor       | '#fff'           | Cor do botão Menu fechado                       |
| openMenuButtonColor   | '#fff'           | Cor do botão Menu aberto                        |
| accentColor           | '#5227FF'        | CSS var `--sm-accent` nos links                 |
| changeMenuColorOnOpen | true             | Muda cor do botão ao abrir                      |
| isFixed               | false            | `position: fixed` no wrapper                    |
| closeOnClickAway      | true             | Fecha ao clicar fora                            |
| onMenuOpen            | undefined        | Callback ao abrir                               |
| onMenuClose           | undefined        | Callback ao fechar                              |

### Uso

```jsx
import { StaggeredMenu } from './StaggeredMenu';

<StaggeredMenu
  colors={['#0a0a0a', '#1a1a1a']}
  accentColor="#fafafa"
  menuButtonColor="#fafafa"
  items={[
    { label: 'Work',    link: '/work'    },
    { label: 'About',   link: '/about'   },
    { label: 'Journal', link: '/journal' },
    { label: 'Contact', link: '/contact' },
  ]}
  socialItems={[
    { label: 'Twitter', link: 'https://twitter.com/...' },
    { label: 'GitHub',  link: 'https://github.com/...'  },
  ]}
/>
```

### StaggeredMenu.css

```css
/* Wrapper */
.staggered-menu-wrapper { position: relative; width: 100%; }
.staggered-menu-wrapper.fixed-wrapper { position: fixed; top: 0; left: 0; right: 0; z-index: 1000; }

/* Header */
.staggered-menu-header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 20px 32px; position: relative; z-index: 50;
}
.sm-logo-img { display: block; }

/* Toggle button */
.sm-toggle {
  display: flex; align-items: center; gap: 10px;
  background: none; border: none; cursor: pointer;
  font-family: inherit; outline: none;
}
.sm-toggle-textWrap { height: 1em; overflow: hidden; }
.sm-toggle-textInner { display: flex; flex-direction: column; }
.sm-toggle-line {
  font-size: 11px; font-weight: 600; letter-spacing: .18em;
  text-transform: uppercase; height: 1em; display: flex;
  align-items: center; white-space: nowrap; color: inherit;
}

/* Icon +/× */
.sm-icon {
  position: relative; width: 20px; height: 20px;
  display: flex; align-items: center; justify-content: center;
}
.sm-icon-line {
  position: absolute; width: 16px; height: 1.5px;
  background: currentColor; border-radius: 2px; transform-origin: center;
}
.sm-icon-line-v { transform: rotate(90deg); }

/* Pre-layers */
.sm-prelayers { position: fixed; inset: 0; pointer-events: none; z-index: 90; }
.sm-prelayer  { position: absolute; inset: 0; }

/* Panel */
.staggered-menu-panel {
  position: fixed; inset: 0; z-index: 95; overflow-y: auto;
  display: flex; flex-direction: column;
  padding: 100px 60px 48px;
  pointer-events: none;
}
.staggered-menu-panel[aria-hidden="false"] { pointer-events: all; }

/* Items */
.sm-panel-inner { display: flex; flex-direction: column; flex: 1; }
.sm-panel-list  { list-style: none; flex: 1; display: flex; flex-direction: column; justify-content: center; gap: 4px; }
.sm-panel-itemWrap { overflow: hidden; padding: 2px 0; }
.sm-panel-item {
  display: flex; align-items: center; gap: 16px;
  text-decoration: none; cursor: pointer;
}
.sm-panel-item[data-index]::before {
  content: '0' attr(data-index);
  font-size: 11px; font-weight: 500; letter-spacing: .1em;
  opacity: var(--sm-num-opacity, 0);
  color: rgba(0,0,0,.35); min-width: 22px;
}
.sm-panel-list:not([data-numbering]) .sm-panel-item[data-index]::before { display: none; }
.sm-panel-itemLabel {
  font-size: clamp(2.8rem, 6vw, 4.5rem);
  line-height: .92; color: #05050f;
  transform-origin: left bottom; display: block;
  will-change: transform;
}
.sm-panel-item:hover .sm-panel-itemLabel { color: var(--sm-accent, #5227FF); }

/* Socials */
.sm-socials {
  border-top: 1px solid rgba(0,0,0,.1); padding-top: 20px;
  display: flex; align-items: center; gap: 24px; margin-top: 24px;
}
.sm-socials-title {
  font-size: 9px; font-weight: 700; letter-spacing: .22em;
  text-transform: uppercase; color: rgba(0,0,0,.4);
}
.sm-socials-list { list-style: none; display: flex; gap: 16px; }
.sm-socials-link {
  font-size: 11px; font-weight: 600; letter-spacing: .12em;
  text-transform: uppercase; color: #05050f; text-decoration: none;
}
.sm-socials-link:hover { opacity: .55; }

/* Position variant */
[data-position="left"] .sm-prelayers,
[data-position="left"] .staggered-menu-panel { left: 0; right: auto; width: min(480px, 100vw); }
[data-position="right"] .sm-prelayers,
[data-position="right"] .staggered-menu-panel { right: 0; left: auto; width: min(480px, 100vw); }
```

### Lógica da animação (resumo GSAP)

```
OPEN:
  pre-layers → panel: xPercent 100→0 com stagger 70ms entre layers
  itemLabels: yPercent 140→0, rotate 10→0, stagger 100ms
  números:    --sm-num-opacity 0→1, stagger 80ms
  socials:    opacity 0→1, y 25→0, stagger 70ms

CLOSE:
  todos → xPercent 0→100, duration 320ms, power3.in
  reset states (yPercent, opacity) no onComplete

ICON: rotate 0→225° (open), 225→0° (close)
TEXT: yPercent cycling entre linhas (Menu/Close) com power4.out
```

---

## 03 — TargetCursor (Cursor com corners que travam no elemento alvo)

Cursor customizado com 4 corners que se expandem para cobrir o elemento `.cursor-target` ao hover,
com efeito parallax interno e spin contínuo quando livre.
Requer: `gsap` instalado.

### Props

| Prop               | Default          | Descrição                                          |
|--------------------|------------------|----------------------------------------------------|
| targetSelector     | '.cursor-target' | CSS selector dos elementos que ativam o snap       |
| spinDuration       | 2                | Duração de uma volta completa (segundos)           |
| hideDefaultCursor  | true             | Esconde o cursor nativo                            |
| hoverDuration      | 0.2              | Velocidade de snap nos corners (segundos)          |
| parallaxOn         | true             | Corners seguem o mouse dentro do target            |
| cursorColor        | '#ffffff'        | Cor base dos corners e dot                         |
| cursorColorOnTarget| undefined        | Cor dos corners ao entrar no target                |

### Uso

```jsx
import TargetCursor from './TargetCursor';

// No root da app ou do container
<TargetCursor
  targetSelector=".cursor-target"
  cursorColor="#0a0a0a"
  cursorColorOnTarget="#0a0a0a"
  spinDuration={3}
  parallaxOn={true}
/>

// Nos elementos interativos
<button className="cursor-target">Clica aqui</button>
<a href="/work" className="cursor-target">Work</a>
```

### TargetCursor.css

```css
.target-cursor-wrapper {
  position: fixed;
  top: 0; left: 0;
  width: 0; height: 0;
  pointer-events: none;
  z-index: 9999;
  /* xPercent/yPercent: -50% aplicado via GSAP */
}

.target-cursor-dot {
  position: absolute;
  width: 6px; height: 6px;
  border-radius: 50%;
  top: -3px; left: -3px;
  will-change: transform;
}

.target-cursor-corner {
  position: absolute;
  width: 12px; height: 12px;
  border-style: solid;
  border-width: 0;
  will-change: transform;
}

/* Cada corner: borda só nos dois lados corretos */
.corner-tl { border-top-width: 3px; border-left-width:  3px; top: -18px; left: -18px; }
.corner-tr { border-top-width: 3px; border-right-width: 3px; top: -18px; left:   6px; }
.corner-br { border-bottom-width: 3px; border-right-width: 3px; top: 6px; left:  6px; }
.corner-bl { border-bottom-width: 3px; border-left-width:  3px; top: 6px; left: -18px; }
```

### Como funciona (lógica core)

```
ESTADO LIVRE:
  → cursor segue mouse (gsap.to x/y, duration 0.1, power3.out)
  → wrapper gira continuamente (spinTimeline repeat:-1)
  → corners ficam nas posições padrão (-18px / +6px)

AO ENTRAR NO TARGET (.cursor-target via mouseover):
  → spin pausa, rotation vai para 0
  → calcula 4 posições dos corners baseado no getBoundingClientRect() do target
  → gsap.ticker adiciona tickerFn que interpola corners para as posições do target
  → activeStrengthRef 0→1 (controla força do snap)
  → parallaxOn: corners continuam se movendo levemente com o mouse dentro

AO SAIR DO TARGET (mouseleave):
  → ticker remove, corners voltam para posição padrão (power3.out)
  → 50ms depois spin retoma do ângulo atual (sem salto)

MOUSEDOWN: dot e wrapper escalam para 0.7 / 0.9
MOUSEUP:   voltam para 1

MOBILE: retorna null (detectado por touch + screen width)
```

### Problema do containing block

```js
// ⚠️ position:fixed é relativo ao viewport MAS se um ancestral tiver
// transform/filter/perspective, ele vira o containing block.
// O componente detecta isso e compensa:

const getContainingBlock = (element) => {
  let node = element?.parentElement;
  while (node && node !== document.documentElement) {
    const style = getComputedStyle(node);
    if (style.transform !== 'none' || style.filter !== 'none' || ...) {
      return node; // encontrou o bloco
    }
    node = node.parentElement;
  }
  return null; // sem containing block, usa viewport
};
// Offset é subtraído das coordenadas do mouse
```

---

## Combinações recomendadas

```
DotField + StaggeredMenu  → hero com fundo interativo + nav fullscreen
DotField + TargetCursor   → landing page com cursor snapping nos CTAs
StaggeredMenu + TargetCursor → portfolio completo com cursor customizado nos nav items
Todos os 3               → experiência imersiva nível Awwwards
```

---

## Notas de performance

```js
// DotField: GPU safe pois só usa ctx.fill() + transform
// StaggeredMenu: GSAP só anima transform + opacity (sem layout thrashing)
// TargetCursor: gsap.ticker é mais eficiente que rAF manual para múltiplos tweens

// ⚠️ TargetCursor: não usar em containers com `transform` sem testar o offset fix
// ⚠️ DotField: dotSpacing < 8 com dotRadius > 2 = muitos dots = janks em mobile
// ⚠️ StaggeredMenu: sempre testar `closeOnClickAway` com `isFixed=true`
```

---

## 04 — GooeyNav (Navegação com partículas gooey)

Navbar com efeito de blob líquido entre os itens e partículas coloridas ao trocar de aba.
Puro React + CSS. Zero dependências além do React.
Inspiration: React Bits. Funciona em dark e light bg — apenas ajusta `mix-blend-mode`.

### Props

| Prop                | Type         | Default              | Descrição                                              |
|---------------------|--------------|----------------------|--------------------------------------------------------|
| items               | `{ label, href }[]` | `[]`        | Array de itens da nav                                  |
| animationTime       | number       | 600                  | Duração (ms) da animação principal                     |
| particleCount       | number       | 15                   | Quantidade de bolhas por transição                     |
| particleDistances   | [n, n]       | [90, 10]             | Distância externa e interna do spread das partículas   |
| particleR           | number       | 100                  | Fator de raio na rotação aleatória das partículas      |
| timeVariance        | number       | 300                  | Variação aleatória (ms) nas animações de partícula     |
| colors              | number[]     | [1,2,3,1,2,3,1,4]   | Índices de cor (`--color-1` a `--color-4`) das bolhas  |
| initialActiveIndex  | number       | 0                    | Item ativo no mount                                    |
| onNavigate          | (i) => void  | undefined            | Callback com index ao clicar                           |

### CSS vars de cor das partículas

```css
/* No B&W preset, use tons monocromáticos para as bolinhas */
:root {
  --color-1: #fafafa;   /* white  */
  --color-2: #b8b4ad;   /* ghost  */
  --color-3: #6a6a6a;   /* dust   */
  --color-4: #2a2a2a;   /* smoke  */
}

/* Versão colorida (fora do B&W) */
:root {
  --color-1: #4ade80;
  --color-2: #60a5fa;
  --color-3: #e879f9;
  --color-4: #fb923c;
}
```

### Uso

```jsx
import GooeyNav from './GooeyNav';

const items = [
  { label: 'Home',    href: '/' },
  { label: 'Work',    href: '/work' },
  { label: 'Contact', href: '/contact' },
];

// Navegação simples
<div style={{ position: 'relative', height: 60 }}>
  <GooeyNav items={items} />
</div>

// Com controle de página e customização
<GooeyNav
  items={items}
  particleCount={12}
  particleDistances={[80, 8]}
  particleR={80}
  animationTime={500}
  timeVariance={200}
  colors={[1, 2, 3, 4, 1, 2]}
  initialActiveIndex={0}
  onNavigate={(index) => setPage(index)}
/>
```

### GooeyNav.jsx (self-contained, CSS embutido via prop ou arquivo externo)

```jsx
import { useRef, useEffect, useState } from 'react';
// Importe o GooeyNav.css separado ou injete o CSS via <style> tag

const GooeyNav = ({
  items,
  animationTime = 600,
  particleCount = 15,
  particleDistances = [90, 10],
  particleR = 100,
  timeVariance = 300,
  colors = [1, 2, 3, 1, 2, 3, 1, 4],
  initialActiveIndex = 0,
  onNavigate,
}) => {
  const containerRef = useRef(null);
  const navRef       = useRef(null);
  const filterRef    = useRef(null);
  const textRef      = useRef(null);
  const [activeIndex, setActiveIndex] = useState(initialActiveIndex);

  const noise  = (n = 1) => n / 2 - Math.random() * n;

  const getXY = (distance, pointIndex, totalPoints) => {
    const angle = ((360 + noise(8)) / totalPoints) * pointIndex * (Math.PI / 180);
    return [distance * Math.cos(angle), distance * Math.sin(angle)];
  };

  const createParticle = (i, t, d, r) => {
    const rotate = noise(r / 10);
    return {
      start: getXY(d[0], particleCount - i, particleCount),
      end:   getXY(d[1] + noise(7), particleCount - i, particleCount),
      time:  t,
      scale: 1 + noise(0.2),
      color: colors[Math.floor(Math.random() * colors.length)],
      rotate: rotate > 0 ? (rotate + r / 20) * 10 : (rotate - r / 20) * 10,
    };
  };

  const makeParticles = (element) => {
    const d = particleDistances, r = particleR;
    element.style.setProperty('--time', `${animationTime * 2 + timeVariance}ms`);
    for (let i = 0; i < particleCount; i++) {
      const t = animationTime * 2 + noise(timeVariance * 2);
      const p = createParticle(i, t, d, r);
      element.classList.remove('active');
      setTimeout(() => {
        const particle = document.createElement('span');
        const point    = document.createElement('span');
        particle.classList.add('particle');
        particle.style.setProperty('--start-x', `${p.start[0]}px`);
        particle.style.setProperty('--start-y', `${p.start[1]}px`);
        particle.style.setProperty('--end-x',   `${p.end[0]}px`);
        particle.style.setProperty('--end-y',   `${p.end[1]}px`);
        particle.style.setProperty('--time',    `${p.time}ms`);
        particle.style.setProperty('--scale',   `${p.scale}`);
        particle.style.setProperty('--color',   `var(--color-${p.color}, white)`);
        particle.style.setProperty('--rotate',  `${p.rotate}deg`);
        point.classList.add('point');
        particle.appendChild(point);
        element.appendChild(particle);
        requestAnimationFrame(() => element.classList.add('active'));
        setTimeout(() => { try { element.removeChild(particle); } catch {} }, t);
      }, 30);
    }
  };

  const updateEffectPosition = (element) => {
    if (!containerRef.current || !filterRef.current || !textRef.current) return;
    const cr  = containerRef.current.getBoundingClientRect();
    const pos = element.getBoundingClientRect();
    const s   = { left: `${pos.x - cr.x}px`, top: `${pos.y - cr.y}px`, width: `${pos.width}px`, height: `${pos.height}px` };
    Object.assign(filterRef.current.style, s);
    Object.assign(textRef.current.style, s);
    textRef.current.innerText = element.innerText;
  };

  const handleClick = (e, index) => {
    if (activeIndex === index) return;
    setActiveIndex(index);
    onNavigate?.(index);
    updateEffectPosition(e.currentTarget);
    filterRef.current?.querySelectorAll('.particle').forEach(p => filterRef.current.removeChild(p));
    if (textRef.current) {
      textRef.current.classList.remove('active');
      void textRef.current.offsetWidth; // force reflow
      textRef.current.classList.add('active');
    }
    if (filterRef.current) makeParticles(filterRef.current);
  };

  useEffect(() => {
    if (!navRef.current || !containerRef.current) return;
    const activeLi = navRef.current.querySelectorAll('li')[activeIndex];
    if (activeLi) {
      updateEffectPosition(activeLi);
      textRef.current?.classList.add('active');
    }
    const ro = new ResizeObserver(() => {
      const cur = navRef.current?.querySelectorAll('li')[activeIndex];
      if (cur) updateEffectPosition(cur);
    });
    ro.observe(containerRef.current);
    return () => ro.disconnect();
  }, [activeIndex]);

  return (
    <div className="gooey-nav-container" ref={containerRef}>
      <nav>
        <ul ref={navRef}>
          {items.map((item, index) => (
            <li
              key={index}
              className={activeIndex === index ? 'active' : ''}
              onClick={(e) => handleClick(e, index)}
            >
              <a href={item.href || '#'} onClick={e => e.preventDefault()}>
                {item.label}
              </a>
            </li>
          ))}
        </ul>
      </nav>
      <span className="effect filter" ref={filterRef} />
      <span className="effect text"   ref={textRef}   />
    </div>
  );
};

export default GooeyNav;
```

### GooeyNav.css

```css
:root {
  --linear-ease: linear(
    0, 0.068, 0.19 2.7%, 0.804 8.1%, 1.037, 1.199 13.2%, 1.245,
    1.27 15.8%, 1.274, 1.272 17.4%, 1.249 19.1%, 0.996 28%,
    0.949, 0.928 33.3%, 0.926, 0.933 36.8%, 1.001 45.6%,
    1.013, 1.019 50.8%, 1.018 54.4%, 1 63.1%,
    0.995 68%, 1.001 85%, 1
  );
}

.gooey-nav-container { position: relative; }
.gooey-nav-container nav { display: flex; position: relative; transform: translate3d(0,0,0.01px); }
.gooey-nav-container nav ul {
  display: flex; gap: 2em; list-style: none; padding: 0 1em; margin: 0;
  position: relative; z-index: 3; color: white;
}
.gooey-nav-container nav ul li {
  border-radius: 100vw; position: relative; cursor: pointer;
  transition: background-color 0.3s ease, color 0.3s ease, box-shadow 0.3s ease;
  box-shadow: 0 0 0.5px 1.5px transparent; color: white;
}
.gooey-nav-container nav ul li a {
  display: inline-block; padding: 0.6em 1em;
  text-decoration: none; color: inherit;
}
.gooey-nav-container nav ul li::after {
  content: ''; position: absolute; inset: 0;
  border-radius: 10px; background: white;
  opacity: 0; transform: scale(0);
  transition: all 0.3s ease; z-index: -1;
}
.gooey-nav-container nav ul li.active        { color: black; text-shadow: none; }
.gooey-nav-container nav ul li.active::after { opacity: 1; transform: scale(1); }

.gooey-nav-container .effect {
  position: absolute; left: 0; top: 0; width: 0; height: 0;
  pointer-events: none; display: grid; place-items: center; z-index: 1;
}
.gooey-nav-container .effect.text   { color: white; transition: color 0.3s ease; }
.gooey-nav-container .effect.text.active { color: black; }
.gooey-nav-container .effect.filter {
  filter: blur(7px) contrast(100) blur(0);
  mix-blend-mode: lighten;
}
.gooey-nav-container .effect.filter::before {
  content: ''; position: absolute; inset: -75px; z-index: -2; background: black;
}
.gooey-nav-container .effect.filter::after {
  content: ''; position: absolute; inset: 0;
  background: white; transform: scale(0); opacity: 0;
  z-index: -1; border-radius: 100vw;
}
.gooey-nav-container .effect.active::after { animation: pill 0.3s ease both; }
@keyframes pill { to { transform: scale(1); opacity: 1; } }

.particle, .point { display: block; opacity: 0; width: 20px; height: 20px; border-radius: 100%; transform-origin: center; }
.particle {
  --time: 5s;
  position: absolute;
  top: calc(50% - 8px); left: calc(50% - 8px);
  animation: particle calc(var(--time)) ease 1 -350ms;
}
.point { background: var(--color); opacity: 1; animation: point calc(var(--time)) ease 1 -350ms; }

@keyframes particle {
  0%   { transform: rotate(0deg) translate(var(--start-x), var(--start-y)); opacity: 1; animation-timing-function: cubic-bezier(0.55,0,1,0.45); }
  70%  { transform: rotate(calc(var(--rotate)*.5)) translate(calc(var(--end-x)*1.2),calc(var(--end-y)*1.2)); opacity: 1; animation-timing-function: ease; }
  85%  { transform: rotate(calc(var(--rotate)*.66)) translate(var(--end-x),var(--end-y)); opacity: 1; }
  100% { transform: rotate(calc(var(--rotate)*1.2)) translate(calc(var(--end-x)*.5),calc(var(--end-y)*.5)); opacity: 1; }
}
@keyframes point {
  0%  { transform: scale(0); opacity: 0; animation-timing-function: cubic-bezier(0.55,0,1,0.45); }
  25% { transform: scale(calc(var(--scale)*.25)); }
  38% { opacity: 1; }
  65% { transform: scale(var(--scale)); opacity: 1; animation-timing-function: ease; }
  85% { transform: scale(var(--scale)); opacity: 1; }
  100%{ transform: scale(0); opacity: 0; }
}
```

### Integração no B&W preset

```jsx
// Header com GooeyNav sobre fundo escuro
<header style={{ background: 'var(--ink)', padding: '20px 40px', display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
  <span style={{ fontFamily: 'Syne', fontWeight: 900, color: 'var(--white)', letterSpacing: '-0.03em' }}>LOGO</span>
  <GooeyNav
    items={[
      { label: 'Work',    href: '/work'    },
      { label: 'About',   href: '/about'   },
      { label: 'Journal', href: '/journal' },
    ]}
    particleCount={10}
    colors={[1, 2, 3, 4]}
    onNavigate={(i) => console.log('Navegou para', i)}
  />
</header>
```

### Notas de uso

```
⚠️  O container pai PRECISA ter position: relative
⚠️  mix-blend-mode: lighten → funciona só sobre fundos escuros (#000 / var(--ink))
    Para fundo claro: trocar para mix-blend-mode: multiply + inverter cores (blob preto)
⚠️  Não usar dentro de containers com overflow: hidden — mata as partículas
⚠️  ResizeObserver já cuida do reposicionamento no resize
✅  onNavigate retorna o index (number) — ideal pra page switching sem router
✅  href no <a> é opcional (use '#' se for SPA com onNavigate)
✅  particleCount acima de 20 pode causar leve jank em mobile
```

---

## Combinações recomendadas

```
DotField + StaggeredMenu  → hero com fundo interativo + nav fullscreen
DotField + TargetCursor   → landing page com cursor snapping nos CTAs
StaggeredMenu + TargetCursor → portfolio completo com cursor customizado nos nav items
GooeyNav + TargetCursor   → header com blob nav + cursor corners nos links
GooeyNav + DotField       → hero page com nav líquida sobre fundo de pontos
Todos os 4               → experiência imersiva nível Awwwards
```

---

## Notas de performance

```js
// DotField: GPU safe pois só usa ctx.fill() + transform
// StaggeredMenu: GSAP só anima transform + opacity (sem layout thrashing)
// TargetCursor: gsap.ticker é mais eficiente que rAF manual para múltiplos tweens
// GooeyNav: partículas são spans reais no DOM, removidos via setTimeout após animação

// ⚠️ TargetCursor: não usar em containers com `transform` sem testar o offset fix
// ⚠️ DotField: dotSpacing < 8 com dotRadius > 2 = muitos dots = janks em mobile
// ⚠️ StaggeredMenu: sempre testar `closeOnClickAway` com `isFixed=true`
// ⚠️ GooeyNav: particleCount > 20 em mobile = leve jank — prefira 8-12 em mobile
```

---

## ElasticTab (ExclusiveTab — Lightswind Pro clone)

Segmented control com blob elástico que estica até o destino e encolhe com spring. Duas fases: stretch rápido → squish com cubic-bezier overshoot. Zero hover de fundo, zero flash no init.

### Como usar

```html
<!-- HTML: wrapper + blob + botões -->
<div class="tab-wrapper" id="my-tab">
  <div class="elastic-blob" id="my-blob"></div>
  <button class="tab-btn active">Tab A</button>
  <button class="tab-btn">Tab B</button>
  <button class="tab-btn">Tab C</button>
</div>
```

```js
// JS: initElasticTab(wrapperId, blobId, onChangeFn?)
initElasticTab('my-tab', 'my-blob', btn => {
  console.log('ativo:', btn.textContent);
});
```

### CSS — base + variantes

```css
/* ── BASE ── */
.tab-wrapper {
  display: inline-flex;
  position: relative;
  padding: 5px;
  border-radius: 999px;
  background: transparent; /* SEM fundo no default — blob é o único bg */
}

.elastic-blob {
  position: absolute;
  top: 5px;
  left: 0;
  height: calc(100% - 10px);
  width: 0;
  border-radius: 999px;
  background: var(--pill); /* #e63946 por padrão */
  pointer-events: none;
  z-index: 0;
  opacity: 0; /* invisível até JS posicionar */
  will-change: left, width;
}
.elastic-blob.ready { opacity: 1; }

.tab-btn {
  position: relative;
  z-index: 2;
  font-family: 'Syne', sans-serif;
  font-size: 0.82rem;
  font-weight: 700;
  letter-spacing: 0.03em;
  padding: 0.55rem 1.35rem;
  border: none;
  background: transparent !important; /* SEMPRE transparent — nunca sobrescrever */
  color: var(--muted);
  border-radius: 999px;
  cursor: pointer;
  transition: color 0.2s ease;
  white-space: nowrap;
  user-select: none;
  -webkit-tap-highlight-color: transparent;
  outline: none;
}
.tab-btn:hover,
.tab-btn:focus,
.tab-btn:active { background: transparent !important; }
.tab-btn.active  { color: var(--white); }

/* ── VARIANTE: dark ── */
.tab-wrapper.dark                 { background: var(--ink); }
.tab-wrapper.dark .tab-btn        { color: rgba(250,250,250,0.38); }
.tab-wrapper.dark .tab-btn.active { color: var(--ink); }
.tab-wrapper.dark .elastic-blob   { background: var(--white); }

/* ── VARIANTE: outline ── */
.tab-wrapper.outline {
  background: transparent;
  border: 1.5px solid rgba(10,10,10,0.14);
}
.tab-wrapper.outline .tab-btn        { color: var(--muted); }
.tab-wrapper.outline .tab-btn.active { color: var(--white); }
.tab-wrapper.outline .elastic-blob   { background: var(--ink); }

/* ── VARIANTE: green ── */
.tab-wrapper.green                 { background: var(--accent); } /* #1a6b3a */
.tab-wrapper.green .tab-btn        { color: rgba(255,255,255,0.42); }
.tab-wrapper.green .tab-btn.active { color: var(--accent); }
.tab-wrapper.green .elastic-blob   { background: var(--white); }
```

### JS — engine completa

```js
function initElasticTab(wrapperId, blobId, onChangeFn) {
  const wrapper = document.getElementById(wrapperId);
  const blob    = document.getElementById(blobId);
  const btns    = Array.from(wrapper.querySelectorAll('.tab-btn'));
  let current   = wrapper.querySelector('.tab-btn.active');
  let busy      = false;
  let timer     = null;

  function metrics(btn) {
    const wr = wrapper.getBoundingClientRect();
    const br = btn.getBoundingClientRect();
    return { left: br.left - wr.left - 5, width: br.width };
  }

  function place(btn, animate) {
    const { left, width } = metrics(btn);
    if (!animate) blob.style.transition = 'none';
    blob.style.left  = left  + 'px';
    blob.style.width = width + 'px';
  }

  function activate(btn) {
    if (btn === current || busy) return;

    const from  = metrics(current);
    const to    = metrics(btn);
    const right = to.left > from.left;

    // troca classe imediatamente — texto muda de cor na hora
    btns.forEach(b => b.classList.remove('active'));
    btn.classList.add('active');
    current = btn;
    busy = true;

    // fase 1 — estica cobrindo os dois
    const sl = right ? from.left : to.left;
    const sw = right
      ? (to.left + to.width - from.left)
      : (from.left + from.width - to.left);

    blob.style.transition = 'left .18s ease-in, width .18s ease-in';
    blob.style.left  = sl + 'px';
    blob.style.width = sw + 'px';

    // fase 2 — encolhe no destino com spring overshoot
    clearTimeout(timer);
    timer = setTimeout(() => {
      blob.style.transition =
        'left .35s cubic-bezier(.34,1.6,.64,1), width .35s cubic-bezier(.34,1.6,.64,1)';
      blob.style.left  = to.left  + 'px';
      blob.style.width = to.width + 'px';
      setTimeout(() => { busy = false; }, 360);
    }, 155);

    if (onChangeFn) onChangeFn(btn);
  }

  // delegação no wrapper — captura clique em qualquer filho do botão
  wrapper.addEventListener('click', e => {
    const btn = e.target.closest('.tab-btn');
    if (btn) activate(btn);
  });

  // init: duplo rAF garante layout computado antes de tornar visível
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      place(current, false);
      blob.getBoundingClientRect(); // força reflow
      blob.classList.add('ready');
    });
  });
}

// Resize handler — reposiciona sem animação
window.addEventListener('resize', () => {
  // ex: [['wrapper-id', 'blob-id'], ...]
  [['my-tab', 'my-blob']].forEach(([wId, bId]) => {
    const w = document.getElementById(wId);
    const b = document.getElementById(bId);
    const a = w?.querySelector('.tab-btn.active');
    if (!a || !b) return;
    const wr = w.getBoundingClientRect();
    const ar = a.getBoundingClientRect();
    b.style.transition = 'none';
    b.style.left  = (ar.left - wr.left - 5) + 'px';
    b.style.width = a.offsetWidth + 'px';
  });
});
```

### Variantes disponíveis

```
Default   → sem classe extra   | blob: var(--pill)  | texto ativo: branco
Dark      → .dark              | blob: branco       | texto ativo: preto  | bg: var(--ink)
Outline   → .outline           | blob: var(--ink)   | texto ativo: branco | border: 1.5px
Green     → .green             | blob: branco       | texto ativo: verde  | bg: var(--accent)
```

### Notas críticas

```
⚠️  wrapper PRECISA ter position: relative (já garantido pelo display:inline-flex + padding)
⚠️  blob começa opacity:0 — só aparece após duplo rAF (zero flash de posicionamento)
⚠️  background: transparent !important no .tab-btn é obrigatório — browsers adicionam :hover bg
⚠️  delegação de evento no wrapper (não em cada btn) — funciona com elementos filhos no botão
✅  onChangeFn recebe o elemento <button> ativo — use btn.dataset.panel pra trocar painéis
✅  fase 1 (stretch): 155ms ease-in | fase 2 (squish): 350ms cubic-bezier(.34,1.6,.64,1)
✅  funciona em qualquer direção: direita, esquerda, salto de posições
✅  múltiplas instâncias na mesma página — cada initElasticTab é isolada
```

---

## 05 — InteractiveGridBackground (Grade de linhas reativa com rastro dinâmico)

Grade interativa em canvas que reage ao movimento do cursor do mouse, criando rastros com decaimento suave por tempo. Quando inativa por mais de 2 segundos, inicia uma animação autônoma de partículas flutuando.

### Props

| Prop | Type | Default | Descrição |
|---|---|---|---|
| gridSize | number | 50 | Tamanho de cada célula da grade (px) |
| gridColor | string | '#cbcbcb' | Cor da grade no modo claro |
| darkGridColor | string | '#303030' | Cor da grade no modo escuro |
| effectColor | string | 'rgba(45, 45, 48, 0.6)' | Cor do rastro no modo claro (formato rgba) |
| darkEffectColor | string | 'rgba(161, 161, 170, 0.6)'| Cor do rastro no modo escuro (formato rgba) |
| trailLength | number | 3 | Multiplicador de comprimento do rastro |
| width | number | undefined | Largura customizada do canvas (opcional) |
| height | number | undefined | Altura customizada do canvas (opcional) |
| idleSpeed | number | 0.2 | Velocidade das partículas autônomas no estado inativo |
| glow | boolean | true | Se verdadeiro, adiciona efeito de brilho neon |
| glowRadius | number | 20 | Intensidade do raio de brilho neon |
| showFade | boolean | true | Ativa vinheta de fade radial nas bordas |
| fadeIntensity | number | 20 | Porcentagem inicial de transparência do fade radial |
| idleRandomCount | number | 5 | Número de células aleatórias que se movem no estado inativo |

### Uso

```tsx
import InteractiveGridBackground from './components/InteractiveGridBackground';

<div style={{ position: 'relative', width: '100%', height: '100vh' }}>
  <InteractiveGridBackground
    gridSize={50}
    trailLength={4}
    glow={true}
    effectColor="rgba(99, 102, 241, 0.6)"
  />
</div>
```

### InteractiveGridBackground.tsx

```tsx
"use client";

import React, { useEffect, useRef, useState } from "react";

export interface InteractiveGridBackgroundProps
  extends React.HTMLProps<HTMLDivElement> {
  gridSize?: number;
  gridColor?: string;
  darkGridColor?: string;
  effectColor?: string;
  darkEffectColor?: string;
  trailLength?: number;
  width?: number;
  height?: number;
  idleSpeed?: number;
  glow?: boolean;
  glowRadius?: number;
  children?: React.ReactNode;
  showFade?: boolean;
  fadeIntensity?: number;
  idleRandomCount?: number; // ✅ how many random cells move during idle
}

const InteractiveGridBackground: React.FC<InteractiveGridBackgroundProps> = ({
  gridSize = 50,
  gridColor = "#cbcbcb",
  darkGridColor = "#303030",
  effectColor = "rgba(45, 45, 48, 0.6)",
  darkEffectColor = "rgba(161, 161, 170, 0.6)",
  trailLength = 3,
  width,
  height,
  idleSpeed = 0.2,
  glow = true,
  glowRadius = 20,
  children,
  showFade = true,
  fadeIntensity = 20,
  idleRandomCount = 5,
  className,
  ...props
}) => {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const containerRef = useRef<HTMLDivElement>(null);
  const [isDarkMode, setIsDarkMode] = useState(false);

  const lineColor = isDarkMode ? darkGridColor : gridColor;

  const trailRef = useRef<{ x: number; y: number; alpha: number }[]>([]);
  const idleTargetsRef = useRef<{ x: number; y: number }[]>([]);
  const idlePositionsRef = useRef<{ x: number; y: number }[]>([]);
  const mouseActiveRef = useRef(false);
  const lastMouseTimeRef = useRef(Date.now());

  // Detect dark mode
  useEffect(() => {
    const updateDarkMode = () => {
      setIsDarkMode(document.documentElement.classList.contains("dark"));
    };
    updateDarkMode();
    const observer = new MutationObserver(() => updateDarkMode());
    observer.observe(document.documentElement, { attributes: true });
    return () => observer.disconnect();
  }, []);

  // Mouse tracking
  useEffect(() => {
    let rect: DOMRect | null = null;
    const container = containerRef.current;

    const updateRect = () => {
      if (container) {
        rect = container.getBoundingClientRect();
      }
    };

    // Initialize rect
    updateRect();

    // Update rect on resize or scroll
    window.addEventListener("resize", updateRect);
    window.addEventListener("scroll", updateRect, { passive: true });

    const handleMouseMove = (e: MouseEvent) => {
      if (!container) return;
      if (!rect) {
        rect = container.getBoundingClientRect();
      }

      const rawX = e.clientX - rect.left;
      const rawY = e.clientY - rect.top;

      if (rawX < 0 || rawY < 0 || rawX > rect.width || rawY > rect.height)
        return;

      mouseActiveRef.current = true;
      lastMouseTimeRef.current = Date.now();

      const snappedX = Math.floor(rawX / gridSize);
      const snappedY = Math.floor(rawY / gridSize);

      const last = trailRef.current[0];
      if (!last || last.x !== snappedX || last.y !== snappedY) {
        trailRef.current.unshift({ x: snappedX, y: snappedY, alpha: 1.0 });
        if (trailRef.current.length > 50) trailRef.current.pop();
      } else {
        last.alpha = 1.0;
      }
    };

    window.addEventListener("mousemove", handleMouseMove);
    return () => {
      window.removeEventListener("resize", updateRect);
      window.removeEventListener("scroll", updateRect);
      window.removeEventListener("mousemove", handleMouseMove);
    };
  }, [gridSize]);

  // Drawing logic
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    const canvasWidth = width || window.innerWidth;
    const canvasHeight = height || window.innerHeight;
    canvas.width = canvasWidth;
    canvas.height = canvasHeight;

    const cols = Math.floor(canvasWidth / gridSize);
    const rows = Math.floor(canvasHeight / gridSize);

    const glowColor = isDarkMode ? darkEffectColor : effectColor;

    // Initialize idle positions if they don't exist yet
    if (idleTargetsRef.current.length === 0) {
      idleTargetsRef.current = Array.from({ length: idleRandomCount }, () => ({
        x: Math.floor(Math.random() * cols),
        y: Math.floor(Math.random() * rows),
      }));
      idlePositionsRef.current = idleTargetsRef.current.map((p) => ({ ...p }));
    }

    const draw = () => {
      ctx.clearRect(0, 0, canvasWidth, canvasHeight);

      // Decay trail cells based on trailLength
      const decayRate = 1 / (trailLength * 8);
      trailRef.current.forEach((cell) => {
        cell.alpha -= decayRate;
      });
      // Filter out dead cells
      trailRef.current = trailRef.current.filter((cell) => cell.alpha > 0);

      // Idle animation logic
      const idleThreshold = 2000;
      if (Date.now() - lastMouseTimeRef.current > idleThreshold) {
        mouseActiveRef.current = false;

        idlePositionsRef.current.forEach((pos, i) => {
          const target = idleTargetsRef.current[i];
          if (!target) return;
          const dx = target.x - pos.x;
          const dy = target.y - pos.y;

          if (Math.abs(dx) < 0.01 && Math.abs(dy) < 0.01) {
            // new random target when reached
            idleTargetsRef.current[i] = {
              x: Math.floor(Math.random() * cols),
              y: Math.floor(Math.random() * rows),
            };
          } else {
            pos.x += dx * idleSpeed;
            pos.y += dy * idleSpeed;
          }

          const roundedX = Math.round(pos.x);
          const roundedY = Math.round(pos.y);
          
          const existing = trailRef.current.find(c => c.x === roundedX && c.y === roundedY);
          if (existing) {
            existing.alpha = Math.min(1.0, existing.alpha + 0.1);
          } else {
            trailRef.current.unshift({ x: roundedX, y: roundedY, alpha: 0.8 });
          }
        });
      }

      // Draw trail glow
      trailRef.current.forEach((cell) => {
        const clampedAlpha = Math.max(0, Math.min(1, cell.alpha));
        const rgbaColor = glowColor.replace(/[\d.]+\)$/g, `${clampedAlpha.toFixed(3)})`);

        ctx.fillStyle = rgbaColor;
        if (glow) {
          ctx.shadowColor = rgbaColor;
          ctx.shadowBlur = glowRadius;
        } else {
          ctx.shadowBlur = 0;
        }

        ctx.fillRect(cell.x * gridSize, cell.y * gridSize, gridSize, gridSize);
      });

      animationFrameId = requestAnimationFrame(draw);
    };

    let animationFrameId = requestAnimationFrame(draw);
    return () => cancelAnimationFrame(animationFrameId);
  }, [
    gridSize,
    width,
    height,
    gridColor,
    darkGridColor,
    effectColor,
    darkEffectColor,
    isDarkMode,
    trailLength,
    idleSpeed,
    glow,
    glowRadius,
    idleRandomCount,
  ]);

  return (
    <div
      ref={containerRef}
      className={`relative ${className}`}
      style={{ width: width || "100%", height: height || "100%" }}
      {...props}
    >
      <canvas
        ref={canvasRef}
        className="absolute top-0 left-0 z-0 pointer-events-none"
        style={{
          backgroundImage: `linear-gradient(to right, ${lineColor} 1px, transparent 1px), linear-gradient(to bottom, ${lineColor} 1px, transparent 1px)`,
          backgroundSize: `${gridSize}px ${gridSize}px`,
        }}
      />

      {showFade && (
        <div
          className="pointer-events-none absolute inset-0 bg-white dark:bg-black transition-colors duration-300"
          style={{
            maskImage: `radial-gradient(ellipse at center, transparent ${fadeIntensity}%, black)`,
            WebkitMaskImage: `radial-gradient(ellipse at center, transparent ${fadeIntensity}%, black)`,
          }}
        />
      )}
      <div className="relative z-10 w-full h-full">{children}</div>
    </div>
  );
};

export default InteractiveGridBackground;
```

---

## 06 — MagicRings (Anéis concêntricos WebGL Shader com Three.js)

Anéis expansivos e orgânicos renderizados via shader personalizado (WebGL) usando Three.js. Reage ao hover e cliques do mouse com deformação e explosões.

### Props

| Prop | Type | Default | Descrição |
|---|---|---|---|
| color | string | '#fc42ff' | Cor inicial (gradiente interno dos anéis) |
| colorTwo | string | '#42fcff' | Cor secundária (gradiente externo dos anéis) |
| speed | number | 1.0 | Fator de velocidade da expansão e oscilações |
| ringCount | number | 6 | Quantidade de anéis (máximo 10) |
| attenuation | number | 10 | Taxa de atenuação/suavidade da borda do anel |
| lineThickness | number | 2 | Espessura das linhas que formam o anel |
| baseRadius | number | 0.35 | Raio interno inicial do primeiro anel |
| radiusStep | number | 0.1 | Espaçamento de raio adicionado a cada camada subsequente |
| scaleRate | number | 0.1 | Fator de crescimento e oscilação |
| opacity | number | 1.0 | Opacidade geral da camada shader |
| blur | number | 0 | Filtro de desfoque (CSS blur em px) aplicado no contêiner |
| noiseAmount | number | 0.1 | Fator de ruído granulado analógico |
| rotation | number | 0 | Ângulo de rotação inicial em graus |
| ringGap | number | 1.5 | Fator de gap geométrico entre os anéis |
| fadeIn | number | 0.7 | Limiar de fade-in gradual da animação |
| fadeOut | number | 0.5 | Limiar de fade-out gradual da animação |
| followMouse | boolean | false | Ativa o deslocamento central seguindo o mouse (parallax) |
| mouseInfluence | number | 0.2 | Intensidade de influência da posição do mouse no centro |
| hoverScale | number | 1.2 | Escala multiplicadora ao passar o mouse por cima |
| parallax | number | 0.05 | Intensidade de paralaxe de profundidade entre as camadas de anéis |
| clickBurst | boolean | false | Se ativo, cliques criam ondas rápidas de explosão de anel |

### Uso

```tsx
import MagicRings from './components/MagicRings';

<div style={{ position: 'relative', width: '100%', height: '100vh' }}>
  <MagicRings
    color="#fc42ff"
    colorTwo="#42fcff"
    speed={1.0}
    followMouse={true}
    clickBurst={true}
  />
</div>
```

### MagicRings.tsx

```tsx
import { useEffect, useRef } from 'react';
import * as THREE from 'three';

import './MagicRings.css';

const vertexShader = `
void main() {
  gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}
`;

const fragmentShader = `
precision highp float;

uniform float uTime, uAttenuation, uLineThickness;
uniform float uBaseRadius, uRadiusStep, uScaleRate;
uniform float uOpacity, uNoiseAmount, uRotation, uRingGap;
uniform float uFadeIn, uFadeOut;
uniform float uMouseInfluence, uHoverAmount, uHoverScale, uParallax, uBurst;
uniform vec2 uResolution, uMouse;
uniform vec3 uColor, uColorTwo;
uniform int uRingCount;

const float HP = 1.5707963;
const float CYCLE = 3.45;

float fade(float t) {
  return t < uFadeIn ? smoothstep(0.0, uFadeIn, t) : 1.0 - smoothstep(uFadeOut, CYCLE - 0.2, t);
}

float ring(vec2 p, float ri, float cut, float t0, float px) {
  float t = mod(uTime + t0, CYCLE);
  float r = ri + t / CYCLE * uScaleRate;
  float d = abs(length(p) - r);
  float a = atan(abs(p.y), abs(p.x)) / HP;
  float th = max(1.0 - a, 0.5) * px * uLineThickness;
  float h = (1.0 - smoothstep(th, th * 1.5, d)) + 1.0;
  d += pow(cut * a, 3.0) * r;
  return h * exp(-uAttenuation * d) * fade(t);
}

void main() {
  float px = 1.0 / min(uResolution.x, uResolution.y);
  vec2 p = (gl_FragCoord.xy - 0.5 * uResolution.xy) * px;
  float cr = cos(uRotation), sr = sin(uRotation);
  p = mat2(cr, -sr, sr, cr) * p;
  p -= uMouse * uMouseInfluence;
  float sc = mix(1.0, uHoverScale, uHoverAmount) + uBurst * 0.3;
  p /= sc;
  vec3 c = vec3(0.0);
  float rcf = max(float(uRingCount) - 1.0, 1.0);
  for (int i = 0; i < 10; i++) {
    if (i >= uRingCount) break;
    float fi = float(i);
    vec2 pr = p - fi * uParallax * uMouse;
    vec3 rc = mix(uColor, uColorTwo, fi / rcf);
    c = mix(c, rc, vec3(ring(pr, uBaseRadius + fi * uRadiusStep, pow(uRingGap, fi), i == 0 ? 0.0 : 2.95 * fi, px)));
  }
  c *= 1.0 + uBurst * 2.0;
  float n = fract(sin(dot(gl_FragCoord.xy + uTime * 100.0, vec2(12.9898, 78.233))) * 43758.5453);
  c += (n - 0.5) * uNoiseAmount;
  gl_FragColor = vec4(c, max(c.r, max(c.g, c.b)) * uOpacity);
}
`;

export interface MagicRingsProps {
  color?: string;
  colorTwo?: string;
  speed?: number;
  ringCount?: number;
  attenuation?: number;
  lineThickness?: number;
  baseRadius?: number;
  radiusStep?: number;
  scaleRate?: number;
  opacity?: number;
  blur?: number;
  noiseAmount?: number;
  rotation?: number;
  ringGap?: number;
  fadeIn?: number;
  fadeOut?: number;
  followMouse?: boolean;
  mouseInfluence?: number;
  hoverScale?: number;
  parallax?: number;
  clickBurst?: boolean;
}

export default function MagicRings({
  color = '#fc42ff',
  colorTwo = '#42fcff',
  speed = 1,
  ringCount = 6,
  attenuation = 10,
  lineThickness = 2,
  baseRadius = 0.35,
  radiusStep = 0.1,
  scaleRate = 0.1,
  opacity = 1,
  blur = 0,
  noiseAmount = 0.1,
  rotation = 0,
  ringGap = 1.5,
  fadeIn = 0.7,
  fadeOut = 0.5,
  followMouse = false,
  mouseInfluence = 0.2,
  hoverScale = 1.2,
  parallax = 0.05,
  clickBurst = false,
}: MagicRingsProps) {
  const mountRef = useRef<HTMLDivElement>(null);
  const propsRef = useRef<Required<MagicRingsProps> | null>(null);
  const mouseRef = useRef<[number, number]>([0, 0]);
  const smoothMouseRef = useRef<[number, number]>([0, 0]);
  const hoverAmountRef = useRef<number>(0);
  const isHoveredRef = useRef<boolean>(false);
  const burstRef = useRef<number>(0);

  propsRef.current = {
    color, colorTwo, speed, ringCount, attenuation, lineThickness,
    baseRadius, radiusStep, scaleRate, opacity, blur, noiseAmount,
    rotation, ringGap, fadeIn, fadeOut, followMouse, mouseInfluence,
    hoverScale, parallax, clickBurst,
  };

  useEffect(() => {
    const mount = mountRef.current;
    if (!mount) return;

    let renderer: THREE.WebGLRenderer;
    try {
      renderer = new THREE.WebGLRenderer({ alpha: true });
    } catch {
      return;
    }

    if (!renderer.capabilities.isWebGL2) {
      renderer.dispose();
      return;
    }

    renderer.setClearColor(0x000000, 0);
    mount.appendChild(renderer.domElement);

    const scene = new THREE.Scene();
    const camera = new THREE.OrthographicCamera(-0.5, 0.5, 0.5, -0.5, 0.1, 10);
    camera.position.z = 1;

    const uniforms = {
      uTime: { value: 0 },
      uAttenuation: { value: 0 },
      uResolution: { value: new THREE.Vector2() },
      uColor: { value: new THREE.Color() },
      uColorTwo: { value: new THREE.Color() },
      uLineThickness: { value: 0 },
      uBaseRadius: { value: 0 },
      uRadiusStep: { value: 0 },
      uScaleRate: { value: 0 },
      uRingCount: { value: 0 },
      uOpacity: { value: 1 },
      uNoiseAmount: { value: 0 },
      uRotation: { value: 0 },
      uRingGap: { value: 1.6 },
      uFadeIn: { value: 0.5 },
      uFadeOut: { value: 0.75 },
      uMouse: { value: new THREE.Vector2() },
      uMouseInfluence: { value: 0 },
      uHoverAmount: { value: 0 },
      uHoverScale: { value: 1 },
      uParallax: { value: 0 },
      uBurst: { value: 0 },
    };

    const material = new THREE.ShaderMaterial({ vertexShader, fragmentShader, uniforms, transparent: true });
    const quad = new THREE.Mesh(new THREE.PlaneGeometry(1, 1), material);
    scene.add(quad);

    const resize = () => {
      const w = mount.clientWidth;
      const h = mount.clientHeight;
      const dpr = Math.min(window.devicePixelRatio, 2);
      renderer.setSize(w, h);
      renderer.setPixelRatio(dpr);
      uniforms.uResolution.value.set(w * dpr, h * dpr);
    };
    resize();
    window.addEventListener('resize', resize);

    const ro = new ResizeObserver(() => resize());
    ro.observe(mount);

    const onMouseMove = (e: MouseEvent) => {
      const rect = mount.getBoundingClientRect();
      mouseRef.current[0] = (e.clientX - rect.left) / rect.width - 0.5;
      mouseRef.current[1] = -((e.clientY - rect.top) / rect.height - 0.5);
    };
    const onMouseEnter = () => { isHoveredRef.current = true; };
    const onMouseLeave = () => {
      isHoveredRef.current = false;
      mouseRef.current[0] = 0;
      mouseRef.current[1] = 0;
    };
    const onClick = () => { burstRef.current = 1; };

    mount.addEventListener('mousemove', onMouseMove);
    mount.addEventListener('mouseenter', onMouseEnter);
    mount.addEventListener('mouseleave', onMouseLeave);
    mount.addEventListener('click', onClick);

    let frameId: number;
    const animate = (t: number) => {
      frameId = requestAnimationFrame(animate);
      const p = propsRef.current;
      if (!p) return;

      smoothMouseRef.current[0] += (mouseRef.current[0] - smoothMouseRef.current[0]) * 0.08;
      smoothMouseRef.current[1] += (mouseRef.current[1] - smoothMouseRef.current[1]) * 0.08;
      hoverAmountRef.current += ((isHoveredRef.current ? 1 : 0) - hoverAmountRef.current) * 0.08;
      burstRef.current *= 0.95;
      if (burstRef.current < 0.001) burstRef.current = 0;

      uniforms.uTime.value = t * 0.001 * p.speed;
      uniforms.uAttenuation.value = p.attenuation;
      uniforms.uColor.value.set(p.color);
      uniforms.uColorTwo.value.set(p.colorTwo);
      uniforms.uLineThickness.value = p.lineThickness;
      uniforms.uBaseRadius.value = p.baseRadius;
      uniforms.uRadiusStep.value = p.radiusStep;
      uniforms.uScaleRate.value = p.scaleRate;
      uniforms.uRingCount.value = p.ringCount;
      uniforms.uOpacity.value = p.opacity;
      uniforms.uNoiseAmount.value = p.noiseAmount;
      uniforms.uRotation.value = (p.rotation * Math.PI) / 180;
      uniforms.uRingGap.value = p.ringGap;
      uniforms.uFadeIn.value = p.fadeIn;
      uniforms.uFadeOut.value = p.fadeOut;
      uniforms.uMouse.value.set(smoothMouseRef.current[0], smoothMouseRef.current[1]);
      uniforms.uMouseInfluence.value = p.followMouse ? p.mouseInfluence : 0;
      uniforms.uHoverAmount.value = hoverAmountRef.current;
      uniforms.uHoverScale.value = p.hoverScale;
      uniforms.uParallax.value = p.parallax;
      uniforms.uBurst.value = p.clickBurst ? burstRef.current : 0;

      renderer.render(scene, camera);
    };
    frameId = requestAnimationFrame(animate);

    return () => {
    cancelAnimationFrame(frameId);
    window.removeEventListener('resize', resize);
    ro.disconnect();
    mount.removeEventListener('mousemove', onMouseMove);
    mount.removeEventListener('mouseenter', onMouseEnter);
    mount.removeEventListener('mouseleave', onMouseLeave);
    mount.removeEventListener('click', onClick);
    if (renderer && renderer.domElement && mount.contains(renderer.domElement)) {
    mount.removeChild(renderer.domElement);
    }
    renderer.dispose();
    material.dispose();
    };
    }, []);

    return <div ref={mountRef} className="magic-rings-container" style={blur > 0 ? { filter: `blur(${blur}px)` } : undefined} />;
}
```

---

## 04 — LiquidGlassButton (Vidro real estilo visionOS/iOS 26)

Botão de vidro líquido com refração real via SVG `feDisplacementMap` — não é blur
fake com fundo opaco, é distorção de verdade do que está atrás. Inclui tilt 3D
que segue o cursor, borda especular animada (luz correndo pela borda no hover),
glare sutil e ripple no clique. Vanilla HTML/CSS/JS, self-contained, sem libs.

**Por que `feDisplacementMap` e não só `backdrop-filter: blur()`:**
Blur sozinho só borra o fundo — fica parecendo plástico fosco. O filtro de
distorção (`feTurbulence` + `feDisplacementMap`) deforma o conteúdo atrás do
vidro como uma lente de verdade, que é o que vende a ilusão de material físico.

### Props / Variáveis customizáveis

| Variável CSS         | Default              | Descrição                              |
|-----------------------|---------------------|------------------------------------------|
| `--glass-tint`         | `rgba(250,250,250,.035)` | Opacidade do vidro (quase zero = mais real) |
| `--glass-border`       | `rgba(250,250,250,.35)`  | Cor da borda fina do vidro              |
| `data-glass`           | —                    | Atributo que ativa o tilt 3D + glare via JS |
| `.dark`                | classe modifier      | Variante de vidro escuro                |
| `.vision-fab`          | classe modifier      | Variante circular só com ícone          |

### Uso no B&W preset

```html
<a href="#" class="vision-btn" data-glass>
  <div class="vision-glare"></div>
  <span class="vision-text">Explorar</span>
</a>
```

### LiquidGlassButton (HTML/CSS/JS self-contained)

```html
<!-- Filtro de refração — essencial. Sem isso é só blur, não lente. -->
<svg width="0" height="0" style="position:absolute">
  <filter id="lensDistort" x="-20%" y="-20%" width="140%" height="140%">
    <feTurbulence type="fractalNoise" baseFrequency="0.008 0.012" numOctaves="2" seed="7" result="noise"/>
    <feDisplacementMap in="SourceGraphic" in2="noise" scale="18" xChannelSelector="R" yChannelSelector="G"/>
  </filter>
</svg>

<style>
.vision-btn{
  position:relative;
  height:56px;
  padding:0 30px;
  border-radius:28px;
  display:flex; align-items:center; gap:10px;
  text-decoration:none; cursor:pointer; user-select:none;
  isolation:isolate;
  font-family:'Martian Mono', monospace;

  /* material quase transparente — o segredo do liquid glass real */
  background: rgba(250,250,250,0.035);
  backdrop-filter: blur(3px) url(#lensDistort) saturate(140%) brightness(104%);
  -webkit-backdrop-filter: blur(3px) saturate(140%) brightness(104%);
  border:1px solid rgba(250,250,250,0.35);

  box-shadow:
    inset 0 1.5px 0 rgba(250,250,250,.9),
    inset 0 -1px 0 rgba(250,250,250,.15),
    inset 1px 0 0 rgba(250,250,250,.25),
    inset -1px 0 0 rgba(250,250,250,.1),
    0 10px 24px rgba(10,10,10,.28),
    0 2px 6px rgba(10,10,10,.20);

  transform: perspective(1200px) rotateX(var(--rx,0deg)) rotateY(var(--ry,0deg)) scale(var(--s,1));
  transition: transform .5s cubic-bezier(.25,1,.5,1),
              box-shadow .45s cubic-bezier(.25,1,.5,1),
              background-color .35s ease;
}

.vision-btn.dark{
  background: rgba(10,10,10,0.06);
  border-color: rgba(250,250,250,0.18);
  box-shadow:
    inset 0 1.5px 0 rgba(250,250,250,.4),
    inset 0 -1px 0 rgba(250,250,250,.05),
    0 10px 24px rgba(10,10,10,.4),
    0 2px 6px rgba(10,10,10,.25);
}

/* borda especular — luz correndo pela borda no hover */
.vision-btn::after{
  content:''; position:absolute; inset:0; border-radius:inherit; z-index:2; pointer-events:none;
  padding:1px;
  background: conic-gradient(from var(--edge-angle,0deg),
    transparent 0%, rgba(250,250,250,.7) 8%, transparent 18%, transparent 100%);
  -webkit-mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0);
  -webkit-mask-composite: xor; mask-composite: exclude;
  opacity:0; transition:opacity .4s ease;
  animation: spin-edge 3.2s linear infinite paused;
}
.vision-btn:hover::after{ opacity:1; animation-play-state:running; }
@keyframes spin-edge{ to{ --edge-angle:360deg; } }
@property --edge-angle{ syntax:'<angle>'; inherits:false; initial-value:0deg; }

/* glare sutil que segue o cursor */
.vision-glare{
  position:absolute; inset:0; z-index:3; border-radius:inherit; pointer-events:none;
  opacity:0; transition:opacity .3s ease;
  background: radial-gradient(circle 55px at var(--x,50%) var(--y,50%), rgba(250,250,250,.35), transparent 70%);
  mix-blend-mode: overlay;
}
.vision-btn:hover .vision-glare{ opacity:1; }

.vision-text{
  position:relative; z-index:4;
  color:rgba(250,250,250,.95);
  font-size:15px; font-weight:500; letter-spacing:.02em;
  text-shadow:0 1px 4px rgba(10,10,10,.25);
  pointer-events:none; white-space:nowrap;
}

/* pulso de clique */
.ripple{
  position:absolute; border-radius:50%; z-index:1; pointer-events:none;
  background:radial-gradient(circle, rgba(250,250,250,.5) 0%, transparent 70%);
  transform:translate(-50%,-50%) scale(0);
  animation: ripple-out .65s cubic-bezier(.25,1,.5,1) forwards;
}
@keyframes ripple-out{ to{ transform:translate(-50%,-50%) scale(1); opacity:0; } }

.vision-fab{ height:56px; width:56px; padding:0; border-radius:50%; justify-content:center; }
</style>

<script>
document.querySelectorAll('[data-glass]').forEach((btn) => {
  let rect;

  btn.addEventListener('mouseenter', () => {
    rect = btn.getBoundingClientRect();
    btn.style.transition = 'transform .05s linear, box-shadow .4s cubic-bezier(.25,1,.5,1)';
  });

  btn.addEventListener('mousemove', (e) => {
    if (!rect) return;
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    const glare = btn.querySelector('.vision-glare');
    glare.style.setProperty('--x', `${x}px`);
    glare.style.setProperty('--y', `${y}px`);

    const cx = rect.width / 2, cy = rect.height / 2;
    const max = 7;
    btn.style.setProperty('--rx', `${((y - cy) / cy) * -max}deg`);
    btn.style.setProperty('--ry', `${((x - cx) / cx) * max}deg`);
  });

  btn.addEventListener('mouseleave', () => {
    btn.style.transition = 'transform .55s cubic-bezier(.25,1,.5,1), box-shadow .4s cubic-bezier(.25,1,.5,1)';
    btn.style.setProperty('--rx', '0deg');
    btn.style.setProperty('--ry', '0deg');
    const glare = btn.querySelector('.vision-glare');
    glare.style.setProperty('--x', '50%');
    glare.style.setProperty('--y', '50%');
  });

  btn.addEventListener('mousedown', (e) => {
    const r = btn.getBoundingClientRect();
    const ripple = document.createElement('div');
    ripple.className = 'ripple';
    const size = Math.max(r.width, r.height) * 1.6;
    ripple.style.width = ripple.style.height = `${size}px`;
    ripple.style.left = `${e.clientX - r.left}px`;
    ripple.style.top = `${e.clientY - r.top}px`;
    btn.appendChild(ripple);
    ripple.addEventListener('animationend', () => ripple.remove());
  });
});
</script>
```

**Notas de uso:**
- Precisa de fundo com contraste (texto, padrão, imagem) atrás pra refração aparecer — em fundo liso a distorção não se vê.
- `--rx`/`--ry`/`--s` são setadas via JS inline, não precisa declarar no CSS base.
- Pra usar em fundo claro (`--paper`), inverte os valores de `rgba(250,250,250,...)` pra `rgba(10,10,10,...)` nas bordas e no glare.

