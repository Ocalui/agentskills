---
name: remotion
description: Build programmatic videos with Remotion and React. Use when creating, animating, or rendering videos using code — covering composition setup, frame-based animation, interpolation, spring physics, media embedding, and CLI rendering.
---

# Remotion

Remotion creates videos programmatically using React. Each frame is a React render at a specific point in time. There is no timeline editor — animation logic lives entirely in code.

## Setup

```bash
npx create-video@latest        # scaffold new project
npx remotion studio            # open visual dev environment
npx remotion render            # render to video file
```

**Version pinning (critical):** All `remotion` and `@remotion/*` packages must be the exact same version. Always install with `--save-exact` or pin in package.json.

## Core Hooks

### `useCurrentFrame()`
Returns the current frame number (0-indexed). Inside a `<Sequence>`, returns frame relative to the sequence's start.

```tsx
import {useCurrentFrame} from 'remotion';

const MyComp = () => {
  const frame = useCurrentFrame(); // 0, 1, 2, ...
  return <div style={{opacity: frame / 30}} />;
};
```

### `useVideoConfig()`
Returns composition metadata. Throws if called outside a `<Composition>` or `<Player>`.

```tsx
import {useVideoConfig} from 'remotion';

const {width, height, fps, durationInFrames} = useVideoConfig();
// width: 1920, height: 1080, fps: 30, durationInFrames: 150
```

## Animation Utilities

### `interpolate()`
Maps an input value linearly across a range. Use `extrapolateRight: 'clamp'` to prevent values from going out of bounds.

```tsx
import {interpolate} from 'remotion';

const frame = useCurrentFrame();

// Fade in over first 20 frames, stay at 1 after
const opacity = interpolate(frame, [0, 20], [0, 1], {
  extrapolateRight: 'clamp',
});

// Slide in from left
const x = interpolate(frame, [0, 30], [-200, 0], {
  extrapolateRight: 'clamp',
});
```

### `spring()`
Physics-based animation — bouncy, natural motion. Always pass `frame` and `fps`.

```tsx
import {spring} from 'remotion';

const frame = useCurrentFrame();
const {fps} = useVideoConfig();

const scale = spring({
  frame,
  fps,
  from: 0,
  to: 1,
  config: {stiffness: 100, damping: 10},
  durationInFrames: 40,
});
```

**`config` presets:**
- Default: `{mass: 1, damping: 10, stiffness: 100, overshootClamping: false}`
- Bouncy: low `damping`, high `stiffness`
- Stiff: high `damping`, high `stiffness`

## Core Components

### `<Composition>`
Defines a renderable video. Register all compositions in `src/Root.tsx`.

```tsx
<Composition
  id="MyVideo"              // unique, alphanumeric + hyphens
  component={MyComp}        // or lazyComponent={() => import('./MyComp')}
  width={1920}
  height={1080}
  fps={30}
  durationInFrames={150}    // 5 seconds at 30fps
  defaultProps={{title: 'Hello'}}
/>
```

### `<Sequence>`
Shows children only during a time window. Resets `useCurrentFrame()` to 0 at its `from` point.

```tsx
import {Sequence} from 'remotion';

// Show intro for first 60 frames, then main content
<>
  <Sequence durationInFrames={60}>
    <Intro />
  </Sequence>
  <Sequence from={60}>
    <MainContent />
  </Sequence>
</>
```

### `<AbsoluteFill>`
Full-width/height absolute positioned div. Use for layering — last child renders on top.

```tsx
import {AbsoluteFill} from 'remotion';

<AbsoluteFill style={{backgroundColor: 'black'}}>
  <AbsoluteFill>
    <BackgroundVideo />
  </AbsoluteFill>
  <AbsoluteFill>
    <TitleOverlay />     {/* renders on top */}
  </AbsoluteFill>
</AbsoluteFill>
```

### `<Series>`
Sequence of clips that play back-to-back (shorthand for multiple `<Sequence>` with auto-calculated `from`).

```tsx
import {Series} from 'remotion';

<Series>
  <Series.Sequence durationInFrames={40}><Clip1 /></Series.Sequence>
  <Series.Sequence durationInFrames={60}><Clip2 /></Series.Sequence>
</Series>
```

