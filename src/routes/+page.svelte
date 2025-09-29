<!-- +page.svelte -->
<script>
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';

  // UI state
  let value = 1, listening = false, lastRoll = 0;
  let trackX = true, trackY = true;

  // live sensor values
  let ax = 0, ay = 0, az = 0;           // acceleration (no gravity)
  let gx = 0, gy = 0, gz = 0;           // accelerationIncludingGravity
  let alpha = 0, beta = 0, gamma = 0;   // orientation (for subtle rotation)

  // motion → roll settings
  const THRESHOLD = 13;        // shake sensitivity
  const COOLDOWN_MS = 800;     // avoid multi triggers

  // die pips
  const pips = {1:[4],2:[0,8],3:[0,4,8],4:[0,2,6,8],5:[0,2,4,6,8],6:[0,2,3,5,6,8]};
  const roll = () => (value = 1 + Math.floor(Math.random()*6));

  // map sensor → transform (px/deg), gently clamped
  const clamp = (n, min, max) => Math.max(min, Math.min(max, n));
  function dieTransform() {
    // translate by gravity-influenced axes (feels more “physical”)
    const tx = trackY ? clamp(gx * 2, -12, 12) : 0;   // horizontal sway
    const ty = trackX ? clamp(-gy * 2, -12, 12) : 0;  // vertical sway
    const rz = clamp(gamma * 0.4, -15, 15);           // slight twist from orientation
    return `translate3d(${tx}px, ${ty}px, 0) rotate(${rz}deg)`;
  }

  // simple smoothing to avoid jitter
  let sAx=0, sAy=0, sAz=0, sGx=0, sGy=0, sGz=0, sGamma=0;
  const SMOOTH = 0.25;
  const smooth = (prev, next) => prev + (next - prev) * SMOOTH;

  function handleMotion(e) {
    const a = e.acceleration || {};
    const ag = e.accelerationIncludingGravity || {};
    ax = sAx = smooth(sAx, a.x ?? 0);
    ay = sAy = smooth(sAy, a.y ?? 0);
    az = sAz = smooth(sAz, a.z ?? 0);
    gx = sGx = smooth(sGx, ag.x ?? 0);
    gy = sGy = smooth(sGy, ag.y ?? 0);
    gz = sGz = smooth(sGz, ag.z ?? 0);

    const now = Date.now();
    const shakeX = trackX && Math.abs(ax) > THRESHOLD;
    const shakeY = trackY && Math.abs(ay) > THRESHOLD;
    if ((shakeX || shakeY) && now - lastRoll > COOLDOWN_MS) {
      lastRoll = now; roll();
      // trigger a quick spin via CSS class toggle
      spinning = true; clearTimeout(spinT); spinT = setTimeout(()=> (spinning=false), 500);
    }
  }

  // orientation (optional, for nicer rotation)
  function handleOrientation(e) {
    alpha = e.alpha ?? 0;
    beta  = e.beta  ?? 0;
    gamma = sGamma = smooth(sGamma, e.gamma ?? 0);
  }

  let spinning = false, spinT;

  async function start() {
    if (!browser) return;
    // iOS permission gate
    if (typeof DeviceMotionEvent !== 'undefined' && DeviceMotionEvent.requestPermission) {
      const p = await DeviceMotionEvent.requestPermission();
      if (p !== 'granted') return;
    }
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

  onMount(start);
  onDestroy(stop);
</script>

<div class="min-h-screen bg-gray-100 flex items-center justify-center p-6">
  <div class="space-y-5 w-full max-w-sm">
    <!-- Controls -->
    <div class="flex items-center gap-3">
      <label class="flex items-center gap-2"><input type="checkbox" bind:checked={trackX} class="accent-gray-700"> X</label>
      <label class="flex items-center gap-2"><input type="checkbox" bind:checked={trackY} class="accent-gray-700"> Y</label>
      {#if !listening}
        <button on:click={start} class="px-3 py-1.5 rounded-lg bg-gray-900 text-white text-sm">Bewegung aktivieren</button>
      {:else}
        <button on:click={roll} class="px-3 py-1.5 rounded-lg border border-gray-300 text-sm">Roll</button>
      {/if}
    </div>

    <!-- Live sensor HUD -->
    <div class="rounded-xl bg-white shadow p-3 text-[13px] leading-5 grid grid-cols-2 gap-x-4">
      <div>
        <div class="font-semibold text-gray-700 mb-1">acceleration</div>
        <div>X: {ax.toFixed(2)}</div>
        <div>Y: {ay.toFixed(2)}</div>
        <div>Z: {az.toFixed(2)}</div>
      </div>
      <div>
        <div class="font-semibold text-gray-700 mb-1">accel+gravity</div>
        <div>X: {gx.toFixed(2)}</div>
        <div>Y: {gy.toFixed(2)}</div>
        <div>Z: {gz.toFixed(2)}</div>
      </div>
      <div class="col-span-2 pt-1 text-gray-600">γ (orientation): {gamma.toFixed(1)}°</div>
    </div>

    <!-- Die -->
    <div
      class={`h-36 w-36 mx-auto bg-white rounded-2xl shadow-xl grid grid-cols-3 gap-2 p-4
              transition-transform duration-150 ease-out will-change-transform
              ${spinning ? 'animate-dice-spin' : ''}`}
      style={`transform:${dieTransform()}`}
    >
      {#each Array(9) as _, i}
        <div class="flex items-center justify-center">
          {#if pips[value].includes(i)}<div class="w-4 h-4 bg-gray-900 rounded-full"></div>{/if}
        </div>
      {/each}
    </div>

    <div class="text-center text-sm text-gray-600">Result: <span class="font-semibold">{value}</span></div>
  </div>
</div>

<style>
  /* Professional motion: physics-like sway via transform + a brief spin on roll */
  @keyframes dice-spin {
    0%   { transform: scale(1) rotate(0deg); }
    35%  { transform: scale(1.05) rotate(120deg); }
    70%  { transform: scale(1.02) rotate(240deg); }
    100% { transform: scale(1) rotate(360deg); }
  }
  .animate-dice-spin { animation: dice-spin 0.5s cubic-bezier(.22,.61,.36,1); }
</style>
