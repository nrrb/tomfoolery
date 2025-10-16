# tomfoolery

Copy and paste these JavaScript snippets into the Developer Console of your browser for fun.

## Disco Mode

```js
(() => {
  if (window.__discoMode?.stop) {
    window.__discoMode.stop()
    return
  }

  const els = Array.from(document.querySelectorAll('*')).filter(el => {
    const cs = getComputedStyle(el)
    return cs.display !== 'none' && cs.visibility !== 'hidden'
  })

  const originals = new Map()
  for (const el of els) {
    originals.set(el, {
      background: el.style.backgroundColor || '',
      transition: el.style.transition || ''
    })
    el.style.transition = 'background-color 0.1s linear'
  }

  let raf = null
  let active = true
  let hue = 0

  const tick = () => {
    if (!active) return
    hue = (hue + 20) % 360
    for (const el of els) {
      const s = 60 + Math.random() * 40 // saturation
      const l = 40 + Math.random() * 20 // lightness
      el.style.backgroundColor = `hsl(${(hue + el.offsetTop / 5) % 360}, ${s}%, ${l}%)`
    }
    raf = requestAnimationFrame(tick)
  }

  const stop = () => {
    active = false
    if (raf) cancelAnimationFrame(raf)
    for (const el of els) {
      const orig = originals.get(el)
      el.style.backgroundColor = orig.background
      el.style.transition = orig.transition
    }
    delete window.__discoMode
    delete window.stopDiscoMode
    console.log('🪩 Disco Mode stopped and colors restored.')
  }

  window.__discoMode = { stop }
  window.stopDiscoMode = stop

  console.log('🕺 Disco Mode activated! Call stopDiscoMode() to restore.')
  tick()
})()

```

## Wobble Mode

```js
(() => {
  if (window.__wobbleMode?.stop) {
    window.__wobbleMode.stop()
    return
  }

  const els = Array.from(document.querySelectorAll('*')).filter(el => {
    const cs = getComputedStyle(el)
    return cs.display !== 'none' && cs.visibility !== 'hidden'
  })

  const originals = new Map()
  for (const el of els) {
    originals.set(el, el.style.transform || '')
    el.style.transformOrigin = 'center center'
    el.style.willChange = 'transform'
  }

  let raf = null
  let start = performance.now()

  const tick = now => {
    const t = (now - start) / 1000
    for (const el of els) {
      const base = originals.get(el) || ''
      const wobble = Math.sin(t * 2 + el.offsetTop / 50) * 5 // degrees
      el.style.transform = `${base} rotate(${wobble}deg)`
    }
    raf = requestAnimationFrame(tick)
  }

  const stop = () => {
    if (raf) cancelAnimationFrame(raf)
    for (const el of els) {
      el.style.transform = originals.get(el)
      el.style.willChange = ''
    }
    delete window.__wobbleMode
    delete window.stopWobbleMode
    console.log('Wobble Mode stopped and styles restored.')
  }

  window.__wobbleMode = { stop }
  window.stopWobbleMode = stop

  console.log('Wobble Mode activated! Call stopWobbleMode() to restore.')
  raf = requestAnimationFrame(tick)
})()
```

## Drunken Walk

