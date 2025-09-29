<!-- +page.svelte -->
<script>
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';

  /* =========================
     State
  ========================== */
  let value = 1;
  let listening = false;
  let supported = true;

  // Axes toggles
  let trackX = true, trackY = true;

  // Sensitivity & timing (tweak in UI)
  let sensitivity = 13;     // threshold for "shake" (lower = easier)
  let cooldownMs = 700;     // time between rolls
  let lastRoll = 0;

  // Live sensor values (smoothed)
  let ax=0, ay=0, az=0;           // acceleration (no gravity)
  let gx=0, gy=0, gz=0;           // accelerationIncludingGravity
  let gamma=0;                    // orientation (for subtle twist)

  // Calibration (baseline offsets for gravity-including axes)
  let baseGx=0, baseGy=0;

  // Visual transform (interpolated per frame)
  let transform = 'translate3d(0,0,0) rotate(0deg)';
  let spinClass = '';
  let spinDurMs = 450; // adaptive per shake strength

  /* =========================
     Helpers
  ========================== */
  const clamp = (n, min, max) => Math.max(min, Math.min(max, n));
  const lerp = (a, b, t) => a + (b - a) * t;

  // low-pass smoothing
  const SMOOTH = 0.25;
  let sAx=0, sAy=0, sAz=0, sGx=0, sGy=0, sGz=0, sGamma=0;

  function smoothUpdate(a, b) { return a + (b - a) * SMOOTH; }

  // Map sensor → die target transform
  function makeTargetTransform() {
    // subtract calibration so "level" phone ≈ center
    const cgx = gx - baseGx;
    const cgy = gy - baseGy;

    // Map gravity-including axes to px translation (feels physical)
    const tx = trackY ? clamp(cgx * 2.0, -14, 14) : 0;     // left/right
    const ty = trackX ? clamp(-cgy * 2.0, -14, 14) : 0;    // up/down
    const rz = clamp(gamma * 0.4, -15, 15);                // subtle twist

    return { tx, ty, rz };
  }

  function applyTransform({ tx, ty, rz }) {
    transform = `translate3d(${tx.toFixed(1)}px, ${ty.toFixed(1)}px, 0) rotate(${rz.toFixed(1)}deg)`;
  }

  function roll() {
    value = 1 + Math.floor(Math.random() * 6);
  }

  function triggerSpin(magnitude=1) {
    // adapt duration by shake strength (cap 300–700ms)
    spinDurMs = clamp(300 + magnitude * 120, 300, 700);
    // restart animation by toggling class
    spinClass = '';
    requestAnimationFrame(() => { spinClass = 'animate-dice-spin'; });
  }

  /* =========================
     Motion / Orientation
  ========================== */
  function handleMotion(e) {
    const a = e.acceleration || {};
    const ag = e.accelerationIncludingGravity || {};

    sAx = smoothUpdate(sAx, a.x ?? 0);
    sAy = smoothUpdate(sAy, a.y ?? 0);
    sAz = smoothUpdate(sAz, a.z ?? 0);

    sGx = smoothUpdate(sGx, ag.x ?? 0);
    sGy = smoothUpdate(sGy, ag.y ?? 0);
    sGz = smoothUpdate(sGz, ag.z ?? 0);

    ax = sAx; ay = sAy; az = sAz;
    gx = sGx; gy = sGy; gz = sGz;

    // Shake detection uses acceleration (no gravity)
    const strength = Math.max(Math.abs(ax), Math.abs(ay));
    const now = Date.now();
    const hitX = trackX && Math.abs(ax) > sensitivity;
    const hitY = trackY && Math.abs(ay) > sensitivity;

    if ((hitX || hitY) && now - lastRoll > cooldownMs) {
      lastRoll = now;
      roll();
      triggerSpin(strength);
    }
  }

  function handleOrientation(e) {
    sGamma = smoothUpdate(sGamma, e.gamma ?? 0);
    gamma = sGamma;
  }

  async function start() {
    if (!browser) return;
    supported = 'DeviceMotionEvent' in window || 'DeviceOrientationEvent' in window;

    if (!supported) return;

    // iOS permission gate
    try {
      // @ts-ignore - requestPermission exists on iOS Safari
      if (window.DeviceMotionEvent?.requestPermission) {
        const p = await window.DeviceMotionEvent.requestPermission();
        if (p !== 'granted') return;
      }
    } catch { /* ignore */ }

    window.addEventListener('devicemotion', handleMotion, { passive: true });
    window.addEventListener('deviceorientation', handleOrientation, { passive: true });
    listening = true;
  }

  function stop() {
    if (!browser) return;
    window.removeEventListener('devicemotion', handleMotion);
    window.removeEventListener('deviceorientation', handleOrientation);
    listening = false;
  }

  function calibrate() {
    // capture current gravity baseline (phone at rest)
    baseGx = gx;
    baseGy = gy;
  }

  // Animation loop: interpolate toward current target transform for smooth UX
  let raf;
  function tick() {
    const { tx, ty, rz } = makeTargetTransform();

    // simple easing per frame (lerp from current numeric transform)
    // parse current transform back to numbers (kept tiny and fast)
    const m = /translate3d\((-?\d+\.?\d*)px,\s*(-?\d+\.?\d*)px,.*rotate\((-?\d+\.?\d*)deg\)/.exec(transform);
    const curTx = m ? parseFloat(m[1]) : 0;
    const curTy = m ? parseFloat(m[2]) : 0;
    const curRz = m ? parseFloat(m[3]) : 0;

    applyTransform({
      tx: lerp(curTx, tx, 0.25),
      ty: lerp(curTy, ty, 0.25),
      rz: lerp(curRz, rz, 0.20),
    });

    raf = requestAnimationFrame(tick);
  }

  onMount(() => {
    start();
    raf = requestAnimationFrame(tick);
  });

  onDestroy(() => {
    stop();
    if (raf) cancelAnimationFrame(raf);
  });

  const pips = {
    1:[4], 2:[0,8], 3:[0,4,8], 4:[0,2,6,8], 5:[0,2,4,6,8], 6:[0,2,3,5,6,8]
  };
</script>

<div class="min-h-screen bg-gray-50 flex items-center justify-center p-6">
  <div class="w-full max-w-md space-y-5">
    <!-- Header / status -->
    <div class="flex items-center justify-between">
      <h1 class="text-lg font-semibold text-gray-900">Shaking Dice</h1>
      {#if !supported}
        <span class="text-xs px-2 py-1 rounded bg-red-100 text-red-700">Motion not supported</span>
      {:else if !listening}
        <span class="text-xs px-2 py-1 rounded bg-amber-100 text-amber-800">Enable motion</span>
      {:else}
        <span class="text-xs px-2 py-1 rounded bg-emerald-100 text-emerald-800">Live</span>
      {/if}
    </div>

    <!-- Controls -->
    <div class="rounded-xl bg-white shadow p-3 flex flex-wrap gap-3 items-center">
      {#if !listening}
        <button on:click={start} class="px-3 py-1.5 rounded-lg bg-gray-900 text-white text-sm">Enable motion</button>
      {/if}
      <button on:click={roll} class="px-3 py-1.5 rounded-lg border border-gray-300 text-sm">Roll</button>

      <label class="flex items-center gap-2 ml-auto text-sm">
        <input type="checkbox" bind:checked={trackX} class="accent-gray-800"> X
      </label>
      <label class="flex items-center gap-2 text-sm">
        <input type="checkbox" bind:checked={trackY} class="accent-gray-800"> Y
      </label>
    </div>

    <!-- Tuning -->
    <div class="rounded-xl bg-white shadow p-3 space-y-3">
      <div class="flex items-center gap-3">
        <label class="text-sm text-gray-700 w-24">Sensitivity</label>
        <input type="range" min="6" max="25" step="1" bind:value={sensitivity} class="w-full accent-gray-800" />
        <div class="w-10 text-right text-xs text-gray-500">{sensitivity}</div>
      </div>

      <div class="flex items-center gap-3">
        <label class="text-sm text-gray-700 w-24">Cooldown</label>
        <input type="range" min="300" max="1500" step="50" bind:value={cooldownMs} class="w-full accent-gray-800" />
        <div class="w-10 text-right text-xs text-gray-500">{cooldownMs}ms</div>
      </div>

      <div class="flex items-center gap-3">
        <button on:click={calibrate} class="px-3 py-1.5 rounded-lg bg-gray-100 border border-gray-300 text-sm">Calibrate</button>
        <span class="text-xs text-gray-500">Hold phone still, then tap</span>
      </div>
    </div>

    <!-- Sensor HUD (collapsible) -->
    <details class="rounded-xl bg-white shadow p-3">
      <summary class="cursor-pointer select-none text-sm text-gray-800">Sensor HUD</summary>
      <div class="grid grid-cols-2 gap-4 mt-3 text-[13px] leading-5">
        <div>
          <div class="font-semibold text-gray-700 mb-1">acceleration</div>
          <div>X: {ax.toFixed(2)}</div>
          <div>Y: {ay.toFixed(2)}</div>
          <div>Z: {az.toFixed(2)}</div>
        </div>
        <div>
          <div class="font-semibold text-gray-700 mb-1">accel + gravity</div>
          <div>X: {gx.toFixed(2)}</div>
          <div>Y: {gy.toFixed(2)}</div>
          <div>Z: {gz.toFixed(2)}</div>
        </div>
        <div class="col-span-2 text-gray-600">γ (orientation): {gamma.toFixed(1)}°</div>
      </div>
    </details>

    <!-- Die -->
    <div
      class={`h-40 w-40 mx-auto bg-white rounded-2xl shadow-xl grid grid-cols-3 gap-2 p-5
              will-change-transform ${spinClass}`}
      style={`transform:${transform}; animation-duration:${spinDurMs}ms`}
      aria-label={`Die showing ${value}`}
      role="img"
    >
      {#each Array(9) as _, i}
        <div class="flex items-center justify-center">
          {#if pips[value].includes(i)}<div class="w-4 h-4 bg-gray-900 rounded-full"></div>{/if}
        </div>
      {/each}
    </div>

    <div class="text-center text-sm text-gray-600">Result: <span class="font-semibold">{value}</span></div>

    <!-- Desktop hint -->
    <p class="text-center text-xs text-gray-400">Tip: On iOS, tap “Enable motion” first. On desktop, press “Roll”.</p>
  </div>
</div>

<style>
  /* Professional, minimal roll animation (duration set inline for adaptivity) */
  @keyframes dice-spin {
    0%   { transform: scale(1) rotate(0deg); }
    33%  { transform: scale(1.05) rotate(130deg); }
    66%  { transform: scale(1.02) rotate(260deg); }
    100% { transform: scale(1) rotate(360deg); }
  }
  .animate-dice-spin { animation-name: dice-spin; animation-timing-function: cubic-bezier(.22,.61,.36,1); }
  @media (prefers-reduced-motion: reduce) {
    .animate-dice-spin { animation: none !important; }
  }
</style>
