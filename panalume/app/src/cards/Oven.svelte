<script module>
  import { states } from '../lib/ha.svelte.js';
  // Whirlpool / Maytag / KitchenAid ovens (the Whirlpool integration): each oven
  // has "<x>_state" (standby / preheating / cooking) beside "<x>_cook_mode" and
  // "<x>_current_temperature"; a double oven has an upper and a lower one.
  const has = (id) => (id && states.has(id) ? id : '');
  const isOvenState = (id) => /^sensor\..+_state$/.test(id) && (states.get(id)?.attributes.options || []).includes('preheating');
  const cavity = (b) => ({
    state: `sensor.${b}_state`, temp: has(`sensor.${b}_current_temperature`), target: has(`number.${b}_target_temperature`),
    mode: has(`select.${b}_cook_mode`), light: has(`light.${b}_light`), stop: has(`button.${b}_stop`),
  });
  // Ovens grouped by device: "oven_upper_oven" and "oven_lower_oven" are one oven ("oven").
  function ovens() {
    const by = new Map();
    for (const id of states.keys()) {
      if (!isOvenState(id)) continue;
      const b = id.slice(7, -6);
      const dev = b.replace(/_(upper|lower|top|bottom|main|second)(_oven)?$/, '');
      if (!by.has(dev)) by.set(dev, []);
      by.get(dev).push(b);
    }
    // Upper first.
    return [...by].map(([dev, bs]) => ({ dev, bs: bs.sort((a, c) => (/upper|top|main/.test(c) ? 1 : 0) - (/upper|top|main/.test(a) ? 1 : 0)) }));
  }
  function fill(o) {
    const [a, b] = o.bs.map(cavity);
    const out = { name: (states.get(a.state)?.attributes.friendly_name || '').split(' ')[0] || 'Oven' };
    for (const [k, v] of Object.entries(a)) out[`a_${k}`] = v;
    for (const [k, v] of Object.entries(b || {})) out[`b_${k}`] = v;
    return out;
  }
  function autofill(ctx) {
    const used = new Set(Object.values(ctx?.cards || {}).filter((c) => c.type === 'oven').map((c) => c.props?.a_state));
    const all = ovens();
    const o = all.find((x) => !used.has(fill(x).a_state)) || all[0];
    return o ? fill(o) : {};
  }
  function devices(props) {
    const all = ovens();
    if (all.length < 2) return null;
    const cur = all.find((x) => fill(x).a_state === props.a_state);
    return { current: cur ? cur.dev.replace(/_/g, ' ') : '', others: all.filter((x) => x !== cur).map((x) => ({ id: x.dev, name: x.dev.replace(/_/g, ' '), props: fill(x) })) };
  }
  const ovenFields = (p, label) => [
    { key: `${p}_state`, label: `${label}: state`, type: 'entity', domain: 'sensor' },
    { key: `${p}_temp`, label: `${label}: temperature`, type: 'entity', domain: 'sensor' },
    { key: `${p}_target`, label: `${label}: target temperature`, type: 'entity', domain: ['number', 'sensor'] },
    { key: `${p}_mode`, label: `${label}: cook mode`, type: 'entity', domain: ['select', 'sensor'] },
    { key: `${p}_light`, label: `${label}: light`, type: 'entity', domain: ['light', 'switch'] },
    { key: `${p}_stop`, label: `${label}: stop button`, type: 'entity', domain: 'button' },
  ];
  export const meta = {
    type: 'oven', name: 'Oven', icon: 'mdi:stove', category: 'Controls',
    size: { w: 460, h: 260 }, tap: 'none',
    defaults: {},
    autofill,
    devices,
    sections: [{ key: 'progress', label: 'Preheat progress' }, { key: 'light', label: 'Light buttons' }, { key: 'stop', label: 'Stop buttons' }],
    fields: [
      { key: 'name', label: 'Name', type: 'text' },
      { key: 'a_name', label: 'First oven’s name (blank: from the entity, e.g. Upper oven)', type: 'text' },
      ...ovenFields('a', 'First oven'),
      { key: 'b_name', label: 'Second oven’s name (blank: from the entity)', type: 'text' },
      ...ovenFields('b', 'Second oven (double ovens)'),
    ],
  };
</script>

