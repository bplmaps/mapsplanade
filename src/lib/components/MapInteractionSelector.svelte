<script>
  export let type = 'click';
  export let onChange;
</script>

<form>
  <label for="type">Action type &nbsp;</label>
  <select id="type" bind:value={type} on:change={() => onChange(type)}>
    <option value="click">Click</option>
    <option value="singleclick">Single-click</option>
    <option value="pointermove">Hover</option>
    <option value="altclick">Alt+Click</option>
    <option value="none">None</option>
  </select>
</form>
