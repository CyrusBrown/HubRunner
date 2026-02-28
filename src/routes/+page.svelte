
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import WebSocket from '@tauri-apps/plugin-websocket';

  // Device IDs
  const DEV_FUEL_RED = 'FUEL_SENSOR_RED';
  const DEV_FUEL_BLUE = 'FUEL_SENSOR_BLUE';
  const DEV_MOTORS_RED = 'HUB_MOTORS_RED';
  const DEV_MOTORS_BLUE = 'HUB_MOTORS_BLUE';

  // Commands Sent TO Devices
  const CMD_GET_COUNTS = 'GET_INTERNAL_COUNT';
  const CMD_RESET_COUNTS = 'RESET_INTERNAL_COUNT';
  const CMD_MOTORS_ON = 'ALL_MOTORS_ON';
  const CMD_MOTORS_OFF = 'ALL_MOTORS_OFF';

  // Events Received FROM Devices
  const EVT_FUEL_DETECTED = 'FUEL_DETECTED';
  const EVT_INTERNAL_COUNTS = 'INTERNAL_COUNT';

  let ws: WebSocket | null = null;
  let poll_interval: number;

  // Actual display scores
  let red_score = $state(0);
  let blue_score = $state(0);

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
          const event = parsed.data?.event;

          // 1. Live increment (Fast UI update)
          if (event === EVT_FUEL_DETECTED) {
            if (parsed.device_id === DEV_FUEL_RED) red_score += parsed.data.count;
            if (parsed.device_id === DEV_FUEL_BLUE) blue_score += parsed.data.count;
          } 
          // 2. Authoritative Sync (Overrides local count with hardware truth)
          else if (event === EVT_INTERNAL_COUNTS) {
            if (parsed.device_id === DEV_FUEL_RED) red_score = parsed.data.count;
            if (parsed.device_id === DEV_FUEL_BLUE) blue_score = parsed.data.count;
          }
        } catch (e) {
          // Ignore parsing errors for simplicity
        }
      });

      // Poll every 3 seconds for the true internal count
      poll_interval = window.setInterval(() => {
        send_ws(DEV_FUEL_RED, CMD_GET_COUNTS);
        send_ws(DEV_FUEL_BLUE, CMD_GET_COUNTS);
      }, 3000);

    } catch (err) {
      console.error("Connection failed", err);
    }
  });

  onDestroy(() => {
    if (poll_interval) clearInterval(poll_interval);
    if (ws) ws.disconnect();
  });

  // Reset Commands
  function reset_red() {
    red_score = 0; // Optimistic UI update
    send_ws(DEV_FUEL_RED, CMD_RESET_COUNTS);
  }

  function reset_blue() {
    blue_score = 0; // Optimistic UI update
    send_ws(DEV_FUEL_BLUE, CMD_RESET_COUNTS);
  }

  // Motor Commands
  function toggle_motors(alliance: 'red' | 'blue') {
    if (alliance === 'red') {
      red_motors_on = !red_motors_on;
      send_ws(DEV_MOTORS_RED, red_motors_on ? CMD_MOTORS_ON : CMD_MOTORS_OFF);
    } else {
      blue_motors_on = !blue_motors_on;
      send_ws(DEV_MOTORS_BLUE, blue_motors_on ? CMD_MOTORS_ON : CMD_MOTORS_OFF);
    }
  }
</script>

<main class="container">

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

  <div class="divider"></div>

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

