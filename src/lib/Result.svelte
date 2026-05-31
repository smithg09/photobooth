<script>
  import { onMount, onDestroy } from 'svelte';
  import { getStripById } from './strip-db.js';

  export let go;
  export let finalStrip;
  export let setFinalStrip;
  export let resultId;
  export let setResultId;
  export let setPhotos;

  const CELEBRATION = [
    'Fresh out of the booth. Certified fridge-door material.',
    'Four frames, one tiny time capsule.',
    'Nostalgia processed successfully. No retakes required.',
    'Proof your camera roll still has classics in it.',
  ];

  function buildConfetti() {
    return Array.from({ length: 16 }, (_, i) => ({
      id: i,
      x: Math.round(8 + Math.random() * 84),
      delay: Math.round(Math.random() * 320),
      drift: Math.round(-24 + Math.random() * 48),
      rot: Math.round(-28 + Math.random() * 56),
      tone: i % 2 === 0 ? 'var(--glow)' : 'var(--primary)',
    }));
  }

  let toast = '';
  let celebrate = true;
  let pieces = [];
  let note = '';
  let toastTimer = null;

  onMount(() => {
    const now = new Date();
    note = CELEBRATION[(now.getDate() + now.getMonth()) % CELEBRATION.length];
    pieces = buildConfetti();
    const t = setTimeout(() => { celebrate = false; }, 1300);

    if (!finalStrip && resultId) {
      getStripById(resultId).then((s) => {
        if (s?.dataUrl) setFinalStrip(s.dataUrl);
        else go('lobby');
      }).catch(() => go('lobby'));
    }

    return () => { clearTimeout(t); clearTimeout(toastTimer); };
  });

  onDestroy(() => { clearTimeout(toastTimer); });

  function showToast(m) {
    toast = m;
    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => { toast = ''; }, 2200);
  }

  function download() {
    if (!finalStrip) return;
    const a = document.createElement('a');
    a.download = 'flashback-strip.png'; a.href = finalStrip; a.click();
  }

  function print() {
    if (!finalStrip) return;
    const w = window.open('', '_blank');
    if (!w) { showToast('Popup blocked — allow popups to print.'); return; }
    w.document.write(`<html><head><title>Flashback Strip</title><style>body{margin:0;min-height:100vh;display:grid;place-items:center;background:#15110d}img{max-height:95vh;box-shadow:0 12px 26px rgba(0,0,0,0.4)}</style></head><body><img src="${finalStrip}" alt="Photobooth strip" /></body></html>`);
    w.document.close(); w.focus(); setTimeout(() => w.print(), 400);
  }

  async function share() {
    if (!finalStrip) return;
    if (navigator.share && navigator.canShare) {
      try {
        const res = await fetch(finalStrip); const blob = await res.blob();
        const file = new File([blob], 'flashback-strip.png', { type: 'image/png' });
        if (navigator.canShare({ files: [file] })) {
          await navigator.share({ files: [file], title: 'My Flashback Booth strip' });
          showToast('Shared.'); return;
        }
      } catch { /* fall through */ }
    }
    const link = resultId
      ? `${location.origin}${location.pathname}#/result/${resultId}`
      : location.href;
    try { await navigator.clipboard.writeText(link); showToast('Share link copied.'); }
    catch { showToast('Share is unavailable in this browser.'); }
  }

  function restart() {
    setPhotos([]); setFinalStrip(null); setResultId(null); go('lobby');
  }
</script>

