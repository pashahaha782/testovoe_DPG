<script>
  import { onMount, tick } from 'svelte';
  import gsap from 'gsap';
  import { ScrollTrigger } from 'gsap/ScrollTrigger';
  import { SplitText } from 'gsap/SplitText';
  import Swiper from 'swiper';
  import { FreeMode } from 'swiper/modules';
  import 'swiper/css';
  import 'swiper/css/free-mode';
  import * as maplibregl from 'maplibre-gl';
  import 'maplibre-gl/dist/maplibre-gl.css';
  import mapWorker from 'maplibre-gl/dist/maplibre-gl-worker.mjs?url';
  import { slide } from 'svelte/transition';
  import { getJSON, getJSONSync } from './lib/api.js';
  import { uiStore, flatsStore, favoritesStore } from './lib/store.js';
  import { monthlyPayment } from './lib/mortgage.js';
  import { newsDate, monthTitle, pounds, perMonth, area } from './lib/format.js';
  import Lightbox from './components/Lightbox.svelte';
  import FlatList from './components/FlatList.svelte';
  import FlatModal from './components/FlatModal.svelte';
  import ModalLead from './components/ModalLead.svelte';
  import ModalCallback from './components/ModalCallback.svelte';

  import quayAerial from './assets/about/quay-aerial.png';
  import interior1 from './assets/about/interior-1.png';
  import interior2 from './assets/about/interior-2.png';
  import iconPier from './assets/icons/pier.png';
  import iconCrane from './assets/icons/crane.png';
  import iconWarehouse from './assets/icons/warehouse.png';
  import iconTree from './assets/icons/tree.png';
  import iconHeight from './assets/icons/height.png';
  import iconWave from './assets/icons/wave.png';
  import iconBoard from './assets/icons/board.png';

  gsap.registerPlugin(ScrollTrigger, SplitText);
  maplibregl.setWorkerUrl(mapWorker);

  export let config = window.appConfig || {};

  $: title = config.siteName || 'Larch Quay';
  $: phone = config.phone || '+44 131 496 0418';
  $: phoneHref = 'tel:' + phone.replace(/\s/g, '');

  let sections = {};
  $: about = sections.about || {};
  $: aboutParts = (about.html || '')
    .split('</p>')
    .filter((p) => p.trim())
    .map((p) => p + '</p>');
  $: facts = about.facts || [];
  $: film = about.film || {};
  $: advantages = sections.advantages || [];
  $: places = sections.infrastructure || [];
  $: construction = sections.construction || [];
  $: faq = sections.faq || [];
  $: visit = sections.visit || {};
  $: visitFields = visit.fields || {};
  $: footer = sections.footer || {};
  $: contact = footer.contact || {};
  $: cookie = sections.cookie || {};
  $: callback = sections.callback || {};

  const icons = {
    pier: iconPier,
    crane: iconCrane,
    warehouse: iconWarehouse,
    tree: iconTree,
    height: iconHeight,
    wave: iconWave,
    board: iconBoard,
  };

  const featurePhotos = [
    { src: '/media/gallery/render-08.png', alt: 'The pier with the water tram' },
    { src: '/media/gallery/render-02.png', alt: 'The cranes lit at dusk' },
    { src: '/media/gallery/render-06.png', alt: 'The arched windows of the Larch Store' },
    { src: '/media/gallery/render-03.png', alt: 'The courtyard with young birches' },
    { src: '/media/gallery/render-05.png', alt: 'A living room in Quayside with windows from floor to ceiling' },
    { src: '/media/gallery/render-04.png', alt: 'The water along the granite edge' },
    { src: interior2, alt: 'Larch boards on a facade, amber turning silver' },
  ];
  const photoSizes = [
    { w: 320, ratio: '4 / 5' },
    { w: 440, ratio: '3 / 2' },
    { w: 280, ratio: '1 / 1' },
  ];

  const gallery = [
    { src: '/media/gallery/render-07.png', alt: 'Terrace, with roof gardens on every step' },
    { src: '/media/gallery/render-06.png', alt: 'The Larch Store, built in 1912' },
    { src: '/media/gallery/render-08.png', alt: 'The pier and the water tram' },
    { src: '/media/gallery/render-01.png', alt: 'Quayside and Crane House from the water on an overcast morning' },
    { src: '/media/gallery/render-03.png', alt: 'The courtyard with young birches' },
    { src: '/media/gallery/render-02.png', alt: 'The three cranes at dusk' },
    { src: '/media/gallery/render-05.png', alt: 'A living room in Quayside looking at the cranes' },
    { src: '/media/gallery/render-09.png', alt: 'A bedroom with a larch floor' },
    { src: '/media/gallery/render-04.png', alt: 'The promenade along the granite edge' },
  ];
  const galleryModules = [
    { type: 'tall', items: [0] },
    { type: 'pair', items: [1, 2] },
    { type: 'wide', items: [3] },
    { type: 'tall', items: [4] },
    { type: 'wide', items: [5] },
    { type: 'tall', items: [6] },
    { type: 'wide', items: [7] },
    { type: 'tall', items: [8] },
  ];

  const buildings = [
    { name: 'Quayside', title: 'Quayside, 21 floors, keys in Q4 2027' },
    { name: 'Crane House', title: 'Crane House, 16 floors, keys in Q2 2028' },
    { name: 'Yard', title: 'Yard, 12 floors, keys in Q3 2028' },
    { name: 'Terrace', title: 'Terrace, 9 floors, keys in Q3 2028' },
  ];
  const sorts = [
    { value: 'price', label: 'Price, low to high' },
    { value: 'price-desc', label: 'Price, high to low' },
    { value: 'area', label: 'Area' },
    { value: 'floor', label: 'Floor' },
  ];

  const offerIds = ['QS-1209', 'CH-907', 'YD-708', 'TR-609'];
  let offers = [];

  function salePrice(price) {
    return Math.round((price * 0.93) / 100) * 100;
  }


  let flats = [];
  let prices = {};
  let found = [];
  let favorites = JSON.parse(localStorage.getItem('favorites') || '[]');
  let activeFlat = null;
  let groups = [];
  let rooms = [];
  let areaFrom = null;
  let areaTo = null;
  let priceFrom = null;
  let priceTo = null;
  let floorFrom = null;
  let floorTo = null;
  let query = '';
  let sort = 'price';

  $: roomOptions = (config.filters && config.filters.rooms) || [0, 1, 2, 3, 4];

  $: {
    let list = flats.map((f) => ({
      ...f,
      price: prices[f.id] || f.price,
      current: !!activeFlat && activeFlat.id === f.id,
    }));
    if (rooms.length) list = list.filter((f) => rooms.indexOf(f.rooms) > -1);
    if (areaFrom) list = list.filter((f) => f.area >= areaFrom);
    if (areaTo) list = list.filter((f) => f.area <= areaTo);
    if (priceFrom) list = list.filter((f) => f.price >= priceFrom);
    if (priceTo) list = list.filter((f) => f.price <= priceTo);
    if (floorFrom) list = list.filter((f) => f.floor >= floorFrom);
    if (floorTo) list = list.filter((f) => f.floor <= floorTo);
    if (query) list = list.filter((f) => found.indexOf(f.id) > -1);
    list.sort((a, b) => {
      if (sort === 'price-desc') return b.price - a.price;
      if (sort === 'area') return a.area - b.area;
      if (sort === 'floor') return a.floor - b.floor;
      return a.price - b.price;
    });
    groups = buildings.map((b) => ({ ...b, flats: list.filter((f) => f.building === b.name) }));
    window.appState.filters = { rooms, areaFrom, areaTo, priceFrom, priceTo, floorFrom, floorTo, query, sort };
    window.appState.favorites = favorites;
  }

  let topBar;
  let topBand;
  let heroEl;
  let heroStage;
  let heroBg;
  let heroVideo;
  let heroTitle;
  let heroCopy;
  let aboutPhoto;
  let aboutImg;
  let galleryEl;
  let gallerySwiper;
  let photoReady = [];
  let openingEl;
  let openingWindow;
  let openingImg;
  let videoReady = false;
  let lastY = 0;
  let scrollFrame = 0;
  let lightboxIndex = -1;

  function queueScroll() {
    if (scrollFrame) return;
    scrollFrame = requestAnimationFrame(() => {
      scrollFrame = 0;
      onScroll();
    });
  }

  function onScroll() {
    const y = window.scrollY;
    const vh = window.innerHeight;
    const heroTop = heroEl.offsetTop;
    const heroHeight = heroEl.offsetHeight;

    let p = (y - heroTop) / (heroHeight - vh);
    if (p < 0) p = 0;
    if (p > 1) p = 1;
    if (p < 1 && heroVideo && heroVideo.duration) {
      const next = p * heroVideo.duration;
      if (Math.abs(heroVideo.currentTime - next) > 1 / 30) {
        heroVideo.currentTime = next;
      }
    }
    heroBg.style.top = y * 0.35 + 'px';
    heroCopy.style.opacity = Math.max(0, 1 - p * 3);

    topBar.style.height = Math.max(80, 84 - y / 60) + 'px';
    if (y > lastY && y > 320) {
      topBar.style.top = '-' + (topBar.offsetHeight + 60) + 'px';
    } else {
      topBar.style.top = '0px';
    }
    lastY = y;
    topBand.style.width = y > heroTop + heroHeight - topBar.offsetHeight ? '100%' : '0%';
    if (p < 1) return;

    document.querySelectorAll('.section-title').forEach((title) => {
      const r = title.getBoundingClientRect();
      let k = (r.top - vh / 2) / vh;
      if (k < -1) k = -1;
      if (k > 1) k = 1;
      title.style.top = k * 4 + 'px';
    });

    document.querySelectorAll('.feature-photo img').forEach((img) => {
      const r = img.parentNode.getBoundingClientRect();
      let k = (r.top + r.height / 2 - vh / 2) / (vh / 2 + r.height / 2);
      if (k < -1) k = -1;
      if (k > 1) k = 1;
      img.style.top = -k * 6 + 'px';
    });

    document.querySelectorAll('.gallery-shot img').forEach((img) => {
      const r = img.parentNode.getBoundingClientRect();
      let k = (r.top + r.height / 2 - vh / 2) / vh;
      if (k < -1) k = -1;
      if (k > 1) k = 1;
      img.style.top = -6 - k * 6 + 'px';
    });

    document.querySelectorAll('.flat, .mortgage-row, .news-item').forEach((row) => {
      const r = row.getBoundingClientRect();
      let d = r.top - vh * 0.85;
      if (d < 0) d = 0;
      row.style.top = (d / (d + 100)) * 8 + 'px';
    });

    const panels = document.querySelectorAll('.progress-panel');
    panels.forEach((panel, i) => {
      const next = panels[i + 1];
      if (!next) return;
      let k = 1 - next.getBoundingClientRect().top / window.innerHeight;
      if (k < 0) k = 0;
      if (k > 1) k = 1;
      panel.querySelector('.progress-shade').style.opacity = k * 0.6;
      panel.querySelectorAll('.progress-img').forEach((img) => {
        img.style.transform = 'scale(' + (1.08 - k * 0.06) + ')';
      });
    });
  }

  function onResize() {
    if (gallerySwiper) {
      gallerySwiper.params.slidesOffsetBefore = 32;
      gallerySwiper.update();
    }
    onScroll();
  }

  function openLightbox(i) {
    uiStore.update((s) => ({ ...s, lightboxIndex: i }));
  }

  function openFlat(flat) {
    uiStore.update((s) => ({ ...s, modalOpen: true, activeFlat: flat }));
  }

  function toggleFavorite(id) {
    if (favorites.indexOf(id) > -1) {
      favorites = favorites.filter((x) => x !== id);
    } else {
      favorites = [...favorites, id];
    }
    favoritesStore.set(favorites);
    localStorage.setItem('favorites', JSON.stringify(favorites));
  }

  function toggleRooms(r) {
    rooms = rooms.indexOf(r) > -1 ? rooms.filter((x) => x !== r) : [...rooms, r];
  }

  function onSearch(e) {
    query = e.target.value;

    const q = query.trim().toUpperCase();
    found = flats.filter((f) => f.id.indexOf(q) > -1).map((f) => f.id);
  }

  function clearFilters() {
    rooms = [];
    areaFrom = null;
    areaTo = null;
    priceFrom = null;
    priceTo = null;
    floorFrom = null;
    floorTo = null;
    query = '';
    found = [];
    sort = 'price';
  }

  function preparePayments(list) {
    const { rates, terms, downPayments } = window.appConfig.mortgage;
    const payments = {};
    list.forEach((flat) => {
      const rows = [];
      rates.forEach((rate) => {
        terms.forEach((term) => {
          downPayments.forEach((down) => {
            const loan = flat.price - (flat.price * down) / 100;
            const monthly = monthlyPayment(loan, rate, term);
            const total = monthly * term * 12;
            rows.push({ rate, term, down, monthly, total, overpay: total - loan });
          });
        });
      });
      payments[flat.id] = rows;
    });
    window.appState.payments = payments;
  }

  function openCallback() {
    uiStore.update((s) => ({ ...s, callbackOpen: true }));
  }

  function intro() {
    const split = SplitText.create(heroTitle.querySelectorAll('.hero-line'), {
      type: 'lines',
      mask: 'lines',
      linesClass: 'hero-mask-line',
    });
    gsap.set([heroTitle, heroCopy], { visibility: 'visible' });
    gsap.from(split.lines, {
      yPercent: 105,
      duration: 0.9,
      ease: 'expo.out',
      stagger: 0.11,
      delay: 0.15,
      onComplete: () => {
        gsap
          .timeline({ scrollTrigger: { trigger: heroStage, start: 'top top', end: '+=480', scrub: true } })
          .fromTo(split.lines[0], { yPercent: 0, opacity: 1 }, { yPercent: -105, opacity: 0, ease: 'none', immediateRender: false })
          .fromTo(split.lines[1], { yPercent: 0, opacity: 1 }, { yPercent: 105, opacity: 0, ease: 'none', immediateRender: false }, 0);
      },
    });
    gsap.from(heroCopy.children, { top: 24, opacity: 0, duration: 0.7, ease: 'power3.out', delay: 0.6 });
  }

  function reveal(targets, trigger, vars) {
    const list = typeof targets === 'string' ? document.querySelectorAll(targets) : targets;
    if (!list || !list.length) return;
    gsap.from(list, {
      opacity: 0,
      y: 24,
      duration: 0.75,
      ease: 'power3.out',
      stagger: 0.08,
      immediateRender: false,
      scrollTrigger: { trigger: trigger || list[0], start: 'top 92%', once: true },
      ...vars,
    });
  }

  function motion() {
    reveal('#about .section-title, #about .about-lead', '#about');
    reveal('#about .about-photo', '#about .about-mosaic', { y: 0, clipPath: 'inset(0 0 100% 0)', duration: 1.1, ease: 'power4.inOut' });
    reveal('#about .about-paper, #about .about-tile', '#about .about-mosaic', { delay: 0.35 });
    reveal('#about .about-fact', '#about .about-facts', { y: 0, top: 20, stagger: 0.07 });
    reveal('#about .about-film > div', '#about .about-film');

    reveal('#features .features-head', '#features');
    document.querySelectorAll('#features .feature-row').forEach((row) => {
      reveal(row.querySelectorAll('.feature-title, .feature-text'), row);
      reveal(row.querySelectorAll('.feature-photo'), row, { y: 0, clipPath: 'inset(100% 0 0 0)', duration: 1, ease: 'power4.inOut', delay: 0.12 });
    });

    gsap
      .timeline({ scrollTrigger: { trigger: openingEl, start: 'top top', end: 'bottom bottom', scrub: true } })
      .fromTo(openingWindow, { width: '40%', height: '76%' }, { width: '100%', height: '100%', ease: 'none', duration: 0.75 })
      .fromTo(openingImg, { scale: 1.15 }, { scale: 1, ease: 'none', duration: 0.75 }, 0)
      .to({}, { duration: 0.25 });

    reveal('#gallery .gallery-head > *', '#gallery .gallery-head');
    reveal('#gallery .gallery-shot', '#gallery .gallery-row', { y: 0, x: 120, stagger: 0.07, duration: 1, ease: 'expo.out' });

    reveal('#catalog .section-title', '#catalog');
    reveal('#catalog .filter, #catalog .catalog-clear', '#catalog', { y: 0, top: 18, stagger: 0.06, delay: 0.1 });
    reveal('#catalog .sheet', '#catalog', { y: 48, duration: 1, delay: 0.2 });

    reveal('#location .location-card', '#location', { y: 40, duration: 0.9 });
    reveal('#location .location-place', '#location', { y: 0, x: -16, stagger: 0.04, delay: 0.3 });

    reveal('#mortgage .section-title, #mortgage .mortgage-line', '#mortgage');
    reveal('#mortgage .calc-control, #mortgage .mortgage-ask', '#mortgage .calc', { stagger: 0.09 });
    reveal('#mortgage .mortgage-group, #mortgage .mortgage-photo', '#mortgage .mortgage-list', { stagger: 0.08 });

    reveal('#questions .section-title, #questions .questions-facts li', '#questions', { stagger: 0.06 });
    reveal('#questions .faq-item', '#questions .faq', { y: 0, top: 16, stagger: 0.05 });
    reveal('#questions .questions-tall', '#questions', { y: 0, clipPath: 'inset(0 0 100% 0)', duration: 1.2, ease: 'power4.inOut' });
    reveal('#questions .questions-square', '#questions .questions-pics', { y: 60, delay: 0.5 });

    reveal('#visit .visit-card', '#visit', { y: 56, duration: 1 });

    reveal('#footer .footer-mark, #footer .footer-menu a', '#footer', { y: 0, top: 28, stagger: 0.06 });
    reveal('#footer .footer-info > div, #footer .footer-legal p', '#footer .footer-info', { delay: 0.3 });
  }

  let callbackOpen = false;
  let leadOpen = false;
  let cookieVisible = false;
  let chatDock;

  const footerMenu = [
    { label: 'Homes', href: '#catalog' },
    { label: 'Location', href: '#location' },
    { label: 'Mortgage', href: '#mortgage' },
    { label: 'Progress', href: '#progress' },
    { label: 'News', href: '#news' },
    { label: 'Questions', href: '#questions' },
  ];

  const officeFacts = [
    'Sales office, 1 Crane Walk, Larch Quay',
    'Open daily, 10:00 to 19:00',
    'Water tram to Larch Quay Pier, or bus 14 to Crane Walk',
  ];

  function openLead() {
    uiStore.update((s) => ({ ...s, leadOpen: true }));
  }

  function closeCookies(value) {
    localStorage.setItem('cookies', value);
    cookieVisible = false;
    tick().then(() => ScrollTrigger.refresh());
  }

  // map
  let mapEl;
  let map;
  let mapMarkers = [];
  let mapActive = -1;

  $: if (map && places.length) addMarkers(places);

  function paintMap() {
    map.setPaintProperty('background', 'background-color', '#eeecea');
    map.setPaintProperty('landuse_residential', 'fill-color', '#e8e5e1');
    map.setPaintProperty('park', 'fill-color', '#dcdfd5');
    map.setPaintProperty('landcover_wood', 'fill-color', '#d6dacf');
    map.setPaintProperty('water', 'fill-color', '#c5cbc1');
    map.setPaintProperty('waterway', 'line-color', '#c5cbc1');
    map.setPaintProperty('building', 'fill-color', '#e1ddd8');
    map.setPaintProperty('building', 'fill-outline-color', '#d6d1cb');
    map.setPaintProperty('road_area_pier', 'fill-color', '#eeecea');
    map.setPaintProperty('road_pier', 'line-color', '#eeecea');
    map.setPaintProperty('water_name_point_label', 'text-color', '#6b6763');
    map.setPaintProperty('water_name_line_label', 'text-color', '#6b6763');
    ['highway-name-minor', 'highway-name-major', 'label_other', 'label_village', 'label_town', 'label_city'].forEach((id) => {
      map.setPaintProperty(id, 'text-color', '#4a4846');
      map.setPaintProperty(id, 'text-halo-color', '#eeecea');
    });
  }

  function addMarkers(list) {
    if (mapMarkers.length) return;
    const quay = document.createElement('div');
    quay.className = 'map-quay';
    new maplibregl.Marker({ element: quay })
      .setLngLat(config.mapCenter)
      .setPopup(new maplibregl.Popup({ offset: 16, closeButton: false }).setHTML('<strong>Larch Quay</strong><br>' + (config.address || '')))
      .addTo(map);
    list.forEach((place) => {
      const dot = document.createElement('div');
      dot.className = 'map-dot';
      const marker = new maplibregl.Marker({ element: dot })
        .setLngLat(place.lngLat)
        .setPopup(new maplibregl.Popup({ offset: 10, closeButton: false }).setHTML('<strong>' + place.name + '</strong><br>' + place.distance))
        .addTo(map);
      mapMarkers.push(marker);
    });
  }

  function flyToPlace(i) {
    mapMarkers.forEach((marker, n) => {
      if (n !== i && marker.getPopup().isOpen()) marker.togglePopup();
    });
    mapActive = i;
    map.flyTo({ center: places[i].lngLat, zoom: 15.2, speed: 0.9, padding: { left: 640 } });
    if (!mapMarkers[i].getPopup().isOpen()) mapMarkers[i].togglePopup();
  }

  // mortgage
  const mortgage = window.appConfig.mortgage;
  const rates = mortgage.rates;
  const terms = mortgage.terms;
  const deposits = mortgage.downPayments;
  let deposit = mortgage.defaults.deposit;
  let term = mortgage.defaults.term;
  let rateIndex = rates.indexOf(mortgage.defaults.rate);
  let calcFlats = [];
  let calcRows = [];
  const digits = ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9'];

  function figure(value) {
    return perMonth(value).replace(' a month', '').split('');
  }

  const calcGroups = [
    { rooms: 0, title: 'Studios', label: 'Studio' },
    { rooms: 1, title: 'One bedroom', label: '1 bedroom' },
    { rooms: 2, title: 'Two bedrooms', label: '2 bedrooms' },
    { rooms: 3, title: 'Three bedrooms', label: '3 bedrooms' },
    { rooms: 4, title: 'Four bedrooms', label: '4 bedrooms' },
  ];

  function pickFlats(list) {
    const best = {};
    list.forEach((flat) => {
      if (flat.status !== 'free') return;
      const key = flat.rooms + '-' + flat.building;
      if (!best[key] || flat.price < best[key].price) {
        best[key] = flat;
      }
    });
    return Object.values(best).sort((a, b) => a.rooms - b.rooms || a.price - b.price);
  }

  function onCalc(e) {
    if (e) {
      const el = e.target;
      if (el.name === 'deposit') deposit = Number(el.value);
      if (el.name === 'term') term = Number(el.value);
      if (el.name === 'rate') rateIndex = Number(el.value);
    }
    const rate = rates[rateIndex];
    const minTerm = terms[0];
    const maxTerm = terms[terms.length - 1];
    const all = calcFlats.map((flat) => {
      const loan = flat.price - (flat.price * deposit) / 100;
      const payments = {};
      for (let t = minTerm; t <= maxTerm; t++) {
        payments[t] = monthlyPayment(loan, rate, t);
      }
      return { ...flat, loan, payments, monthly: payments[term] };
    });
    calcRows = pickFlats(all);
    window.appState.mortgage = { deposit, term, rate, flats: all };
  }

  // progress
  let progressAir = [];

  function showShot(i, air) {
    progressAir[i] = air;
    progressAir = progressAir;
  }

  // news
  let news = [];
  let newsActive = 0;
  let newsRows = [];

  function onNewsScroll() {
    const line = window.innerHeight / 2;
    newsRows.forEach((row, i) => {
      if (row && row.getBoundingClientRect().top < line) {
        newsActive = i;
      }
    });
  }

  // questions
  let faqOpen = -1;

  function toggleFaq(i) {
    faqOpen = faqOpen === i ? -1 : i;
  }

  // visit
  let visitName = '';
  let visitPhone = '';
  let visitTime = '';
  let visitSent = false;
  let visitMessage = '';

  function sendVisit() {
    const res = getJSONSync('/api/lead.json');
    window.appState.user = { name: visitName, phone: visitPhone, time: visitTime };
    visitSent = res.ok;
    visitMessage = res.message || visit.success;
  }

  function hoursDays(line) {
    return line.slice(0, line.lastIndexOf(' '));
  }

  function hoursTime(line) {
    return line.slice(line.lastIndexOf(' ') + 1);
  }

  onMount(() => {
    Promise.all(offerIds.map((id) => getJSON('/api/flats/' + id + '.json'))).then((list) => {
      offers = list;
    });

    getJSON('/api/sections.json').then((data) => {
      sections = data;
      heroBg.style.backgroundImage = 'url(' + window.appConfig.heroImage + ')';
      tick().then(() => {
        onScroll();
        motion();
        ScrollTrigger.refresh();
      });
    });

    getJSON('/api/flats.json').then((list) => {
      list.forEach((flat) => {
        delete flat.preview;
      });
      flats = list;
      flatsStore.set(list);
      window.appState.flats = list;
      tick().then(() => ScrollTrigger.refresh());
    });

    getJSON('/api/flats.json?v=2').then(preparePayments);

    setInterval(() => {
      if (document.hidden) return;
      getJSON('/api/prices.json').then((data) => {
        const map = {};
        let changed = false;
        data.items.forEach((p) => {
          map[p.id] = p.price;
          if (prices[p.id] !== p.price) changed = true;
        });
        if (changed) prices = map;
      });
    }, window.appConfig.pricesPollInterval);

    uiStore.subscribe((s) => {
      lightboxIndex = s.lightboxIndex;
      activeFlat = s.activeFlat;
      callbackOpen = s.callbackOpen;
      leadOpen = s.leadOpen;
    });

    ScrollTrigger.create({
      trigger: heroStage,
      start: 'top top',
      end: '+=200%',
      pin: true,
    });

    document.fonts.ready.then(intro);

    window.addEventListener('scroll', queueScroll, { passive: true });
    window.addEventListener('resize', onResize);

    try {
      gallerySwiper = new Swiper(galleryEl, {
        modules: [FreeMode],
        slidesPerView: 'auto',
        spaceBetween: 12,
        slidesOffsetBefore: 32,
        slidesOffsetAfter: 32,
        freeMode: { enabled: true, momentumRatio: 0.6 },
        grabCursor: true,
        speed: 1200,
      });
    } catch (e) {}

    setInterval(() => {
      if (document.hidden || !gallerySwiper || lightboxIndex > -1) return;
      if (galleryEl.getBoundingClientRect().top > window.innerHeight) return;
      if (gallerySwiper.isEnd) {
        gallerySwiper.slideTo(0);
      } else {
        gallerySwiper.slideNext();
      }
    }, 4500);

    getJSON('/api/flats.json?_=' + Date.now()).then((list) => {
      calcFlats = list;
      onCalc();
    });

    map = new maplibregl.Map({
      container: mapEl,
      style: config.mapStyle,
      center: config.mapCenter,
      zoom: config.mapZoom,
      scrollZoom: false,
      attributionControl: { compact: false },
    });
    map.setPadding({ left: 640 });
    map.addControl(new maplibregl.NavigationControl({ showCompass: false }), 'bottom-right');
    map.on('load', paintMap);

    getJSON('/api/news.json').then((list) => {
      news = list;
      tick().then(() => {
        reveal('#news .section-title', '#news');
        reveal('#news .news-item', '#news .news-list', { stagger: 0.07 });
        reveal('#news .news-photo', '#news', { y: 0, clipPath: 'inset(0 0 100% 0)', duration: 1.1, ease: 'power4.inOut' });
        ScrollTrigger.refresh();
      });
    });

    window.addEventListener('scroll', onNewsScroll);

    if (!localStorage.getItem('cookies')) {
      setTimeout(() => {
        cookieVisible = true;
        tick().then(() => ScrollTrigger.refresh());
      }, 1500);
    }

    setTimeout(() => {
      const chat = document.querySelector('.cw-btn');
      if (chat) {
        chatDock.appendChild(chat);
      }
    }, 3000);
  });
