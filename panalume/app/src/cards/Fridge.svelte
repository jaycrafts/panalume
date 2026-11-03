<script module>
  import { states } from '../lib/ha.svelte.js';
  // Samsung fridges (SmartThings or LocalThings: …_fridge_temperature or
  // …_cooler_temperature next to …_freezer_temperature), and any other
  // integration that names them the same way.
  const has = (id) => (id && states.has(id) ? id : '');
  const keys = () => [...states.keys()];
  function fridges() {
    const out = [];
    for (const id of states.keys()) {
      const m = /^sensor\.(.+)_(fridge|cooler)_temperature$/.exec(id);
      if (m && states.has(`sensor.${m[1]}_freezer_temperature`)) out.push({ p: m[1], fridge: id });
    }
    return out;
  }
  // The device's name without the model code: "Samsung Refrigerator (TP2X_REF_20K) Fridge temperature" → "Samsung Refrigerator".
  const deviceName = (id) => (states.get(id)?.attributes.friendly_name || '').replace(/\s*(fridge|cooler)\s+temperature$/i, '').replace(/\s*\([^)]*\)\s*/g, ' ').trim();
  function fill(f) {
    const p = f.p;
    const of = (re) => keys().filter((k) => k.includes(`.${p}_`) && re.test(k)).sort();
    return {
      name: deviceName(f.fridge), fridge: f.fridge, freezer: has(`sensor.${p}_freezer_temperature`),
      fridge_set: has(`number.${p}_fridge_setpoint`) || has(`number.${p}_cooler_setpoint`) || has(`number.${p}_fridge_target_temperature`),
      freezer_set: has(`number.${p}_freezer_setpoint`) || has(`number.${p}_freezer_target_temperature`),
      doors: of(/^binary_sensor\..*_door(_open)?$/),
      zones: of(/^select\..*_zone_mode$/),
      features: of(/^switch\..*(ice|rapid|sabbath|power_cool|power_freeze|vacation)/),
      ice_status: of(/^sensor\..*ice.*making_status$/),
      defrost: has(`binary_sensor.${p}_defrost_active`), filter: has(`sensor.${p}_filter_usage`) || has(`sensor.${p}_water_filter_usage`), filter_status: has(`sensor.${p}_filter_status`),
      alarm: has(`sensor.${p}_alarm_code`), power: has(`sensor.${p}_power`), energy: has(`sensor.${p}_energy_this_month`), energy_last: has(`sensor.${p}_energy_last_month`),
    };
  }
  function autofill(ctx) {
    const used = new Set(Object.values(ctx?.cards || {}).filter((c) => c.type === 'fridge').map((c) => c.props?.fridge));
    const all = fridges();
    const f = all.find((x) => !used.has(x.fridge)) || all[0];
    return f ? fill(f) : {};
  }
  function devices(props) {
    const all = fridges();
    if (all.length < 2) return null;
    const cur = all.find((x) => x.fridge === props.fridge);
    const label = (x) => deviceName(x.fridge) || x.p.replace(/_/g, ' ');
    return { current: cur ? label(cur) : '', others: all.filter((x) => x !== cur).map((x) => ({ id: x.fridge, name: label(x), props: fill(x) })) };
  }
  export const meta = {
    type: 'fridge', name: 'Refrigerator', icon: 'mdi:fridge-outline', category: 'Controls',
    size: { w: 440, h: 300 }, tap: 'none',
    defaults: { toggle_features: false },
    autofill,
    devices,
    sections: [{ key: 'doors', label: 'Doors' }, { key: 'zones', label: 'Zones' }, { key: 'features', label: 'Ice & features' }, { key: 'filter', label: 'Water filter' }, { key: 'energy', label: 'Power & energy' }],
    fields: [
      { key: 'name', label: 'Name', type: 'text' },
      { key: 'fridge', label: 'Fridge temperature', type: 'entity', domain: 'sensor' },
      { key: 'fridge_set', label: 'Fridge setpoint', type: 'entity', domain: ['number', 'climate'] },
      { key: 'freezer', label: 'Freezer temperature', type: 'entity', domain: 'sensor' },
      { key: 'freezer_set', label: 'Freezer setpoint', type: 'entity', domain: ['number', 'climate'] },
      { key: 'doors', label: 'Doors', type: 'entities', domain: 'binary_sensor', section: 'doors' },
      { key: 'zones', label: 'Convertible zones (mode)', type: 'entities', domain: ['select', 'sensor'], section: 'zones' },
      { key: 'features', label: 'Ice makers and modes (switches)', type: 'entities', domain: ['switch', 'binary_sensor', 'input_boolean'], section: 'features' },
      { key: 'ice_status', label: 'Ice making status', type: 'entities', domain: 'sensor', section: 'features' },
      { key: 'defrost', label: 'Defrosting', type: 'entity', domain: 'binary_sensor', section: 'features' },
      { key: 'toggle_features', label: 'Tapping a feature switches it on / off', type: 'bool', section: 'features' },
      { key: 'filter', label: 'Water filter used %', type: 'entity', domain: 'sensor', section: 'filter' },
      { key: 'filter_status', label: 'Water filter status', type: 'entity', domain: 'sensor', section: 'filter' },
      { key: 'alarm', label: 'Alarm code', type: 'entity', domain: 'sensor' },
      { key: 'power', label: 'Power', type: 'entity', domain: 'sensor', section: 'energy' },
      { key: 'energy', label: 'Energy this month', type: 'entity', domain: 'sensor', section: 'energy' },
      { key: 'energy_last', label: 'Energy last month', type: 'entity', domain: 'sensor', section: 'energy' },
    ],
  };
