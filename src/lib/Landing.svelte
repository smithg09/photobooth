<script>
  import { onMount } from 'svelte';
  import { getRecentStrips, clearAllStrips } from './strip-db.js';
  import strip1 from '../assets/sample-strip-1.png';
  import strip2 from '../assets/sample-strip-2.png';
  import strip3 from '../assets/sample-strip-3.png';
  import strip4 from '../assets/sample-strip-4.png';

  export let go;
  export let openStrip;

  const VIBE_LINES = [
    'Strike a pose — the booth handles the rest.',
    'Grab a friend and rehearse your dramatic faces.',
    'Pro tip: B&W mode is instant nostalgia.',
    'Your next favourite fridge-door souvenir.',
  ];

  let strips = [];
  let vibe = '';
  let session = 'Afternoon';

  function sessionWindow(hour) {
    if (hour < 12) return 'Morning';
    if (hour < 17) return 'Afternoon';
    if (hour < 21) return 'Evening';
    return 'Late night';
  }

  onMount(async () => {
    const now = new Date();
    session = sessionWindow(now.getHours());
    const seed = now.getFullYear() + now.getMonth() + now.getDate();
    vibe = VIBE_LINES[seed % VIBE_LINES.length];
    try { strips = await getRecentStrips(8); } catch { strips = []; }
  });

  async function clearHistory() {
    if (!window.confirm('Delete every saved strip from this browser?')) return;
    await clearAllStrips();
    strips = [];
  }

  function scrollToDryline() {
    document.getElementById('dryline')?.scrollIntoView({ behavior: 'smooth', block: 'start' });
  }

  const fmt = (ts) =>
    new Intl.DateTimeFormat('en-US', { month: 'short', day: 'numeric', year: 'numeric' }).format(ts);
</script>

