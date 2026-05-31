<script>
  import { onMount } from 'svelte';
  import { get } from 'svelte/store';
  import Landing from './lib/Landing.svelte';
  import Selection from './lib/Selection.svelte';
  import Processing from './lib/Processing.svelte';
  import Result from './lib/Result.svelte';
  import { currentPage, currentResultId, photos, finalStrip, selectedFrame, selectedFilter, customEmoji, customColor, customText } from './lib/store.js';

  const PAGE_LABEL = { lobby: 'Lobby', studio: 'Studio', darkroom: 'Darkroom', pickup: 'Pickup' };

  let syncingFromHash = false;

  function setHash(path) {
    const nextHash = `#/${path}`;
    if (window.location.hash !== nextHash) {
      window.history.replaceState(null, '', `${window.location.pathname}${window.location.search}${nextHash}`);
    }
  }

  function applyRouteFromHash() {
    syncingFromHash = true;
    const path = window.location.hash.replace(/^#\/?/, '');
    const parts = path.split('/').filter(Boolean);

    if (parts.length === 0) { currentResultId.set(null); currentPage.set('lobby'); }
    else if (parts[0] === 'studio') { currentResultId.set(null); currentPage.set('studio'); }
    else if (parts[0] === 'darkroom') { currentPage.set('darkroom'); }
    else if (parts[0] === 'result' && parts[1]) {
      const id = Number(parts[1]);
      currentResultId.set(Number.isFinite(id) ? id : null);
      currentPage.set('pickup');
    } else { currentResultId.set(null); currentPage.set('lobby'); }

    syncingFromHash = false;
  }

  function go(next) {
    currentPage.set(next);
    const id = get(currentResultId);
    const map = { lobby: '', studio: 'studio', darkroom: 'darkroom', pickup: id ? `result/${id}` : 'result' };
    setHash(map[next] || '');
    if (next !== 'pickup') window.scrollTo({ top: 0, behavior: 'smooth' });
  }

  function openStrip(s) {
    if (!s?.dataUrl) return;
    finalStrip.set(s.dataUrl);
    currentResultId.set(s.id ?? null);
    go('pickup');
  }

  onMount(() => {
    document.body.dataset.theme = 'afterhours';
    document.body.dataset.type = 'signage';
    document.body.dataset.texture = 'paper';

    applyRouteFromHash();
    window.addEventListener('hashchange', applyRouteFromHash);

    const unsubPage = currentPage.subscribe((page) => {
      if (syncingFromHash) return;
      if (page === 'lobby') setHash('');
      else if (page === 'studio') setHash('studio');
      else if (page === 'darkroom') setHash('darkroom');
      else if (page === 'pickup') {
        const id = get(currentResultId);
        if (id) setHash(`result/${id}`);
      }
    });

    const unsubId = currentResultId.subscribe((id) => {
      if (syncingFromHash) return;
      if (get(currentPage) === 'pickup' && id) setHash(`result/${id}`);
    });

    return () => {
      window.removeEventListener('hashchange', applyRouteFromHash);
      unsubPage();
      unsubId();
    };
  });
</script>

<div class="room">
  <div class="curtain curtain-left"></div>
  <div class="curtain curtain-right"></div>

  <div class="booth">
    <header class="marquee">
      <div class="marquee-bulbs">
        {#each Array(9) as _}
          <span class="bulb"></span>
        {/each}
      </div>
      <div class="marquee-sign">Photobooth</div>
      <div class="marquee-sub">
        <span class="marquee-plate">Est. 1988</span>
        <span class="room-bulb">Now open · {PAGE_LABEL[$currentPage] ?? 'Lobby'}</span>
      </div>
    </header>

    <div class="cabinet">
      <div class="cabinet-inner">
        <div class="cabinet-face">
          {#if $currentPage === 'lobby'}
            <Landing {go} {openStrip} />
          {:else if $currentPage === 'studio'}
            <Selection
              {go}
              filter={$selectedFilter}
              setFilter={(v) => selectedFilter.set(v)}
              frameId={$selectedFrame}
              setFrameId={(v) => selectedFrame.set(v)}
              customEmoji={$customEmoji}
              setCustomEmoji={(v) => customEmoji.set(v)}
              customColor={$customColor}
              setCustomColor={(v) => customColor.set(v)}
              customText={$customText}
              setCustomText={(v) => customText.set(v)}
              setPhotos={(v) => photos.set(v)}
            />
          {:else if $currentPage === 'darkroom'}
            <Processing
              {go}
              photos={$photos}
              filter={$selectedFilter}
              frameId={$selectedFrame}
              customEmoji={$customEmoji}
              customColor={$customColor}
              customText={$customText}
              setFinalStrip={(v) => finalStrip.set(v)}
              setResultId={(v) => currentResultId.set(v)}
            />
          {:else if $currentPage === 'pickup'}
            <Result
              {go}
              finalStrip={$finalStrip}
              setFinalStrip={(v) => finalStrip.set(v)}
              resultId={$currentResultId}
              setResultId={(v) => currentResultId.set(v)}
              setPhotos={(v) => photos.set(v)}
            />
          {/if}
        </div>
      </div>
    </div>
  </div>
</div>
