<script module>
  import { states } from '../lib/ha.svelte.js';
  // Fills in a car that isn't on another fuel vehicle card yet: Ford (FordConnect
  // Query / FordPass: …_fuel next to …_odometer) and other integrations with a
  // …_fuel_level or …_fuel sensor beside an odometer or mileage sensor.
  function cars() {
    const out = [];
    for (const id of states.keys()) {
      const m = /^sensor\.([a-z0-9_]+?)_fuel(?:_level)?$/.exec(id);
      if (m && ['odometer', 'mileage', 'oil', 'oil_life'].some((k) => states.has(`sensor.${m[1]}_${k}`))) out.push({ p: m[1], fuel: id });
    }
    return out;
  }
  function fill(car) {
    const has = (...ids) => ids.find((id) => states.has(id)) || '';
    const p = car.p;
    const out = {
      fuel: car.fuel, range: has(`sensor.${p}_fuel_range`, `sensor.${p}_range`),
      lock: has(`lock.${p}_doorlock`, `lock.${p}_doors`, `lock.${p}_lock`, `sensor.${p}_doorlock`, `binary_sensor.${p}_doors_lock`),
      doors: has(`sensor.${p}_doorstatus`, `binary_sensor.${p}_doors`, `binary_sensor.${p}_door`), windows: has(`sensor.${p}_windowposition`, `binary_sensor.${p}_windows`),
      oil: has(`sensor.${p}_oil`, `sensor.${p}_oil_life`), battery: has(`sensor.${p}_battery`, `sensor.${p}_battery_12v`), tires: has(`sensor.${p}_tirepressure`, `sensor.${p}_tire_pressure`),
      ignition: has(`sensor.${p}_ignitionstatus`, `binary_sensor.${p}_ignition`, `binary_sensor.${p}_engine`), remote: has(`switch.${p}_ignition`, `sensor.${p}_remotestartstatus`),
      alarm: has(`sensor.${p}_alarm`, `alarm_control_panel.${p}_alarm`), mileage: has(`sensor.${p}_odometer`, `sensor.${p}_mileage`), location: has(`device_tracker.${p}_tracker`, `device_tracker.${p}`),
    };
    // Ford names its sensors just "Fuel", "Odometer"…, so there may be no shared car name: leave it blank.
    return { name: nameOf(Object.values(out).filter((id) => id.includes('.'))), ...out };
  }
  // For the Device box when the car has no name: "fcq_1ab…xyz4827" → "fcq …4827".
  const label = (car) => fill(car).name || car.p.split('_').map((w) => (w.length >= 12 ? `…${w.slice(-4)}` : w)).join(' ');
  function autofill(ctx) {
    const used = new Set(Object.values(ctx?.cards || {}).filter((c) => c.type === 'fuel_vehicle').map((c) => c.props?.fuel));
    const all = cars();
    const car = all.find((c) => !used.has(c.fuel)) || all[0];
    return car ? fill(car) : {};
  }
  function devices(props) {
    const all = cars();
    if (all.length < 2) return null;
    const cur = all.find((c) => c.fuel === props.fuel);
    return { current: cur ? label(cur) : '', others: all.filter((c) => c !== cur).map((c) => ({ id: c.fuel, name: label(c), props: fill(c) })) };
  }
  import { nameOf } from '../lib/device.js';
  export const meta = {
    type: 'fuel_vehicle', name: 'Ford/Lincoln Gas Vehicle', icon: 'mdi:car', category: 'Energy',
    size: { w: 420, h: 240 }, tap: 'none',
    defaults: { tire_unit: '' },
    autofill,
    devices,
    sections: [{ key: 'ring', label: 'Fuel ring' }, { key: 'range', label: 'Range' }, { key: 'status', label: 'Engine status' }, { key: 'lock', label: 'Lock' }, { key: 'doors', label: 'Doors' }, { key: 'windows', label: 'Windows' }, { key: 'alarm', label: 'Alarm' }, { key: 'oil', label: 'Oil life' }, { key: 'battery', label: '12V battery' }, { key: 'tires', label: 'Tyre pressures' }, { key: 'mileage', label: 'Mileage' }, { key: 'location', label: 'Location' }, { key: 'image', label: 'Car image' }],
    fields: [
      { key: 'name', label: 'Name', type: 'text' },
      { key: 'fuel', label: 'Fuel level %', type: 'entity', domain: 'sensor' },
      { key: 'range', label: 'Range (blank: the fuel sensor’s range, if it has one)', type: 'entity', domain: 'sensor', section: 'range' },
      { key: 'ignition', label: 'Ignition / engine', type: 'entity', domain: ['sensor', 'binary_sensor'], section: 'status' },
      { key: 'remote', label: 'Remote start (status sensor, switch or button)', type: 'entity', domain: ['sensor', 'binary_sensor', 'switch', 'button'], section: 'status' },
      { key: 'lock', label: 'Doors lock', type: 'entity', domain: ['lock', 'binary_sensor', 'sensor'], section: 'lock' },
      { key: 'doors', label: 'Doors', type: 'entity', domain: ['binary_sensor', 'sensor'], section: 'doors' },
      { key: 'windows', label: 'Windows', type: 'entity', domain: ['binary_sensor', 'sensor'], section: 'windows' },
      { key: 'alarm', label: 'Alarm', type: 'entity', domain: ['sensor', 'alarm_control_panel', 'binary_sensor'], section: 'alarm' },
      { key: 'oil', label: 'Oil life %', type: 'entity', domain: 'sensor', section: 'oil' },
      { key: 'battery', label: '12V battery', type: 'entity', domain: 'sensor', section: 'battery' },
      { key: 'tires', label: 'Tyre pressures', type: 'entity', domain: ['sensor', 'binary_sensor'], section: 'tires' },
      { key: 'tire_unit', label: 'Tyre pressure unit', type: 'select', options: [{ value: '', label: 'As reported' }, { value: 'psi', label: 'psi' }, { value: 'bar', label: 'bar' }, { value: 'kpa', label: 'kPa' }], section: 'tires' },
      { key: 'mileage', label: 'Mileage', type: 'entity', domain: 'sensor', section: 'mileage' },
      { key: 'location', label: 'Location (device tracker)', type: 'entity', domain: ['device_tracker', 'person'], section: 'location' },
      { key: 'image', label: 'Car image URL', type: 'text', section: 'image' },
    ],
  };
