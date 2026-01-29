# Deploying with Git LFS (GLB model)

The 3D model is tracked with **Git LFS**. On Vercel you must pull the real file before build, otherwise the app bundles the LFS *pointer* (text) and you get: `"version ht"... is not valid JSON`.

## Option 1: Enable Git LFS on Vercel (recommended)

1. **Vercel Dashboard** → your project → **Settings** → **Git**
2. Turn **on** “Git LFS”
3. **Redeploy** (Deployments → ⋮ on latest → Redeploy)

Vercel will then fetch LFS objects during clone, so the real `.glb` is present when `vite build` runs.

---

## Option 2: Host the GLB elsewhere (no LFS in build)

If LFS keeps causing issues, host the model yourself and load it by URL:

1. Upload `3dmodel.glb` to a static host (e.g. [Vercel Blob](https://vercel.com/docs/storage/vercel-blob), S3, Cloudflare R2, or a repo’s Releases / raw URL).
2. In `src/components/ThreeScene.vue`, replace the asset import with that URL:

```js
// Remove: import modelUrl from '../assets/models/3dmodel.glb?url';
const modelUrl = 'https://your-cdn-or-blob-url/3dmodel.glb';
```

Then the build no longer depends on the binary in the repo.