<script>
  // An oven, or both halves of a double oven, at a glance: off, preheating
  // (with how far along) or cooking (and for how long), the light, and a stop
  // button that needs a second tap. It watches and stops; it doesn't start.
  import Icon from '../components/Icon.svelte';
  import { t, ent } from '../lib/tpl.js';
  import { formatNumber, toggle } from '../lib/entity.js';
  import { callService } from '../lib/ha.svelte.js';
  import { clock } from '../lib/clock.svelte.js';
  import { app } from '../lib/config.svelte.js';
  let { props, w = 460, h = 260 } = $props();
  const show = (k) => !props.hide?.[k];
  const info = (e) => { if (e) app.popup = { entity: e.entity_id }; };
  const gone = (e) => !e || e.state === 'unavailable' || e.state === 'unknown';
  // last_changed is seconds since 1970 from the live feed (a date string elsewhere).
  const ms = (v) => (v == null ? NaN : typeof v === 'number' ? v * 1000 : Date.parse(v));
  const MODES = { bake: 'Bake', convection_bake: 'Convection bake', broil: 'Broil', convection_broil: 'Convection broil', convection_roast: 'Convection roast', keep_warm: 'Keep warm', air_fry: 'Air fry', proof: 'Proof', dehydrate: 'Dehydrate', self_clean: 'Self clean' };
  const modeName = (s) => MODES[s] || String(s || '').replace(/_/g, ' ').replace(/^./, (c) => c.toUpperCase());
  const modeIcon = (s) => (/broil/.test(s) ? 'mdi:grill' : /air_fry/.test(s) ? 'mdi:fan' : /convection/.test(s) ? 'mdi:fan' : /warm/.test(s) ? 'mdi:thermometer-low' : /clean/.test(s) ? 'mdi:auto-fix' : 'mdi:stove');
  const num = (e) => (gone(e) ? NaN : Number(e.state));

  function oven(p) {
    const st = ent(props[`${p}_state`]);
    if (!st) return null;
    const mode = ent(props[`${p}_mode`]);
    const tempE = ent(props[`${p}_temp`]), targetE = ent(props[`${p}_target`]);
    const raw = gone(st) ? '' : st.state;
    const modeKey = gone(mode) ? '' : mode.state;
    const on = (raw && raw !== 'standby' && raw !== 'off') || (modeKey && modeKey !== 'standby');
    const phase = !on ? 'off' : /preheat/.test(raw) ? 'preheat' : 'cook';
    const unit = tempE?.attributes.unit_of_measurement || targetE?.attributes.unit_of_measurement || '°F';
    const cur = num(tempE), tgt = num(targetE);
    // Preheat progress from room temperature to the target.
    const room = /C/.test(unit) ? 22 : 72;
    const pct = phase === 'preheat' && !isNaN(cur) && tgt > room ? Math.max(0, Math.min(99, Math.round(((cur - room) / (tgt - room)) * 100))) : NaN;
    // On since the cook mode was set (else since the state left standby); ready = cooking began within a minute.
    const since = ms((mode && !gone(mode) ? mode : st).last_changed);
    const mins = on && !isNaN(since) ? Math.max(0, Math.floor((clock.now - since) / 60e3)) : NaN;
    const ready = phase === 'cook' && clock.now - ms(st.last_changed) < 60e3;
    const label = t(props[`${p}_name`]) || (st.attributes.friendly_name || '').replace(/^\S+\s+/, '').replace(/\s+state$/i, '') || 'Oven';
    return { p, st, label, on, phase, mode: modeKey, cur, tgt, unit, pct, mins, ready, light: ent(props[`${p}_light`]), stop: ent(props[`${p}_stop`]), tempE };
  }
  const list = $derived(['a', 'b'].map(oven).filter(Boolean));
  const name = $derived(t(props.name) || 'Oven');
  const onOnes = $derived(list.filter((o) => o.on));
  const status = $derived.by(() => {
    if (!list.length) return { text: '', cls: '' };
    const r = onOnes.find((o) => o.ready);
    if (r) return { text: `${list.length > 1 ? r.label : 'Oven'} ready!`, cls: 'ready' };
    if (!onOnes.length) return { text: 'Off', cls: '' };
    if (onOnes.length > 1) return { text: 'Both ovens on', cls: 'hot' };
    const o = onOnes[0];
    return { text: `${list.length > 1 ? o.label + ' ' : ''}${o.phase === 'preheat' ? 'preheating' : 'cooking'}`.replace(/^./, (c) => c.toUpperCase()), cls: o.phase === 'preheat' ? 'warm' : 'hot' };
  });

  // Stop needs a second tap within 4 seconds.
  let armed = $state('');
  let disarm;
  function stop(o) {
    if (armed !== o.p) { armed = o.p; clearTimeout(disarm); disarm = setTimeout(() => (armed = ''), 4000); return; }
    armed = '';
    clearTimeout(disarm);
    callService('button', 'press', {}, { entity_id: o.stop.entity_id });
  }
  const deg = (v, u) => (isNaN(v) ? '—' : `${formatNumber(v, 0)}${u}`);
  const dur = (m) => (m < 60 ? `${m} min` : `${Math.floor(m / 60)} h ${String(m % 60).padStart(2, '0')} min`);
  const narrow = $derived(w < 340 && list.length > 1);
</script>