```js
(() => {
  if (window.__drunkenWalk?.stop) {
    window.__drunkenWalk.stop()
    return
  }

  // Tunables
  const RADIUS_PX = 8        // max wander distance from original spot
  const ACCEL = 0.8          // higher = more twitchy
  const DAMPING = 0.86       // 0–1; lower = more sluggish
  const ROT_MAX_DEG = 2      // small rotation wobble cap

  const skip = new Set(['HTML','HEAD','META','LINK','STYLE','SCRIPT','TITLE'])
  const els = Array.from(document.querySelectorAll('*')).filter(el => {
    if (skip.has(el.tagName)) return false
    const cs = getComputedStyle(el)
    if (cs.display === 'none' || cs.visibility === 'hidden') return false
    const r = el.getBoundingClientRect()
    return r.width > 0 && r.height > 0
  })

  const st = new Map()
  for (const el of els) {
    st.set(el, {
      base: el.style.transform || '',
      ox: 0, oy: 0, vx: 0, vy: 0,
      ang: 0, vang: 0,
      phase: (el.offsetTop + el.offsetLeft) * 0.001 // desync elements
    })
    el.style.transformOrigin = 'center center'
    el.style.willChange = 'transform'
  }

  let raf = null
  let last = performance.now()

  const clamp = (v, a, b) => Math.max(a, Math.min(b, v))

  const tick = now => {
    const dt = Math.min(0.033, (now - last) / 1000) // seconds
    last = now

    for (const el of els) {
      const s = st.get(el)
      if (!s) continue

      // random accelerations (Brownian-ish) with damping to keep near center
      const jitter = (Math.random() - 0.5) * ACCEL
      const jitter2 = (Math.random() - 0.5) * ACCEL
      s.vx = s.vx * DAMPING + jitter
      s.vy = s.vy * DAMPING + jitter2

      s.ox += s.vx * (dt * 60) // scale to roughly frame-rate independent px
      s.oy += s.vy * (dt * 60)

      // keep inside radius with soft clamp (reflect at edge)
      if (s.ox > RADIUS_PX || s.ox < -RADIUS_PX) { s.vx *= -0.9; s.ox = clamp(s.ox, -RADIUS_PX, RADIUS_PX) }
      if (s.oy > RADIUS_PX || s.oy < -RADIUS_PX) { s.vy *= -0.9; s.oy = clamp(s.oy, -RADIUS_PX, RADIUS_PX) }

      // tiny rotational wobble tied to motion & a sine to look organic
      const t = now * 0.001 + s.phase
      s.vang = s.vang * 0.9 + (Math.sin(t * 1.7) + (Math.random() - 0.5)) * 0.02
      s.ang += s.vang
      s.ang = clamp(s.ang, -ROT_MAX_DEG * Math.PI/180, ROT_MAX_DEG * Math.PI/180)

      el.style.transform = `${s.base} translate(${s.ox}px, ${s.oy}px) rotate(${s.ang}rad)`.trim()
    }

    raf = requestAnimationFrame(tick)
  }

  const stop = () => {
    if (raf) cancelAnimationFrame(raf)
    for (const el of els) {
      const s = st.get(el)
      if (!s) continue
      el.style.transform = s.base
      el.style.willChange = ''
    }
    st.clear()
    delete window.__drunkenWalk
    delete window.stopDrunkenWalk
    console.log('🍺 Drunken Walk stopped and styles restored.')
  }

  window.__drunkenWalk = { stop }
  window.stopDrunkenWalk = stop

  console.log('🍻 Drunken Walk activated! Call stopDrunkenWalk() to restore.')
  requestAnimationFrame(tick)
})()
```

## Sarcasm Mode

```js
(() => {
  if (window.__sarcasmMode?.stop) {
    window.__sarcasmMode.stop()
    return
  }

  // Elements whose text we should ignore entirely
  const SKIP_TAGS = new Set(['SCRIPT','STYLE','NOSCRIPT','IFRAME','TITLE','META','LINK'])
  const isSkippable = el =>
    SKIP_TAGS.has(el.tagName) ||
    el.isContentEditable ||
    el.closest?.('input, textarea, [contenteditable="true"]')

  // Map original text so we can restore later
  const originals = new WeakMap()

  const isAlpha = ch => /[A-Za-z]/.test(ch)

  const toSarcasm = str => {
    // Start upper or lower at random for some variety
    let upper = Math.random() < 0.5
    let out = ''
    for (let i = 0; i < str.length; i++) {
      const ch = str[i]
      if (isAlpha(ch)) {
        out += upper ? ch.toUpperCase() : ch.toLowerCase()
        upper = !upper
      } else {
        // keep punctuation/whitespace/symbols exactly as-is
        out += ch
      }
    }
    return out
  }

  // Walk the DOM and transform text nodes
  const process = root => {
    const walker = document.createTreeWalker(
      root,
      NodeFilter.SHOW_TEXT,
      {
        acceptNode(node) {
          if (!node.nodeValue || !node.nodeValue.trim()) return NodeFilter.FILTER_REJECT
          const parent = node.parentElement
          if (!parent || isSkippable(parent)) return NodeFilter.FILTER_REJECT
          return NodeFilter.FILTER_ACCEPT
        }
      }
    )
    const nodes = []
    let n
    while ((n = walker.nextNode())) nodes.push(n)
    for (const textNode of nodes) {
      if (!originals.has(textNode)) {
        originals.set(textNode, textNode.nodeValue)
      }
      textNode.nodeValue = toSarcasm(originals.get(textNode))
    }
  }

  // Initial pass
  process(document.body)

  // If new DOM gets added, sarcasm it too (ignores edits to existing text to keep restore simple)
  const observer = new MutationObserver(muts => {
    for (const m of muts) {
      for (const node of m.addedNodes || []) {
        if (node.nodeType === Node.ELEMENT_NODE && !isSkippable(node)) {
          process(node)
        } else if (node.nodeType === Node.TEXT_NODE) {
          const parent = node.parentElement
          if (parent && !isSkippable(parent)) {
            if (!originals.has(node)) originals.set(node, node.nodeValue)
            node.nodeValue = toSarcasm(originals.get(node))
          }
        }
      }
    }
  })
  observer.observe(document.body, { childList: true, subtree: true })

  const stop = () => {
    observer.disconnect()
    // Restore all tracked text nodes
    originals.forEach((orig, node) => {
      if (node && node.nodeType === Node.TEXT_NODE) {
        try { node.nodeValue = orig } catch {}
      }
    })
    originals.clear()
    delete window.__sarcasmMode
    delete window.stopSarcasmMode
    console.log('🙃 Sarcasm Mode stopped and text restored.')
  }

  window.__sarcasmMode = { stop }
  window.stopSarcasmMode = stop

  console.log('🙃 Sarcasm Mode activated! Call stopSarcasmMode() to restore.')
})()
```

