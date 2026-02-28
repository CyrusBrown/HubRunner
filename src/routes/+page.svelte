<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import WebSocket from '@tauri-apps/plugin-websocket';

  const DEV_FUEL_RED = 'FUEL_SENSOR_RED';
  const DEV_FUEL_BLUE = 'FUEL_SENSOR_BLUE';
  const DEV_MOTORS_RED = 'HUB_MOTORS_RED';
  const DEV_MOTORS_BLUE = 'HUB_MOTORS_BLUE';

  let ws: WebSocket | null = null;
  let poll_interval: number;

  let red_raw_count = $state(0);
  let blue_raw_count = $state(0);
  let red_offset = $state(0);
  let blue_offset = $state(0);

  let red_score = $derived(Math.max(0, red_raw_count - red_offset));
  let blue_score = $derived(Math.max(0, blue_raw_count - blue_offset));

  let red_motors_on = $state(false);
  let blue_motors_on = $state(false);

  function send_ws(device_id: string, command: string) {
    if (!ws) return;
    ws.send(JSON.stringify({ type: 'device_command', device_id, command, payload: {} }));
  }

  onMount(async () => {
    try {
      ws = await WebSocket.connect('ws://10.0.100.5:8000/connect');
      ws.addListener((msg) => {
        try {
          const parsed = JSON.parse(msg.data as string);
          if (parsed.data?.event === 'INTERNAL_COUNT') {
            if (parsed.device_id === DEV_FUEL_RED) red_raw_count = parsed.data.count;
            if (parsed.device_id === DEV_FUEL_BLUE) blue_raw_count = parsed.data.count;
          }
        } catch (e) {}
      });

      poll_interval = window.setInterval(() => {
        send_ws(DEV_FUEL_RED, 'GET_INTERNAL_COUNTS');
        send_ws(DEV_FUEL_BLUE, 'GET_INTERNAL_COUNTS');
      }, 1000);
    } catch (err) {
      console.error("Connection failed", err);
    }
  });

  onDestroy(() => {
    if (poll_interval) clearInterval(poll_interval);
    if (ws) ws.disconnect();
  });

  const reset_red = () => red_offset = red_raw_count;
  const reset_blue = () => blue_offset = blue_raw_count;

  function toggle_motors(alliance: 'red' | 'blue') {
    if (alliance === 'red') {
      red_motors_on = !red_motors_on;
      send_ws(DEV_MOTORS_RED, red_motors_on ? 'ALL_MOTORS_ON' : 'ALL_MOTORS_OFF');
    } else {
      blue_motors_on = !blue_motors_on;
      send_ws(DEV_MOTORS_BLUE, blue_motors_on ? 'ALL_MOTORS_ON' : 'ALL_MOTORS_OFF');
    }
  }
</script>

<main class="container">
  <div class="alliance">
    <h2 class="label-red">RED ALLIANCE</h2>
    <div class="score">{red_score}</div>
    <div class="controls">
      <button onclick={reset_red}>Reset Score</button>
      <button onclick={() => toggle_motors('red')} class:active={red_motors_on}>
        Motors: {red_motors_on ? 'ON' : 'OFF'}
      </button>
    </div>
  </div>

  <div class="divider"></div>

  <div class="alliance">
    <h2 class="label-blue">BLUE ALLIANCE</h2>
    <div class="score">{blue_score}</div>
    <div class="controls">
      <button onclick={reset_blue}>Reset Score</button>
      <button onclick={() => toggle_motors('blue')} class:active={blue_motors_on}>
        Motors: {blue_motors_on ? 'ON' : 'OFF'}
      </button>
    </div>
  </div>
</main>

<style>
  :root {
    --text-main: #334155;
    --text-light: #64748b;
    --border-color: #e2e8f0;
    --button-bg: #f1f5f9;
    --button-text: #475569;
    --button-active: #cbd5e1;
  }

  /* Reset & Base Font */
  :global(body) {
    margin: 0;
    padding: 0;
    font-family: system-ui, -apple-system, sans-serif;
    background-color: #ffffff;
    color: var(--text-main);
  }

  .container {
    max-width: 900px;
    margin: 4rem auto;
    display: flex;
    align-items: flex-start;
  }

  .alliance {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 2rem;
  }

  .divider {
    width: 1px;
    align-self: stretch;
    background-color: var(--border-color);
  }

  h2 {
    font-size: 0.85rem;
    letter-spacing: 0.1em;
    margin: 0;
    font-weight: 600;
  }

  .label-red { color: #ef4444; }
  .label-blue { color: #3b82f6; }

  .score {
    font-size: 5rem;
    margin: 2rem 0;
    font-variant-numeric: tabular-nums;
  }

  .controls {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    width: 100%;
    max-width: 200px;
  }

  button {
    appearance: none;
    background: var(--button-bg);
    border: 1px solid var(--border-color);
    color: var(--button-text);
    padding: 0.6rem;
    border-radius: 4px;
    font-size: 0.9rem;
    cursor: pointer;
    transition: all 0.1s ease;
  }

  button:hover {
    background: #e2e8f0;
  }

  button.active {
    background: var(--button-active);
    border-color: #94a3b8;
    box-shadow: inset 0 2px 4px rgba(0,0,0,0.05);
  }
</style>
