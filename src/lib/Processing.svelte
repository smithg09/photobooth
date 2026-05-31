<script>
  import { onMount, onDestroy } from 'svelte';
  import { saveStrip } from './strip-db.js';
  import { generateStripDataUrl } from './strip-renderer.js';

  export let go;
  export let photos;
  export let filter;
  export let frameId;
  export let customEmoji;
  export let customColor;
  export let customText;
  export let setFinalStrip;
  export let setResultId;

  const DK_TOTAL = 5;
  const DK_MESSAGES = [
    'Warming chemical baths and calibrating contrast.',
    'Aligning frame perforations and levelling highlights.',
    'Dusting off nostalgia grain for that analog edge.',
    'Sealing the strip for pockets, fridges, and scrapbooks.',
  ];
  const DK_STEPS = [
    { n: '01', name: 'Chemical bath', at: 5 },
    { n: '02', name: 'Exposure',      at: 35 },
    { n: '03', name: 'Fixing',        at: 50 },
    { n: '04', name: 'Dispense',      at: 70 },
  ];

  let secondsLeft = DK_TOTAL;
  let ready = false;
  let msg = DK_MESSAGES[0];
  let disposed = false;

  $: progress = ((DK_TOTAL - secondsLeft) / DK_TOTAL) * 100;

  onMount(() => {
    disposed = false;
    if (!photos || photos.length === 0) { go('studio'); return; }

    let countHandle;
    const countdownDone = new Promise((resolve) => {
      countHandle = setInterval(() => {
        secondsLeft = Math.max(0, secondsLeft - 1);
        if (secondsLeft === 0) { clearInterval(countHandle); resolve(); }
      }, 1000);
    });

    let msgIndex = 0;
    const msgHandle = setInterval(() => {
      if (disposed) return;
      msgIndex = (msgIndex + 1) % DK_MESSAGES.length;
      if (!ready) msg = DK_MESSAGES[msgIndex];
    }, 1400);

    (async () => {
      try {
        const url = await generateStripDataUrl({
          photos, selectedFilter: filter, selectedFrame: frameId,
          customEmoji, customColor, footerLabel: customText,
        });
        if (!url) throw new Error('no strip');
        if (disposed) return;
        setFinalStrip(url);
        const id = await saveStrip(url);
        setResultId(id ?? null);
        ready = true;
        msg = 'Strip generated and archived. Sending to the pickup counter.';
        await countdownDone;
        if (!disposed) go('pickup');
      } catch {
        clearInterval(countHandle); clearInterval(msgHandle);
        if (!disposed) { alert('Something went wrong while developing. Please try again.'); go('studio'); }
      }
    })();

    return () => {
      disposed = true;
      clearInterval(countHandle);
      clearInterval(msgHandle);
    };
  });

  onDestroy(() => { disposed = true; });
</script>