</script>

{#if cookieVisible}
  <div class="cookie">
    <p class="cookie-text">{cookie.text || ''}</p>
    <div class="cookie-actions">
      <button class="cookie-btn" type="button" on:click={() => closeCookies('accepted')}>{cookie.accept || 'Accept'}</button>
      <button class="cookie-link" type="button" on:click={() => closeCookies('settings')}>{cookie.settings || 'Settings'}</button>
    </div>
  </div>
{/if}

<header class="top fixed-top" bind:this={topBar}>
  <div class="top-band" bind:this={topBand}></div>
  <a class="top-logo" href="#hero">{title}</a>
  <nav class="top-nav">
    <a class="link-line" href="#catalog">Homes</a>
    <a class="link-line" href="#location">Location</a>
    <a class="link-line" href="#progress">Progress</a>
    <a class="link-line" href="#questions">Questions</a>
  </nav>
  <a class="top-phone link-line" href={phoneHref} style="margin-left: 40px">{phone}</a>
  <a class="top-call link-line" href="#visit" on:click|preventDefault={openCallback}>Call me back</a>
  <a class="btn btn-accent top-btn" href="#visit" style="margin-left: 28px">Book a viewing</a>
</header>

<main>
  <section id="hero" class="hero" bind:this={heroEl}>
    <div class="hero-stage" bind:this={heroStage}>
      <div class="hero-bg" bind:this={heroBg} style="background-color: #c6c7c6"></div>
      <video
        class="hero-video"
        bind:this={heroVideo}
        src={config.heroVideo}
        muted
        playsinline
        preload="auto"
        on:loadeddata={() => (videoReady = true)}
        style="opacity: {videoReady ? 1 : 0}"
      ></video>
      <div class="hero-scrim hero-scrim-top"></div>
      <div class="hero-scrim hero-scrim-bottom"></div>
      <h1 class="hero-title" bind:this={heroTitle}>
        <span class="hero-line hero-line-1">Larch</span>
        <span class="hero-line hero-line-2">Quay</span>
      </h1>
      <div class="hero-copy" bind:this={heroCopy}>
        <p style="max-width: 36ch">Homes on the old timber quay, nine minutes by water tram from the old town.</p>
        <a class="btn btn-accent" href="#catalog" style="margin-top: 28px">See the homes</a>
      </div>
    </div>
  </section>

  <section id="about" class="about wrap">
    <h2 class="section-title">{about.title || ''}</h2>
    <div class="about-lead" style="margin-top: 24px">{@html aboutParts[0] || ''}</div>

    <div class="row g-3 about-mosaic" style="margin-top: 72px">
      <div class="col-8">
        <div class="about-photo" bind:this={aboutPhoto} style="aspect-ratio: 16 / 10">
          <img src={quayAerial} alt="Larch Quay from the air: four buildings, the Larch Store and three cranes on the granite edge" bind:this={aboutImg} />
          <div class="about-paper" style="width: 320px">{@html aboutParts[1] || ''}</div>
        </div>
      </div>
      <div class="col-4">
        <div class="about-tile" style="aspect-ratio: 448 / 342">
          <svg viewBox="0 0 440 340" preserveAspectRatio="xMidYMid slice" role="img" aria-label={about.mapLabel || ''}>
            <path
              class="about-water"
              fill-rule="evenodd"
              d="M0 164C22 160 44 172 70 176C96 180 118 168 150 170C176 172 190 186 214 182L292 170L290 140L326 136L330 158L372 150L388 120C404 108 420 118 440 112V256C420 262 400 252 372 258C330 266 300 248 262 252L214 256C172 262 150 248 110 258C70 268 40 252 0 262ZM400 126C396 154 378 182 348 202L342 196C370 178 386 152 390 126Z"
            />
            <circle class="about-dot" cx="252" cy="160" r="8" />
          </svg>
        </div>
        <dl class="about-facts" style="margin-top: 28px">
          {#each facts as fact}
            <div class="about-fact">
              <dt>{fact.label}</dt>
              <dd>{fact.value}</dd>
            </div>
          {/each}
        </dl>
      </div>
    </div>

    <div class="row g-3 about-film" style="margin-top: 120px">
      <div class="col-8">
        <div class="ratio ratio-16x9">
          <iframe
            src="https://www.youtube.com/embed/{config.youtubeId}?rel=0"
            title={film.title || 'The quay at dusk'}
            allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
            allowfullscreen
            style="border: 0"
          ></iframe>
        </div>
      </div>
      <div class="col-4 about-film-text">
        <p class="about-film-title"><em>{film.title || ''}</em></p>
        <p class="about-film-caption" style="margin-top: 20px">{film.caption || ''}</p>
      </div>
    </div>
  </section>

  <section id="features" class="features wrap">
    <div class="row">
      <div class="col-4">
        <div class="features-head">
          <h2 class="section-title features-title">Built around<br />the water</h2>
        </div>
      </div>
      <div class="col-8">
        {#each advantages as adv, i}
          <div class="row feature-row">
            <div class="col-6">
              <h3 class="feature-title">
                <img class="feature-icon" src={icons[adv.icon]} alt="" style="width: 24px; height: 24px" />
                {adv.title}
              </h3>
              <p class="feature-text">{@html adv.text}</p>
            </div>
            <div class="col-6 feature-side">
              <div class="feature-photo" style="width: {photoSizes[i % 3].w}px; aspect-ratio: {photoReady[i] ? photoSizes[i % 3].ratio : 'auto'}">
                <img
                  src={featurePhotos[i] ? featurePhotos[i].src : ''}
                  alt={featurePhotos[i] ? featurePhotos[i].alt : ''}
                  on:load={() => (photoReady[i] = true)}
                />
              </div>
            </div>
          </div>
        {/each}
      </div>
    </div>
  </section>

  <section id="gallery" class="gallery">
    <div class="opening" bind:this={openingEl}>
      <div class="opening-stage">
        <div class="opening-window" bind:this={openingWindow}>
          <img src="/media/photos/crane-dusk.png" alt="A restored portal crane lit on the quay at dusk" bind:this={openingImg} />
        </div>
      </div>
    </div>
    <div class="gallery-head" style="padding: 0 32px">
      <h2 class="section-title">The quay in pictures</h2>
      <span class="gallery-count">{gallery.length} views</span>
    </div>
    <div class="swiper gallery-row" bind:this={galleryEl}>
      <div class="swiper-wrapper">
        {#each galleryModules as m}
          {#if m.type === 'tall'}
            <div class="swiper-slide gallery-module" style="width: calc(68vh * 3 / 4); height: 68vh">
              <div class="gallery-shot" on:click={() => openLightbox(m.items[0])}>
                <img src={gallery[m.items[0]].src} alt={gallery[m.items[0]].alt} />
              </div>
            </div>
          {:else if m.type === 'pair'}
            <div class="swiper-slide gallery-module gallery-pair" style="width: calc((68vh - 12px) / 2 * 4 / 3); height: 68vh">
              {#each m.items as idx}
                <div class="gallery-shot" on:click={() => openLightbox(idx)}>
                  <img src={gallery[idx].src} alt={gallery[idx].alt} />
                </div>
              {/each}
            </div>
          {:else}
            <div class="swiper-slide gallery-module" style="width: calc(68vh * 16 / 10); height: 68vh">
              <div class="gallery-shot" on:click={() => openLightbox(m.items[0])}>
                <img src={gallery[m.items[0]].src} alt={gallery[m.items[0]].alt} />
              </div>
            </div>
          {/if}
        {/each}
      </div>
    </div>
  </section>

  <section id="offers" class="offers wrap">
    <div class="offers-head">
      <h2 class="section-title offers-title">Special offers</h2>
      <div class="offers-wrap">
        <p class="offers-note">Four homes at 7% off the list price <br /> until 31 December.</p>
      </div>
    </div>
    <div class="offers-grid">
      {#each offers as flat (flat.id)}
        <div class="offer">
          <div class="offer-photo">
            <img src={'/media/offers/offers-' + flat.id + '.jpg'} alt={flat.building} />
          </div>
          <div class="offer-info">
            <p class="offer-place">{flat.building}, floor {flat.floor}</p>
            <p class="offer-home">{flat.rooms} bedrooms, {flat.area} m²</p>
            <p class="offer-aspect">{flat.aspect}, {flat.balcony}</p>
            <p class="offer-prices">
              <span class="offer-now">{pounds(salePrice(flat.price))}</span>
              <span class="offer-was">{pounds(flat.price)}</span>
            </p>
            <a class="offer-book" href="#visit" on:click|preventDefault={openCallback}>Book a viewing</a>
          </div>
        </div>
      {/each}
    </div>
  </section>

  <section id="catalog" class="catalog wrap">
    <div class="row">
      <div class="col-4">
        <h2 class="section-title">Choose a home</h2>
        <div class="catalog-side">
          <div class="filter" style="margin-top: 44px">
            <p class="filter-label">Bedrooms</p>
            <div class="chips">
              {#each roomOptions as r}
                <button type="button" class="chip" class:chip-on={rooms.indexOf(r) > -1} on:click={() => toggleRooms(r)}>
                  {r === 0 ? 'Studio' : r}
                </button>
              {/each}
            </div>
          </div>

          <div class="filter">
            <p class="filter-label">Area, m²</p>
            <div class="pair">
              <input class="field" type="number" placeholder="from" bind:value={areaFrom} />
              <input class="field" type="number" placeholder="to" bind:value={areaTo} />
            </div>
          </div>

          <div class="filter">
            <p class="filter-label">Price, £</p>
            <div class="pair">
              <input class="field" type="number" placeholder="from" bind:value={priceFrom} />
              <input class="field" type="number" placeholder="to" bind:value={priceTo} />
            </div>
          </div>

          <div class="filter">
            <p class="filter-label">Floor</p>
            <div class="pair">
              <input class="field" type="number" placeholder="from" bind:value={floorFrom} />
              <input class="field" type="number" placeholder="to" bind:value={floorTo} />
            </div>
          </div>

          <div class="filter" style="margin-top: 28px">
            <input class="field" type="text" placeholder="Home number, for example QS-1204" value={query} on:input={onSearch} />
          </div>

          <div class="filter" style="margin-top: 8px">
            <select class="field field-select" bind:value={sort} aria-label="Sort">
              {#each sorts as s}
                <option value={s.value}>{s.label}</option>
              {/each}
            </select>
          </div>

          <a class="catalog-clear link-line" href="#catalog" on:click|preventDefault={clearFilters}>Clear filters</a>
        </div>
      </div>
      <div class="col-8">
        <FlatList {groups} total={flats.length} {favorites} {config} onOpen={openFlat} onFavorite={toggleFavorite} />
      </div>
    </div>
  </section>
  <section id="location" class="location">
    <div class="location-map" bind:this={mapEl}></div>
    <div class="location-card">
      <h2 class="location-title">Around the quay</h2>
      <ul class="location-list" style="margin-top: 28px">
        {#each places as place, i}
          <li class="location-place" class:location-place-on={mapActive === i} on:click={() => flyToPlace(i)}>
            <span class="location-name">{place.name}</span>
            <span class="location-time">{place.distance}</span>
          </li>
        {/each}
      </ul>
    </div>
  </section>

  <section id="mortgage" class="mortgage wrap">
    <div class="row">
      <div class="col-3 mortgage-side">
        <h2 class="section-title">Monthly payments</h2>
        <p class="mortgage-line" style="margin-top: 28px">Prices from the catalogue, shown as a monthly repayment. Rates are indicative.</p>

        <div class="calc" style="margin-top: 56px">
          <label class="calc-control">
            <span class="calc-label">Deposit</span>
            <span class="calc-value">{deposit} %</span>
            <input
              class="calc-range"
              type="range"
              name="deposit"
              min={deposits[0]}
              max={deposits[deposits.length - 1]}
              step="1"
              value={deposit}
              on:input={onCalc}
            />
          </label>
          <label class="calc-control">
            <span class="calc-label">Term</span>
            <span class="calc-value">{term} years</span>
            <input class="calc-range" type="range" name="term" min={terms[0]} max={terms[terms.length - 1]} step="1" value={term} on:input={onCalc} />
          </label>
          <label class="calc-control">
            <span class="calc-label">Rate</span>
            <span class="calc-value">{rates[rateIndex]} %</span>
            <input class="calc-range" type="range" name="rate" min="0" max={rates.length - 1} step="1" value={rateIndex} on:input={onCalc} />
          </label>
        </div>

        <a class="link-line mortgage-ask" href="#visit" on:click|preventDefault={openCallback}>Ask about a mortgage</a>
      </div>

      <div class="offset-1 col-8">
        <div class="mortgage-list">
          {#each calcGroups as group}
            <div class="mortgage-group">
              <h3 class="mortgage-group-title">{group.title}</h3>
              {#each calcRows as row, i}
                {#if row.rooms === group.rooms}
                  <div class="mortgage-row">
                    <div class="mortgage-row-main">
                      <span>{group.label}, {area(row.area)}</span>
                      <span class="mortgage-pay" aria-label={perMonth(row.monthly)}>
                        {#each figure(row.monthly) as ch, k}
                          {#if digits.indexOf(ch) > -1}
                            <span class="roll-col" aria-hidden="true"><span class="roll-strip" style="top: -{Number(ch) * 1.4}em; transition-delay: {k * 45}ms">{#each digits as d}<span>{d}</span>{/each}</span></span>
                          {:else}
                            <span aria-hidden="true">{ch}</span>
                          {/if}
                        {/each}
                        <span aria-hidden="true">&nbsp;a month</span>
                      </span>
                    </div>
                    <p class="mortgage-row-note">{pounds(row.price)} in {row.building}</p>
                  </div>
                {/if}
              {/each}
            </div>
          {/each}
          <div class="mortgage-photo">
            <img src={interior1} alt="A kitchen with windows from floor to ceiling, looking into the courtyard birches" />
          </div>
        </div>
      </div>
    </div>
  </section>

  <section id="progress" class="progress-section">
    <div class="progress-panel">
      {#if construction.length}
        <img class="progress-img progress-img-on" src={construction[construction.length - 1].photos[0]} />
      {/if}
      <div class="progress-shade"></div>
      <div class="progress-band">
        <span class="progress-month">2026</span>
        <h2 class="progress-title">Construction</h2>
        <span class="progress-fact">March to October</span>
      </div>
    </div>
    {#each construction as item, i}
      <div class="progress-panel">
        {#each item.photos as photo, n}
          <img
            class="progress-img"
            class:progress-img-on={progressAir[i] ? n === 0 : n === 1}
            src={photo}
            alt={monthTitle(item.month) + ', ' + (n === 0 ? 'from the air' : 'on the ground')}
          />
        {/each}
        <div class="progress-shade"></div>
        <div class="progress-band">
          <span class="progress-month">{monthTitle(item.month)}</span>
          <h3 class="progress-title">{item.title}</h3>
          <div class="progress-fact">{@html item.html}</div>
        </div>
        <div class="progress-switch">
          <button type="button" class:progress-switch-on={!progressAir[i]} on:click={() => showShot(i, false)}>On the ground</button>
          <button type="button" class:progress-switch-on={progressAir[i]} on:click={() => showShot(i, true)}>From the air</button>
        </div>
      </div>
    {/each}
  </section>

  <section id="news" class="news wrap">
    <div class="row">
      <div class="col-5">
        <div class="news-photo">
          {#each news as item, i}
            <img class:news-photo-on={i === newsActive} src={item.image} alt={item.title} />
          {/each}
        </div>
      </div>
      <div class="offset-1 col-6">
        <h2 class="section-title">News from the quay</h2>
        <div class="news-list" style="margin-top: 48px">
          {#each news as item, i}
            <div
              class="news-item"
              class:news-item-on={i === newsActive}
              bind:this={newsRows[i]}
              on:mouseenter={() => (newsActive = i)}
            >
              <p class="news-date">{newsDate(item.date)}</p>
              <h3 class="news-title">{item.title}</h3>
              <div class="news-text">{@html item.html}</div>
            </div>
          {/each}
        </div>
      </div>
    </div>
  </section>

  <section id="questions" class="questions">
    <div class="row questions-field">
      <div class="col-5 questions-side">
        <h2 class="section-title">Questions</h2>
        <ul class="questions-facts" style="margin-top: 32px">
          {#each officeFacts as fact}
            <li>{fact}</li>
          {/each}
        </ul>

        <div class="faq" style="margin-top: 36px">
          {#each faq as item, i}
            <div class="faq-item" class:faq-item-open={faqOpen === i}>
              <div class="faq-q" on:click={() => toggleFaq(i)}>
                <span>{item.q}</span>
                <span class="faq-chevron"></span>
              </div>
              {#if faqOpen === i}
                <div class="faq-a" transition:slide={{ duration: 320 }}>{@html item.a}</div>
              {/if}
            </div>
          {/each}
        </div>

        <div class="questions-actions" style="margin-top: 40px">
          <button class="btn btn-accent" type="button" on:click={openLead}>Book a viewing</button>
          <a class="link-line" href="#visit" on:click|preventDefault={openCallback} style="margin-left: 28px">Call me back</a>
        </div>
      </div>

      <div class="offset-1 col-6">
        <div class="questions-pics">
          <div class="questions-tall">
            <img src="/media/photos/interior-living.png" alt="A living room in Quayside looking out over the water" />
          </div>
          <div class="questions-square">
            <img src="/media/photos/facade-detail.png" />
          </div>
        </div>
      </div>
    </div>
  </section>

  <section id="visit" class="visit">
    <img class="visit-photo" src="/media/photos/dusk-promenade.png" alt="The promenade along the granite edge at dusk" />
    <div class="visit-card">
      <h2 class="visit-title">{visit.title || ''}</h2>
      <p class="visit-address" style="margin-top: 18px">{visit.address || ''}</p>
      <dl class="visit-hours" style="margin-top: 20px">
        {#each visit.hours || [] as line}
          <div>
            <dt>{hoursDays(line)}</dt>
            <dd>{hoursTime(line)}</dd>
          </div>
        {/each}
      </dl>
      <p class="visit-phone" style="margin-top: 16px">
        <a class="link-line" href={'tel:' + (visit.phone || '').replace(/\s/g, '')}>{visit.phone || ''}</a>
      </p>

      <form class="visit-form" on:submit|preventDefault={sendVisit} style="margin-top: 28px">
        <input class="visit-field" type="text" name="name" placeholder={visitFields.name} bind:value={visitName} />
        <input class="visit-field" type="tel" name="phone" placeholder={visitFields.phone} bind:value={visitPhone} />
        <select class="visit-field visit-select" name="time" bind:value={visitTime}>
          <option value="">{visitFields.time}</option>
          {#each visitFields.timeOptions || [] as option}
            <option value={option}>{option}</option>
          {/each}
        </select>
        {#if visitSent}
          <p class="visit-done">{visitMessage}</p>
        {:else}
          <p class="visit-note">{visit.note || ''}</p>
        {/if}
        <button class="btn btn-accent visit-btn" type="submit">{visit.button || ''}</button>
      </form>
    </div>
  </section>

  <footer id="footer" class="footer">
    <div class="footer-photo">
      <img src="/media/photos/cranes-dusk.png" />
    </div>
    <div class="footer-side">
      <a class="footer-mark" href="#hero">{title}</a>
      <nav class="footer-menu" style="margin-top: 40px">
        {#each footerMenu as item}
          <a href={item.href}>{item.label}</a>
        {/each}
      </nav>
      <div class="footer-info" style="margin-top: 64px">
        <div class="footer-contact">
          <a class="link-line" href={'tel:' + (contact.phone || '').replace(/\s/g, '')}>{contact.phone || ''}</a>
          <a class="link-line" href={'mailto:' + (contact.email || '')}>{contact.email || ''}</a>
          <span>{contact.address || ''}</span>
        </div>
        <div class="footer-social">
          {#each footer.social || [] as item}
            <a class="link-line" href={item.url}>{item.name}</a>
          {/each}
        </div>
      </div>
      <div class="footer-legal">
        <p>{footer.legal || ''}</p>
        <p>{footer.credits || ''}</p>
        <div class="footer-chat" bind:this={chatDock}></div>
      </div>
    </div>
  </footer>
</main>

{#if lightboxIndex > -1}
  <Lightbox items={gallery} start={lightboxIndex} />
{/if}

{#if activeFlat}
  <FlatModal flat={activeFlat} {favorites} {config} onOpen={openFlat} onFavorite={toggleFavorite} />
{/if}

{#if leadOpen}
  <ModalLead texts={visit} {config} />
{/if}

{#if callbackOpen}
  <ModalCallback texts={callback} {config} />
{/if}

<style>
  .top {
    z-index: 9999 !important;
    display: flex;
    align-items: center;
    justify-content: flex-end;
    height: 84px;
    padding: 0 var(--margin);
    font-size: 15px;
    transition: top 0.45s var(--ease-out);
    will-change: top;
  }

  .top-band {
    position: absolute;
    top: 0;
    right: 0;
    width: 0%;
    height: 100%;
    background: var(--bg);
    box-shadow: 0 20px 80px rgba(51, 50, 49, 0.02);
    transition: width 0.7s var(--ease-in-out);
    will-change: width;
    z-index: -1;
  }

  .top-logo {
    font-weight: 600;
    letter-spacing: -0.01em;
    margin-right: 40px;
    will-change: transform;
  }

  .top-nav {
    display: flex;
    gap: 24px;
  }

  .top-nav a {
    will-change: transform;
  }

  .top-call {
    margin-left: 24px;
  }

  .top-btn {
    font-size: 18px !important;
  }

  .hero {
    position: relative;
  }

  .hero-stage {
    position: relative;
    width: 100%;
    height: 100vh;
    overflow: hidden;
    display: grid;
    grid-template-rows: auto 1fr auto;
    background: #c6c7c6;
  }

  .hero-bg {
    position: absolute;
    top: 0;
    left: -40px;
    right: -40px;
    height: calc(100% + 80px);
    background-size: cover;
    background-position: center;
    filter: blur(24px);
    transform: scale(1.08);
    will-change: top, filter;
  }

  .hero-video {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: opacity 1.2s var(--ease-soft);
    will-change: opacity;
  }

  .hero-scrim {
    position: absolute;
    pointer-events: none;
    will-change: opacity;
  }

  .hero-scrim-top {
    top: 0;
    left: 0;
    width: 62%;
    height: 52%;
    background: radial-gradient(ellipse at 0% 0%, rgba(238, 236, 234, 0.62) 0%, rgba(238, 236, 234, 0) 70%);
  }

  .hero-scrim-bottom {
    right: 0;
    bottom: 0;
    width: 80%;
    height: 52%;
    background: radial-gradient(ellipse at 100% 100%, rgba(238, 236, 234, 0.62) 0%, rgba(238, 236, 234, 0) 70%);
  }

  .hero-title {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 2;
    display: grid;
    grid-template-rows: auto 1fr auto;
    font-size: var(--step-display);
    visibility: hidden;
  }

  .hero-line {
    position: relative;
    display: block;
    font-size: var(--step-display);
    font-weight: 300 !important;
    font-optical-sizing: auto;
    line-height: 0.9 !important;
    letter-spacing: -0.05em;
    white-space: nowrap;
    will-change: transform;
  }

  .hero-line-1 {
    grid-row: 1;
    padding: 0 var(--margin);
    padding-top: calc(72px + 0.032em);
  }

  .hero-line-2 {
    grid-row: 3;
    padding-left: 28vw;
    padding-bottom: 0;
    margin-bottom: calc(24px - 0.203em);
  }

  .hero-copy {
    position: absolute;
    z-index: 3;
    top: 38%;
    left: var(--margin);
    visibility: hidden;
    font-size: var(--step-body);
    line-height: 1.5;
  }

  .hero-copy > * {
    position: relative;
    top: 0;
    will-change: top, opacity;
  }

  .about {
    padding-top: 180px;
  }

  .about-lead {
    max-width: 50%;
    font-size: var(--step-lead);
    line-height: 1.35;
  }

  .about-photo {
    position: relative;
    overflow: hidden;
    background: var(--surface);
    clip-path: inset(0);
    will-change: clip-path;
  }

  .about-photo img {
    position: relative;
    top: 0;
    left: 0;
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: 15% 50%;
    transition: opacity 0.9s var(--ease-soft);
    animation: about-drift 26s ease-in-out infinite alternate;
    will-change: object-position, opacity;
  }

  @keyframes about-drift {
    from {
      object-position: 15% 50%;
    }
    to {
      object-position: 85% 50%;
    }
  }

  .about-paper {
    position: absolute;
    top: 0;
    right: 0;
    padding: 20px;
    background: var(--paper);
    font-size: var(--step-body);
    line-height: 1.5;
    will-change: transform;
  }

  .about-tile {
    background: var(--field-1);
    overflow: hidden;
    will-change: transform;
  }

  .about-tile svg {
    display: block;
    width: 100%;
    height: 100%;
  }

  .about-water {
    fill: var(--ink);
  }

  .about-dot {
    fill: var(--ink);
  }

  .about-facts {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 24px var(--gutter);
  }

  .about-fact {
    position: relative;
    will-change: transform, opacity, top;
  }

  .about-fact dt {
    font-size: var(--step-small);
    color: var(--muted);
  }

  .about-fact dd {
    margin-top: 4px !important;
    font-size: var(--step-body);
    line-height: 1.35;
  }

  .about-film iframe {
    will-change: transform;
  }

  .about-film-text {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
  }

  .about-film-title {
    max-width: 7em;
    font-size: var(--step-head);
    font-weight: 300;
    line-height: 0.95;
    letter-spacing: -0.03em;
  }

  .about-film-caption {
    max-width: 28ch;
    margin-top: 24px !important;
    color: var(--muted);
  }

  .features {
    padding-top: 200px;
    padding-bottom: 160px;
  }

  .features-head {
    position: sticky;
    top: 140px;
    padding-right: 24px;
    will-change: transform, opacity;
  }

  .features-title {
    white-space: nowrap;
  }

  .feature-row {
    margin-bottom: 72px;
    will-change: transform;
  }

  .feature-row:last-child {
    margin-bottom: 0;
  }

  .feature-title {
    display: flex;
    align-items: center;
    gap: 14px;
    font-size: var(--step-lead) !important;
    line-height: 1.15 !important;
    letter-spacing: -0.01em;
  }

  .feature-icon {
    flex: none;
    object-fit: contain;
    filter: drop-shadow(0 4px 10px rgba(51, 50, 49, 0.05));
    will-change: transform;
  }

  .feature-text {
    max-width: 34ch;
    margin-top: 14px !important;
  }

  .feature-side {
    display: flex;
    justify-content: flex-end;
    align-items: flex-start;
  }

  .feature-photo {
    overflow: hidden;
    background: var(--surface);
    clip-path: inset(0);
    transition: box-shadow 0.5s var(--ease-out);
    will-change: transform, box-shadow, clip-path;
  }

  .feature-photo:hover {
    box-shadow: 0 40px 100px rgba(51, 50, 49, 0.08);
  }

  .feature-photo img {
    position: relative;
    top: 0;
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
    transform: scale(1.05);
    will-change: transform, top;
  }

  .gallery {
    padding-top: 0;
    padding-bottom: 180px;
  }

  .opening {
    position: relative;
    height: 200vh;
    margin-bottom: 150px;
  }

  .opening-stage {
    position: sticky;
    top: 0;
    height: 100vh;
    overflow: hidden;
  }

  .opening-window {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    width: 40%;
    height: 76%;
    margin: auto;
    overflow: hidden;
    background: var(--ink);
    will-change: width, height;
  }

  .opening-window img {
    position: absolute;
    top: 50%;
    left: 50%;
    display: block;
    width: 100vw;
    height: 100vh;
    max-width: none;
    margin: -50vh 0 0 -50vw;
    object-fit: cover;
    object-position: 50% 60%;
    will-change: transform;
  }

  .gallery-head {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
    margin-bottom: 32px;
  }

  .gallery-count {
    font-size: var(--step-body);
    color: var(--muted);
  }

  .gallery-row {
    width: 100%;
    will-change: transform;
  }

  .gallery-module {
    display: flex;
    flex-direction: column;
    gap: 12px;
    will-change: transform, width;
  }

  .gallery-shot {
    position: relative;
    flex: 1;
    min-height: 0;
    overflow: hidden;
    cursor: pointer;
    background: var(--surface);
    transition: box-shadow 0.6s var(--ease-out);
    will-change: transform, box-shadow;
  }

  .gallery-shot:hover {
    box-shadow: 0 50px 120px rgba(51, 50, 49, 0.1);
  }

  .gallery-shot img {
    position: relative;
    top: -6px;
    display: block;
    width: 100%;
    height: calc(100% + 12px);
    object-fit: cover;
    transition: transform 0.9s var(--ease-out);
    will-change: transform;
  }

  .gallery-shot:hover img {
    transform: scale(1.04);
  }

  .catalog {
    padding-top: 168px;
    padding-bottom: 176px;
    background: var(--surface);
  }

  .catalog-side {
    position: sticky;
    top: 112px;
  }

  .filter {
    position: relative;
    margin-top: 22px;
    will-change: top, opacity;
  }

  .filter-label {
    margin-bottom: 8px !important;
    font-size: var(--step-small);
    line-height: 1.4;
    color: var(--muted);
  }

  .chips {
    display: flex;
    gap: 6px;
  }

  .chip {
    min-width: 54px;
    height: 46px;
    padding: 0 16px;
    border: 0;
    border-radius: 0;
    background: var(--paper);
    font-size: var(--step-body);
    line-height: 1;
    cursor: pointer;
    transition: background-color 160ms ease, color 160ms ease, transform 140ms var(--ease-out);
    will-change: transform;
  }

  .chip:hover {
    background: var(--bg);
  }

  .chip:active {
    transform: scale(0.95);
  }

  .chip:focus-visible {
    outline: 2px solid var(--ink);
    outline-offset: 2px;
  }

  .chip-on,
  .chip-on:hover {
    background: var(--accent);
    color: var(--accent-ink);
  }

  .pair {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 6px;
  }

  .field {
    display: block;
    width: 100%;
    height: 48px;
    padding: 0 14px;
    border: 0;
    border-radius: 0;
    background: var(--paper);
    color: var(--ink);
    font-family: var(--font);
    font-size: var(--step-body);
    font-variant-numeric: tabular-nums;
    -moz-appearance: textfield;
    appearance: textfield;
    transition: background-color 160ms ease, outline-color 160ms ease;
  }

  .field::placeholder {
    color: var(--muted);
    opacity: 1;
  }

  .field::-webkit-outer-spin-button,
  .field::-webkit-inner-spin-button {
    -webkit-appearance: none;
    margin: 0;
  }

  .field:hover {
    background-color: #fbfaf8;
  }

  .field:focus {
    outline: 2px solid var(--ink);
    outline-offset: -2px;
  }

  .field-select {
    padding-right: 44px;
    appearance: none;
    cursor: pointer;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'%3E%3Cpath d='M1 1.5l5 5 5-5' fill='none' stroke='%23333231' stroke-width='1.6'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 16px center;
  }

  .catalog-clear {
    display: inline-block;
    margin-top: 22px;
    font-size: var(--step-body);
  }

  .cookie {
    position: relative;
    z-index: 1;
    display: flex;
    align-items: center;
    gap: 24px;
    height: 84px;
    padding: 0 var(--margin);
    background: #ffffff;
    color: #333333;
    font-family: 'Montserrat', Arial, sans-serif;
    font-size: 13px;
    line-height: 1.45;
  }

  .cookie-text {
    max-width: 250px;
  }

  .cookie-actions {
    display: flex;
    align-items: center;
    gap: 6px;
  }

  .cookie-btn,
  .cookie-link {
    height: 36px;
    padding: 0 16px;
    border: 0;
    border-radius: 0;
    font-family: 'Montserrat', Arial, sans-serif !important;
    font-size: 13px;
    font-weight: 600;
    cursor: pointer;
    transition: background-color 160ms ease, transform 140ms var(--ease-out);
  }

  .cookie-btn {
    background: #333231;
    color: #ffffff;
  }

  .cookie-btn:hover {
    background: #1f1e1d;
  }

  .cookie-link {
    background: #ececec;
    color: #333333;
  }

  .cookie-link:hover {
    background: #e0e0e0;
  }

  .cookie-btn:active,
  .cookie-link:active {
    transform: scale(0.96);
  }

  .location {
    position: relative;
    height: 100vh;
    min-width: 1200px;
    overflow: hidden;
    background: var(--surface);
  }

  .location-map {
    position: absolute;
    top: 0;
    left: 0;
    display: block;
    width: 100%;
    height: 100%;
  }

  .location-card {
    position: absolute;
    top: 0;
    bottom: 0;
    left: calc(var(--margin) + (100% - 2 * var(--margin) - 3 * var(--gutter)) / 4 + var(--gutter));
    width: calc((100% - 2 * var(--margin) - 3 * var(--gutter)) / 4);
    height: max-content;
    min-height: 52vh;
    margin: auto 0;
    padding: 36px 24px 32px;
    background: var(--ink);
    color: var(--paper);
    will-change: transform;
  }

  .location-title {
    font-size: var(--step-lead) !important;
    font-weight: 400 !important;
    line-height: 1.1 !important;
    letter-spacing: -0.01em;
    color: var(--paper) !important;
  }

  .location-list {
    display: grid;
    gap: 11px;
    margin: 0;
    padding: 0;
    list-style: none;
  }

  .location-place {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    font-size: 15px;
    line-height: 1.35;
    font-variant-numeric: tabular-nums;
  }

  .location-place {
    cursor: pointer;
    transition: color 160ms ease;
  }

  .location-place:hover .location-name,
  .location-place-on .location-name {
    text-decoration: underline;
    text-underline-offset: 0.22em;
    text-decoration-thickness: 0.07em;
  }

  .location-time {
    flex: none;
    max-width: 54%;
    text-align: right;
    color: #7d7873;
  }

  :global(.map-quay) {
    width: 22px;
    height: 22px;
    border: 3px solid var(--paper);
    background: var(--accent);
    cursor: pointer;
  }

  :global(.map-dot) {
    width: 13px;
    height: 13px;
    border: 2px solid var(--paper);
    border-radius: 50%;
    background: var(--ink);
    cursor: pointer;
    transition: background-color 0.2s ease;
  }

  :global(.map-dot:hover) {
    background: var(--accent);
  }

  :global(.maplibregl-popup-content) {
    padding: 12px 16px !important;
    border-radius: 0 !important;
    background: var(--paper) !important;
    color: var(--ink);
    font-family: var(--font);
    font-size: 15px;
    line-height: 1.4;
  }

  :global(.maplibregl-popup-content strong) {
    font-weight: 500;
  }

  :global(.maplibregl-popup-tip) {
    display: none;
  }

  .location :global(.maplibregl-ctrl-bottom-right) {
    right: 22px;
    bottom: 14px;
  }

  :global(.maplibregl-ctrl-attrib) {
    font-family: var(--font);
    font-size: 13px;
  }

  .mortgage {
    min-width: 1200px;
    padding-top: 184px;
    padding-bottom: 184px;
  }

  .mortgage-side {
    position: sticky;
    top: 112px;
    align-self: flex-start;
  }

  .mortgage-line {
    margin-top: 28px !important;
    max-width: 30ch;
    font-size: var(--step-body);
    line-height: 1.55;
  }

  .calc {
    display: grid;
    gap: 34px;
  }

  .calc-control {
    display: grid;
    grid-template-columns: 1fr auto;
    align-items: baseline;
    row-gap: 14px;
    cursor: pointer;
  }

  .calc-label {
    font-size: var(--step-small);
    color: var(--muted);
  }

  .calc-value {
    font-size: var(--step-head);
    font-weight: 300;
    line-height: 0.9;
    letter-spacing: -0.03em;
    font-variant-numeric: tabular-nums;
    text-align: right;
  }

  .calc-range {
    grid-column: 1 / -1;
    width: 100%;
    height: 28px;
    margin: 0;
    background: transparent;
    -webkit-appearance: none;
    appearance: none;
    cursor: pointer;
  }

  .calc-range::-webkit-slider-runnable-track {
    height: 4px;
    background: var(--field-1);
  }

  .calc-range::-moz-range-track {
    height: 4px;
    background: var(--field-1);
  }

  .calc-range::-webkit-slider-thumb {
    width: 20px;
    height: 20px;
    margin-top: -8px;
    border: 0;
    border-radius: 0;
    background: var(--ink);
    -webkit-appearance: none;
    appearance: none;
    transition: transform 140ms var(--ease-out), background-color 160ms ease;
  }

  .calc-range::-moz-range-thumb {
    width: 20px;
    height: 20px;
    border: 0;
    border-radius: 0;
    background: var(--ink);
  }

  .calc-range:hover::-webkit-slider-thumb {
    background: var(--accent);
  }

  .calc-range:active::-webkit-slider-thumb {
    transform: scale(1.2);
  }

  .calc-range:focus {
    outline: none;
  }

  .calc-range:focus-visible {
    outline: 2px solid var(--ink);
    outline-offset: 4px;
  }

  .mortgage-ask {
    display: inline-block;
    margin-top: 40px;
    font-size: var(--step-body);
  }

  .mortgage-photo {
    width: 100%;
    margin-top: 12px;
    break-inside: avoid;
    aspect-ratio: 1 / 1;
    overflow: hidden;
    background: var(--surface);
  }

  .mortgage-photo img {
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: 58% 50%;
  }

  .mortgage-list {
    column-count: 2;
    column-gap: 64px;
    padding-top: 10px;
  }

  .mortgage-group {
    break-inside: avoid;
    padding-bottom: 48px;
  }

  .mortgage-group-title {
    margin-bottom: 14px !important;
    font-size: var(--step-lead) !important;
    line-height: 1.2 !important;
    letter-spacing: -0.01em;
  }

  .mortgage-row {
    position: relative;
    padding: 11px 0;
    will-change: transform;
  }

  .mortgage-row-main {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    font-size: var(--step-body);
    line-height: 1.4;
  }

  .mortgage-pay {
    display: inline-flex;
    align-items: flex-start;
    font-variant-numeric: tabular-nums;
    white-space: nowrap;
  }

  .roll-col {
    display: inline-block;
    height: 1.4em;
    overflow: hidden;
  }

  .roll-strip {
    position: relative;
    display: flex;
    flex-direction: column;
    transition: top 0.9s var(--ease-expo);
    will-change: top;
  }

  .roll-strip span {
    display: block;
    height: 1.4em;
    line-height: 1.4em;
  }

  .mortgage-row-note {
    margin-top: 2px !important;
    font-size: var(--step-small);
    line-height: 1.45;
    color: var(--muted);
    font-variant-numeric: tabular-nums;
  }

  .progress-section {
    position: relative;
    min-width: 1200px;
  }

  .progress-panel {
    position: sticky;
    top: 0;
    height: 100vh;
    overflow: hidden;
    background: var(--ink);
  }

  .progress-img {
    position: absolute;
    top: 0;
    left: 0;
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
    opacity: 0;
    transform: scale(1.08);
    transition: opacity 0.6s var(--ease-soft);
    will-change: opacity, transform;
  }

  .progress-img-on {
    opacity: 1;
  }

  .progress-shade {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: #1e1d1c;
    opacity: 0;
    pointer-events: none;
    will-change: opacity;
  }

  .progress-band {
    position: absolute;
    top: 50%;
    left: 0;
    right: 0;
    display: flex;
    align-items: center;
    height: 90px;
    margin-top: -45px;
    padding: 0 var(--margin);
    background: rgba(30, 29, 28, 0.58);
    color: var(--paper);
  }

  .progress-month {
    flex: none;
    width: calc((100% - 11 * var(--gutter)) / 6 + 2 * var(--gutter));
    font-size: 15px;
    font-variant-numeric: tabular-nums;
  }

  .progress-title {
    flex: 1;
    font-size: var(--step-lead) !important;
    font-weight: 400 !important;
    line-height: 1.1 !important;
    letter-spacing: 0.02em;
    text-transform: uppercase;
    color: var(--paper) !important;
  }

  .progress-fact {
    flex: none;
    font-size: 15px;
    text-align: right;
  }

  .progress-switch {
    position: absolute;
    left: var(--margin);
    bottom: 32px;
    display: flex;
    gap: 4px;
    padding: 6px;
    background: rgba(30, 29, 28, 0.58);
  }

  .progress-switch button {
    height: 40px;
    padding: 0 16px;
    border: 0;
    border-radius: 0;
    background: transparent;
    color: rgba(247, 245, 242, 0.62);
    font-size: 15px;
    cursor: pointer;
    transition: background-color 200ms ease, color 200ms ease, transform 140ms var(--ease-out);
  }

  .progress-switch button:hover {
    color: var(--paper);
  }

  .progress-switch button:active {
    transform: scale(0.96);
  }

  .progress-switch button:focus-visible {
    outline: 2px solid var(--paper);
    outline-offset: 2px;
  }

  .progress-switch .progress-switch-on,
  .progress-switch .progress-switch-on:hover {
    background: var(--paper);
    color: var(--ink);
  }

  .news {
    min-width: 1200px;
    padding-top: 184px;
    padding-bottom: 184px;
  }

  .news-photo {
    position: sticky;
    top: 12vh;
    aspect-ratio: 4 / 5;
    overflow: hidden;
    background: var(--surface);
    clip-path: inset(0);
    will-change: clip-path;
  }

  .news-photo img {
    position: absolute;
    top: 0;
    left: 0;
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
    opacity: 0;
    transition: opacity 0.3s ease;
    will-change: opacity;
  }

  .news-photo .news-photo-on {
    opacity: 1;
  }

  .news-list {
    display: grid;
  }

  .news-item {
    position: relative;
    padding: 22px 0;
    cursor: default;
    will-change: transform, opacity;
  }

  .news-item .news-title {
    color: var(--muted) !important;
    transition: color 0.3s ease;
  }

  .news-item-on .news-title {
    color: var(--ink) !important;
  }

  .news-date {
    font-size: var(--step-small);
    color: #aaa59f;
    font-variant-numeric: tabular-nums;
  }

  .news-title {
    margin-top: 8px !important;
    font-size: var(--step-lead) !important;
    line-height: 1.15 !important;
    letter-spacing: -0.01em;
  }

  .news-text {
    display: -webkit-box;
    max-width: 46ch;
    margin-top: 12px;
    overflow: hidden;
    color: var(--muted);
    font-size: var(--step-body);
    line-height: 1.55;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
  }

  .news-text :global(p) {
    display: inline;
  }

  .news-text :global(p + p)::before {
    content: ' ';
  }

  .questions {
    min-width: 1200px;
    padding: 0 var(--margin);
    background: var(--field-1);
  }

  .questions-side {
    padding-top: 104px;
    padding-bottom: 72px;
  }

  .questions-facts {
    display: grid;
    gap: 4px;
    margin: 0;
    padding: 0;
    list-style: none;
    font-size: var(--step-body);
    line-height: 1.45;
  }

  .faq-item {
    position: relative;
    padding: 13px 0;
    will-change: top, opacity;
  }

  .faq-q {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 24px;
    font-size: var(--step-body);
    line-height: 1.35;
    cursor: pointer;
    user-select: none;
  }

  .faq-chevron {
    position: relative;
    flex: none;
    width: 14px;
    height: 14px;
    transition: transform 0.3s var(--ease-out);
    will-change: transform;
  }

  .faq-chevron::before,
  .faq-chevron::after {
    content: '';
    position: absolute;
    top: 6px;
    width: 9px;
    height: 2px;
    background: var(--ink);
  }

  .faq-chevron::before {
    left: 0;
    transform: rotate(45deg);
  }

  .faq-chevron::after {
    right: 0;
    transform: rotate(-45deg);
  }

  .faq-q:hover .faq-chevron {
    transform: translateY(2px);
  }

  .faq-item-open .faq-chevron,
  .faq-item-open .faq-q:hover .faq-chevron {
    transform: rotate(180deg);
  }

  .faq-a {
    max-width: 52ch;
    padding-top: 10px;
    font-size: var(--step-body);
    line-height: 1.55;
  }

  .faq-a :global(p + p) {
    margin-top: 8px !important;
  }

  .questions-actions {
    display: flex;
    align-items: center;
    font-size: var(--step-body);
  }

  .questions-pics {
    position: relative;
    margin-top: -96px;
    padding-bottom: 80px;
  }

  .questions-tall {
    overflow: hidden;
    background: var(--surface);
    clip-path: inset(0);
    will-change: clip-path;
  }

  .questions-tall img,
  .questions-square img {
    display: block;
    width: 100%;
    height: auto;
  }

  .questions-square {
    position: absolute;
    left: -64px;
    bottom: 40px;
    width: 33.333%;
    overflow: hidden;
    background: var(--surface);
  }

  .visit {
    position: relative;
    height: 100vh;
    min-width: 1200px;
    overflow: hidden;
    background: var(--ink);
  }

  .visit-photo {
    position: absolute;
    top: 0;
    left: 0;
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .visit-card {
    position: absolute;
    top: 108px;
    right: 60px;
    width: 380px;
    padding: 32px 32px 0;
    background: var(--paper);
  }

  .visit-title {
    font-size: var(--step-lead) !important;
    font-weight: 300 !important;
    line-height: 1.1 !important;
    letter-spacing: -0.01em;
  }

  .visit-address {
    margin-top: 18px !important;
  }

  .visit-phone {
    margin-top: 16px !important;
  }

  .visit-address,
  .visit-phone {
    font-size: var(--step-body);
    line-height: 1.4;
  }

  .visit-phone {
    font-variant-numeric: tabular-nums;
  }

  .visit-hours {
    margin-top: 20px !important;
    display: grid;
    gap: 2px;
  }

  .visit-hours div {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    font-size: var(--step-body);
    line-height: 1.45;
  }

  .visit-hours dd {
    font-variant-numeric: tabular-nums;
  }

  .visit-form {
    display: grid;
    gap: 8px;
  }

  .visit-field {
    display: block;
    width: 100%;
    height: 48px;
    padding: 0 14px;
    border: 0;
    border-radius: 0;
    background: var(--surface);
    color: var(--ink);
    font-family: var(--font);
    font-size: var(--step-body);
    transition: background-color 160ms ease;
  }

  .visit-field::placeholder {
    color: #aaa59f;
    opacity: 1;
  }

  .visit-field:hover {
    background-color: #dcd8d3;
  }

  .visit-field:focus {
    outline: 2px solid var(--ink);
    outline-offset: -2px;
  }

  .visit-select {
    appearance: none;
    cursor: pointer;
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'%3E%3Cpath d='M1 1.5l5 5 5-5' fill='none' stroke='%23333231' stroke-width='1.6'/%3E%3C/svg%3E");
    background-repeat: no-repeat;
    background-position: right 16px center;
  }

  .visit-note,
  .visit-done {
    margin-top: 6px !important;
    font-size: var(--step-small);
    line-height: 1.45;
  }

  .visit-note {
    color: #aaa59f;
  }

  .visit-btn {
    width: calc(100% + 64px);
    margin: 20px -32px 0;
  }

  .footer {
    display: grid;
    grid-template-columns: 1fr 1fr;
    height: 100vh;
    min-height: 760px;
    min-width: 1200px;
    background: var(--bg);
  }

  .footer-photo {
    position: relative;
    overflow: hidden;
    background: var(--ink);
  }

  .footer-photo img {
    position: absolute;
    top: 0;
    left: 0;
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
    object-position: 34% 70%;
    transform: scale(1.3);
    transform-origin: 30% 85%;
  }

  .footer-side {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    padding: 64px var(--margin) 32px 128px;
  }

  .footer-mark {
    position: relative;
    font-size: 15px;
    font-weight: 600;
    letter-spacing: -0.01em;
  }

  .footer-menu {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    gap: 26px;
  }

  .footer-menu a {
    position: relative;
    left: 0;
    font-size: 2.6vw;
    font-weight: 500;
    line-height: 1;
    letter-spacing: -0.025em;
    transition: left 0.35s var(--ease-out), color 0.2s ease;
    will-change: left;
  }

  .footer-menu:hover a {
    color: var(--muted) !important;
  }

  .footer-menu a:hover {
    left: 12px;
    color: var(--ink) !important;
  }

  .footer-info {
    display: flex;
    gap: 64px;
    font-size: 15px;
    line-height: 1.6;
  }

  .footer-contact,
  .footer-social {
    display: flex;
    flex-direction: column;
    align-items: flex-start;
  }

  .footer-contact {
    font-variant-numeric: tabular-nums;
  }

  .footer-legal {
    margin-top: auto;
    font-size: var(--step-small);
    line-height: 1.5;
    color: var(--muted);
  }

  .footer-chat {
    margin-top: 20px;
  }

  :global(.cw-btn) {
    position: relative !important;
    right: auto !important;
    bottom: auto !important;
    height: 40px !important;
    padding: 0 18px !important;
    background: #4a4846 !important;
    box-shadow: none !important;
    font-size: 13px !important;
    line-height: 40px !important;
  }

  :global(.cw-btn:hover) {
    background: var(--ink) !important;
  }

  .offers {
    max-width: 100vw;
    padding-top: 160px;
    padding-bottom: 160px;
  }

  .offers-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 24px;
    margin-bottom: 64px;
  }

  .offers-title {
    font-size: 64px !important;
  }

  .offers-wrap {
    max-width: 420px;
    width: 100%;
  }

  .offers-note {
    font-size: 24px;
    line-height: 1.35;
  }

  .offers-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 24px;
  }

  .offer-photo {
    margin-bottom: 20px;
    overflow: hidden;
    background: var(--surface);
    aspect-ratio: 4 / 5;
  }

  .offer-photo img {
    display: block;
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
  .offer-info {
    display: flex;
    flex-direction: column;
    gap: 6px;
  }
  .offer-place,
  .offer-was,
  .offer-aspect{
    font-size: 14px;
    line-height: 1.45;
    color: var(--muted);
  }

  .offer-home {
    font-size: 24px;
    line-height: 1.35;
  }


  .offer-prices {
    display: flex;
    margin-top: 14px !important;
    align-items: baseline;
    gap: 12px;
  }

  .offer-now {
    font-size: 24px;
    font-weight: 500;
    line-height: 1.2;
  }

  .offer-was {
    text-decoration: line-through;
  }

  .offer-book {
    display: inline-block;
    margin-top: 16px;
    font-size: 14px;
    text-decoration: underline !important;
    text-underline-offset: 0.2em;
  }

  @media (max-width: 1200px) {
    .offers {
      padding-top: 120px;
      padding-bottom: 120px;
    }

    .offers-title {
      font-size: 48px !important;
    }

    .offers-head {
      display: block;
      margin-bottom: 48px;
    }

    .offers-note {
      max-width: 360px;
      margin-top: 12px !important;
      text-align: left;
    }

    .offers-grid {
      grid-template-columns: repeat(2, 1fr);
      column-gap: 24px;
      row-gap: 48px;
    }
  }

  @media (max-width: 767px) {
    .offers {
      padding-top: 96px;
      padding-bottom: 96px;
    }

    .offers.wrap {
      padding-left: 16px;
      padding-right: 0;
    }

    .offers-head {
      padding-right: 16px;
    }

    .offers-title {
      font-size: 34px !important;
    }

    .offers-note {
      max-width: none;
    }

    .offers-grid {
      display: flex;
      gap: 12px;
      overflow-x: auto;
      scroll-snap-type: x mandatory;
      padding-right: 16px;
      scrollbar-width: none;
    }

    .offers-grid::-webkit-scrollbar {
      display: none;
    }

    .offer {
      flex: none;
      width: 280px;
      scroll-snap-align: start;
    }

    .offer-home {
      font-size: 20px;
    }
  }
</style>
