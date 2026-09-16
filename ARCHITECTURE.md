# Architecture

Everything is in `index.html`: the styles, one card per sensor, and one script.
Helpers `setCard(id, state)` and `setVal(id, html)` mark a card active or in
error and fill it; each sensor's code only calls those.

## Permissions

iOS allows a motion permission request only as the direct result of a tap, and
not after any asynchronous step. So the page asks in two taps:

1. **Motion.** `requestMotionPermission()` calls
   `DeviceMotionEvent.requestPermission()` where it exists (iOS 13+), and
   otherwise starts straight away. On success, `motionGranted()` adds the
   `devicemotion` and `deviceorientation` listeners, starts the chart loop and
   dead reckoning, and enables the second button.
2. **Camera, microphone and location.** `requestMediaPermissions()` requests
   the rear camera and the microphone with `getUserMedia`, and starts
   `navigator.geolocation.watchPosition`. When both media requests have settled,
   speech recognition is set up.

The screen orientation, visual viewport, network and client cards need no
permission and fill in on load.

## Motion

- **Accelerometer** reads `accelerationIncludingGravity`; the g-force is its
  magnitude ÷ 9.80665 (standard gravity).
- **Gyroscope** reads `rotationRate`.
- **Orientation** reads `alpha` (heading), `beta` (pitch) and `gamma` (roll),
  and rotates the compass needle by `alpha`.

Each event also writes to a ring buffer, and `motionChartLoop()` redraws the
three history charts once per animation frame.

**Dead reckoning** takes `acceleration` (gravity already removed by the OS), or,
where that is missing, `accelerationIncludingGravity` minus 9.81 on z. It adds
`acceleration × dt` to a velocity vector, and speed × `dt` to a distance. After
8 consecutive samples under 0.15 m/s² it assumes the phone is still and zeroes
the velocity (a zero-velocity update, ZUPT). Summing noisy acceleration drifts
fast, so a warning appears after 10 s.

## Location

`watchPosition` (high accuracy) fills the geolocation card and places a marker
on a [Leaflet](https://leafletjs.com/) map. On load the page also fetches
`https://ipapi.co/json/` and places a second marker at the IP address's
estimated position, then shows the distance between the two. Tiles come from
`tile.openstreetmap.org`, darkened by a CSS filter on the tile layer.

## Camera

The stream plays in a `<video>` and is drawn, once per animation frame, onto a
128 × 72 canvas. From those pixels the page builds 256-bin R, G, B and
luminance histograms, the average colour, and a dominant colour, and pushes the
averages into history charts. The small canvas keeps this cheap on a phone.

## Microphone

The stream feeds an `AnalyserNode` (FFT size 2048, smoothing 0.75).

- **Level and dBFS** come from the RMS of the time-domain samples, with a
  decaying peak hold.
- **Pitch** is found by autocorrelation of the time-domain samples (ignored
  below an RMS of 0.01), and shown as the nearest note with its deviation in
  cents.
- **Chords:** spectral peaks between 80 Hz and 5 kHz, at least 30 dB above the
  median bin, become notes. Their pitch classes are compared with 13 chord
  shapes (major, minor, diminished, augmented, 7, maj7, m7, dim7, sus4, sus2,
  add9, m7♭5, power chord) with each note tried as the root.
- **Oscilloscope, spectrogram and spectrum** draw the time-domain data, a
  scrolling waterfall and the current spectrum.

## Speech

Uses `SpeechRecognition` or `webkitSpeechRecognition`, with continuous
recognition and interim results, shown as a running transcript. This is the
browser's own recogniser, which may send audio to its maker's servers.

## Device and network cards

- **Network / IP** fetches Cloudflare's `/cdn-cgi/trace` from the same origin
  and parses its `key=value` lines. It only works behind Cloudflare.
- **Client / device** reads `navigator`, `screen`, `Intl`, `performance.memory`
  where present, and the GPU name from WebGL's `WEBGL_debug_renderer_info`
  extension.
- **Screen orientation** uses `screen.orientation`, falling back to
  `window.orientation`; **visual viewport** uses `window.visualViewport`. Both
  update on change.

## Deployment files

| File | Purpose |
|---|---|
| `_headers` | `Content-Security-Policy` allowing inline script, unpkg (Leaflet), Google Fonts, OpenStreetMap tiles, ipapi.co, and `blob:` media; plus `nosniff`, `DENY` framing and a referrer policy (OpenStreetMap's tile policy requires a referrer) |
| `wrangler.toml` | Worker `ios-sensors`, custom domain, `assets.directory = "./"` |
| `.assetsignore` | Keeps `wrangler.toml` and the repository docs off the site |