<div class="screen darkroom page-enter">
  <div class="darkroom-grid">
    <div class="dk-info">
      <span class="status-pill"><span class="dot"></span>Darkroom active</span>
      <div class="dk-title">
        <span class="l1">Developing</span>
        <span class="l2">your strip</span>
      </div>
      <div class="dk-steps" aria-hidden="true">
        {#each DK_STEPS as s}
          <div class="dk-step {progress >= s.at ? 'active' : ''}">
            <span class="dk-num">{s.n}</span>
            <span class="dk-name">{s.name}</span>
          </div>
        {/each}
      </div>
      {#if progress >= 75}
        <div class="dk-log" aria-live="polite"><span class="cur">&gt;</span>{msg}</div>
      {/if}
    </div>

    <div class="dk-machine" aria-hidden="true">
      <div class="dk-timer">{secondsLeft}</div>
      <div class="dev-tray">
        <p class="dev-label">Chemical bath · developing</p>
        <div class="dev-grid">
          {#each Array(4) as _, i}
            <div class="dev-cell" style="--i:{i}">
              {#if photos[i]}
                <img src={photos[i]} alt="" />
              {/if}
              <div class="dev-overlay"></div>
            </div>
          {/each}
        </div>
        <div class="dev-wash"></div>
      </div>
    </div>
  </div>
</div>

<style>
  .darkroom { min-height: clamp(440px, 60vh, 640px); display: grid; align-content: center; }
  .darkroom-grid { display: grid; grid-template-columns: 1fr; gap: clamp(2rem, 6vw, 5rem); align-items: center; }
  @media (min-width: 860px) { .darkroom-grid { grid-template-columns: 1.05fr 0.95fr; } }

  .dk-info { display: grid; gap: clamp(1.1rem, 3vw, 1.8rem); }
  .dk-title { display: grid; line-height: 0.84; }
  .dk-title .l1 { font-family: var(--font-sign); font-weight: 700; text-transform: uppercase; font-size: clamp(2.8rem, 7vw, 5rem); color: var(--primary); letter-spacing: 0.01em; }
  .dk-title .l2 { font-family: var(--font-body); font-style: italic; font-size: clamp(2.6rem, 6vw, 4.4rem); color: var(--ink); padding-left: clamp(1.5rem, 5vw, 3.5rem); margin-top: -0.1em; }

  .dk-steps { display: grid; gap: 1.1rem; }
  .dk-step { display: grid; grid-template-columns: auto 1fr; align-items: center; gap: 0.9rem; opacity: 0.32; transform: translateX(-10px); transition: opacity 0.5s ease, transform 0.5s var(--ease-out-quart); padding-bottom: 0.9rem; border-bottom: 1px solid color-mix(in oklch, var(--ink) 12%, transparent); }
  .dk-step.active { opacity: 1; transform: translateX(0); border-bottom-color: color-mix(in oklch, var(--glow) 50%, transparent); }
  .dk-num  { font-family: var(--font-mono); font-weight: 600; font-size: 0.9rem; color: var(--wood-d); padding: 0.2rem 0.42rem; border-radius: 5px; background: linear-gradient(180deg, var(--brass), var(--brass-d)); }
  .dk-name { font-family: var(--font-sign); text-transform: uppercase; font-size: clamp(1.1rem, 2.4vw, 1.5rem); letter-spacing: 0.06em; color: var(--ink); }

  .dk-log { font-family: var(--font-mono); font-size: 0.78rem; line-height: 1.6; color: var(--ink-soft); padding: 0.9rem 1rem; border-radius: 8px; border-left: 3px solid var(--primary); background: color-mix(in oklch, var(--enamel-2) 70%, transparent); }
  .dk-log .cur { color: var(--primary); font-weight: 700; margin-right: 0.4rem; animation: blink 1s step-end infinite; }

  .dk-machine { position: relative; display: grid; place-items: center; }
  .dk-timer { position: absolute; inset: 0; display: grid; place-items: center; font-family: var(--font-sign); font-weight: 700; font-size: clamp(11rem, 24vw, 20rem); color: color-mix(in oklch, var(--glow) 10%, transparent); pointer-events: none; line-height: 1; }

  .dev-tray {
    position: relative; z-index: 1;
    border-radius: 18px;
    padding: clamp(0.9rem, 2vw, 1.3rem);
    background: linear-gradient(160deg, #1c1008, #0e0804);
    box-shadow:
      inset 0 1px 0 rgba(255,255,255,0.04),
      inset 0 0 60px rgba(0,0,0,0.6),
      0 28px 50px -16px rgba(0,0,0,0.7);
    border: 1.5px solid var(--wood-d);
    overflow: hidden;
  }
  .dev-label {
    font-family: var(--font-mono); text-transform: uppercase; letter-spacing: 0.18em;
    font-size: 0.52rem; text-align: center; margin-bottom: 0.8rem;
    color: color-mix(in oklch, var(--glow) 55%, transparent);
  }
  .dev-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; }
  .dev-cell {
    width: clamp(100px, 13vw, 130px);
    height: clamp(128px, 16vw, 164px);
    border-radius: 4px; position: relative; overflow: hidden;
    background: #08050302;
  }
  .dev-cell img { width: 100%; height: 100%; object-fit: cover; display: block;
    filter: saturate(0) brightness(0.15);
    animation: devPhoto 1.6s calc(var(--i) * 0.65s + 0.4s) var(--ease-out-quart) both;
  }
  .dev-overlay {
    position: absolute; inset: 0; pointer-events: none;
    background: linear-gradient(160deg, rgba(160,60,8,0.82), rgba(60,16,2,0.94));
    animation: devOverlay 1.6s calc(var(--i) * 0.65s + 0.4s) var(--ease-out-quart) both;
  }
  .dev-wash {
    position: absolute; inset: 0; pointer-events: none;
    background: linear-gradient(180deg, transparent 0%, rgba(200,90,20,0.12) 50%, transparent 100%);
    animation: washWave 2.2s ease-in-out infinite;
  }

  @keyframes devPhoto   { from { filter: saturate(0) brightness(0.15); } to { filter: saturate(1.05) brightness(1); } }
  @keyframes devOverlay { from { opacity: 1; } to { opacity: 0; } }
  @keyframes washWave   { 0%, 100% { transform: translateY(-120%); } 50% { transform: translateY(120%); } }
  @keyframes blink      { 50% { opacity: 0; } }
</style>
