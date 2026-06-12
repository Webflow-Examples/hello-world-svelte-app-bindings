<script lang="ts">
  import * as Sentry from '@sentry/svelte';

  /**
   * Join the app's base mount path with an API path (same helper as
   * BindingsStatus). `import.meta.env.BASE_URL` is populated by Vite from
   * the `base` config (wired to COSMIC_MOUNT_PATH at build time).
   */
  function buildAppUrl(path: string): string {
    const base = (import.meta.env.BASE_URL ?? '').replace(/\/+$/, '');
    const cleanPath = path.replace(/^\/+/, '');
    return `${base}/${cleanPath}`;
  }

  const PING_INTERVAL_MS = 30_000;

  let lastPing = $state('waiting…');
  let pingCount = $state(0);

  /**
   * Calls /api/sentry-ping on load and every 30s. Each round trip produces:
   *  - a server-side Sentry log (emitted inside the worker route), and
   *  - a browser-side Sentry log (emitted here after the response).
   * The buttons trigger a client / server error to verify error capture.
   */
  async function ping() {
    const startedAt = Date.now();
    try {
      const res = await fetch(buildAppUrl('api/sentry-ping'));
      const body = (await res.json()) as { requestedAt?: string };

      Sentry.logger.info('client: sentry-ping completed', {
        status: res.status,
        durationMs: Date.now() - startedAt,
        serverTime: body.requestedAt ?? 'unknown',
      });

      pingCount += 1;
      lastPing = new Date().toLocaleTimeString();
    } catch (err) {
      Sentry.logger.error('client: sentry-ping failed', {
        durationMs: Date.now() - startedAt,
        message: err instanceof Error ? err.message : String(err),
      });
      lastPing = 'failed — see console / Sentry';
    }
  }

  $effect(() => {
    // Deferred so the first ping (and its state writes) runs outside the effect body.
    const initial = setTimeout(() => void ping(), 0);
    const id = setInterval(() => void ping(), PING_INTERVAL_MS);
    return () => {
      clearTimeout(initial);
      clearInterval(id);
    };
  });
</script>

{#if !import.meta.env.VITE_SENTRY_DSN}
  <section class="wf-bindings" aria-label="Sentry status">
    <p class="wf-subtitle">
      Sentry is not configured — set <code>VITE_SENTRY_DSN</code> (browser) and
      <code>SENTRY_DSN</code> (worker) in your environment variables to enable it.
    </p>
  </section>
{:else}
  <section class="wf-bindings" aria-label="Sentry status">
    <p class="wf-subtitle">
      Sentry demo · pings <code>/api/sentry-ping</code> every {PING_INTERVAL_MS / 1000}s · pings
      sent: {pingCount} · last: {lastPing}
    </p>
    <div class="wf-cta">
      <button
        class="wf-btn wf-btn-ghost"
        type="button"
        onclick={() => {
          throw new Error('sentry-ping: intentional test error (browser)');
        }}
      >
        Trigger client error
      </button>
      <button
        class="wf-btn wf-btn-ghost"
        type="button"
        onclick={() => void fetch(buildAppUrl('api/sentry-ping?error=1'))}
      >
        Trigger server error
      </button>
    </div>
  </section>
{/if}
