# genuary-2025

My entries for Genuary 2025

# Sveltekit project with Node runtime:

Run project:

```bash
npm run dev
```

Download ffmpeg:

```bash
curl 'https://unpkg.com/@ffmpeg/core@0.12.10/dist/esm/ffmpeg-core.js' -o static/ffmpeg/ffmpeg-core.js
curl 'https://unpkg.com/@ffmpeg/core@0.12.10/dist/esm/ffmpeg-core.wasm' -o static/ffmpeg/ffmpeg-core.wasm
```

ffmpeg multi-threaded: (not working)

```bash
curl 'https://unpkg.com/@ffmpeg/core-mt@0.12.10/dist/esm/ffmpeg-core.js' -o static/ffmpeg/ffmpeg-core.js
curl 'https://unpkg.com/@ffmpeg/core-mt@0.12.10/dist/esm/ffmpeg-core.wasm' -o static/ffmpeg/ffmpeg-core.wasm
curl 'https://unpkg.com/@ffmpeg/core-mt@0.12.10/dist/esm/ffmpeg-core.worker.js' -o static/ffmpeg/ffmpeg-core.worker.js
```

## Mixbox

This project uses [mixbox](https://github.com/scrtwpns/mixbox) for prompt 16. Mixbox is licensed for non-commercial use under the CC BY-NC
4.0 license: [View License](https://github.com/scrtwpns/mixbox/tree/master?tab=License-1-ov-file#readme)