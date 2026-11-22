<script>
  import { onMount } from 'svelte';
  import { getJSON } from '../lib/api.js';
  import { uiStore } from '../lib/store.js';
  import FlatCardCompact from './FlatCardCompact.svelte';

  export let flat;
  export let favorites = [];
  export let config = {};
  export let onOpen = () => {};
  export let onFavorite = () => {};

  let detail = {};
  let similar = [];

  $: item = { ...detail, ...flat };
  $: bedrooms = item.rooms === 0 ? 'Studio' : item.rooms;
  $: balcony = item.balcony === 'no balcony' ? 'No' : (item.balcony || '').replace(/^balcony /, '').replace(/^./, (c) => c.toUpperCase());
  $: if (flat) load(flat);

  function money(v) {
    return '£' + Number(v || 0).toLocaleString('en-GB');
  }

  function load(f) {
    getJSON('/api/flats/' + f.id + '.json').then((data) => {
      detail = data;
      const list = window.appState.flats || [];
      similar = list.filter((x) => data.similar && data.similar.indexOf(x.id) > -1);
    });
  }

  function close() {
    document.body.style.overflow = '';
    uiStore.update((s) => ({ ...s, modalOpen: false, activeFlat: null }));
  }

  function book() {
    document.body.style.overflow = '';
    uiStore.update((s) => ({ ...s, modalOpen: false, activeFlat: null, callbackOpen: true }));
  }

  function onKey(e) {
    if (e.key === 'Escape') close();
  }

  onMount(() => {
    document.body.style.overflow = 'hidden';
    window.addEventListener('keydown', onKey);
  });
</script>

<div class="flat-modal" on:click|self={close}>
  <div class="flat-modal-box" role="dialog" aria-modal="true" aria-label={'Home ' + item.id}>
    <button class="flat-modal-close" type="button" on:click={close} style="top: 22px; right: 28px">
      Close <i class="fa-solid fa-xmark"></i>
    </button>

    <div class="flat-modal-plan">
      {#if detail.planImage}
        <img src={detail.planImage} alt={'Plan of home ' + item.id} />
      {/if}
    </div>

    <div class="flat-modal-info">
      <h2 class="flat-modal-title">Home {item.id}</h2>

      <dl class="flat-modal-rows" style="margin-top: 36px">
        <div><dt>Building</dt><dd>{item.building}</dd></div>
        <div><dt>Floor</dt><dd>{item.floor} of {item.floorsTotal}</dd></div>
        <div><dt>Bedrooms</dt><dd>{bedrooms}</dd></div>
        <div><dt>Area</dt><dd>{item.area} m²</dd></div>
        <div><dt>Kitchen</dt><dd>{item.kitchenArea} m²</dd></div>
        <div><dt>Living area</dt><dd>{item.livingArea} m²</dd></div>
        <div><dt>Ceiling</dt><dd>{item.ceiling} m</dd></div>
        <div><dt>Windows face</dt><dd>{item.windows}</dd></div>
        <div><dt>Balcony</dt><dd>{balcony}</dd></div>
        <div><dt>Finish</dt><dd>{item.finishing}</dd></div>
        <div><dt>Price</dt><dd>{money(item.price)}</dd></div>
        <div><dt>Price per m²</dt><dd>{money(item.pricePerMeter)}</dd></div>
      </dl>

      <div class="flat-modal-actions" style="margin-top: 36px">
        <button class="btn btn-accent" type="button" on:click={book}>Book a viewing</button>
        <a class="link-line" href={detail.planImage} download style="margin-left: 28px">Download the plan</a>
      </div>

      <h3 class="flat-modal-similar" style="margin-top: 56px">Similar homes</h3>
      <div class="flat-modal-list">
        {#each similar as s}
          <FlatCardCompact flat={s} {favorites} {config} {onOpen} {onFavorite} />
        {/each}
      </div>
    </div>
  </div>
</div>

<style>
  .flat-modal {
    position: fixed;
    inset: 0;
    z-index: 99999;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 32px;
    background: rgba(51, 50, 49, 0.72);
    animation: flat-modal-in 0.35s var(--ease-out);
    will-change: opacity;
  }

  .flat-modal-box {
    position: relative;
    display: grid;
    grid-template-columns: 1.1fr 1fr;
    width: 100%;
    max-width: 1376px;
    height: 100%;
    max-height: 860px;
    background: var(--paper);
    overflow: hidden;
    animation: flat-modal-up 0.5s var(--ease-out);
    will-change: transform, opacity;
  }

  .flat-modal-close {
    position: absolute;
    z-index: 2;
    display: inline-flex;
    align-items: center;
    gap: 10px;
    padding: 6px 0;
    border: 0;
    background: transparent;
    font-size: 15px;
    cursor: pointer;
    transition: left 0.3s ease, opacity 0.2s ease;
  }

  .flat-modal-close:hover {
    opacity: 0.7;
  }

  .flat-modal-plan {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 64px 56px;
    background: var(--paper);
  }

  .flat-modal-plan img {
    display: block;
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .flat-modal-info {
    padding: 72px 56px 48px 24px;
    overflow-y: auto;
    overscroll-behavior: contain;
  }

  .flat-modal-title {
    font-size: clamp(2rem, 1.3rem + 2vw, 3rem) !important;
  }

  .flat-modal-rows {
    margin-top: 32px !important;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 14px 24px;
  }

  .flat-modal-rows dt {
    font-size: var(--step-small);
    line-height: 1.4;
    color: var(--muted);
  }

  .flat-modal-rows dd {
    font-size: var(--step-body);
    line-height: 1.4;
    font-variant-numeric: tabular-nums;
  }

  .flat-modal-actions {
    display: flex;
    align-items: center;
  }

  .flat-modal-similar {
    margin-top: 56px !important;
    margin-bottom: 12px !important;
    font-size: var(--step-lead) !important;
    line-height: 1.2 !important;
  }

  @keyframes flat-modal-in {
    from {
      opacity: 0;
    }
  }

  @keyframes flat-modal-up {
    from {
      opacity: 0;
      transform: translateY(24px);
    }
  }
</style>