<div class="screen pickup page-enter">
  <header class="pickup-head">
    <p class="eyebrow eyebrow--accent">Ready for pickup</p>
    <h2 class="display-title">Your strip is printed &amp; saved</h2>
    <p class="note">{note}</p>
    {#if resultId}
      <p class="pickup-id">Strip no. {resultId}</p>
    {/if}
  </header>

  <div class="pickup-grid">
    <div class="tray">
      <span class="tray-pin l"></span>
      <span class="tray-pin r"></span>
      {#if celebrate}
        <div class="confetti" aria-hidden="true">
          {#each pieces as p (p.id)}
            <i style="left:{p.x}%; animation-delay:{p.delay}ms; background:color-mix(in oklch, {p.tone} 76%, var(--enamel)); --drift:{p.drift}px; --rot:{p.rot}deg;"></i>
          {/each}
        </div>
      {/if}
      {#if finalStrip}
        <img class="final-strip" src={finalStrip} alt="Completed photobooth strip" />
      {:else}
        <span class="preview-loading">Loading strip…</span>
      {/if}
      <span class="tray-slot"></span>
    </div>

    <div class="pickup-actions">
      <p class="lead">Save it, print it, or share it with one tap.</p>
      <button class="btn btn--primary" on:click={download}>Download PNG</button>
      <button class="btn btn--ghost" on:click={print}>Print strip</button>
      <button class="btn btn--ghost" on:click={share}>Share</button>
      <button class="btn btn--brass" on:click={restart}>Back to lobby</button>
    </div>
  </div>

  {#if toast}
    <div class="toast" role="status">{toast}</div>
  {/if}
</div>

<style>
  .pickup { display: grid; gap: clamp(0.9rem, 2vw, 1.3rem); }
  .pickup-head { display: grid; gap: 0.4rem; }
  .pickup-head :global(h2) { font-size: clamp(2rem, 5vw, 3.4rem); }
  .note { justify-self: start; font-family: var(--font-mono); text-transform: uppercase; letter-spacing: 0.1em; font-size: 0.58rem; color: var(--ink-soft); padding: 0.3rem 0.6rem; border-radius: 5px; background: color-mix(in oklch, var(--glow) 14%, transparent); }
  .pickup-id { font-family: var(--font-mono); text-transform: uppercase; letter-spacing: 0.12em; font-size: 0.6rem; color: var(--ink-faint); }

  .pickup-grid { display: grid; grid-template-columns: 1fr; gap: clamp(1rem, 2.4vw, 1.6rem); align-items: start; }
  @media (min-width: 820px) { .pickup-grid { grid-template-columns: 1fr 0.72fr; } }

  .tray { position: relative; display: grid; place-items: center; min-height: 360px; padding: clamp(1.4rem, 3vw, 2.2rem) 1rem 2.4rem; border-radius: 14px; background: radial-gradient(120% 70% at 50% 0%, color-mix(in oklch, var(--glow) 12%, transparent), transparent 55%), linear-gradient(180deg, var(--enamel), var(--enamel-2)); box-shadow: inset 0 0 0 1px color-mix(in oklch, var(--ink) 12%, transparent), inset 0 4px 14px rgba(0,0,0,0.1); overflow: hidden; }
  .tray-pin { position: absolute; top: 16px; width: 13px; aspect-ratio: 1; border-radius: 50%; background: radial-gradient(circle at 38% 32%, var(--brass), var(--brass-d) 70%, var(--wood-d)); box-shadow: inset 0 1px 1px rgba(255,255,255,0.6), 0 1px 2px rgba(0,0,0,0.5); }
  .tray-pin.l { left: 22px; } .tray-pin.r { right: 22px; }
  .tray-slot { position: absolute; left: 50%; bottom: 0; transform: translateX(-50%); width: clamp(180px, 40%, 260px); height: 18px; border-radius: 10px 10px 0 0; background: linear-gradient(180deg, #000, var(--wood-d)); box-shadow: inset 0 4px 8px rgba(0,0,0,0.8); }
  .final-strip { max-height: min(64vh, 600px); width: auto; border-radius: 4px; box-shadow: 0 22px 34px -10px rgba(0,0,0,0.55); transform: rotate(-2deg); animation: dispense 700ms 100ms var(--ease-out-expo) both; }
  .final-strip:hover { transform: rotate(-0.6deg) translateY(-3px); transition: transform 240ms var(--ease-out-quint); }

  .pickup-actions { display: grid; gap: 0.55rem; align-content: start; }
  .lead { font-family: var(--font-body); font-size: 1.05rem; line-height: 1.4; color: var(--ink-soft); margin-bottom: 0.2rem; }
  .pickup-actions :global(.btn) { width: 100%; justify-content: flex-start; }

  .confetti { position: absolute; inset: 0; pointer-events: none; }
  .confetti i { position: absolute; top: -16px; width: 8px; height: 12px; border-radius: 2px; animation: drop 900ms var(--ease-out-quint) forwards; }

  .toast { position: fixed; left: 50%; bottom: 26px; transform: translateX(-50%); z-index: 60; font-family: var(--font-mono); text-transform: uppercase; letter-spacing: 0.12em; font-size: 0.62rem; color: var(--on-dark); padding: 0.6rem 1rem; border-radius: 8px; background: var(--wood-d); box-shadow: 0 12px 24px -8px rgba(0,0,0,0.6); animation: pageIn 220ms var(--ease-out-quart); }

  @keyframes dispense { from { opacity: 0; transform: rotate(-3deg) translateY(-16px) scale(0.98); } to { opacity: 1; transform: rotate(-2deg) translateY(0) scale(1); } }
  @keyframes drop { from { opacity: 0; transform: translateY(-4px) rotate(0); } 12% { opacity: 1; } to { opacity: 0; transform: translateY(220px) translateX(var(--drift)) rotate(var(--rot)); } }
</style>