</script>

<script>
  // A fridge-freezer at a glance: both compartments against their setpoints
  // (coloured by food-safety ranges), doors (and how long one has been open),
  // the convertible zones' modes, ice makers and modes, the water filter and
  // what it's using. Tapping a temperature opens its setpoint.
  import Icon from '../components/Icon.svelte';
  import { t, ent } from '../lib/tpl.js';
  import { formatNumber, toggle } from '../lib/entity.js';
  import { clock } from '../lib/clock.svelte.js';
  import { app } from '../lib/config.svelte.js';
  let { props, w = 440, h = 300 } = $props();
  const show = (k) => !props.hide?.[k];
  const info = (e) => { if (e) app.popup = { entity: e.entity_id }; };
  const gone = (e) => !e || e.state === 'unavailable' || e.state === 'unknown';
  const list = (v) => (Array.isArray(v) ? v : v ? [v] : []).map((id) => ent(id)).filter(Boolean);
  const fridgeE = $derived(ent(props.fridge));
  const freezerE = $derived(ent(props.freezer));
  const name = $derived(t(props.name) || 'Fridge');
  // What's left of an entity's name once the device's is taken off: "… Rapid freezing" → "Rapid freezing".
  const prefix = $derived((fridgeE?.attributes.friendly_name || '').replace(/(fridge|cooler)\s+temperature$/i, ''));
  const short = (e) => { const n = e?.attributes.friendly_name || e?.entity_id || ''; return (prefix && n.startsWith(prefix) ? n.slice(prefix.length) : n).replace(/\s+(enabled|open)$/i, '').replace(/^./, (c) => c.toUpperCase()); };

  // Food safety: a fridge at or below 40 °F (4 °C), a freezer at or below 0 °F (-18 °C).
  const isF = (e) => /F/.test(e?.attributes.unit_of_measurement || '°F');
  function level(e, kind) {
    const v = Number(e?.state);
    if (gone(e) || isNaN(v)) return '';
    const [ok, warn] = kind === 'fridge' ? (isF(e) ? [40, 45] : [4.4, 7.2]) : (isF(e) ? [0, 10] : [-17.8, -12.2]);
    return v <= ok ? 'ok' : v <= warn ? 'warn' : 'bad';
  }
  const comps = $derived([
    { key: 'fridge', label: 'Fridge', icon: 'mdi:fridge-top', e: fridgeE, set: ent(props.fridge_set) },
    { key: 'freezer', label: 'Freezer', icon: 'mdi:snowflake', e: freezerE, set: ent(props.freezer_set) },
  ].filter((c) => c.e));
  const temp = (e) => (gone(e) ? '—' : `${formatNumber(e.state, 0)}${e.attributes.unit_of_measurement || ''}`);
  const setText = (s) => { if (gone(s)) return ''; const v = s.entity_id.startsWith('climate.') ? s.attributes.temperature : s.state; return v == null ? '' : `Set ${formatNumber(v, 0)}${s.attributes.unit_of_measurement || ''}`; };

  // Doors: which are open, and for how long (a door left open pulses after a minute).
  const doors = $derived(list(props.doors));
  const open = $derived(doors.filter((e) => e.state === 'on'));
  // last_changed is seconds since 1970 from the live feed (a date string elsewhere).
  const ms = (v) => (v == null ? NaN : typeof v === 'number' ? v * 1000 : Date.parse(v));
  const openedAt = $derived(open.map((e) => ms(e.last_changed)).filter((x) => !isNaN(x)));
  const openFor = $derived(openedAt.length ? Math.max(0, Math.floor((clock.now - Math.min(...openedAt)) / 60e3)) : 0);
  const doorText = $derived(!open.length ? 'Doors closed' : `${open.length === 1 ? short(open[0]).replace(/\s+door$/i, '') + ' door' : `${open.length} doors`} open${openFor ? ` · ${openFor} min` : ''}`);

  // Convertible zones: Samsung's raw modes ("cv_ttype_rf9000a_meat_fish") as words.
  const NICE = { softfreeze: 'Soft freeze', meat_fish: 'Meat & fish', fruit_veggies: 'Fruit & veggies', cool_drinks: 'Cool drinks' };
  const mode = (s) => { const k = String(s || '').replace(/^(cv_)?ttype_(rf\d+\w*?_)?/i, '').toLowerCase(); return NICE[k] || k.replace(/_/g, ' ').replace(/^./, (c) => c.toUpperCase()); };
  const zones = $derived(list(props.zones).map((e) => {
    const zn = short(e).replace(/\s+mode$/i, '');
    const door = doors.find((d) => short(d).toLowerCase().startsWith(zn.toLowerCase().replace(/\s+zone$/, '')));
    return { e, name: zn, mode: gone(e) ? '—' : mode(e.state), open: door?.state === 'on' };
  }));

  // Ice makers always show (on or off); other modes only while they're on.
  // Each ice maker's status pairs with it by name: "Cubed ice making status" ↔ "Cubed ice enabled".
  const making = $derived(new Set(list(props.ice_status).filter((e) => /run|making|on$/i.test(e.state)).map((e) => short(e).replace(/\s+making\s+status$/i, '').toLowerCase())));
  const feats = $derived(list(props.features).filter((e) => /ice/i.test(e.entity_id) || e.state === 'on'));
  const featNote = (e) => (!/ice/i.test(e.entity_id) ? '' : e.state !== 'on' ? ' off' : making.has(short(e).toLowerCase()) ? ' · making' : '');
  const defrostE = $derived(ent(props.defrost));
  const tap = (e) => (props.toggle_features && ['switch', 'input_boolean'].includes(e.entity_id.split('.')[0]) ? toggle(e) : info(e));

  const filterE = $derived(ent(props.filter));
  const filterStE = $derived(ent(props.filter_status));
  const filterUsed = $derived(Number(filterE?.state));
  const filterBad = $derived(/replace/i.test(filterStE?.state || '') || filterUsed >= 100);
  const filterWarn = $derived(!filterBad && (/wash|soon/i.test(filterStE?.state || '') || filterUsed >= 90));

  const alarmE = $derived(ent(props.alarm));
  const alarm = $derived(!gone(alarmE) && !/^(none|no_?alarm|normal|off|0)$/i.test(alarmE.state) ? alarmE.state.replace(/_/g, ' ').toLowerCase().replace(/^./, (c) => c.toUpperCase()) : '');
  const alarmText = $derived(/alarm/i.test(alarm) ? alarm : `Alarm: ${alarm}`);
  const powerE = $derived(ent(props.power));
  const energyE = $derived(ent(props.energy));
  const lastE = $derived(ent(props.energy_last));

  const compact = $derived(h < 190 || w < 300);
