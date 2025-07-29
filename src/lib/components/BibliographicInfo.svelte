<script>
  import { hovered } from "$lib/stores.js";

  let metadata = {
    title: "title_info_primary_tsi",
    creator: "name_tsim",
    date: "date_tsim",
  };

  let item = $state(null);
  let lastId = $state(null);
  let loading = $state(false);
  let hoveredId = $state(null);
  let url = $state(null)

  $effect(() => {
    const unsubscribe = hovered.subscribe(($hovered) => {
      if ($hovered && typeof $hovered.get === "function") {
        const id = $hovered.get("id");
        hoveredId = id != null ? String(id) : null;
      } else {
        hoveredId = null;
      }
    });
    return unsubscribe;
  });

  $effect(() => {
    if (hoveredId) {
      if (hoveredId !== lastId || item === null) {
        const currentId = hoveredId;
        lastId = hoveredId;
        loading = true;

        fetch(`https://collections.leventhalmap.org/search/${hoveredId}.json`)
          .then((r) => r.json())
          .then((data) => {
            if (currentId === hoveredId) {
              item = data;
            }
          })
          .catch(() => {
            if (currentId === hoveredId) {
              item = null;
            }
          })
          .finally(() => {
            if (currentId === hoveredId) {
              loading = false;
            }
          });
      }
    } else {
      item = null;
      lastId = null;
      loading = false;
    }
  });
</script>

<div class="m-8 space-y-2">
  {#if loading}
    {#each Object.entries(metadata) as [label]}
      <div class="flex items-start gap-4">
        <div
          class="w-32 text-md p-1 text-center font-bold rounded shadow border bg-blue-50 border-blue-300 text-blue-300 px-2"
        >
          {label.toUpperCase()}
        </div>
        <div
          class="bg-blue-50 py-1 px-3 rounded text-blue-200 italic text-sm animate-pulse"
        >
          Loading {label}...
        </div>
      </div>
    {/each}
  {:else if item}
    {#each Object.entries(metadata) as [label, field]}
      <div class="flex items-start gap-4">
        <div
          class="w-32 p-1 text-md uppercase text-center font-bold rounded shadow border bg-blue-50 border-blue-300 text-blue-300 px-2"
        >
          {label}
        </div>
        <div class="text-gray-400 p-1">
          {item.response.document[field] &&
          item.response.document[field].length > 60
            ? `${item.response.document[field].toString().slice(0, 60)}...`
            : item.response.document[field]}
          {console.log(item.response.document[field].toString().slice(0, 60))}
        </div>
      </div>
    {/each}
  {:else}
    {#each Object.entries(metadata) as [label]}
      <div class="flex items-start gap-4">
        <div
          class="w-32 text-md p-1 uppercase text-center font-bold rounded shadow border bg-blue-50 border-blue-300 text-blue-300 px-2"
        >
          {label}
        </div>
      </div>
    {/each}
  {/if}
</div>
