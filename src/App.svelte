<script lang="ts">
  import { onMount } from "svelte";
  import WaveSurfer from "wavesurfer.js";
  import RegionsPlugin from "wavesurfer.js/dist/plugins/regions.esm.js";
  import Hover from "wavesurfer.js/dist/plugins/hover.esm.js";
  import {
    Play,
    Pause,
    Repeat,
    BrushCleaning,
    FlagTriangleRight,
    Volume,
    Volume2,
    VolumeX,
    Volume1,
    SquarePlus,
  } from "lucide-svelte";

  let wavesurfer: WaveSurfer | null = null;
  let regions: any = null;
  let showClearConfirm = false;
  let isPlaying = false;
  let isLooping = false;
  let volume = 0.4; // valor inicial (40%)
  let hasFile = false;
  const keyBindings = ["Q", "W", "E", "A", "S", "D", "Z", "X", "C"];
  type Marker = { id: string; time: number; key: string };
  let markers: Marker[] = [];

  function getCssVariable(name: string) {
    return getComputedStyle(document.documentElement)
      .getPropertyValue(name)
      .trim();
  }

  function handleFile(event: Event) {
    const input = event.target as HTMLInputElement;
    const file = input?.files?.[0];

    if (!file) {
      hasFile = false;
      return;
    }

  if (file.type !== "audio/ogg" && !file.name.endsWith(".ogg")) {
    alert("Please upload a .ogg file.");
    input.value = ""; // reset input
    hasFile = false;
    return;
  }

    hasFile = true;
    const url = URL.createObjectURL(file);

    markers = []; // reset pads

    wavesurfer?.destroy();
    regions = RegionsPlugin.create();

    wavesurfer = WaveSurfer.create({
      container: "#waveform",
      waveColor: getCssVariable("--color-base-300"),
      progressColor: getCssVariable("--color-neutral"),
      cursorColor: getCssVariable("--color-neutral-content"),
      barWidth: 2,
      normalize: false,
      barHeight: 1,
      cursorWidth: 3,
      url,
      fillParent: true,
      plugins: [
        regions,
        Hover.create({
          lineColor: "#ff0000",
          lineWidth: 2,
          labelBackground: getCssVariable("--color-base-100"),
          labelColor: "#fff",
          labelSize: "11px",
          labelPreferLeft: false,
        }),
      ],
    });

    // setear volumen inicial
    wavesurfer.setVolume(volume);

    wavesurfer.on("finish", () => {
      isPlaying = false;
      if (isLooping) {
        wavesurfer?.play(0);
        isPlaying = true;
      }
    });
  }

  function addMarker() {
    if (!wavesurfer || !regions) return;
    if (markers.length >= keyBindings.length) {
      alert("No more pads available 🥁");
      return;
    }

    const time = wavesurfer.getCurrentTime();
    const key = keyBindings[markers.length];
    const id = `marker-${time}-${key}`;

    markers = [...markers, { id, time, key }];

    regions.addRegion({
      id,
      start: time,
      end: time + 0.2,
      content: key,
      color: getCssVariable("--color-accent"),
      drag: false,
      resize: false,
    });
  }

  function clearMarkers() {
    markers = [];
    regions?.clearRegions();
  }

  function triggerMarker(marker: Marker) {
    if (!wavesurfer) return;
    wavesurfer.setTime(marker.time);
    wavesurfer.play();
    isPlaying = true;
  }

  function togglePlay() {
    if (!wavesurfer) return;
    if (isPlaying) {
      wavesurfer.pause();
      isPlaying = false;
    } else {
      wavesurfer.play();
      isPlaying = true;
    }
  }

  function toggleLoop() {
    isLooping = !isLooping;
  }

  function confirmClear() {
    clearMarkers();
    showClearConfirm = false;
  }

  function cancelClear() {
    showClearConfirm = false;
  }

  function handleClearClick() {
    if (markers.length === 0) return;

    showClearConfirm = true;

    // Ocultar automáticamente después de 3 segundos
    setTimeout(() => {
      showClearConfirm = false;
    }, 5000);
  }

  function handleKeydown(e: KeyboardEvent) {
    const key = e.key.toUpperCase();

    if (key === " ") {
      e.preventDefault();
      togglePlay();
      return;
    }

    if (key === "R") {
      toggleLoop();
      return;
    }

    if (key === "F") {
      addMarker();
      return;
    }

    if (key === "DELETE" || key === "BACKSPACE") {
      handleClearClick();
      return;
    }

    const marker = markers.find((m) => m.key === key);
    if (marker) triggerMarker(marker);
  }

  onMount(() => {
    window.addEventListener("keydown", handleKeydown);
    return () => window.removeEventListener("keydown", handleKeydown);
  });
