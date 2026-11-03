<script module>
  export const meta = {
    type: 'run', name: 'Run (automations, scripts, scenes)', icon: 'mdi:play-circle-outline', category: 'Controls',
    size: { w: 420, h: 180 }, tap: 'none',
    defaults: { title: '', items: [], names: '', hide_names: false, confirm: false, conditions: false, hide_last: false },
    fields: [
      { key: 'title', label: 'Title', type: 'text' },
      { key: 'items', label: 'Automations, scripts and scenes', type: 'entities', domain: ['automation', 'script', 'scene'] },
      { key: 'names', label: 'Names (one per line, in the same order; a blank line keeps the entity’s name)', type: 'textarea' },
      { key: 'hide_names', label: 'Hide names (just the icon, e.g. when the title says it)', type: 'bool' },
      { key: 'confirm', label: 'Tap twice to run (asks “Tap again to run”)', type: 'bool' },
      { key: 'conditions', label: 'Automations: only run when their conditions are met', type: 'bool' },
      { key: 'hide_last', label: 'Hide when each last ran', type: 'bool' },
    ],
  };
</script>

<script>
  // One tap to run an automation (its actions, as "Run actions" does in Home
  // Assistant, without switching it off), a script or a scene, with a spinner
  // and then "Done" or "Failed" so you know it happened.
  import Icon from '../components/Icon.svelte';
  import { t, ent } from '../lib/tpl.js';
  import { send } from '../lib/ha.svelte.js';
  import { relTime } from '../lib/entity.js';
  import { clock } from '../lib/clock.svelte.js';
  let { props, w = 420, h = 180 } = $props();
  const gone = (e) => !e || e.state === 'unavailable' || e.state === 'unknown';
  const dom = (id) => id.split('.')[0];

  const names = $derived(String(t(props.names) || '').split('\n'));
  // An icon from the entity, else from words in its name, else by type.
  const WORDS = [[/light|lamp/i, 'mdi:lightbulb-group-outline'], [/lock|unlock/i, 'mdi:lock-outline'], [/garage/i, 'mdi:garage'], [/door/i, 'mdi:door'], [/holiday|christmas|decoration/i, 'mdi:pine-tree'], [/fan/i, 'mdi:fan'], [/tv|movie/i, 'mdi:television'], [/music|speaker|sound/i, 'mdi:music'], [/temp|heat|cool|thermostat/i, 'mdi:thermostat'], [/alarm|security|arm/i, 'mdi:shield-home-outline'], [/night|sleep|bed/i, 'mdi:weather-night'], [/morning|wake/i, 'mdi:weather-sunset-up'], [/charg/i, 'mdi:battery-charging'], [/notif|announce|speak/i, 'mdi:bullhorn-outline']];
  const iconFor = (e, label) => e.attributes.icon || WORDS.find(([re]) => re.test(label))?.[1] || { automation: 'mdi:robot-outline', script: 'mdi:script-text-outline', scene: 'mdi:palette-outline' }[dom(e.entity_id)] || 'mdi:play';
  const items = $derived((props.items || []).map((id, i) => {
    const e = ent(id);
    if (!e) return null;
    const label = (names[i] || '').trim() || (e.attributes.friendly_name || id).trim();
    const d = dom(e.entity_id);
    // When it last ran: automations and scripts say so; a scene's state is when it was last set.
    const last = d === 'scene' ? e.state : e.attributes.last_triggered;
    return { e, d, label, icon: iconFor(e, label), last: gone(e) && d !== 'scene' ? null : last, autoOff: d === 'automation' && e.state === 'off', down: e.state === 'unavailable' };
  }).filter(Boolean));

  // Per tile: '' | 'armed' | 'running' | 'done' | 'failed', and why it failed.
  let st = $state({});
  let why = $state({});
  const timers = {};
  const set = (id, v, ms) => { st = { ...st, [id]: v }; clearTimeout(timers[id]); if (ms) timers[id] = setTimeout(() => (st = { ...st, [id]: '' }), ms); };
  async function run(it) {
    const id = it.e.entity_id;
    if (st[id] === 'running' || it.down) return;
    if (props.confirm && st[id] !== 'armed') return set(id, 'armed', 4000);
    set(id, 'running');
    const [service, data] = it.d === 'automation' ? ['trigger', { skip_condition: !props.conditions }] : ['turn_on', {}];
    try {
      await send({ type: 'call_service', domain: it.d, service, service_data: data, target: { entity_id: id } });
      set(id, 'done', 3000);
    } catch (err) {
      why = { ...why, [id]: err?.message || 'Home Assistant refused it' };
      set(id, 'failed', 6000);
    }
  }
  const lastText = (v) => { clock.now; if (!v) return 'Never run'; const r = relTime(v); return typeof r === 'string' ? r.replace(/^./, (c) => c.toUpperCase()) : ''; };
  // As many columns as fit (tiles at least ~110 px), so a few actions sit in one row.
  const cols = $derived(Math.max(1, Math.min(items.length, Math.floor(w / 110) || 1)));