<div class="screen lobby page-enter">
  <div class="lobby-hero">
    <div class="lobby-copy">
      <p class="eyebrow eyebrow--accent">{session} session · now open</p>
      <h2 class="display-title lobby-h">
        Four snapshots,<br /><em>one timeless strip.</em>
      </h2>
      <p class="serif-lead">
        The classic street-corner photo booth, restored for your browser. Snap live
        photos or bring your own — no coins required.
      </p>
      <p class="lobby-tag">{vibe}</p>
      <div class="lobby-actions">
        <button class="btn btn--primary" on:click={() => go('studio')}>Step inside</button>
        <button class="btn btn--ghost" on:click={scrollToDryline}>Past strips</button>
      </div>
    </div>

    <div class="viewing-window" aria-hidden="true">
      <div class="vw-head">
        <div>
          <span class="vw-sign">Photobooth</span>
          <span class="vw-sub">Studio preview</span>
        </div>
        <span class="vw-badge">Samples</span>
      </div>
      <div class="vw-glass">
        <img class="sample-strip s-l" src={strip4} alt="" />
        <img class="sample-strip s-r" src={strip2} alt="" />
        <img class="sample-strip s-main" src={strip1} alt="" />
        <img class="sample-strip s-l" src={strip3} alt="" />
      </div>
      <div class="vw-rail">
        <span class="vw-coin"></span>
        <span class="vw-cap">4 shots · 1 strip · instant keepsake</span>
        <span class="vw-led"></span>
      </div>
    </div>
  </div>

  <section class="strip-history" id="dryline">
    <div class="history-head">
      <h3 class="display-title">Recent strips</h3>
      {#if strips.length > 0}
        <button class="linklike" on:click={clearHistory}>Clear saved</button>
      {/if}
    </div>
    {#if strips.length === 0}
      <div class="history-empty">
        <p>Your printed strips will appear here.</p>
        <button class="btn btn--primary" on:click={() => go('studio')}>Start a session</button>
      </div>
    {:else}
      <div class="strip-grid">
        {#each strips as s (s.id)}
          <button class="strip-card" on:click={() => openStrip(s)}>
            <div class="strip-card-img">
              <img src={s.dataUrl} alt="Saved strip" />
            </div>
            <span class="strip-card-date">{fmt(s.createdAt)}</span>
          </button>
        {/each}
      </div>
    {/if}
  </section>
</div>

<style>
  .lobby { display: grid; gap: clamp(1.5rem, 4vw, 2.8rem); }

  .lobby-hero { display: grid; grid-template-columns: 1fr; gap: clamp(1.4rem, 4vw, 2.6rem); align-items: center; }
  @media (min-width: 880px) { .lobby-hero { grid-template-columns: 1.02fr 0.98fr; } }

  .lobby-copy { display: grid; gap: clamp(0.85rem, 2vw, 1.2rem); max-width: 30ch; }
  .lobby-copy > :global(*) { animation: revealUp 540ms var(--ease-out-quint) both; }
  .lobby-copy > :global(*:nth-child(2)) { animation-delay: 60ms; }
  .lobby-copy > :global(*:nth-child(3)) { animation-delay: 120ms; }
  .lobby-copy > :global(*:nth-child(4)) { animation-delay: 180ms; }
  .lobby-copy > :global(*:nth-child(5)) { animation-delay: 240ms; }

  .lobby-h { font-size: clamp(2.4rem, 6vw, 4.1rem); }
  .lobby-h :global(em) { font-family: var(--font-body); font-style: italic; font-weight: 500; text-transform: none; letter-spacing: -0.01em; color: color-mix(in oklch, var(--primary) 80%, var(--ink)); }

  .lobby-tag {
    justify-self: start; font-family: var(--font-mono); text-transform: uppercase;
    letter-spacing: 0.14em; font-size: 0.66rem; color: var(--wood-d);
    padding: 0.34rem 0.7rem; border-radius: 4px;
    background: linear-gradient(180deg, var(--brass), var(--brass-d));
    box-shadow: inset 0 1px 0 rgba(255,255,255,0.5), 0 2px 5px rgba(0,0,0,0.35);
  }
  .lobby-actions { display: flex; flex-wrap: wrap; gap: 0.7rem; margin-top: 0.3rem; }

  .viewing-window {
    border-radius: 16px; padding: clamp(0.8rem, 2vw, 1.1rem);
    background: linear-gradient(160deg, var(--wood-2), var(--wood-d));
    box-shadow: inset 0 1px 0 rgba(255,255,255,0.1), 0 26px 50px -20px rgba(0,0,0,0.7);
    border: 1px solid rgba(0,0,0,0.4);
    animation: revealUp 640ms 120ms var(--ease-out-expo) both;
  }
  .vw-head { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 0.7rem; }
  .vw-sign { font-family: var(--font-body); font-weight: 500; text-transform: uppercase; letter-spacing: 0.2em; font-size: 1.1rem; color: var(--on-dark); }
  .vw-sub  { display: block; font-family: var(--font-mono); font-size: 0.5rem; letter-spacing: 0.2em; text-transform: uppercase; color: color-mix(in oklch, var(--on-dark) 60%, transparent); margin-top: 0.2rem; }
  .vw-badge { font-family: var(--font-mono); font-size: 0.5rem; letter-spacing: 0.16em; text-transform: uppercase; color: var(--wood-d); padding: 0.2rem 0.46rem; border-radius: 3px; background: linear-gradient(180deg, var(--brass), var(--brass-d)); }

  .vw-glass {
    position: relative; border-radius: 10px; padding: clamp(1.6rem, 4vw, 2.6rem) 1rem;
    display: flex; justify-content: center; align-items: center; gap: 0.6rem;
    background: radial-gradient(120% 80% at 50% 0%, rgba(255,255,255,0.06), transparent 60%), linear-gradient(180deg, #181410, #0d0b09);
    box-shadow: inset 0 6px 22px rgba(0,0,0,0.7); overflow: hidden;
  }
  .vw-glass::after { content: ''; position: absolute; inset: 0; background: linear-gradient(120deg, rgba(255,255,255,0.14) 0%, transparent 24%); pointer-events: none; }

  .sample-strip { width: auto; border-radius: 2px; background: var(--enamel); box-shadow: 0 6px 16px rgba(0,0,0,0.55); }
  .sample-strip.s-main { height: clamp(190px, 30vw, 280px); z-index: 2; }
  .sample-strip.s-l    { height: clamp(180px, 28vw, 268px); transform: rotate(-6deg) translateY(12px); z-index: 1; }
  .sample-strip.s-r    { height: clamp(196px, 30vw, 290px); transform: rotate(5deg)  translateY(16px); z-index: 1; }

  .vw-rail { margin-top: 0.8rem; display: grid; grid-template-columns: auto 1fr auto; align-items: center; gap: 0.6rem; padding: 0.5rem 0.85rem; border-radius: 999px; background: color-mix(in oklch, var(--wood) 60%, #000); box-shadow: inset 0 2px 5px rgba(0,0,0,0.5), 0 1px 0 rgba(255,255,255,0.08); }
  .vw-coin { width: 26px; height: 6px; border-radius: 3px; background: #000; box-shadow: inset 0 2px 3px rgba(0,0,0,0.9), 0 1px 0 rgba(255,255,255,0.18); }
  .vw-cap  { font-family: var(--font-mono); font-size: 0.5rem; letter-spacing: 0.16em; text-transform: uppercase; text-align: center; color: color-mix(in oklch, var(--on-dark) 70%, transparent); }
  .vw-led  { width: 11px; height: 11px; border-radius: 50%; background: radial-gradient(circle at 38% 34%, var(--glow-soft), var(--glow) 55%, var(--brass-d)); box-shadow: 0 0 10px var(--glow); animation: bulbFlicker 1.5s ease-in-out infinite; }

  .strip-history { display: grid; gap: 1rem; }
  .history-head { display: flex; justify-content: space-between; align-items: center; gap: 1rem; }
  .history-head :global(h3) { font-size: clamp(1.4rem, 3.6vw, 2.1rem); }

  .strip-grid {
    display: flex;
    flex-wrap: wrap;
    gap: clamp(0.75rem, 2vw, 1.2rem);
    align-items: flex-start;
  }

  .strip-card {
    display: flex; flex-direction: column; align-items: center; gap: 0.5rem;
    background: none; cursor: pointer;
    animation: revealUp 480ms var(--ease-out-quint) both;
    transition: transform 220ms var(--ease-out-quart), box-shadow 220ms var(--ease-out-quart);
  }
  .strip-card:hover { transform: translateY(-5px); }
  .strip-card:hover .strip-card-img {
    box-shadow:
      0 2px 6px rgba(0,0,0,0.3),
      0 20px 40px -10px rgba(0,0,0,0.55),
      0 0 0 2px color-mix(in oklch, var(--glow) 60%, transparent);
  }

  .strip-card-img {
    position: relative;
    border-radius: 8px;
    padding: 6px 6px 10px;
    background: linear-gradient(180deg, #1a1108, #0f0c07);
    box-shadow:
      0 3px 8px rgba(0,0,0,0.3),
      0 12px 28px -8px rgba(0,0,0,0.5),
      inset 0 1px 0 rgba(255,255,255,0.06);
    transition: box-shadow 220ms var(--ease-out-quart);
  }
  .strip-card-img::after {
    content: 'View';
    position: absolute;
    inset: 0;
    display: grid;
    place-items: center;
    border-radius: 8px;
    background: rgba(0,0,0,0.5);
    font-family: var(--font-sign);
    font-size: 0.7rem;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: var(--on-dark);
    opacity: 0;
    transition: opacity 180ms;
  }
  .strip-card:hover .strip-card-img::after { opacity: 1; }

  .strip-card-img img {
    display: block;
    height: clamp(160px, 18vh, 200px);
    width: auto;
    border-radius: 3px;
  }

  .strip-card-date {
    font-family: var(--font-mono); font-size: 0.52rem;
    letter-spacing: 0.12em; text-transform: uppercase;
    color: var(--ink-faint); text-align: center;
  }

  .history-empty { padding: 1.4rem 1.2rem; border-radius: 12px; border: 1.5px dashed color-mix(in oklch, var(--ink) 26%, transparent); display: grid; gap: 0.7rem; justify-items: start; color: var(--ink-soft); font-family: var(--font-body); font-size: 1.05rem; }

  .linklike { font-family: var(--font-mono); text-transform: uppercase; letter-spacing: 0.14em; font-size: 0.6rem; color: var(--ink-soft); padding: 0.4rem 0.7rem; border-radius: 6px; background: color-mix(in oklch, var(--enamel-2) 70%, transparent); box-shadow: inset 0 0 0 1px color-mix(in oklch, var(--ink) 16%, transparent); transition: transform 160ms; }
  .linklike:hover { transform: translateY(-1px); }

  @keyframes revealUp { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
</style>
