<!-- +page.svelte -->
<script>
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';

  let value = 1, trackX = true, trackY = true, listening = false, last = 0, shaking = false;
  const THRESHOLD = 13, COOLDOWN_MS = 800, SHAKE_MS = 500;

  const pips = {1:[4],2:[0,8],3:[0,4,8],4:[0,2,6,8],5:[0,2,4,6,8],6:[0,2,3,5,6,8]};

  function roll(){
    value = 1 + Math.floor(Math.random()*6);
    shaking = true; setTimeout(()=> shaking=false, SHAKE_MS);
  }

  function handleMotion(e){
    const a = e.accelerationIncludingGravity || e.acceleration || {};
    const ax = Math.abs(a.x ?? 0), ay = Math.abs(a.y ?? 0);
    const now = Date.now();
    if (((trackX && ax>THRESHOLD) || (trackY && ay>THRESHOLD)) && now-last>COOLDOWN_MS){
      last = now; roll();
    }
  }

  async function start(){
    if (!browser) return;
    if (typeof DeviceMotionEvent!=='undefined' && DeviceMotionEvent.requestPermission){
      const perm = await DeviceMotionEvent.requestPermission();
      if (perm!=='granted') return;
    }
    window.addEventListener('devicemotion', handleMotion, { passive:true });
    listening = true;
  }

  function stop(){
    if (!browser) return;
    window.removeEventListener('devicemotion', handleMotion);
    listening = false;
  }

  onMount(start);
  onDestroy(stop);
</script>

<div class="min-h-screen bg-gray-100 flex items-center justify-center p-6">
  <div class="space-y-4">
    <div class="flex items-center gap-3">
      <label class="flex items-center gap-2"><input type="checkbox" bind:checked={trackX} class="accent-gray-700"> X</label>
      <label class="flex items-center gap-2"><input type="checkbox" bind:checked={trackY} class="accent-gray-700"> Y</label>
      {#if !listening}<button on:click={start} class="px-3 py-1 rounded-lg bg-gray-800 text-white text-sm">Enable motion</button>{/if}
      <button on:click={roll} class="px-3 py-1 rounded-lg border border-gray-300 text-sm">Roll</button>
    </div>

    <div class={`h-36 w-36 bg-white rounded-2xl shadow-xl transition duration-300 grid grid-cols-3 gap-2 p-4 ${shaking ? 'animate-dice' : 'rotate-3 hover:rotate-6'}`}>
      {#each Array(9) as _, i}
        <div class="flex items-center justify-center">
          {#if pips[value].includes(i)}<div class="w-4 h-4 bg-gray-800 rounded-full"></div>{/if}
        </div>
      {/each}
    </div>

    <div class="text-center text-sm text-gray-600">Result: <span class="font-semibold">{value}</span></div>
  </div>
</div>

<style>
  /* “Modern” shake: subtle 3D jitters with ease */
  @keyframes dice-shake {
    0%   { transform: rotate(3deg) translate3d(0,0,0); }
    20%  { transform: rotate(6deg) translate3d(2px,-1px,0); }
    40%  { transform: rotate(-4deg) translate3d(-2px,1px,0); }
    60%  { transform: rotate(5deg) translate3d(1px,2px,0); }
    80%  { transform: rotate(-3deg) translate3d(-1px,-2px,0); }
    100% { transform: rotate(3deg) translate3d(0,0,0); }
  }
  .animate-dice { animation: dice-shake 0.5s ease; }
</style>