</script>

<!-- NAVBAR -->
<div class="navbar bg-base-100 shadow-sm">
  <div class="flex-1">
    <a
      class="btn btn-ghost flex-col items-start gap-0 p-2"
      href="https://www.github.com/bemondev/tinysmplr"
      target="_blank"
    >
      <span class="text-sm font-bold text-base-content">tiny.</span>
      <span class="text-lg font-extrabold text-neutral -mt-1">SMPLR</span>
    </a>
  </div>
  <div class="flex-none">
    <ul class="menu menu-horizontal px-1">
      <li>
        <a class="btn btn-ghost" href="https://www.bemon.dev" target="_blank"
          >@bemondev</a
        >
      </li>
    </ul>
  </div>
</div>

<!-- MAIN -->
<div class="min-h-dvh flex items-center justify-center bg-base-200 p-2">
  <div class="card w-full max-w-2xl shadow-xl bg-base-100">
    <div class="card-body items-center text-center gap-6">
      <!-- Input -->
      <div class="flex flex-col md:flex-row gap-4 w-full justify-center">
        <input
          type="file"
          on:change={handleFile}
          class="file-input file-input-bordered w-full md:w-1/2"
          accept=".ogg"
        />
      </div>

      <!-- Waveform o mensaje -->
      <div class="stack w-full h-32">
        {#if !hasFile}
          <div
            class="w-full h-32 flex flex-row gap-2 items-center justify-center text-2xl"
          >
            <SquarePlus class="text-neutral h-10 w-10" />
            <span class="italic">Try uploading something...</span>
          </div>
        {/if}
        <div
          id="waveform"
          class="w-full h-32 bg-base-100 rounded-md align-middle self-center justify-center items-center"
        ></div>
      </div>

      <!-- Controles + Cassette + Volumen -->
      <div
        class="w-full flex flex-col md:flex-row md:justify-between items-center gap-4 md:gap-6"
      >
        <!-- Controles + Cassette juntos en mobile -->
        <div
          class="flex flex-row items-center gap-4 w-full md:w-2/3 justify-center md:justify-start"
        >
          <!-- Controles siempre en fila -->
          <div
            class="flex flex-row gap-4 flex-wrap md:flex-nowrap justify-center md:justify-start"
          >
            <!-- Play -->
            <div class="flex flex-col items-center gap-1">
              <div class="tooltip" data-tip="Play/Pause">
                <button
                  class="btn btn-success btn-circle shadow-md"
                  on:click={togglePlay}
                >
                  {#if isPlaying}
                    <Pause class="w-6 h-6 text-base-100 stroke-3" />
                  {:else}
                    <Play class="w-6 h-6 text-base-100 stroke-3" />
                  {/if}
                </button>
              </div>
              <kbd class="kbd kbd-sm hidden md:block">space</kbd>
            </div>

            <!-- Loop -->
            <div class="flex flex-col items-center gap-1">
              <div class="tooltip" data-tip="Loop">
                <button
                  class="btn btn-warning btn-circle stroke-3 shadow-md {isLooping
                    ? ''
                    : 'btn-outline stroke-3'}"
                  on:click={toggleLoop}
                >
                  <Repeat class="w-6 h-6" />
                </button>
              </div>
              <kbd class="kbd kbd-sm hidden md:block">r</kbd>
            </div>

            <!-- Marker -->
            <div class="flex flex-col items-center gap-1">
              <div class="tooltip" data-tip="Add marker">
                <button
                class="btn btn-info btn-circle shadow-md"
                on:click={addMarker}
              >
                <FlagTriangleRight class="w-6 h-6" />
              </button>
              </div>
              <kbd class="kbd kbd-sm hidden md:block">f</kbd>
            </div>

            <!-- Clear -->
            <div class="flex flex-col items-center gap-1 relative">
              <div class="tooltip" data-tip="Clear markers">
                <button
                class="btn btn-primary btn-circle shadow-md"
                on:click={handleClearClick}
              >
                <BrushCleaning class="w-6 h-6" />
              </button>

              {#if showClearConfirm}
                <div
                  class="absolute top-full mt-2 w-36 p-2 bg-base-100 shadow-lg rounded-lg flex flex-col items-center z-50"
                >
                  <span class="text-sm mb-2">Restart markers?</span>
                  <div class="flex gap-2">
                    <button
                      class="btn btn-sm btn-primary"
                      on:click={confirmClear}>Yes</button
                    >
                    <button class="btn btn-sm btn-ghost" on:click={cancelClear}
                      >No</button
                    >
                  </div>
                </div>
              {/if}
              </div>
              <kbd class="kbd kbd-sm hidden md:block">del</kbd>
            </div>
          </div>

          <!-- Cassette -->
          <div class="flex justify-center md:ml-4 w-40 h-24 flex-shrink-0">
            <div
              class="relative w-40 h-24 bg-base-200 rounded-lg shadow-md flex items-center justify-between px-6"
            >
              <div
                class="w-11 h-11 border-5 border-base-content rounded-full flex items-center justify-center overflow-hidden"
                class:animate-spin={isPlaying}
              >
                <div class="absolute w-6 h-0.5 bg-base-content rotate-0"></div>
                <div class="absolute w-6 h-0.5 bg-base-content rotate-60"></div>
                <div
                  class="absolute w-6 h-0.5 bg-base-content rotate-120"
                ></div>
              </div>
              <div
                class="w-11 h-11 border-5 border-base-content rounded-full flex items-center justify-center overflow-hidden"
                class:animate-spin-reverse={isPlaying}
              >
                <div class="absolute w-6 h-0.5 bg-base-content rotate-0"></div>
                <div class="absolute w-6 h-0.5 bg-base-content rotate-60"></div>
                <div
                  class="absolute w-6 h-0.5 bg-base-content rotate-120"
                ></div>
              </div>
            </div>
          </div>
        </div>

        <!-- Volumen -->
        <div
          class="flex items-center gap-2 w-full md:w-1/3 justify-center md:justify-end"
        >
          {#if Math.round(volume * 100) === 0}
            <VolumeX class="w-5 h-5 text-accent" />
          {:else if Math.round(volume * 100) > 65}
            <Volume2 class="w-5 h-5 text-accent" />
          {:else if Math.round(volume * 100) > 35}
            <Volume1 class="w-5 h-5 text-accent" />
          {:else}
            <Volume class="w-5 h-5 text-accent" />
          {/if}

          <input
            type="range"
            min="0"
            max="1"
            step="0.01"
            bind:value={volume}
            on:input={() => wavesurfer && wavesurfer.setVolume(volume)}
            class="range range-accent w-full"
          />
          <span class="text-sm w-10 text-right">{Math.round(volume * 100)}</span
          >
        </div>
      </div>

      <!-- Pads -->
      <div class="grid grid-cols-3 gap-4 w-full">
        {#each keyBindings as key, i}
          <button
            class="btn py-10 text-4xl {markers[i]
              ? 'btn-accent btn-soft'
              : 'btn-outline'}"
            disabled={!markers[i]}
            on:click={() => markers[i] && triggerMarker(markers[i])}
          >
            {key}
          </button>
        {/each}
      </div>
    </div>
  </div>
</div>