<div class="ov" class:narrow>
  {#if !props.a_state}
    <div class="hint">Pick the oven in the card's settings.</div>
  {:else}
    <div class="head">
      <Icon icon="mdi:stove" size="1.2em" />
      <span class="nm">{name}</span>
      {#if status.text}<span class="chip {status.cls}" class:pulse={status.cls === 'ready'}>{status.text}</span>{/if}
    </div>
    <div class="cavs" data-stop>
      {#each list as o (o.p)}
        <div class="cav {o.phase}">
          <button class="top" onclick={() => info(o.on ? o.tempE || o.st : o.st)}>
            <span class="k">{#if list.length > 1}{o.label}{:else}{o.on ? modeName(o.mode) : 'Oven'}{/if}</span>
            {#if o.on}
              {#if list.length > 1}<span class="md"><Icon icon={modeIcon(o.mode)} size="1em" />{modeName(o.mode)}</span>{/if}
              <span class="v">{deg(o.cur, o.unit)}</span>
              <span class="sub">{o.phase === 'preheat' ? `Preheating to ${deg(o.tgt, o.unit)}` : `Cooking at ${deg(o.tgt, o.unit)}`}{isNaN(o.mins) ? '' : ` · ${dur(o.mins)}`}</span>
              {#if o.phase === 'preheat' && show('progress') && !isNaN(o.pct)}<span class="bar"><i style="width:{o.pct}%"></i></span>{/if}
            {:else}
              <span class="v off">Off</span>
            {/if}
          </button>
          {#if (o.light && show('light')) || (o.on && o.stop && show('stop'))}
            <div class="acts">
              {#if o.light && show('light')}<button class="act" class:lit={o.light.state === 'on'} onclick={() => toggle(o.light)} aria-label="Oven light"><Icon icon={o.light.state === 'on' ? 'mdi:lightbulb-on' : 'mdi:lightbulb-outline'} size="1.1em" /></button>{/if}
              {#if o.on && o.stop && show('stop')}<button class="act stop" class:armed={armed === o.p} onclick={() => stop(o)}><Icon icon="mdi:stop" size="1.1em" /><span>{armed === o.p ? 'Tap again to stop' : 'Stop'}</span></button>{/if}
            </div>
          {/if}
        </div>
      {/each}
    </div>
  {/if}
</div>

<style>
  .ov { height: 100%; display: flex; flex-direction: column; gap: 10px; min-height: 0; }
  .hint { color: var(--muted); font-size: .82em; margin: auto; }
  .head { display: flex; align-items: center; gap: 8px; font-weight: 600; min-width: 0; }
  .head :global(svg) { flex: none; }
  .nm { flex: 1; min-width: 0; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .chip { white-space: nowrap; flex: none; font-size: .75em; font-weight: 500; padding: 4px 8px; border-radius: 8px; background: rgba(255,255,255,.07); color: var(--muted); }
  .chip.warm { color: #ffc861; background: rgba(255,200,97,.14); }
  .chip.hot { color: #ff9a57; background: rgba(255,154,87,.15); }
  .chip.ready { color: #5bd88f; background: rgba(91,216,143,.15); }
  .pulse { animation: pulse 1.4s infinite; }
  @keyframes pulse { 50% { opacity: .45; } }
  .cavs { flex: 1; min-height: 0; display: grid; grid-template-columns: repeat(auto-fit, minmax(0, 1fr)); gap: 8px; }
  .narrow .cavs { grid-template-columns: 1fr; }
  .cav { display: flex; flex-direction: column; min-width: 0; min-height: 0; border-radius: 14px; background: rgba(255,255,255,.05); overflow: hidden; }
  .cav.preheat { background: linear-gradient(180deg, rgba(255,200,97,.12), rgba(255,255,255,.04)); }
  .cav.cook { background: linear-gradient(180deg, rgba(255,154,87,.16), rgba(255,255,255,.04)); }
  .top { flex: 1; min-height: 0; display: flex; flex-direction: column; justify-content: center; align-items: flex-start; gap: 3px; padding: 10px 12px; border: 0; background: none; color: inherit; font: inherit; text-align: left; }
  .k { font-size: .75em; color: var(--muted); }
  .md { display: inline-flex; align-items: center; gap: 4px; font-size: .78em; font-weight: 600; }
  .v { font-size: 2em; font-weight: 600; line-height: 1.05; letter-spacing: -.02em; font-variant-numeric: tabular-nums; }
  .preheat .v { color: #ffc861; }
  .cook .v { color: #ff9a57; }
  .v.off { color: var(--muted); font-size: 1.5em; }
  .sub { font-size: .72em; color: var(--muted); }
  .bar { width: 100%; height: 5px; margin-top: 4px; border-radius: 3px; background: rgba(255,255,255,.08); overflow: hidden; }
  .bar i { display: block; height: 100%; border-radius: 3px; background: #ffc861; transition: width .6s; }
  .acts { display: flex; gap: 6px; padding: 0 10px 10px; }
  .act { display: inline-flex; align-items: center; justify-content: center; gap: 5px; min-width: 36px; height: 34px; padding: 0 10px; border: 0; border-radius: 10px; background: rgba(255,255,255,.07); color: var(--muted); font: inherit; font-size: .78em; font-weight: 600; }
  .act.lit { color: #ffc861; background: rgba(255,200,97,.14); }
  .act.stop { flex: 1; color: #ff7a90; background: rgba(255,122,144,.12); }
  .act.stop.armed { color: #fff; background: #e5546c; }
</style>