</script>

<script>
  // A petrol or diesel car at a glance: fuel and range, whether the engine is
  // running, locks, doors and windows, and its upkeep (oil life, 12V battery,
  // tyre pressures). Each part turns amber or red when it needs attention.
  import Icon from '../components/Icon.svelte';
  import { t, ent } from '../lib/tpl.js';
  import { callService, haImage } from '../lib/ha.svelte.js';
  import { formatNumber, toggle } from '../lib/entity.js';
  import { app } from '../lib/config.svelte.js';
  let { props, w = 420, h = 240 } = $props();
  const show = (k) => !props.hide?.[k];
  const info = (e) => { if (e) app.popup = { entity: e.entity_id }; };
  const dom = (e) => e?.entity_id.split('.')[0];
  const gone = (e) => !e || e.state === 'unavailable' || e.state === 'unknown';
  // Ford and others report "OPEN", "Open", "UNLOCKED"…; binary sensors on/off.
  const isOn = (e, re) => !gone(e) && (dom(e) === 'binary_sensor' ? e.state === 'on' : re.test(e.state));

  const fuelE = $derived(ent(props.fuel));
  const fuel = $derived(Number(fuelE?.state));
  const fuelColor = $derived(fuel < 20 ? '#ff7a90' : fuel < 50 ? '#ffc861' : '#5bd88f');
  const mileE = $derived(ent(props.mileage));
  // Range from its own sensor, else the fuel sensor's range attribute (Ford: fuelRange).
  const rangeE = $derived(ent(props.range));
  const range = $derived.by(() => {
    if (rangeE && !gone(rangeE)) return { v: rangeE.state, u: rangeE.attributes.unit_of_measurement || '' };
    const a = fuelE?.attributes;
    const v = a?.fuelRange ?? a?.fuel_range ?? a?.range;
    return v != null && !isNaN(Number(v)) ? { v, u: a?.rangeUnit || mileE?.attributes.unit_of_measurement || '' } : null;
  });

  // Engine: running from the ignition, or a remote start counting down.
  const ignE = $derived(ent(props.ignition));
  const running = $derived(isOn(ignE, /^(on|run|running|start)$/i));
  const remoteE = $derived(ent(props.remote));
  const remoteCtl = $derived(['switch', 'button'].includes(dom(remoteE)));
  const remoteOn = $derived(!!remoteE && (dom(remoteE) === 'button' ? false : isOn(remoteE, /^(on|active|running)$/i)));
  // Ford's countdown is in seconds.
  const remoteLeft = $derived(Math.ceil((Number(remoteE?.attributes.countdown) || 0) / 60));
  const status = $derived.by(() => {
    if (remoteOn) return `Remote start${remoteLeft ? ` · ${remoteLeft} min left` : ''}`;
    if (ignE && !gone(ignE)) return running ? 'Engine running' : 'Parked · engine off';
    return '';
  });
  function remote() {
    if (dom(remoteE) === 'button') callService('button', 'press', {}, { entity_id: remoteE.entity_id });
    else if (dom(remoteE) === 'switch') toggle(remoteE);
    else info(remoteE);
  }

  // Lock, doors, windows and alarm chips.
  const lockE = $derived(ent(props.lock));
  const unlocked = $derived(!gone(lockE) && (dom(lockE) === 'lock' ? lockE.state !== 'locked' : dom(lockE) === 'binary_sensor' ? lockE.state === 'on' : /unlock/i.test(lockE.state)));
  const human = (k) => k.replace(/([a-z])([A-Z])/g, '$1 $2').replace(/_/g, ' ').toLowerCase().replace(/^./, (c) => c.toUpperCase());
  const doorsE = $derived(ent(props.doors));
  const doorsOpen = $derived(isOn(doorsE, /open|ajar/i));
  const openParts = $derived(doorsE ? Object.entries(doorsE.attributes).filter(([, v]) => typeof v === 'string' && /^(open|ajar)$/i.test(v)).map(([k]) => human(k)) : []);
  const doorsText = $derived(!doorsOpen ? 'Doors closed' : openParts.length === 1 ? `${openParts[0]} open` : openParts.length ? `${openParts.length} doors open` : 'Door open');
  const winE = $derived(ent(props.windows));
  const winOpen = $derived(isOn(winE, /open|vent/i));
  const alarmE = $derived(ent(props.alarm));
  const alarmOn = $derived(!gone(alarmE) && !/^(disarmed|off|undefined|none)$/i.test(alarmE.state));
  const alarmBad = $derived(alarmOn && /trigger|alarm|panic|on$/i.test(alarmE.state) && !/armed/i.test(alarmE.state));

  // Upkeep: oil life, 12V battery and tyres.
  const oilE = $derived(ent(props.oil));
  const oil = $derived(Number(oilE?.state));
  const oilColor = $derived(oil < 15 ? '#ff7a90' : oil < 30 ? '#ffc861' : '#5bd88f');
  const batE = $derived(ent(props.battery));
  const bat = $derived(Number(batE?.state));
  const volts = $derived(Number(batE?.attributes.batteryVoltage ?? batE?.attributes.voltage));
  const batBad = $derived(bat < 40 || (volts > 0 && volts < 11.9));
  // Above ~13.2 V with the engine running, the alternator is charging the battery,
  // so the voltage says nothing about its charge (just after parking it's still
  // settling down from there); a rested battery below ~12.2 V is getting low.
  const charging = $derived(volts >= 13.2 && (running || remoteOn || !ignE || gone(ignE)));
  const voltsLow = $derived(volts > 0 && volts < 12.2 && !running && !remoteOn);
  const tireE = $derived(ent(props.tires));
  const TO = { psi: 14.5038, bar: 1, kpa: 100 };
  function pressure(v) {
    const m = /^\s*([\d.]+)\s*([a-z]*)/i.exec(String(v ?? ''));
    if (!m) return '';
    let n = Number(m[1]), u = (m[2] || tireE?.attributes.unit_of_measurement || '').toLowerCase();
    const want = props.tire_unit;
    if (want && TO[want] && TO[u] && want !== u) { n = (n / TO[u]) * TO[want]; u = want; }
    return formatNumber(n, u === 'bar' ? 1 : 0);
  }
  const tireUnit = $derived(props.tire_unit ? (props.tire_unit === 'kpa' ? 'kPa' : props.tire_unit) : (/[a-z]+$/i.exec(String(tireE?.attributes.frontLeft ?? '').trim())?.[0] || tireE?.attributes.unit_of_measurement || ''));
  const WHEELS = [['frontLeft', 'FL'], ['frontRight', 'FR'], ['rearLeft', 'RL'], ['rearRight', 'RR']];
  const tires = $derived(tireE && tireE.attributes.frontLeft != null ? WHEELS.map(([k, l]) => ({ l, v: pressure(tireE.attributes[k]), bad: !!tireE.attributes[`${k}_state`] && !/normal/i.test(tireE.attributes[`${k}_state`]) })) : null);
  const tiresBad = $derived(!!tireE && !gone(tireE) && (tires ? tires.some((x) => x.bad) : dom(tireE) === 'binary_sensor' ? tireE.state === 'on' : !/normal|ok/i.test(tireE.state)));

  const locE = $derived(ent(props.location));
  const where = $derived(gone(locE) ? '' : locE.state === 'home' ? 'Home' : locE.state === 'not_home' ? 'Away' : human(locE.state));

  // What fits: short cards drop the upkeep row, then the bottom line.
  const short = $derived(h < 150 || w < 320);
  const showUpkeep = $derived(h >= 205 && ((oilE && show('oil')) || (batE && show('battery')) || (tireE && show('tires'))));
  const showFoot = $derived(h >= 165 || (!showUpkeep && h >= 140));
  const name = $derived(t(props.name) || 'Car');
  const R = 42, C = 2 * Math.PI * R;
  let roomW = $state(0), fullW = $state(0);
  const twoRows = $derived(!short && h >= 200);
