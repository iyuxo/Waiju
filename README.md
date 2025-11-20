<p align="center">
  <img src="https://placehold.co/900x220/111827/67e8f9?text=Waiju+Lavalink+Client" alt="Waiju Lavalink Client Banner" />
</p>

<p align="center">
  <a href="https://www.npmjs.com/package/waiju">
    <img src="https://img.shields.io/npm/v/waiju.svg?color=ff79c6&label=npm" alt="npm version">
  </a>
  <a href="https://github.com/waiju-bot/waiju/actions">
    <img src="https://img.shields.io/github/actions/workflow/status/waiju-bot/waiju/ci.yml?label=CI&logo=github" alt="build status">
  </a>
  <a href="https://github.com/waiju-bot/waiju/blob/main/LICENSE">
    <img src="https://img.shields.io/badge/license-MIT-00bcd4.svg" alt="license">
  </a>
  <a href="#installation">
    <img src="https://img.shields.io/badge/node-%3E%3D%2018.0.0-8bc34a" alt="node support">
  </a>
</p>

# Waiju Lavalink Client

**Waiju** is a powerful, drop-in Lavalink client for Discord bots. It ships as a
single JavaScript file (`waiju.js`) yet still supports multi-node orchestration,
rich events, queue utilities, advanced filters, and automatic recovery flows.

## Features

- ✅ Multiple Lavalink nodes with automatic reconnection and failover
- ✅ Player manager per guild with queue + repeat/autoplay helpers
- ✅ Track controls: play, pause, seek, skip, volume, filter chains, crossfade
- ✅ Comprehensive events (`trackStart`, `nodeConnect`, `voiceServerUpdate`, …)
- ✅ Search helpers (YouTube, YTM, SoundCloud, Spotify, local URLs, etc.)
- ✅ Voice payload bridge with smart rate-limited Discord WS dispatch
- ✅ Auto-resume support via resume keys and session rehydration
- ✅ Optional persistent queue/state store for bot restarts
- ✅ REST utilities (`decodeTrack`, `decodeTracks`, stats snapshot)
- ✅ Built-in caching, logging, error handling, and optional debug mode
- ✅ Config file watcher for hot-reloading Lavalink nodes/settings
- ✅ Filter presets + custom chains (bass boost, nightcore, vaporwave, …)
- ✅ Advanced queue helpers: move, remove, shuffle, clear, repeat, autoplay

Everything lives in one file, making Waiju easy to vendor into any bot project or
publish as a package.

## Installation

```bash
npm install waiju
# or
yarn add waiju
```

Waiju depends on `ws`, `undici`, and `tseep`, which are installed automatically.

## Quick Start

```js
const { Client } = require('discord.js');
const { Waiju } = require('waiju');

const client = new Client({ intents: 641 });

const waiju = new Waiju(client, {
  nodes: [
    {
      host: 'lavalink.example.com',
      port: 2333,
      password: 'youshallnotpass',
      secure: true,
      identifier: 'main',
    },
  ],
  send: (payload) => {
    const guild = client.guilds.cache.get(payload.d.guild_id);
    if (guild) guild.shard.send(payload);
  },
  defaultSearchPlatform: 'ytmsearch',
  autoResume: true,
  autoResumeKey: 'waiju-demo-key',
  debug: true,
});

client.on('ready', () => {
  waiju.init(client.user.id);
  console.log(`Logged in as ${client.user.tag}`);
});

waiju.on('trackStart', (player, track) => {
  console.log(`Now playing ${track.info?.title ?? 'Unknown'} in ${player.guildId}`);
});
```

### Creating / Controlling a Player

```js
const player = waiju.players.create(guildId, {
  voiceChannelId: voiceChannelId,
  nodeIdentifier: 'main',
});

const results = await waiju.search('ytmsearch:never gonna give you up');
player.queue.add(results.tracks[0]);
await player.play();
```

Available player helpers include `pause/resume`, `seek`, `skip`, `previous`,
`setVolume`, `setFilters`, `setRepeat`, `setAutoplay`, `setCrossfade`, and more.

### Events

Waiju re-emits Lavalink node events as `EventEmitter` events, including:

- `nodeConnect`, `nodeDisconnect`, `nodeError`
- `playerCreate`, `playerDestroy`, `playerSocketClosed`
- `trackStart`, `trackEnd`, `trackStuck`, `trackRepeat`, `queueEnd`
- `voiceServerUpdate`, `voiceStateUpdate`, `raw`

Listen through either the Waiju instance or individual players:

```js
waiju.on('queueEnd', (player) => {
  console.log(`Queue finished in ${player.guildId}`);
});

player.on('trackRepeat', () => {
  console.log('Track repeat enabled');
});
```

## Configuration Reference

```js
const waiju = new Waiju(client, {
  nodes: [
    {
      host: 'localhost',
      port: 2333,
      identifier: 'local',
      password: 'youshallnotpass',
      secure: false,
    },
  ],
  send: (payload) => client.guilds.cache.get(payload.d.guild_id)?.shard.send(payload),
  defaultSearchPlatform: 'ytsearch',
  autoResume: true,
  autoResumeKey: 'optional-resume-key',
  autoResumeTimeout: 60000,
  retryLimit: 5,
  cacheTTL: 15 * 60 * 1000,
  nodeSelection: 'leastLoad', // 'roundRobin' | 'priority'
  debug: false,
});
```

### Node Selection Strategies

- `leastLoad` (default): picks the connected node with the fewest active players.
- `roundRobin`: cycles through available nodes evenly.
- `priority`: uses `options.nodes` order as priority.

Use `waiju.useNode(id)` to pin the next player/search to a specific node.

## Scripts

- `npm test` – verifies the file loads (basic smoke test).
- `npm run lint` – placeholder command; customize if you add ESLint or similar.

## Example Bot

Check `examples/discord-bot.js` for a discord.js v14 bot featuring an advanced
prefix command set (`play`, `skip`, `stop`, `pause`, `resume`, `queue`, `np`,
`repeat`, `volume`, `autoplay`, `remove`, `move`, `shuffle`, `clear`, `preset`,
`filters off`, `nodes`) plus playlist queuing, node diagnostics, and filter
presets. Copy it, set the environment variables described at the top of the
file, and run with:

```bash
node examples/discord-bot.js
```

## Configuration File

Waiju can watch a JSON config file (see `waiju.config.json`) to hot-reload node
definitions, resume keys, and persistence options without restarting the bot:

```json
{
  "autoResume": true,
  "autoResumeKey": "waiju-example",
  "statePersistence": { "path": "./.cache/waiju-state.json" },
  "nodes": [
    { "identifier": "main", "host": "lava.example.com", "port": 443, "secure": true }
  ]
}
```

Pass `configPath` when instantiating Waiju and it will sync nodes and options
whenever the file changes.

## State Persistence

Add `statePersistence: { path: "./.cache/waiju-state.json" }` (or provide a
custom adapter) to persist queues, filters, volume, and pending tracks. After a
bot restart, Waiju restores the queue locally and continues playback as soon as
it reconnects to Discord voice and Lavalink.

## Filter Presets

Use `FILTER_PRESETS` (exported from `WaijuClient`) or `player.applyPreset(name)`
for built-in mixes such as `bassboost`, `nightcore`, `vaporwave`, and `karaoke`.
Call `player.clearFilters()` or use the `filters off` command in the example bot
to reset everything.

## License

MIT © Waiju Contributors

