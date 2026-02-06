# Frontend Performance

## Code Splitting

### React Lazy Loading

```javascript
import { lazy, Suspense } from 'react';
import { Route, Routes } from 'react-router-dom';

// Lazy load components
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Analytics = lazy(() => import('./pages/Analytics'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard/*" element={<Dashboard />} />
        <Route path="/analytics" element={<Analytics />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

### Dynamic Imports

```javascript
// Load library on demand
async function loadChartLibrary() {
  const { Chart } = await import('chart.js');
  return new Chart(ctx, config);
}

// Intersection Observer for below-fold content
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      import('./HeavyComponent').then(module => {
        // Render component
      });
    }
  });
});
```

## Image Optimization

### Responsive Images

```html
<!-- Modern responsive image -->
<picture>
  <source 
    srcset="image-400.webp 400w,
            image-800.webp 800w,
            image-1200.webp 1200w"
    sizes="(max-width: 600px) 400px,
           (max-width: 1000px) 800px,
           1200px"
    type="image/webp">
  <source 
    srcset="image-400.jpg 400w,
            image-800.jpg 800w,
            image-1200.jpg 1200w"
    sizes="(max-width: 600px) 400px,
           (max-width: 1000px) 800px,
           1200px"
    type="image/jpeg">
  <img 
    src="image-800.jpg" 
    alt="Description"
    loading="lazy"
    decoding="async"
    width="800"
    height="600">
</picture>
```

### Next.js Image Component

```jsx
import Image from 'next/image';

// Automatic optimization
<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  priority={false}  // Above fold = true
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
/>

// External image
<Image
  src="https://example.com/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
  loader={({ src, width, quality }) => {
    return `${src}?w=${width}&q=${quality || 75}`;
  }}
/>
```

## Core Web Vitals

### Largest Contentful Paint (LCP)

```html
<!-- Preload critical resources -->
<link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin>
<link rel="preload" href="/css/critical.css" as="style">
<link rel="preload" href="/hero-image.jpg" as="image" fetchpriority="high">

<!-- Inline critical CSS -->
<style>
  /* Critical CSS above the fold */
  .hero { 
    background: url('/hero-image.jpg'); 
    min-height: 100vh;
  }
</style>

<!-- Preconnect to required origins -->
<link rel="preconnect" href="https://api.example.com">
<link rel="dns-prefetch" href="https://fonts.googleapis.com">
```

### First Input Delay (FID) / Interaction to Next Paint (INP)

```javascript
// Break up long tasks
async function processLargeDataset(data) {
  const chunkSize = 1000;
  
  for (let i = 0; i < data.length; i += chunkSize) {
    const chunk = data.slice(i, i + chunkSize);
    
    // Yield to main thread
    await new Promise(resolve => setTimeout(resolve, 0));
    
    processChunk(chunk);
  }
}

// Use requestIdleCallback for non-critical work
requestIdleCallback(() => {
  analytics.track('page_viewed');
});

// Web Workers for heavy computation
const worker = new Worker('./heavy-worker.js');
worker.postMessage({ data: largeArray });
worker.onmessage = (e) => {
  updateUI(e.data);
};
```

### Cumulative Layout Shift (CLS)

```html
<!-- Always set dimensions -->
<img src="photo.jpg" width="800" height="600" alt="Photo">

<!-- Reserve space for dynamic content -->
<div style="min-height: 200px;">
  {loading ? <Skeleton /> : <Content />}
</div>

<!-- Font display to prevent FOUT -->
<style>
  @font-face {
    font-family: 'CustomFont';
    src: url('/fonts/custom.woff2') format('woff2');
    font-display: swap;  /* or optional, fallback */
  }
</style>
```

## JavaScript Optimization

### Tree Shaking

```javascript
// ✅ Named imports (tree-shakeable)
import { useState, useEffect } from 'react';
import { map, filter } from 'lodash-es';

// ❌ Namespace imports (not tree-shakeable)
import * as _ from 'lodash';
import React from 'react';
```

### Bundle Analysis

```bash
# Webpack Bundle Analyzer
npm install --save-dev @next/bundle-analyzer

# Analyze build
npm run analyze

# VS Code extension: Import Cost
```

### Event Delegation

```javascript
// ❌ Individual listeners
document.querySelectorAll('.button').forEach(btn => {
  btn.addEventListener('click', handleClick);
});

// ✅ Event delegation
document.addEventListener('click', (e) => {
  if (e.target.matches('.button')) {
    handleClick(e);
  }
});
```

## CSS Optimization

### Critical CSS

```javascript
// Critical CSS extraction (Next.js)
// pages/_document.js
import Document, { Html, Head, Main, NextScript } from 'next/document';

class MyDocument extends Document {
  render() {
    return (
      <Html>
        <Head>
          {/* Inline critical CSS */}
          <style dangerouslySetInnerHTML={{ __html: criticalCss }} />
          
          {/* Preload non-critical CSS */}
          <link 
            rel="preload" 
            href="/css/non-critical.css" 
            as="style"
            onLoad="this.onload=null;this.rel='stylesheet'"
          />
        </Head>
        <body>
          <Main />
          <NextScript />
        </body>
      </Html>
    );
  }
}
```

### CSS Containment

```css
/* Isolate rendering */
.card {
  contain: layout style paint;
}

/* Strict containment for off-screen content */
.off-screen {
  content-visibility: auto;
  contain-intrinsic-size: 0 500px;
}
```

## Service Worker Caching

```javascript
// service-worker.js
const CACHE_NAME = 'myapp-v1';
const urlsToCache = [
  '/',
  '/styles.css',
  '/script.js',
  '/offline.html'
];

// Install
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(urlsToCache))
  );
});

// Fetch with cache-first strategy
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then(response => {
        // Return cached or fetch from network
        if (response) {
          return response;
        }
        return fetch(event.request);
      })
  );
});
```

## Performance Budgets

```javascript
// budgets.config.js
module.exports = {
  budgets: [
    {
      path: '/**',
      performanceBudget: {
        bundleSize: 250000,        // 250KB
        imageSize: 100000,         // 100KB
        scriptDuration: 100,       // 100ms
      }
    },
    {
      path: '/dashboard/**',
      performanceBudget: {
        bundleSize: 500000,        // 500KB
        scriptDuration: 200,
      }
    }
  ]
};

// Lighthouse CI
// lighthouserc.js
module.exports = {
  ci: {
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'interactive': ['error', { maxNumericValue: 3500 }],
      }
    }
  }
};
```
