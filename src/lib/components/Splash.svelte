<script>
  import { onMount } from "svelte";
  let svgContent = $state();

  onMount(async () => {
    const res = await fetch("/icons/logo.svg");
    svgContent = await res.text();
  });

  let active = $state(false);

  function toggleActive() {
    active = !active;
  }
</script>

<div
  class:hidden={active}
  class="fixed inset-0 z-20 flex items-center justify-center"
>
  <div class="fixed inset-0 bg-gray-500/90 transition-opacity z-10"></div>

  <div class="relative z-20 flex items-end justify-center p-4 text-center sm:items-center sm:p-0">
    <div
      class="relative overflow-hidden rounded-lg bg-gradient-to-tr from-purple-100 to-gray-50 px-4 pb-4 pt-5 text-left shadow-xl transition-all sm:my-8 sm:w-full sm:max-w-sm sm:p-6"
    >
      <div class="absolute -inset-5 z-0 opacity-30 pointer-events-none">
        {@html svgContent}
      </div>

      <div class="relative z-10">
        <div class="text-4xl text-blue-300 uppercase font-black tracking-widest text-center text-shadow-sm text-shadow-blue-600">
          Mapsplanade
        </div>

        <div class="m-3 text-center text-sm sm:mt-5">
          <p class="mb-4 text-shadow-lg text-shadow-white">
            How long would all the maps in the Leventhal Map & Education Center's
            collections be if you laid them on the ground back to back?
          </p>
        </div>

        <div class="mt-5 sm:mt-6">
          <button
            type="button"
            on:click={toggleActive}
            class="inline-flex w-full justify-center rounded-md bg-green-200 ring-2 text-md px-3 py-2 font-semibold text-slate-900 shadow-sm hover:bg-green-300 focus-visible:outline focus-visible:outline-4 focus-visible:outline-offset-2 focus-visible:outline-orange-600 animate-pulse"
          >
            Walk the Mapsplanade to find out
          </button>
        </div>

        <div class="mt-4">
          <p class="text-xs text-center font-semibold">
            A project of the
            <a
              class="text-blue-900 hover:text-blue-600 transition-colors"
              href="https://leventhalmap.org"
              target="_blank"
              >Leventhal Map & Education Center</a
            >
            at the Boston Public Library, for
            <a
              class="text-blue-900 hover:text-blue-600 transition-colors"
              href="https://gardensofegleston.org/"
              target="_blank"
              >Gardens of Egleston</a
            >
          </p>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  button {
    border-radius: 8px;
    border: 1px solid transparent;
    font-weight: 500;
    font-family: inherit;
    background-color: #def1f1;
    cursor: pointer;
    transition: border-color 0.25s;
  }
  button:hover {
    border-color: #646cff;
  }
  button:focus,
  button:focus-visible {
    outline: 4px auto -webkit-focus-ring-color;
  }
</style>