</script>

<div class="fr" class:compact>
  {#if !props.fridge}
    <div class="hint">Pick the fridge's temperature in the card's settings.</div>
  {:else}
    <div class="head">
      <Icon icon="mdi:fridge-outline" size="1.2em" />
      <span class="nm">{name}</span>
      {#if doors.length && show('doors')}<button class="chip" class:ok={!open.length} class:bad={open.length} class:pulse={openFor >= 1} data-stop onclick={() => info(open[0] || doors[0])}><Icon icon={open.length ? 'mdi:fridge-alert-outline' : 'mdi:check-circle-outline'} size="1em" /><span>{doorText}</span></button>{/if}
    </div>

    {#if alarm}<button class="banner" data-stop onclick={() => info(alarmE)}><Icon icon="mdi:alert" size="1em" />{alarmText}</button>{/if}

    <div class="comps" data-stop>
      {#each comps as c (c.key)}
        {@const lv = level(c.e, c.key)}
        <button class="comp" onclick={() => info(c.set || c.e)}>
          <span class="k"><Icon icon={c.icon} size="1em" />{c.label}</span>
          <span class="v {lv}">{temp(c.e)}</span>
          {#if c.set}<span class="set">{setText(c.set)}</span>{/if}
        </button>
      {/each}
    </div>

    {#if !compact && zones.length && show('zones')}
      <div class="zones" data-stop>
        {#each zones as z (z.e.entity_id)}
          <button class="zone" onclick={() => info(z.e)}><span class="zn">{z.name}</span><b>{z.mode}</b>{#if z.open}<i class="dot" title="Door open"></i>{/if}</button>
        {/each}
      </div>
    {/if}

    {#if !compact && ((feats.length || defrostE?.state === 'on') && show('features') || ((filterE || filterStE) && show('filter')))}
      <div class="chips" data-stop>
        {#if show('features')}
          {#each feats as e (e.entity_id)}
            {@const iceMaker = /ice/i.test(e.entity_id)}
            <button class="chip" class:on={e.state === 'on'} onclick={() => tap(e)}><Icon icon={iceMaker ? 'mdi:cube-outline' : /freez/i.test(e.entity_id) ? 'mdi:snowflake-alert' : /rapid|power/i.test(e.entity_id) ? 'mdi:fan-chevron-up' : 'mdi:toggle-switch-outline'} size="1em" /><span>{short(e)}{featNote(e)}</span></button>
          {/each}
          {#if defrostE?.state === 'on'}<button class="chip on" onclick={() => info(defrostE)}><Icon icon="mdi:snowflake-melt" size="1em" /><span>Defrosting</span></button>{/if}
        {/if}
        {#if (filterE || filterStE) && show('filter')}<button class="chip" class:warn={filterWarn} class:bad={filterBad} onclick={() => info(filterStE || filterE)}><Icon icon="mdi:water-check-outline" size="1em" /><span>Filter{filterBad ? ' · replace' : filterWarn ? ` · ${/wash/i.test(filterStE?.state || '') ? 'wash' : 'soon'}` : ''}{isNaN(filterUsed) ? '' : ` · ${Math.round(filterUsed)}% used`}</span></button>{/if}
      </div>
    {/if}

    {#if !compact && show('energy') && (powerE || energyE)}
      <div class="foot" data-stop>
        {#if powerE && !gone(powerE)}<button onclick={() => info(powerE)}><Icon icon="mdi:flash-outline" size="1em" />{formatNumber(powerE.state, 0)} {powerE.attributes.unit_of_measurement || 'W'}</button>{/if}
        {#if energyE && !gone(energyE)}<button onclick={() => info(energyE)}><Icon icon="mdi:lightning-bolt-outline" size="1em" />{formatNumber(energyE.state, 1)} {energyE.attributes.unit_of_measurement || 'kWh'} this month{#if lastE && !gone(lastE)}<span class="dim"> · {formatNumber(lastE.state, 1)} last month</span>{/if}</button>{/if}
      </div>
    {/if}
  {/if}
</div>

<style>
  .fr { height: 100%; display: flex; flex-direction: column; gap: 10px; min-height: 0; }
  .hint { color: var(--muted); font-size: .82em; margin: auto; }
  .head { display: flex; align-items: center; gap: 8px; font-weight: 600; min-width: 0; }
  .head :global(svg) { flex: none; }
  .nm { flex: 1; min-width: 0; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .chip { white-space: nowrap; flex: none; display: inline-flex; align-items: center; gap: 4px; font: inherit; font-size: .75em; font-weight: 500; padding: 4px 8px; border-radius: 8px; border: 0; background: rgba(255,255,255,.07); color: var(--muted); }
  .chip.on { color: #4fb4ff; background: rgba(79,180,255,.14); }
  .chip.ok { color: #5bd88f; background: rgba(91,216,143,.14); }
  .chip.warn { color: #ffc861; background: rgba(255,200,97,.14); }
  .chip.bad { color: #ff7a90; background: rgba(255,122,144,.15); }
  .chip.pulse { animation: pulse 1.4s infinite; }
  @keyframes pulse { 50% { opacity: .45; } }
  .banner { display: flex; align-items: center; gap: 6px; padding: 6px 10px; border: 0; border-radius: 10px; background: rgba(255,122,144,.15); color: #ff7a90; font: inherit; font-size: .8em; font-weight: 600; text-align: left; }
  .comps { flex: 1; min-height: 0; display: grid; grid-template-columns: repeat(auto-fit, minmax(0, 1fr)); gap: 8px; }
  .comp { display: flex; flex-direction: column; justify-content: center; align-items: flex-start; gap: 2px; min-width: 0; padding: 10px 12px; border: 0; border-radius: 14px; background: rgba(255,255,255,.05); color: inherit; font: inherit; text-align: left; }
  .k { display: flex; align-items: center; gap: 5px; font-size: .75em; color: var(--muted); }
  .v { font-size: 2em; font-weight: 600; line-height: 1.05; letter-spacing: -.02em; font-variant-numeric: tabular-nums; }
  .v.ok { color: #5bd88f; }
  .v.warn { color: #ffc861; }
  .v.bad { color: #ff7a90; }
  .set { font-size: .72em; color: var(--muted); }
  .zones { display: flex; gap: 8px; flex: none; }
  .zone { position: relative; flex: 1; min-width: 0; display: flex; flex-direction: column; align-items: flex-start; gap: 1px; padding: 6px 10px; border: 0; border-radius: 10px; background: rgba(255,255,255,.05); color: inherit; font: inherit; text-align: left; }
  .zn { font-size: .68em; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 100%; }
  .zone b { font-size: .82em; font-weight: 600; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 100%; }
  .dot { position: absolute; top: 8px; right: 8px; width: 7px; height: 7px; border-radius: 50%; background: #ff7a90; }
  .chips { display: flex; gap: 5px; flex-wrap: wrap; flex: none; }
  .foot { display: flex; align-items: center; gap: 14px; flex: none; font-size: .75em; color: var(--muted); min-width: 0; }
  .foot button { display: inline-flex; align-items: center; gap: 4px; padding: 0; border: 0; background: none; color: inherit; font: inherit; white-space: nowrap; }
  .dim { opacity: .7; }
  .compact { gap: 6px; }
  .compact .v { font-size: 1.6em; }
</style>