## Smiley Titles

```js
(() => {
  const smileys = [
    '¯\\_(ツ)_/¯',
    '¯\\\\(°_o)/¯',
    '¯\\_(⊙︿⊙)_/¯',
    '¯\\_(シ)_/¯',
    '¯\\_( ͡° ͜ʖ ͡°)_/¯'
  ]

  const pick = () => smileys[Math.floor(Math.random() * smileys.length)]

  const headers = document.querySelectorAll('h1, h2, h3, h4, h5, h6')
  headers.forEach(h => {
    h.textContent = pick()
  })

  console.log(`Replaced ${headers.length} headers with random smileys ¯\\_(ツ)_/¯`)
})()
```

## Word Scramble

```js
(() => {
  const scrambleText = str => {
    return str.replace(/\w+/g, word => {
      if (word.length <= 3) return word // too short to scramble
      const chars = word.split('')
      const middle = chars.slice(1, -1)
      for (let i = middle.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1))
        ;[middle[i], middle[j]] = [middle[j], middle[i]]
      }
      return chars[0] + middle.join('') + chars[chars.length - 1]
    })
  }

  const walk = node => {
    for (const child of node.childNodes) {
      if (child.nodeType === Node.TEXT_NODE) {
        child.textContent = scrambleText(child.textContent)
      } else if (child.nodeType === Node.ELEMENT_NODE) {
        walk(child)
      }
    }
  }

  walk(document.body)
  console.log('All words scrambled (first and last letters preserved, punctuation untouched).')
})()
```

## Morse Code Mode

