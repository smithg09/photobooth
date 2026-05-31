<script>
    import { onMount, onDestroy } from "svelte";
    import {
        createMockPhotoDataUrls,
        generateStripDataUrl,
    } from "./strip-renderer.js";

    export let go;
    export let filter;
    export let setFilter;
    export let frameId;
    export let setFrameId;
    export let customEmoji;
    export let setCustomEmoji;
    export let customColor;
    export let setCustomColor;
    export let customText;
    export let setCustomText;
    export let setPhotos;

    const FRAMES = [
        {
            id: "film-strip",
            name: "Film Strip",
            note: "Analog perforations down both sides.",
            reaction: "Peak nostalgia. Arcade-night approved.",
        },
        {
            id: "simple-white",
            name: "Classic White",
            note: "Clean portrait strips, no decoration.",
            reaction: "Clean and timeless.",
        },
        {
            id: "simple-black",
            name: "Noir",
            note: "Dark frame for dramatic, high-contrast looks.",
            reaction: "Moody and cinematic.",
        },
        {
            id: "polaroid",
            name: "Vintage",
            note: "Warm paper tint with retro softness.",
            reaction: "Warm and dreamy. Instant throwback.",
        },
        {
            id: "hearts",
            name: "Hearts",
            note: "Playful accents framing each shot.",
            reaction: "Soft chaos. Date-night approved.",
        },
        {
            id: "sparkles",
            name: "Sparkles",
            note: "Party-ready stars and highlights.",
            reaction: "Party mode activated.",
        },
        {
            id: "sketchy",
            name: "Sketchy",
            note: "Hand-drawn border for casual vibes.",
            reaction: "Messy in the best way.",
        },
        {
            id: "floral",
            name: "Botanical",
            note: "Soft borders with tiny leaf accents.",
            reaction: "Fresh and gentle.",
        },
        {
            id: "cyber",
            name: "Cyber Glitch",
            note: "Neon edges on a dark sci-fi backdrop.",
            reaction: "Loud and electric.",
        },
        {
            id: "arcade",
            name: "8-Bit Arcade",
            note: "Pixel decorations and bright colour.",
            reaction: "High score energy.",
        },
        {
            id: "postage",
            name: "Postage",
            note: "Looks like a rare vintage stamp.",
            reaction: "First-class keepsake.",
        },
        {
            id: "notepad",
            name: "Grid Notepad",
            note: "Lined paper with casual scribbles.",
            reaction: "Sweet and scrappy.",
        },
        {
            id: "custom",
            name: "Custom",
            note: "Stamp your own emoji along the edge.",
            reaction: "Make it unmistakably yours.",
        },
    ];

    const DELIGHT = {
        preview: [
            "Pick your frame, then run a quick rehearsal.",
            "Tip: colour for party glow, B&W for album-cover mood.",
            "Frame first, filter second, then lock your best pose.",
        ],
        camera: [
            "Camera live. Hold steady between countdown beeps.",
            "Every shot saves automatically — keep the energy up.",
            "One serious face, one chaotic face, then repeat.",
        ],
        review: [
            "Reorder the panels, then send it to the darkroom.",
            "Happy with the order? Print to develop your strip.",
            "Swap shots until the rhythm feels right.",
        ],
    };

    const TOTAL = 4;
    const MOCKS = createMockPhotoDataUrls(4);

    let frameIndex = Math.max(
        0,
        FRAMES.findIndex((f) => f.id === frameId),
    );
    let mode = "preview"; // preview | camera | review
    let countdown = null;
    let count = 0;
    let showFlash = false;
    let camReady = false;
    let reviewPhotos = [];
    let previewUrl = "";
    let delight = DELIGHT.preview[0];

    let videoEl;
    let canvasEl;
    let fileEl;
    let replaceEl;
    let replaceIndex = -1;
    let streamRef = null;
    let countTimer = null;
    let queueTimer = null;
    let shotsRef = [];
    let delightTimer = null;
    let previewAlive = false;

    $: frame = FRAMES[frameIndex];
    $: modeLabel =
        mode === "preview"
            ? "Compose"
            : mode === "camera"
              ? "Camera session"
              : "Review session";
    $: camTip =
        count === 0
            ? "Opening shot — hold a confident pose."
            : count === TOTAL - 1
              ? "Final shot — give it your most expressive look."
              : "Keep changing expressions so each panel feels different.";

    $: {
        // sync frameId up to parent when frame changes
        if (frame) setFrameId(frame.id);
    }

    $: {
        // rotate delight hint on mode change
        clearInterval(delightTimer);
        const lines = DELIGHT[mode] || DELIGHT.preview;
        delight = lines[0];
        let i = 0;
        if (lines.length >= 2) {
            delightTimer = setInterval(() => {
                i = (i + 1) % lines.length;
                delight = lines[i];
            }, 2400);
        }
    }

    $: {
        // regenerate live preview when settings change
        if (mode !== "camera") {
            previewAlive = false;
            const source =
                mode === "review" && reviewPhotos.length > 0
                    ? reviewPhotos
                    : MOCKS;
            const curFrame = frame;
            const curFilter = filter;
            const curEmoji = customEmoji;
            const curColor = customColor;
            const curText = customText;
            previewAlive = true;
            generateStripDataUrl({
                photos: source,
                selectedFilter: curFilter,
                selectedFrame: curFrame.id,
                customEmoji: curEmoji,
                customColor: curColor,
                footerLabel: curText || "PREVIEW",
            })
                .then((url) => {
                    if (previewAlive) previewUrl = url || "";
                })
                .catch(() => {
                    if (previewAlive) previewUrl = "";
                });
        }
    }

    function changeFrame(dir) {
        if (mode !== "preview") return;
        frameIndex = (frameIndex + dir + FRAMES.length) % FRAMES.length;
    }

    async function startCapture() {
        if (!navigator.mediaDevices?.getUserMedia) {
            alert("Camera capture is not available in this browser.");
            return;
        }
        mode = "camera";
        reviewPhotos = [];
        camReady = false;
        shotsRef = [];
        try {
            const stream = await navigator.mediaDevices.getUserMedia({
                video: {
                    width: { ideal: 640 },
                    height: { ideal: 480 },
                    facingMode: "user",
                },
                audio: false,
            });
            streamRef = stream;
            if (videoEl) {
                videoEl.srcObject = stream;
                await videoEl.play();
            }
            camReady = true;
            count = 0;
            setPhotos([]);
            nextShot(0);
        } catch {
            stopCamera();
            alert("Please allow camera access to take photos.");
            mode = "preview";
        }
    }

    function nextShot(currentCount) {
        if (currentCount >= TOTAL) {
            finishToReview();
            return;
        }
        countdown = 3;
        let c = 3;
        clearInterval(countTimer);
        countTimer = setInterval(() => {
            c -= 1;
            countdown = c;
            if (c <= 0) {
                clearInterval(countTimer);
                takePhoto(currentCount);
            }
        }, 1000);
    }

    function takePhoto(currentCount) {
        const video = videoEl,
            canvas = canvasEl;
        if (!video?.videoWidth) return;
        showFlash = true;
        setTimeout(() => {
            showFlash = false;
        }, 180);
        const ctx = canvas.getContext("2d");
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        ctx.save();
        ctx.translate(canvas.width, 0);
        ctx.scale(-1, 1);
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
        ctx.restore();
        const dataUrl = canvas.toDataURL("image/jpeg", 0.9);
        shotsRef = [...shotsRef, dataUrl];
        const newCount = currentCount + 1;
        count = newCount;
        clearTimeout(queueTimer);
        if (newCount < TOTAL)
            queueTimer = setTimeout(() => nextShot(newCount), 900);
        else queueTimer = setTimeout(finishToReview, 600);
    }

    function stopCamera() {
        camReady = false;
        countdown = null;
        clearInterval(countTimer);
        clearTimeout(queueTimer);
        if (streamRef) {
            streamRef.getTracks().forEach((t) => t.stop());
            streamRef = null;
        }
    }

    function finishToReview() {
        stopCamera();
        const shots = shotsRef.slice(0, TOTAL);
        while (shots.length && shots.length < TOTAL)
            shots.push(shots[shots.length - 1]);
        reviewPhotos = shots;
        setPhotos(shots);
        mode = "review";
    }

    function triggerUpload() {
        stopCamera();
        fileEl?.click();
    }

    function readFileAsUrl(file) {
        return new Promise((res, rej) => {
            const r = new FileReader();
            r.onload = (e) => res(e.target.result);
            r.onerror = rej;
            r.readAsDataURL(file);
        });
    }

    async function onUpload(e) {
        const files = Array.from(e.target.files || []);
        if (files.length === 0) return;
        try {
            const loaded = await Promise.all(
                files.slice(0, 4).map(readFileAsUrl),
            );
            const norm = [...loaded];
            while (norm.length < 4) norm.push(norm[norm.length - 1]);
            reviewPhotos = norm;
            setPhotos(norm);
            mode = "review";
        } catch {
            alert("Could not load those images. Try different files.");
        }
        e.target.value = "";
    }

    function swap(a, b) {
        if (
            a < 0 ||
            b < 0 ||
            a >= reviewPhotos.length ||
            b >= reviewPhotos.length
        )
            return;
        const next = [...reviewPhotos];
        [next[a], next[b]] = [next[b], next[a]];
        reviewPhotos = next;
        setPhotos(next);
    }

    function triggerReplace(i) {
        replaceIndex = i;
        replaceEl?.click();
    }

    async function onReplace(e) {
        const file = e.target.files?.[0];
        if (!file || replaceIndex < 0) return;
        try {
            const url = await readFileAsUrl(file);
            const next = [...reviewPhotos];
            next[replaceIndex] = url;
            reviewPhotos = next;
            setPhotos(next);
        } catch { alert('Could not load that image. Try a different file.'); }
        e.target.value = '';
        replaceIndex = -1;
    }

    function reset() {
        stopCamera();
        reviewPhotos = [];
        mode = "preview";
    }

    onDestroy(() => {
        stopCamera();
        clearInterval(delightTimer);
        previewAlive = false;
    });