</script>

<div class="fv" class:short>
  <div class="top">
    {#if show('ring')}<button class="ring" data-stop onclick={() => info(fuelE)} aria-label="Fuel {isNaN(fuel) ? '' : Math.round(fuel) + '%'}">
      <svg viewBox="0 0 100 100">
        <circle cx="50" cy="50" r={R} fill="none" stroke="rgba(255,255,255,.08)" stroke-width="8" />
        <circle cx="50" cy="50" r={R} fill="none" style="stroke:{fuelColor};transition:stroke-dasharray .6s" stroke-width="8" stroke-linecap="round" stroke-dasharray="{isNaN(fuel) ? 0 : (C * fuel) / 100} {C}" transform="rotate(-90 50 50)" />
      </svg>
      <div class="pct"><Icon icon="mdi:gas-station" size=".55em" style="color:{fuelColor}" /><span>{isNaN(fuel) ? '—' : Math.round(fuel) + '%'}</span></div>
    </button>{/if}
    <div class="info">
      <div class="name">{name}</div>
      {#if range && show('range')}<div class="range">{formatNumber(range.v, 0)} <span>{range.u}</span></div>{/if}
      {#if status && show('status')}<div class="dim" class:run={running || remoteOn}>{#if running || remoteOn}<Icon icon="mdi:engine" size="1em" />{/if}{status}</div>{/if}
      <div class="chipbox" bind:clientWidth={roomW}>
        {#snippet chipset()}
          {#if lockE && show('lock')}<button class="chip" class:bad={unlocked} class:ok={!unlocked && !gone(lockE)} onclick={() => info(lockE)}><Icon icon={unlocked ? 'mdi:lock-open-variant' : 'mdi:lock'} size="1em" /><span class="lbl">{gone(lockE) ? 'Lock ?' : unlocked ? 'Unlocked' : 'Locked'}</span></button>{/if}
          {#if doorsE && show('doors')}<button class="chip" class:bad={doorsOpen} class:ok={!doorsOpen && !gone(doorsE)} onclick={() => info(doorsE)} title={doorsText}><Icon icon={doorsOpen ? 'mdi:car-door' : 'mdi:car-door-lock'} size="1em" /><span class="lbl">{doorsText}</span></button>{/if}
          {#if winE && show('windows') && winOpen}<button class="chip warn" onclick={() => info(winE)}><Icon icon="mdi:car-windshield-outline" size="1em" /><span class="lbl">Window open</span></button>{/if}
          {#if alarmE && show('alarm') && alarmOn}<button class="chip" class:bad={alarmBad} class:ok={!alarmBad} onclick={() => info(alarmE)}><Icon icon={alarmBad ? 'mdi:alarm-light' : 'mdi:shield-car'} size="1em" /><span class="lbl">{alarmBad ? 'Alarm!' : 'Armed'}</span></button>{/if}
          {#if remoteE && remoteCtl}<button class="chip" class:ok={remoteOn} onclick={remote} title={remoteOn ? 'Remote start on' : 'Remote start'}><Icon icon="mdi:remote" size="1em" /><span class="lbl">{remoteOn ? 'Running' : 'Start'}</span></button>{/if}
        {/snippet}
        <!-- Labels matter most when something's wrong: use a second row before dropping them. -->
        <div class="chips" class:two={twoRows} class:bare={fullW > roomW * (twoRows ? 2 : 1) - (twoRows ? 30 : -1)} data-stop>{@render chipset()}</div>
        <div class="chips measure" aria-hidden="true" inert bind:clientWidth={fullW}>{@render chipset()}</div>
      </div>
    </div>
    {#if props.image && show('image')}<img class="car" src={haImage(t(props.image))} alt="" />{/if}
  </div>

  {#if showUpkeep}
    <div class="upkeep" data-stop>
      {#if oilE && show('oil')}<button class="tile" onclick={() => info(oilE)}>
        <span class="k"><Icon icon="mdi:oil" size="1em" />Oil life</span>
        <span class="v" style="color:{isNaN(oil) ? 'inherit' : oilColor}">{isNaN(oil) ? '—' : Math.round(oil) + '%'}</span>
        <span class="bar"><i style="width:{isNaN(oil) ? 0 : Math.max(0, Math.min(100, oil))}%;background:{oilColor}"></i></span>
      </button>{/if}
      {#if batE && show('battery')}<button class="tile" onclick={() => info(batE)}>
        <span class="k"><Icon icon="mdi:car-battery" size="1em" />12V battery</span>
        <span class="v" class:badv={batBad}>{isNaN(bat) ? '—' : Math.round(bat) + '%'}</span>
        <span class="sub" class:chg={charging} class:low={voltsLow} title={charging ? 'The alternator is charging the battery' : voltsLow ? 'Resting voltage is low' : 'Resting voltage'}>{volts > 0 ? `${charging ? 'Charging · ' : ''}${formatNumber(volts, 1)} V` : ''}</span>
      </button>{/if}
      {#if tireE && show('tires')}<button class="tile" onclick={() => info(tireE)}>
        <span class="k"><Icon icon="mdi:car-tire-alert" size="1em" />Tyres{tireUnit ? ` · ${tireUnit}` : ''}</span>
        {#if tires}
          <span class="tires">{#each tires as x}<b class:badv={x.bad} title={x.l}>{x.v || '—'}</b>{/each}</span>
        {:else}<span class="v" class:badv={tiresBad}>{gone(tireE) ? '—' : tiresBad ? 'Check' : 'OK'}</span>{/if}
      </button>{/if}
    </div>
  {/if}

  {#if showFoot && ((mileE && show('mileage')) || (where && show('location')))}
    <div class="foot">
      {#if mileE && show('mileage') && !gone(mileE)}<span><Icon icon="mdi:counter" size="1em" />{formatNumber(mileE.state, 0)} {mileE.attributes.unit_of_measurement || ''}</span>{/if}
      {#if where && show('location')}<button data-stop onclick={() => info(locE)}><Icon icon={where === 'Home' ? 'mdi:home' : 'mdi:map-marker'} size="1em" />{where}</button>{/if}
    </div>
  {/if}
</div>

<style>
  .fv { height: 100%; display: flex; flex-direction: column; gap: 10px; position: relative; }
  .top { flex: 1; min-height: 0; display: flex; align-items: center; gap: 16px; position: relative; }
  .ring { position: relative; height: 100%; max-height: 130px; aspect-ratio: 1; flex: none; container-type: inline-size; padding: 0; border: 0; background: none; color: inherit; font: inherit; }
  .ring svg { width: 100%; height: 100%; }
  .pct { position: absolute; inset: 0; display: flex; flex-direction: column; align-items: center; justify-content: center; font-size: min(1.5em, 21cqi); font-weight: 600; letter-spacing: -.02em; line-height: 1.1; }
  .info { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 4px; z-index: 1; }
  .name { font-weight: 600; font-size: 1.1em; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .range { font-size: 1.6em; font-weight: 600; line-height: 1; }
  .range span { font-size: .5em; color: var(--muted); }
  .dim { display: flex; align-items: center; gap: 5px; color: var(--muted); font-size: .85em; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .dim.run { color: #5bd88f; }
  .chipbox { position: relative; margin-top: 4px; margin-left: -8px; }
  .chips { display: flex; gap: 5px; flex-wrap: wrap; max-height: calc(.9em + 10px); overflow: hidden; }
  .chips.two { max-height: calc(1.8em + 25px); }
  .bare .lbl { display: none; }
  .measure { position: absolute; top: 0; left: 0; flex-wrap: nowrap; width: max-content; visibility: hidden; pointer-events: none; }
  .chip { white-space: nowrap; flex: none; display: inline-flex; align-items: center; gap: 4px; font-size: .75em; padding: 4px 8px; border-radius: 8px; background: rgba(255,255,255,.07); color: var(--muted); border: 0; font-family: inherit; }
  .chip.ok { color: #5bd88f; background: rgba(91,216,143,.14); }
  .chip.warn { color: #ffc861; background: rgba(255,200,97,.14); }
  .chip.bad { color: #ff7a90; background: rgba(255,122,144,.15); }
  .upkeep { display: grid; grid-template-columns: repeat(auto-fit, minmax(0, 1fr)); gap: 8px; flex: none; }
  .tile { display: flex; flex-direction: column; align-items: flex-start; gap: 3px; min-width: 0; padding: 8px 10px; border: 0; border-radius: 12px; background: rgba(255,255,255,.05); color: inherit; font: inherit; text-align: left; }
  .k { display: flex; align-items: center; gap: 5px; font-size: .72em; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 100%; }
  .v { font-size: 1.15em; font-weight: 600; line-height: 1.1; }
  .sub { font-size: .72em; color: var(--muted); min-height: 1em; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; max-width: 100%; }
  .sub.chg { color: #5bd88f; }
  .sub.low { color: #ffc861; }
  .badv { color: #ff7a90; }
  .bar { width: 100%; height: 4px; border-radius: 2px; background: rgba(255,255,255,.08); overflow: hidden; }
  .bar i { display: block; height: 100%; border-radius: 2px; }
  .tires { display: grid; grid-template-columns: 1fr 1fr; gap: 1px 10px; font-size: .8em; font-variant-numeric: tabular-nums; }
  .tires b { font-weight: 600; }
  .foot { display: flex; align-items: center; gap: 14px; font-size: .75em; color: var(--muted); flex: none; min-width: 0; }
  .foot span, .foot button { display: inline-flex; align-items: center; gap: 4px; white-space: nowrap; }
  .foot button { padding: 0; border: 0; background: none; color: inherit; font: inherit; }
  .car { position: absolute; right: -10px; bottom: -10px; height: 75%; opacity: .9; pointer-events: none; }
  .short { gap: 6px; }
  .short .top { gap: 12px; }
  .short .info { gap: 2px; }
  @supports not (font-size: 1cqmin) { .pct { font-size: 1.5em; } }
</style>
