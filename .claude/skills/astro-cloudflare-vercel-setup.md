# Astro Blog on Cloudflare Pages with Vercel Proxy

Guide for setting up an Astro blog deployed on Cloudflare Pages, served via Vercel rewrites on a subdirectory (e.g., `celesto.ai/blog`).

## Architecture

```
celesto.ai (Vercel - React app)
    └── /blog/* → paper-blog.aniket-maurya.workers.dev (Cloudflare Pages - Astro)
```

## Key Configuration

### 1. Astro Config (`astro.config.ts`)

```ts
export default defineConfig({
  site: "https://celesto.ai/blog/",
  base: "/blog",           // Critical: sets base path for all routes and assets
  trailingSlash: "always", // Ensures consistent URL format
  output: "server",        // SSR mode for Cloudflare
  adapter: cloudflare(),
});
```

### 2. Vercel Rewrites (`vercel.json`)

```json
{
  "rewrites": [
    {
      "source": "/blog/(.*)",
      "destination": "https://paper-blog.aniket-maurya.workers.dev/blog/$1"
    },
    {
      "source": "/blog",
      "destination": "https://paper-blog.aniket-maurya.workers.dev/blog/"
    }
  ]
}
```

**Important patterns:**
- Use `(.*)` regex instead of `:path*` or `:path+` - handles trailing slashes correctly
- Need explicit rule for `/blog` (no trailing slash) as `(.*)` requires the trailing slash
- Must forward the full path including `/blog` prefix to Cloudflare

### 3. Internal Links

Since `base: "/blog"` is set, use `import.meta.env.BASE_URL` for all internal links:

```astro
---
const base = import.meta.env.BASE_URL; // Returns "/blog/"
---
<a href={base}>Home</a>
<a href={`${base}posts/`}>Posts</a>
```

Or use a config constant:

```ts
// src/config.ts
export const SITE = {
  basePath: "/blog",
  // ...
};
```

```astro
<a href={`${SITE.basePath}/posts/`}>Posts</a>
```

## Common Issues & Solutions

### Issue: 404 from Vercel on `/blog/posts/`

**Cause:** Vercel rewrite pattern `:path*` doesn't match empty paths (trailing slash only)

**Solution:** Use regex `(.*)` which matches zero or more characters:
```json
{ "source": "/blog/(.*)", "destination": "https://cloudflare.dev/blog/$1" }
```

### Issue: 404 from Cloudflare on `/blog/`

**Cause:** Mismatch between Vercel rewrite and Astro's `base` config
- Vercel strips `/blog` prefix: `/blog/posts/` → Cloudflare `/posts/`
- But Astro with `base: "/blog"` serves at `/blog/posts/`

**Solution:** Preserve the `/blog` prefix in rewrites:
```json
// Wrong - strips /blog
{ "source": "/blog/:path*", "destination": "https://cf.dev/:path*" }

// Correct - preserves /blog
{ "source": "/blog/(.*)", "destination": "https://cf.dev/blog/$1" }
```

### Issue: CSS/Assets not loading (unstyled page)

**Cause:** Assets are at `/_astro/*` but need to be at `/blog/_astro/*`

**Solution:** Ensure `base: "/blog"` is set in Astro config. This makes Astro generate all asset paths with the `/blog` prefix.

### Issue: `trailingSlash: "always"` causing 404 in dev

**Cause:** Accessing `/blog` instead of `/blog/`

**Solution:** Always use trailing slash in URLs, or configure redirect in Vercel:
```json
{
  "redirects": [
    { "source": "/blog", "destination": "/blog/", "permanent": true }
  ]
}
```

## Deployment Commands

```bash
# Build Astro site
npm run build

# Deploy to Cloudflare Pages
wrangler pages deploy dist

# Or if using project name
wrangler pages deploy dist --project-name paper-blog
```

## Testing Checklist

After deployment, verify these URLs return 200:
- `https://celesto.ai/blog` - Home (no trailing slash)
- `https://celesto.ai/blog/` - Home (with trailing slash)
- `https://celesto.ai/blog/posts/` - Posts listing
- `https://celesto.ai/blog/posts/[slug]/` - Individual post
- `https://celesto.ai/blog/_astro/[file].css` - CSS assets load

## Files Modified

When setting up this architecture, these files need attention:

1. `astro.config.ts` - Set `base` and `site`
2. `src/config.ts` - Add `basePath` constant
3. `vercel.json` (in main app repo) - Configure rewrites
4. All components with internal links - Use `BASE_URL` or `SITE.basePath`