```js
(() => {
  if (window.__morseCodeMode?.stop) {
    window.__morseCodeMode.stop()
    return
  }

  // Elements to skip entirely
  const SKIP_TAGS = new Set(['SCRIPT','STYLE','NOSCRIPT','IFRAME','TITLE','META','LINK'])
  const isSkippable = el =>
    SKIP_TAGS.has(el.tagName) ||
    el.isContentEditable ||
    el.closest?.('input, textarea, [contenteditable="true"]')

  // Morse table (letters & digits)
  const M = {
    A: '.-',    B: '-...',  C: '-.-.',  D: '-..',   E: '.',
    F: '..-.',  G: '--.',   H: '....',  I: '..',    J: '.---',
    K: '-.-',   L: '.-..',  M: '--',    N: '-.',    O: '---',
    P: '.--.',  Q: '--.-',  R: '.-.',   S: '...',   T: '-',
    U: '..-',   V: '...-',  W: '.--',   X: '-..-',  Y: '-.--',
    Z: '--..',
    0: '-----', 1: '.----', 2: '..---', 3: '...--', 4: '....-',
    5: '.....', 6: '-....', 7: '--...', 8: '---..', 9: '----.'
  }

  // Letter separator: thin non-breaking space for readability
  const SEP = '\u202F' // U+202F

  // Convert an alphanumeric "word" to Morse, unknown chars left as-is
  const wordToMorse = word => {
    const out = []
    for (const ch of word) {
      const k = ch.toUpperCase()
      out.push(M[k] || ch)
    }
    return out.join(SEP)
  }

  // Replace alphanumeric runs with Morse, keep punctuation/spacing intact
  const toMorse = str =>
    str.replace(/[\p{L}\p{N}]+/gu, wordToMorse)

  // Track originals to restore
  const originals = new WeakMap()

  const process = root => {
    const walker = document.createTreeWalker(
      root,
      NodeFilter.SHOW_TEXT,
      {
        acceptNode(node) {
          if (!node.nodeValue || !node.nodeValue.trim()) return NodeFilter.FILTER_REJECT
          const parent = node.parentElement
          if (!parent || isSkippable(parent)) return NodeFilter.FILTER_REJECT
          return NodeFilter.FILTER_ACCEPT
        }
      }
    )
    const nodes = []
    let n
    while ((n = walker.nextNode())) nodes.push(n)
    for (const textNode of nodes) {
      if (!originals.has(textNode)) originals.set(textNode, textNode.nodeValue)
      textNode.nodeValue = toMorse(originals.get(textNode))
    }
  }

  // Initial pass
  process(document.body)

  // Handle newly added nodes as the DOM changes
  const observer = new MutationObserver(muts => {
    for (const m of muts) {
      for (const node of m.addedNodes || []) {
        if (node.nodeType === Node.ELEMENT_NODE && !isSkippable(node)) {
          process(node)
        } else if (node.nodeType === Node.TEXT_NODE) {
          const parent = node.parentElement
          if (parent && !isSkippable(parent)) {
            if (!originals.has(node)) originals.set(node, node.nodeValue)
            node.nodeValue = toMorse(originals.get(node))
          }
        }
      }
    }
  })
  observer.observe(document.body, { childList: true, subtree: true })

  const stop = () => {
    observer.disconnect()
    originals.forEach((orig, node) => {
      if (node && node.nodeType === Node.TEXT_NODE) {
        try { node.nodeValue = orig } catch {}
      }
    })
    originals.clear()
    delete window.__morseCodeMode
    delete window.stopMorseCodeMode
    console.log('〰️ Morse Code Mode stopped and text restored.')
  }

  window.__morseCodeMode = { stop }
  window.stopMorseCodeMode = stop

  console.log('〰️ Morse Code Mode activated! Alphanumeric words → Morse, punctuation/spacing preserved. Call stopMorseCodeMode() to restore.')
})()
```

## Doom Mode

