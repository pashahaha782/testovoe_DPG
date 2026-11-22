<script>
  import FlatActions from './FlatActions.svelte';

  export let flat;
  export let favorites = [];
  export let config = {};
  export let onOpen = () => {};
  export let onFavorite = () => {};

  const names = ['Studio', '1 bedroom', '2 bedrooms', '3 bedrooms', '4 bedrooms'];
  const statuses = { reserved: 'Reserved', sold: 'Sold' };

  $: title = names[flat.rooms] + ', ' + flat.area + ' m², floor ' + flat.floor;
  $: note = flat.description ? flat.description.split('. ')[0] : flat.aspect || '';
  $: price = '£' + Number(flat.price).toLocaleString('en-GB');
  $: status = statuses[flat.status] || '';
</script>

<div class="flat" class:flat-off={status} class:flat-current={flat.current} on:click={() => onOpen(flat)}>
  <div class="flat-main">
    <p class="flat-title">
      {title}
    </p>
    <p class="flat-note">
      {note}
    </p>
  </div>
  <div class="flat-side">
    {#if status}
      <span class="flat-status">{status}</span>
    {:else}
      <span class="flat-price">{price}</span>
    {/if}
  </div>
  <FlatActions {flat} {favorites} {config} {onFavorite} />
</div>

<style>
  .flat {
    position: relative;
    display: flex;
    align-items: flex-start;
    margin: 0 -48px 0 -20px;
    padding: 8px 8px 9px 20px;
    cursor: pointer;
    transition: background-color 180ms ease;
    will-change: transform, background-color;
  }

  .flat:hover,
  .flat-current {
    background-color: var(--bg);
  }

  .flat-main {
    flex: 1;
    min-width: 0;
  }

  .flat-title {
    font-size: var(--step-body);
    line-height: 1.45;
    font-variant-numeric: tabular-nums;
  }

  .flat-note {
    min-height: 1.4em;
    margin-top: 2px !important;
    font-size: var(--step-small);
    line-height: 1.4;
    color: var(--muted);
  }

  .flat-side {
    flex: none;
    padding-left: 24px;
    font-size: var(--step-body);
    line-height: 1.45;
    text-align: right;
    font-variant-numeric: tabular-nums;
  }

  .flat-off .flat-title,
  .flat-off .flat-status {
    color: var(--muted);
  }

  .flat :global(.flat-actions) {
    margin-top: -6px;
  }
</style>