</script>

<div class="run">
  {#if !(props.items || []).length}
    <div class="hint">Pick the automations, scripts or scenes to run in the card's settings.</div>
  {:else}
    {#if props.title}<div class="title">{t(props.title)}</div>{/if}
    <div class="grid" class:bare={props.hide_names} style="--cols:{cols}" data-stop>
      {#each items as it (it.e.entity_id)}
        {@const s = st[it.e.entity_id] || ''}
        <button class="tile {s}" class:down={it.down} onclick={() => run(it)} title={s === 'failed' ? why[it.e.entity_id] : it.label} disabled={it.down}>
          <span class="ic">
            {#if s === 'running'}<span class="spin"></span>
            {:else if s === 'done'}<Icon icon="mdi:check-bold" size="1.3em" />
            {:else if s === 'failed'}<Icon icon="mdi:close-thick" size="1.3em" />
            {:else}<Icon icon={it.icon} size="1.3em" />{/if}
          </span>
          {#if !props.hide_names}<span class="nm">{it.label}</span>{/if}
          <span class="sub">
            {#if s === 'armed'}Tap again to run
            {:else if s === 'running'}Running…
            {:else if s === 'done'}Done
            {:else if s === 'failed'}Failed{why[it.e.entity_id] ? `: ${why[it.e.entity_id]}` : ''}
            {:else if it.down}Unavailable
            {:else}{props.hide_last ? '' : lastText(it.last)}{#if it.autoOff}<span class="off">· auto off</span>{/if}{/if}
          </span>
        </button>
      {/each}
    </div>
  {/if}
</div>

<style>
  .run { height: 100%; display: flex; flex-direction: column; gap: 8px; min-height: 0; }
  .hint { color: var(--muted); font-size: .82em; margin: auto; text-align: center; }
  .title { font-weight: 600; }
  .grid { flex: 1; min-height: 0; display: grid; grid-template-columns: repeat(var(--cols), minmax(0, 1fr)); grid-auto-rows: minmax(0, 1fr); gap: 8px; overflow: auto; }
  .tile { display: flex; flex-direction: column; align-items: flex-start; justify-content: center; gap: 4px; min-width: 0; min-height: 76px; padding: 10px 12px; border: 0; border-radius: 14px; background: rgba(255,255,255,.06); color: inherit; font: inherit; text-align: left; transition: background .2s, transform .1s; }
  .tile:active { transform: scale(.97); }
  .ic { display: grid; place-items: center; width: 2em; height: 2em; border-radius: 10px; background: color-mix(in srgb, var(--accent) 18%, transparent); color: var(--accent); }
  /* Names hidden: the icon, larger, centred over the status line. */
  .bare .tile { align-items: center; text-align: center; gap: 6px; }
  .bare .ic { width: 2.6em; height: 2.6em; border-radius: 14px; font-size: 1.15em; }
  .nm { max-width: 100%; font-weight: 600; font-size: .9em; line-height: 1.2; overflow: hidden; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; }
  .sub { max-width: 100%; font-size: .72em; color: var(--muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
  .off { color: #ffc861; margin-left: .35em; }
  .tile.armed { background: rgba(255,200,97,.14); }
  .tile.armed .sub { color: #ffc861; font-weight: 600; }
  .tile.running .ic { background: rgba(255,255,255,.08); }
  .tile.done { background: rgba(91,216,143,.14); }
  .tile.done .ic { background: rgba(91,216,143,.22); color: #5bd88f; }
  .tile.done .sub { color: #5bd88f; font-weight: 600; }
  .tile.failed { background: rgba(255,122,144,.14); }
  .tile.failed .ic { background: rgba(255,122,144,.22); color: #ff7a90; }
  .tile.failed .sub { color: #ff7a90; }
  .tile.down { opacity: .45; }
  .spin { width: 1.1em; height: 1.1em; border-radius: 50%; border: 2px solid rgba(255,255,255,.25); border-top-color: var(--accent); animation: spin .8s linear infinite; }
  @keyframes spin { to { transform: rotate(360deg); } }
</style>
