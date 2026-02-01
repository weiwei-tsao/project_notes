# Enhancing SPA Resilience: Handling Chunk Load Errors with Auto-Recovery

During a recent deployment cycle, I encountered a persistent issue that affects many Single Page Applications (SPAs): users reporting white screens or broken navigation shortly after a new version went live. While our `ErrorBoundary` component correctly caught these crashes (displaying an "Oops, something went wrong" message), the user experience was far from ideal.

I decided to investigate the root cause and implement a more proactive solution that resolves these issues without requiring manual user intervention.

## The Challenge: The "Missing Chunk" Phenomenon

The problem typically manifests when:
1.  **A Hot Module Replacement (HMR) occurs** during local development.
2.  **A new deployment happens** while a user still has the application open.

In modern bundlers (like Vite or Webpack), code splitting generates "chunks" with hash-based filenames for caching (e.g., `ServiceRequestsPageRouter-DNVwXkPv.js`). When we deploy a new build, these hashes change.

If a user tries to navigate to a route they haven't visited yet, their browser requests the *old* chunk filename. Since that file likely no longer exists on the server (or the server returns `index.html` as a fallback), the JavaScript module loading fails with a MIME type error or a 404.

The result? The application crashes, and the user is stuck staring at an error boundary.

## The Solution: Smart Auto-Retry

Instead of simply letting the `lazy()` import fail and crashing the app, I devised a wrapper utility called `lazyWithRetry`. This utility intercepts the loading error and attempts a recovery strategy before giving up.

### How It Works

The logic follows a simple but effective decision flow:

1.  **Intercept the Error**: When a dynamic import fails, catch the error.
2.  **Check Context**: Is this likely a chunk loading error?
3.  **Check History**: Have we already tried to "fix" this recently (within the last 10 seconds)?
4.  **Recover**:
    *   **If we haven't refreshed recently**: Force a page reload (`window.location.reload()`). This forces the browser to fetch the *new* `index.html`, which contains the references to the *new* correct chunk hashes.
    *   **If we JUST refreshed**: This is a genuine error (not a stale chunk). Throw the error so the `ErrorBoundary` can display the fallback UI.

### Implementation Details

I created a new utility file `lazyWithRetry.ts` to replace the standard React `lazy` function.

```typescript
// lazyWithRetry.ts
import { lazy, ComponentType, LazyExoticComponent } from 'react';

/**
 * A wrapper around React.lazy that automatically reloads the page
 * if a chunk fails to load (e.g., due to a new deployment).
 */
export const lazyWithRetry = (
  componentImport: () => Promise<{ default: ComponentType<any> }>
): LazyExoticComponent<ComponentType<any>> => {
  return lazy(async () => {
    try {
      const component = await componentImport();
      // If load is successful, clear any previous retry flags 
      // (optional, depending on strictness of the 10s window)
      return component;
    } catch (error) {
      // Check if we've already forced a reload recently to avoid infinite loops
      const storageKey = 'page-has-been-force-refreshed';
      const lastReload = window.sessionStorage.getItem(storageKey);
      const timeSinceLastReload = lastReload ? Date.now() - parseInt(lastReload) : null;
      
      // 10-second cooling period
      const RELOAD_THRESHOLD_MS = 10000;

      if (!timeSinceLastReload || timeSinceLastReload > RELOAD_THRESHOLD_MS) {
        console.warn('Chunk load failed. Attempting auto-recovery via page reload...', error);
        
        // Set the flag with the current timestamp
        window.sessionStorage.setItem(storageKey, Date.now().toString());
        
        // Force reload the page exactly one time
        window.location.reload();
        
        // Return a never-resolving promise to pause rendering while the page reloads
        return new Promise(() => {});
      }

      // If we are here, we've already tried reloading and it failed again.
      // Throw the error naturally to trigger the ErrorBoundary.
      throw error;
    }
  });
};
```

### integrating into the Router

Usage is seamless. In `router/index.tsx`, I simply swapped out the native `lazy` for my new wrapper:

```typescript
// Before
// const ServiceRequestsPage = lazy(() => import('./pages/ServiceRequestsPage'));

// After
import { lazyWithRetry } from '../utils/lazyWithRetry';

const ServiceRequestsPage = lazyWithRetry(() => import('./pages/ServiceRequestsPage'));
```

## The Results

This small change has significantly improved resilience:

*   **Seamless Updates**: Users getting caught between deployments now experience a brief page reload instead of a crash.
*   **Developer Experience**: HMR glitches that previously required a manual refresh are now handled automatically.
*   **Safety**: The `sessionStorage` and time-check logic ensures we never trap the user in an infinite refresh loop.

It's a "set and forget" improvement that respects the user's flow while robustly handling the realities of modern web deployment strategies.