</script>

<div class="screen studio page-enter">
    <div class="studio-top">
        <button
            class="chip-btn"
            on:click={() => {
                stopCamera();
                go("lobby");
            }}>← Back to lobby</button
        >
        <span class="mode-tab">{modeLabel}</span>
    </div>

    <div class="studio-grid">
        <section class="console">
            <h2 class="display-title">Compose your strip</h2>
            <p class="serif-lead" style="font-size: 1.02rem;">
                Pick a frame, choose your film, then capture or upload four
                shots.
            </p>
            <p class="console-hint">{delight}</p>

            <div class="dial">
                <button
                    class="dial-arrow"
                    on:click={() => changeFrame(-1)}
                    disabled={mode !== "preview"}
                    aria-label="Previous frame">‹</button
                >
                <div class="dial-face">
                    <p class="dial-name">{frame.name}</p>
                    {#if frame.id === "custom"}
                        <div class="dial-custom">
                            <label
                                >Emoji
                                <input
                                    class="emoji-in"
                                    value={customEmoji}
                                    maxlength="5"
                                    on:input={(e) =>
                                        setCustomEmoji(e.target.value)}
                                />
                            </label>
                            <label
                                >Colour
                                <input
                                    class="color-in"
                                    type="color"
                                    value={customColor}
                                    on:input={(e) =>
                                        setCustomColor(e.target.value)}
                                />
                            </label>
                        </div>
                    {:else}
                        <p class="dial-note">{frame.note}</p>
                        <p class="dial-reaction">{frame.reaction}</p>
                    {/if}
                </div>
                <button
                    class="dial-arrow"
                    on:click={() => changeFrame(1)}
                    disabled={mode !== "preview"}
                    aria-label="Next frame">›</button
                >
            </div>

            <div class="switch-row">
                <span class="switch-label">Film</span>
                <div class="film-toggle">
                    <span
                        class="knob"
                        style="transform: {filter === 'bw'
                            ? 'translateX(calc(100% + 2px))'
                            : 'translateX(0)'}"
                    ></span>
                    <button
                        class={filter === "color" ? "on" : ""}
                        on:click={() => setFilter("color")}>Colour</button
                    >
                    <button
                        class={filter === "bw" ? "on" : ""}
                        on:click={() => setFilter("bw")}>B&amp;W</button
                    >
                </div>
            </div>

            <div class="field">
                <label for="strip-label">Strip label</label>
                <input
                    id="strip-label"
                    class="emboss-input"
                    value={customText}
                    maxlength="30"
                    placeholder="Add custom text…"
                    on:input={(e) => setCustomText(e.target.value)}
                />
            </div>

            <div class="meta-chips">
                <span>4 shots per strip</span>
                <span>Auto-save enabled</span>
            </div>

            <div class="console-actions">
                {#if mode === "preview"}
                    <button
                        class="btn btn--primary btn--capture"
                        on:click={startCapture}>Take photos</button
                    >
                    <button class="btn btn--ghost" on:click={triggerUpload}
                        >Upload images</button
                    >
                {:else if mode === "camera"}
                    <button class="btn btn--ghost" on:click={reset}
                        >Cancel capture</button
                    >
                {:else if mode === "review"}
                    <div class="reorder">
                        <p class="reorder-title">Reorder shots</p>
                        <div class="reorder-track">
                            {#each reviewPhotos as thumb, i}
                                <div class="reorder-item">
                                    <img src={thumb} alt="shot {i + 1}" />
                                    <span class="n">{i + 1}</span>
                                    <div class="reorder-actions">
                                        <button
                                            disabled={i === 0}
                                            on:click={() => swap(i, i - 1)}
                                            aria-label="Move earlier">‹</button
                                        >
                                        <button
                                            disabled={i ===
                                                reviewPhotos.length - 1}
                                            on:click={() => swap(i, i + 1)}
                                            aria-label="Move later">›</button
                                        >
                                        <button
                                            class="replace-btn"
                                            on:click={() => triggerReplace(i)}
                                            aria-label="Replace photo"><svg width="12" height="12" viewBox="0 0 12 12" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M6 8.5V2M3.5 4L6 2L8.5 4M2 10.5h8" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg></button
                                        >
                                    </div>
                                </div>
                            {/each}
                        </div>
                    </div>
                    <button
                        class="btn btn--primary btn--capture"
                        on:click={() => go("darkroom")}
                        disabled={reviewPhotos.length === 0}>Print strip</button
                    >
                    <button class="btn btn--ghost" on:click={triggerUpload}
                        >Upload different</button
                    >
                    <button class="btn btn--ghost" on:click={reset}
                        >Discard</button
                    >
                {/if}
            </div>
        </section>

        <section class="monitor">
            <div class="monitor-bezel">
                <div class="monitor-head">
                    <span>Live preview</span>
                    <span>{filter === "bw" ? "B&W" : "Colour"}</span>
                </div>
                <span class="monitor-sticker">Snap Magic</span>

                <div class="monitor-screen {mode === 'camera' ? 'live' : ''}">
                    {#if mode === "camera"}
                        <div class="cam">
                            <video
                                bind:this={videoEl}
                                autoplay
                                playsinline
                                muted
                            ></video>
                            <div class="cam-brackets">
                                <b class="tl"></b><b class="tr"></b><b
                                    class="bl"
                                ></b><b class="br"></b>
                            </div>
                            {#if !camReady}
                                <div class="cam-overlay">
                                    <span class="cam-note">Opening camera…</span
                                    >
                                </div>
                            {/if}
                            {#if countdown !== null && countdown > 0}
                                {#key countdown}
                                    <div class="cam-overlay">
                                        <span class="count-ring"
                                            >{countdown}</span
                                        >
                                    </div>
                                {/key}
                            {/if}
                            {#if showFlash}
                                <div class="flash"></div>
                            {/if}
                        </div>
                    {:else if previewUrl}
                        <img
                            class="preview-strip"
                            src={previewUrl}
                            alt="Strip preview"
                        />
                    {:else}
                        <span class="preview-loading">Building preview…</span>
                    {/if}
                </div>

                <div class="monitor-foot">
                    {#if mode === "camera"}
                        Photo {Math.min(count + 1, TOTAL)} / {TOTAL} · {camTip}
                    {:else if filter === "bw"}
                        Monochrome keeps tones cinematic.
                    {:else}
                        Colour keeps every tone and accent.
                    {/if}
                </div>
            </div>
        </section>
    </div>

    <canvas bind:this={canvasEl} style="display:none"></canvas>
    <input
        bind:this={fileEl}
        type="file"
        accept="image/*"
        multiple
        on:change={onUpload}
        style="display:none"
    />
    <input bind:this={replaceEl} type="file" accept="image/*" on:change={onReplace} style="display:none" />
</div>

<style>
    .studio {
        display: grid;
        gap: clamp(0.9rem, 2vw, 1.3rem);
    }
    .studio-top {
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 0.8rem;
    }

    .chip-btn {
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.14em;
        font-size: 0.62rem;
        padding: 0.5rem 0.85rem;
        border-radius: 8px;
        color: var(--ink);
        background: color-mix(in oklch, var(--enamel-2) 80%, transparent);
        box-shadow: inset 0 0 0 1px
            color-mix(in oklch, var(--ink) 18%, transparent);
        min-height: 40px;
        transition: transform 160ms;
    }
    .chip-btn:hover {
        transform: translateY(-1px);
    }
    .mode-tab {
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.18em;
        font-size: 0.6rem;
        color: color-mix(in oklch, var(--primary) 76%, var(--ink));
        padding: 0.42rem 0.8rem;
        border-radius: 999px;
        background: color-mix(in oklch, var(--primary) 12%, transparent);
        box-shadow: inset 0 0 0 1px
            color-mix(in oklch, var(--primary) 26%, transparent);
    }

    .studio-grid {
        display: grid;
        grid-template-columns: 1fr;
        gap: clamp(1rem, 2.4vw, 1.5rem);
        align-items: start;
    }
    @media (min-width: 900px) {
        .studio-grid {
            grid-template-columns: 0.82fr 1.18fr;
        }
    }

    .console {
        position: relative;
        border-radius: var(--radius-panel);
        padding: clamp(1rem, 2.4vw, 1.5rem);
        background: linear-gradient(
            180deg,
            color-mix(in oklch, var(--enamel) 96%, var(--brass) 4%),
            var(--enamel-2)
        );
        box-shadow:
            inset 0 1px 0 rgba(255, 255, 255, 0.7),
            0 18px 30px -18px rgba(0, 0, 0, 0.5),
            inset 0 0 0 1px color-mix(in oklch, var(--ink) 10%, transparent);
        display: grid;
        gap: 0.85rem;
    }
    .console :global(h2) {
        font-size: clamp(1.6rem, 2.4vw, 2.3rem);
    }
    .console-hint {
        width: fit-content;
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.1em;
        font-size: 0.58rem;
        line-height: 1.4;
        color: color-mix(in oklch, var(--primary) 70%, var(--ink));
        padding: 0.3rem 0.6rem;
        border-radius: 6px;
        background: color-mix(in oklch, var(--primary) 9%, transparent);
    }

    .dial {
        display: grid;
        grid-template-columns: auto 1fr auto;
        align-items: center;
        gap: 0.7rem;
        padding: 0.7rem;
        border-radius: 12px;
        background: linear-gradient(
            180deg,
            color-mix(in oklch, var(--wood) 18%, var(--enamel)),
            color-mix(in oklch, var(--wood) 10%, var(--enamel-2))
        );
        box-shadow:
            inset 0 0 0 1px color-mix(in oklch, var(--ink) 14%, transparent),
            inset 0 2px 4px rgba(0, 0, 0, 0.12);
    }
    .dial-arrow {
        width: 42px;
        height: 42px;
        border-radius: 50%;
        display: grid;
        place-items: center;
        font-size: 1.3rem;
        color: var(--ink);
        background: radial-gradient(
            circle at 38% 30%,
            var(--brass),
            var(--brass-2) 60%,
            var(--brass-d)
        );
        box-shadow:
            inset 0 1px 0 rgba(255, 255, 255, 0.5),
            0 3px 6px -2px rgba(0, 0, 0, 0.5);
        transition: transform 150ms;
    }
    .dial-arrow:hover:not(:disabled) {
        transform: scale(1.06);
    }
    .dial-arrow:active:not(:disabled) {
        transform: scale(0.96);
    }
    .dial-arrow:disabled {
        opacity: 0.4;
        cursor: not-allowed;
    }
    .dial-face {
        text-align: center;
        display: grid;
        gap: 0.15rem;
    }
    .dial-name {
        font-family: var(--font-sign);
        font-weight: 600;
        text-transform: uppercase;
        font-size: 1.2rem;
        letter-spacing: 0.02em;
        color: var(--ink);
    }
    .dial-note {
        font-family: var(--font-body);
        font-size: 0.86rem;
        line-height: 1.25;
        color: var(--ink-soft);
    }
    .dial-reaction {
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.1em;
        font-size: 0.52rem;
        color: color-mix(in oklch, var(--primary) 68%, var(--ink));
    }
    .dial-custom {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 0.6rem;
        margin-top: 0.2rem;
    }
    .dial-custom label {
        font-family: var(--font-mono);
        font-size: 0.56rem;
        text-transform: uppercase;
        letter-spacing: 0.1em;
        color: var(--ink-soft);
    }
    .emoji-in {
        width: 3rem;
        font-size: 1.2rem;
        text-align: center;
        padding: 0.2rem;
        border-radius: 6px;
        border: 1.5px solid color-mix(in oklch, var(--ink) 24%, transparent);
        background: var(--enamel);
        color: var(--ink);
    }
    .color-in {
        width: 2rem;
        height: 2rem;
        border: none;
        background: none;
        padding: 0;
        cursor: pointer;
    }

    .switch-row {
        display: flex;
        align-items: center;
        gap: 0.7rem;
    }
    .switch-label {
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.12em;
        font-size: 0.58rem;
        color: var(--ink-soft);
    }
    .film-toggle {
        position: relative;
        display: inline-grid;
        grid-template-columns: 1fr 1fr;
        gap: 2px;
        padding: 3px;
        border-radius: 999px;
        background: color-mix(in oklch, var(--wood) 22%, var(--enamel-2));
        box-shadow:
            inset 0 2px 4px rgba(0, 0, 0, 0.2),
            inset 0 0 0 1px color-mix(in oklch, var(--ink) 16%, transparent);
    }
    .film-toggle button {
        position: relative;
        z-index: 1;
        font-family: var(--font-sign);
        text-transform: uppercase;
        letter-spacing: 0.06em;
        font-size: 0.82rem;
        padding: 0.4rem 1.1rem;
        border-radius: 999px;
        color: var(--ink-soft);
        min-height: 38px;
        transition: color 200ms;
    }
    .film-toggle button.on {
        color: var(--on-dark);
    }
    .knob {
        position: absolute;
        z-index: 0;
        top: 3px;
        bottom: 3px;
        left: 3px;
        width: calc(50% - 4px);
        border-radius: 999px;
        background: linear-gradient(180deg, var(--primary-l), var(--primary-d));
        box-shadow:
            inset 0 1px 0 rgba(255, 255, 255, 0.25),
            0 2px 5px -1px rgba(0, 0, 0, 0.4);
        transition: transform 240ms var(--ease-out-quint);
    }

    .field :global(label) {
        display: block;
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.12em;
        font-size: 0.56rem;
        color: var(--ink-soft);
        margin-bottom: 0.3rem;
    }
    .emboss-input {
        width: 100%;
        font-family: var(--font-mono);
        font-size: 0.92rem;
        padding: 0.55rem 0.7rem;
        border-radius: 8px;
        color: var(--ink);
        border: none;
        background: color-mix(in oklch, var(--enamel-2) 70%, var(--wood) 8%);
        box-shadow:
            inset 0 2px 4px rgba(0, 0, 0, 0.16),
            inset 0 0 0 1px color-mix(in oklch, var(--ink) 14%, transparent);
    }
    .emboss-input:focus {
        outline: none;
        box-shadow:
            inset 0 2px 4px rgba(0, 0, 0, 0.16),
            0 0 0 2px var(--glow);
    }

    .meta-chips {
        display: flex;
        flex-wrap: wrap;
        gap: 0.4rem;
    }
    .meta-chips span {
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.1em;
        font-size: 0.54rem;
        color: var(--ink-soft);
        padding: 0.26rem 0.5rem;
        border-radius: 5px;
        background: color-mix(in oklch, var(--enamel-2) 76%, transparent);
        box-shadow: inset 0 0 0 1px
            color-mix(in oklch, var(--ink) 12%, transparent);
    }

    .console-actions {
        display: grid;
        gap: 0.5rem;
        margin-top: 0.2rem;
    }
    .btn--capture {
        font-size: 1.05rem;
        padding: 0.95rem 1.5rem;
        position: relative;
        overflow: hidden;
    }
    .btn--capture::before {
        content: "";
        position: absolute;
        top: 0;
        left: -40%;
        width: 40%;
        height: 100%;
        background: linear-gradient(
            90deg,
            transparent,
            rgba(255, 255, 255, 0.35),
            transparent
        );
        animation: sheen 3.2s ease-in-out infinite;
    }

    .reorder {
        display: grid;
        gap: 0.5rem;
        padding: 0.7rem;
        border-radius: 10px;
        background: color-mix(in oklch, var(--enamel-2) 70%, transparent);
        box-shadow: inset 0 0 0 1px
            color-mix(in oklch, var(--ink) 12%, transparent);
    }
    .reorder-title {
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.12em;
        font-size: 0.54rem;
        color: var(--ink-soft);
    }
    .reorder-track {
        display: flex;
        gap: 0.4rem;
    }
    .reorder-item {
        flex: 1;
        display: grid;
        gap: 0.3rem;
        justify-items: center;
        padding: 0.3rem;
        border-radius: 7px;
        background: var(--enamel);
        box-shadow: inset 0 0 0 1px
            color-mix(in oklch, var(--ink) 12%, transparent);
    }
    .reorder-item img {
        width: 100%;
        aspect-ratio: 3/4;
        object-fit: cover;
        border-radius: 3px;
        background: var(--ink);
    }
    .reorder-item .n {
        font-family: var(--font-mono);
        font-weight: 600;
        font-size: 0.56rem;
        color: var(--primary);
    }
    .reorder-actions {
        display: flex;
        gap: 0.25rem;
    }
    .reorder-actions button {
        width: 26px;
        height: 26px;
        display: grid;
        place-items: center;
        border-radius: 5px;
        background: linear-gradient(180deg, var(--brass), var(--brass-d));
        color: var(--wood-d);
        box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.4);
    }
    .reorder-actions button:disabled {
        opacity: 0.35;
        cursor: not-allowed;
    }
    .replace-btn {
        background: linear-gradient(180deg, var(--primary-l), var(--primary-d)) !important;
        color: var(--on-dark) !important;
        font-size: 0.9rem;
    }

    .monitor {
        display: grid;
        justify-items: center;
        gap: 0.7rem;
    }
    .monitor-bezel {
        width: min(420px, 100%);
        border-radius: 18px;
        padding: clamp(0.9rem, 2vw, 1.3rem);
        background: linear-gradient(165deg, #2a2622, #14110e);
        box-shadow:
            inset 0 1px 0 rgba(255, 255, 255, 0.08),
            0 28px 50px -22px rgba(0, 0, 0, 0.75),
            inset 0 0 0 1px rgba(0, 0, 0, 0.5);
        display: grid;
        gap: 0.7rem;
    }
    .monitor-head {
        display: flex;
        justify-content: space-between;
        align-items: center;
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.16em;
        font-size: 0.54rem;
        color: color-mix(in oklch, var(--on-dark) 66%, transparent);
    }
    .monitor-sticker {
        align-self: center;
        font-family: var(--font-sign);
        text-transform: uppercase;
        letter-spacing: 0.08em;
        font-size: 0.62rem;
        color: var(--on-dark);
        padding: 0.18rem 0.5rem;
        border-radius: 4px;
        background: linear-gradient(180deg, var(--primary-l), var(--primary-d));
        transform: rotate(-3deg);
    }
    .monitor-screen {
        position: relative;
        border-radius: 10px;
        height: clamp(340px, 70vh, 750px);
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 1rem;
        background:
            radial-gradient(
                120% 80% at 50% 0%,
                rgba(255, 255, 255, 0.05),
                transparent 55%
            ),
            linear-gradient(180deg, #e8e2d6, #d8d0c2);
        box-shadow: inset 0 4px 16px rgba(0, 0, 0, 0.28);
        overflow: hidden;
    }
    .monitor-screen.live {
        background: #000;
    }
    .preview-strip {
        max-height: 100%;
        width: auto;
        object-fit: contain;
        filter: drop-shadow(0 14px 22px rgba(0, 0, 0, 0.4));
        transform: rotate(-2deg);
        animation: floatStrip 4.5s ease-in-out infinite;
    }
    .monitor-foot {
        text-align: center;
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.14em;
        font-size: 0.54rem;
        color: color-mix(in oklch, var(--on-dark) 60%, transparent);
    }

    .cam {
        position: absolute;
        inset: 0;
    }
    .cam video {
        width: 100%;
        height: 100%;
        object-fit: cover;
        transform: scaleX(-1);
    }
    .cam-overlay {
        position: absolute;
        inset: 0;
        display: grid;
        place-items: center;
    }
    .cam-note {
        font-family: var(--font-mono);
        text-transform: uppercase;
        letter-spacing: 0.14em;
        font-size: 0.7rem;
        color: var(--on-dark);
        background: rgba(0, 0, 0, 0.6);
        padding: 0.5rem 0.9rem;
        border-radius: 8px;
    }
    .count-ring {
        width: 92px;
        aspect-ratio: 1;
        border-radius: 50%;
        display: grid;
        place-items: center;
        border: 3px solid color-mix(in oklch, var(--glow) 80%, #fff);
        background: rgba(0, 0, 0, 0.45);
        color: #fff;
        font-family: var(--font-sign);
        font-weight: 600;
        font-size: 2.6rem;
        animation: pop 320ms var(--ease-out-quint);
    }
    .flash {
        position: absolute;
        inset: 0;
        background: #fff;
        animation: flash 190ms ease-out forwards;
    }
    .cam-brackets b {
        position: absolute;
        width: 26px;
        height: 26px;
        border: 2px solid rgba(255, 255, 255, 0.7);
    }
    .cam-brackets b.tl {
        top: 12px;
        left: 12px;
        border-right: none;
        border-bottom: none;
    }
    .cam-brackets b.tr {
        top: 12px;
        right: 12px;
        border-left: none;
        border-bottom: none;
    }
    .cam-brackets b.bl {
        bottom: 12px;
        left: 12px;
        border-right: none;
        border-top: none;
    }
    .cam-brackets b.br {
        bottom: 12px;
        right: 12px;
        border-left: none;
        border-top: none;
    }

    @keyframes floatStrip {
        0%,
        100% {
            transform: rotate(-2deg) translateY(0);
        }
        50% {
            transform: rotate(-1.2deg) translateY(-5px);
        }
    }
    @keyframes sheen {
        0% {
            left: -40%;
        }
        55%,
        100% {
            left: 130%;
        }
    }
    @keyframes pop {
        from {
            transform: scale(1.2);
            opacity: 0;
        }
        to {
            transform: scale(1);
            opacity: 1;
        }
    }
    @keyframes flash {
        from {
            opacity: 1;
        }
        to {
            opacity: 0;
        }
    }
</style>