### `<Loop>`
Repeats children for a given number of iterations.

```tsx
import {Loop} from 'remotion';

<Loop durationInFrames={30} times={5}>
  <Blink />
</Loop>
```

## Media

### Video
Prefer `<OffthreadVideo>` over `<Video>` for rendering — it uses FFmpeg to extract frames, avoiding browser decode issues.

```tsx
import {OffthreadVideo, staticFile} from 'remotion';

<OffthreadVideo src={staticFile('clip.mp4')} />
<OffthreadVideo src="https://example.com/video.mp4" />
```

### Audio

```tsx
import {Audio, staticFile} from 'remotion';

<Audio src={staticFile('track.mp3')} volume={0.8} startFrom={30} />
```

Use `volume` as a number (0–1) or a function for fades: `volume={(f) => interpolate(f, [0, 10], [0, 1])}`.

### Image

```tsx
import {Img, staticFile} from 'remotion';

<Img src={staticFile('logo.png')} />
```

### `staticFile()`
References files in the `public/` directory.

```tsx
import {staticFile} from 'remotion';

const src = staticFile('video.mp4');  // → /public/video.mp4
```

## Async Rendering

Use `delayRender` / `continueRender` to wait for async data before a frame is captured.

```tsx
import {delayRender, continueRender, useEffect, useState} from 'remotion';

const MyComp = () => {
  const [data, setData] = useState(null);
  const [handle] = useState(() => delayRender('Loading data'));

  useEffect(() => {
    fetch('/api/data')
      .then(r => r.json())
      .then(d => {
        setData(d);
        continueRender(handle);
      });
  }, []);

  if (!data) return null;
  return <div>{data.title}</div>;
};
```

## Input Props

Pass dynamic data into compositions at render time.

```tsx
// In composition
import {getInputProps} from 'remotion';
const props = getInputProps(); // { name: 'Alice' }

// Or via component props (preferred with schema)
type Props = {name: string};
const MyComp: React.FC<Props> = ({name}) => <div>{name}</div>;
```

```bash
npx remotion render --props='{"name":"Alice"}'
```

## Rendering

```bash
# Basic render
npx remotion render src/index.ts MyVideo out/video.mp4

# Common options
npx remotion render \
  --codec=h264 \         # h264 (default), h265, vp8, vp9, av1, prores
  --crf=18 \             # quality: lower = better (h264 default: 18)
  --jpeg-quality=80 \    # for image sequences
  --scale=2 \            # 2x resolution scale
  --concurrency=4        # parallel rendering threads
```

## Common Patterns

### Fade in/out
```tsx
const opacity = interpolate(frame, [0, 15, durationInFrames - 15, durationInFrames], [0, 1, 1, 0], {
  extrapolateLeft: 'clamp',
  extrapolateRight: 'clamp',
});
```

### Count frames to seconds
```tsx
const {fps} = useVideoConfig();
const seconds = frame / fps;
const atSecond = (s: number) => s * fps;
```

### Text reveal
```tsx
const chars = "Hello World".split('');
const visibleChars = Math.floor(interpolate(frame, [0, 30], [0, chars.length], {
  extrapolateRight: 'clamp',
}));
return <span>{chars.slice(0, visibleChars).join('')}</span>;
```

## Gotchas

- **All `@remotion/*` packages must be the same version.** Mismatched versions cause silent failures.
- **`useCurrentFrame()` is 0-indexed.** Frame 0 is the first frame.
- **Inside `<Sequence from={30}>`, `useCurrentFrame()` returns 0 at frame 30** of the parent — not 30.
- **`<Img>` blocks rendering** until loaded. For remote images, use `delayRender`.
- **`<Video>` can have decode issues during rendering** — prefer `<OffthreadVideo>`.
- **Licensing:** Free for individuals and small companies (≤3 employees). Requires a Company License from remotion.pro for larger organizations.
- **No side effects in render.** Don't use `Math.random()` directly — use `random()` from remotion for deterministic values.

```tsx
import {random} from 'remotion';
const val = random('seed-string'); // deterministic 0–1
```