```js
(() => {
  if (window.__doomMode?.stop) {
    window.__doomMode.stop()
    return
  }

  // ========= Tunables =========
  let RATE_PER_SEC = 1          // removals per second
  const SKIP_TAGS = new Set(['HTML','HEAD','BODY','SCRIPT','STYLE','LINK','META','TITLE'])
  const MAX_VICTIMS = 10_000
  // ============================

  // Style + HUD
  const style = document.createElement('style')
  style.id = '__doom-style'
  style.textContent = `
    .__doom-hud {
      position: fixed; inset: 16px 16px auto auto; z-index: 2147483647;
      font: 12px/1.3 system-ui, -apple-system, Segoe UI, Roboto, Arial;
      color: #ff4d4d; background: rgba(0,0,0,.75);
      border: 1px solid #ff4d4d; padding: 10px 12px; border-radius: 8px;
      box-shadow: 0 0 20px rgba(255,0,0,.3), inset 0 0 12px rgba(255,0,0,.2);
      backdrop-filter: blur(2px);
      pointer-events: none;
    }
    .__doom-title { font-weight: 700; letter-spacing: .08em; margin-bottom: 6px; text-transform: uppercase }
    .__doom-bar {
      position: relative; height: 8px; background: #220000; border: 1px solid #611; border-radius: 999px; overflow: hidden; margin: 6px 0 8px
    }
    .__doom-fill { position: absolute; left: 0; top: 0; bottom: 0; width: 0%; background: linear-gradient(90deg,#ff4d4d,#ffb199) }
    .__doom-row { display: flex; gap: 10px }
    .__doom-key { color: #ffa0a0 }
    .__doom-blink { animation: __doom-blink 0.9s infinite }
    @keyframes __doom-blink { 0%,100% { opacity: .35 } 50% { opacity: 1 } }
    @keyframes __doom-scan { from { background-position: 0 0 } to { background-position: 0 8px } }
    .__doom-overlay {
      pointer-events: none; position: fixed; inset: 0; z-index: 2147483646;
      background-image: repeating-linear-gradient(180deg, rgba(255,0,0,.05) 0, rgba(255,0,0,.05) 1px, transparent 1px, transparent 8px);
      animation: __doom-scan 1.2s linear infinite
    }
  `
  document.head.appendChild(style)

  const hud = document.createElement('div')
  hud.className = '__doom-hud'
  hud.innerHTML = `
    <div class="__doom-title">SYSTEM FAILURE</div>
    <div class="__doom-row"><div class="__doom-key">Objects lost:</div><div id="__doom-count">0</div></div>
    <div class="__doom-row"><div class="__doom-key">Integrity:</div><div id="__doom-pct">100%</div></div>
    <div class="__doom-bar"><div class="__doom-fill" id="__doom-fill"></div></div>
    <div class="__doom-blink">⚠︎ escalating faults detected</div>
  `
  const overlay = document.createElement('div')
  overlay.className = '__doom-overlay'
  document.body.appendChild(overlay)
  document.body.appendChild(hud)
  const $count = hud.querySelector('#__doom-count')
  const $pct = hud.querySelector('#__doom-pct')
  const $fill = hud.querySelector('#__doom-fill')

  const isVisible = el => {
    const cs = getComputedStyle(el)
    if (cs.display === 'none' || cs.visibility === 'hidden' || cs.opacity === '0') return false
    const r = el.getBoundingClientRect()
    return r.width > 2 && r.height > 2
  }

  const skipHud = el => el.closest('.__doom-hud') || el.classList?.contains('__doom-overlay') || el.id === '__doom-style'

  // Collect only leaf elements (no children)
  const allCandidates = Array.from(document.querySelectorAll('*')).filter(el => {
    if (SKIP_TAGS.has(el.tagName)) return false
    if (skipHud(el)) return false
    if (!isVisible(el)) return false
    if (el.children.length > 0) return false // only leaf nodes
    return true
  })

  const graveyard = []
  const info = {
    removed: 0,
    totalStart: Math.max(1, allCandidates.length),
  }

  const updateHUD = () => {
    $count.textContent = String(info.removed)
    const pct = Math.max(0, 100 - Math.round((info.removed / info.totalStart) * 100))
    $pct.textContent = pct + '%'
    $fill.style.width = (100 - pct) + '%'
  }

  const chooseVictim = () => {
    const alive = allCandidates.filter(el => el.isConnected && !skipHud(el) && !SKIP_TAGS.has(el.tagName))
    if (!alive.length) return null
    return alive[Math.floor(Math.random() * alive.length)]
  }

  const vanish = el => {
    if (!el || !el.parentNode) return
    const rec = { el, parent: el.parentNode, next: el.nextSibling }
    graveyard.push(rec)
    try {
      el.remove()
      info.removed++
      updateHUD()
    } catch {}
  }

  let intervalId = null
  const tick = () => {
    if (info.removed >= MAX_VICTIMS) return
    const victim = chooseVictim()
    vanish(victim)
  }

  const setRate = perSec => {
    const v = Math.max(0.1, Number(perSec) || RATE_PER_SEC)
    RATE_PER_SEC = v
    if (intervalId) clearInterval(intervalId)
    intervalId = setInterval(tick, 1000 / RATE_PER_SEC)
  }

  setRate(RATE_PER_SEC)

  const stop = () => {
    if (intervalId) clearInterval(intervalId)
    for (let i = graveyard.length - 1; i >= 0; i--) {
      const { el, parent, next } = graveyard[i]
      try { parent.insertBefore(el, next || null) } catch {}
    }
    graveyard.length = 0
    try { document.body.removeChild(hud) } catch {}
    try { document.body.removeChild(overlay) } catch {}
    try { document.head.removeChild(style) } catch {}
    delete window.__doomMode
    delete window.stopDoomMode
    delete window.setDoomRate
    console.log('💀 Doom Mode stopped. World restored.')
  }

  window.__doomMode = { stop, setRate }
  window.stopDoomMode = stop
  window.setDoomRate = setRate

  updateHUD()
  console.log('💀 Doom Mode (Leaf Edition) engaged. Only childless elements disappear. Call stopDoomMode() to restore. Use setDoomRate(perSecond) to change speed.')
})()
```
