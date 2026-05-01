<script>
  const WEBHOOK_BASE = 'http://172.10.30.15:8090';
  const APPS_URL = 'http://172.10.30.15:4174/apps.json';
  const TWO_MINUTES = 2 * 60 * 1000; // 120000ms

  let projects = $state([]);
  let loading = $state(true);
  let error = $state(null);
  let hookStates = $state({});
  let lastDeployTime = $state({}); // timestamp del último deploy exitoso

  async function loadProjects() {
    try {
      const res = await fetch(APPS_URL);
      projects = await res.json();
    } catch (e) {
      error = 'No se pudo cargar la lista de proyectos';
    } finally {
      loading = false;
    }
  }

  loadProjects();

  function webhookUrl(project) {
    return WEBHOOK_BASE + project.deploy_url;
  }

  function appHref(project) {
    if (!project.app_url) return null;
    return project.stack === 'python' ? project.app_url + '/docs' : project.app_url;
  }

  function publicHref(project) {
    if (!project.public_url) return null;
    return project.stack === 'python' ? project.public_url + '/docs' : project.public_url;
  }

  function appPort(project) {
    if (!project.app_url) return null;
    try {
      const port = new URL(project.app_url).port;
      return port || null;
    } catch (_) {
      return null;
    }
  }

  async function triggerWebhook(event, project) {
    event.preventDefault();
    
    // Si está cargando, no hacer nada
    if (hookStates[project.id] === 'loading') return;
    
    // Si está en "success", verificar si han pasado 2 minutos
    if (hookStates[project.id] === 'success') {
      const timePassed = Date.now() - (lastDeployTime[project.id] || 0);
      if (timePassed >= TWO_MINUTES) {
        // Han pasado más de 2 minutos, resetear a "idle"
        hookStates[project.id] = 'idle';
        return;
      }
      // Si hace menos de 2 minutos, permitir re-deploy (continuar más abajo)
    }
    
    hookStates[project.id] = 'loading';
    try {
      const minDelay = new Promise(r => setTimeout(r, 3000));
      const request = fetch(webhookUrl(project), { mode: 'no-cors' });
      await Promise.all([minDelay, request]);
      hookStates[project.id] = 'success';
      lastDeployTime[project.id] = Date.now();
    } catch (_) {
      hookStates[project.id] = 'error';
    }
  }
</script>

<div class="app">
  <div class="container">
    <header>
      <h1>Buzzword IA DevOps</h1>
      <p>Panel de despliegue de servicios</p>
    </header>

    <div class="card">
      {#if loading}
        <p class="loading-msg"><span class="spinner"></span> Cargando proyectos...</p>
      {:else if error}
        <p class="error-msg">{error}</p>
      {:else}
        <table>
          <thead>
            <tr>
              <th>Proyecto</th>
              <th>Stack</th>
              <th>Branch</th>
              <th></th>
              <th>Tipo de App</th>
              <th>Puerto</th>
              <th>URL Webhook</th>
              <th>Estado</th>
            </tr>
          </thead>
          <tbody>
            {#each projects as project}
              <tr>
                <td class="project-name">{project.name}</td>
                <td>
                  <span class="badge {project.stack === 'svelte' ? 'badge-svelte' : 'badge-python'}">
                    {project.stack}
                  </span>
                </td>
                <td class="branch">{project.branch}</td>
                <td class="app-cell">
                  {#if project.public_url}
                    <a
                      href={publicHref(project)}
                      class="public-link"
                      target="_blank"
                      rel="noopener noreferrer"
                      title="Dominio público: {publicHref(project)}"
                      aria-label="Abrir dominio público de {project.name}"
                    >
                      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"/>
                        <polyline points="14 2 14 8 20 8"/>
                        <line x1="8" y1="13" x2="16" y2="13"/>
                        <line x1="8" y1="17" x2="16" y2="17"/>
                        <line x1="8" y1="9" x2="10" y2="9"/>
                      </svg>
                    </a>
                  {:else}
                    <span class="muted">—</span>
                  {/if}
                </td>
                <td class="app-chip-cell">
                  {#if project.app_url}
                    <a
                      href={appHref(project)}
                      class="app-link"
                      target="_blank"
                      rel="noopener noreferrer"
                    >
                      {project.stack === 'python' ? 'API ↗' : 'Webapp ↗'}
                    </a>
                  {:else}
                    <span class="muted">—</span>
                  {/if}
                </td>
                <td class="port-cell">
                  {#if appPort(project)}
                    <span class="port">{appPort(project)}</span>
                  {:else}
                    <span class="muted">—</span>
                  {/if}
                </td>
                <td class="webhook-cell">
                  <button
                    type="button"
                    class="deploy-btn"
                    onclick={(e) => triggerWebhook(e, project)}
                    disabled={hookStates[project.id] === 'loading'}
                    aria-label="Desplegar {project.name}"
                    title="Desplegar {project.name}"
                  >
                    🚀
                  </button>
                  <a
                    href={webhookUrl(project)}
                    class="webhook-link"
                    onclick={(e) => triggerWebhook(e, project)}
                  >
                    {webhookUrl(project)}
                  </a>
                </td>
                <td class="status-cell">
                  {#if hookStates[project.id] === 'loading'}
                    <span class="status loading">
                      <span class="spinner"></span> Desplegando...
                    </span>
                  {:else if hookStates[project.id] === 'success'}
                    <span class="status success">✓ Desplegado</span>
                  {:else if hookStates[project.id] === 'error'}
                    <span class="status error">✗ Error</span>
                  {:else}
                    <span class="status idle">— Listo</span>
                  {/if}
                </td>
              </tr>
            {/each}
          </tbody>
        </table>
      {/if}
    </div>

    <footer>
      <p>Flujo: <span class="flow">git pull → dependencias → pm2 restart</span></p>
    </footer>
  </div>
</div>

<style>
  :global(*, *::before, *::after) {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  :global(html, body) {
    width: 100%;
    height: 100%;
    font-family: 'Segoe UI', system-ui, sans-serif;
  }

  .app {
    width: 100%;
    min-height: 100vh;
    padding: 48px 0;
  }

  .container {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 32px;
  }

  header {
    text-align: left;
    padding: 0 40px;
    display: flex;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
  }

  h1 {
    font-size: 1.8rem;
    font-weight: 700;
    color: #1a3a5c;
    letter-spacing: -0.5px;
    margin: 0;
  }

  header p {
    color: #5a7a9a;
    margin: 0;
    margin-left: 12px;
    flex-basis: 100%;
    font-size: 1rem;
  }

  .loading-msg, .error-msg {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 24px 20px;
    font-size: 0.9rem;
  }

  .loading-msg {
    color: #2563eb;
  }

  .error-msg {
    color: #b91c1c;
  }

  .branch {
    font-family: 'Consolas', 'Courier New', monospace;
    font-size: 0.82rem;
    color: #6b7280;
  }

  .card {
    overflow: hidden;
  }

  table {
    width: 100%;
    border-collapse: collapse;
    border: none;
  }

  thead tr {
    background: linear-gradient(90deg, #2563eb, #38bdf8);
  }

  th {
    padding: 14px 20px;
    text-align: left;
    font-size: 0.78rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    color: #ffffff;
    text-transform: uppercase;
    border: none;
  }

  tbody tr {
    transition: background 0.15s;
  }

  tbody tr:last-child {
  }

  tbody tr:hover {
    background: rgba(37, 99, 235, 0.04);
  }

  td {
    padding: 16px 20px;
    vertical-align: middle;
    color: #334155;
    font-size: 0.9rem;
    border: none;
    text-align: left;
  }

  .project-name {
    font-family: 'Consolas', 'Courier New', monospace;
    font-weight: 600;
    color: #1e3a5f;
    font-size: 0.88rem;
    text-align: left;
  }

  .badge {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 70px;
    height: 28px;
    padding: 0 10px;
    border-radius: 20px;
    font-size: 0.76rem;
    font-weight: 600;
    letter-spacing: 0.03em;
    white-space: nowrap;
  }

  .badge-python {
    background: #dcfce7;
    color: #166534;
    border: 1px solid #86efac;
  }

  .badge-svelte {
    background: #ffedd5;
    color: #1e40af;
    border: 1px solid #fed7aa;
  }

  .app-cell {
    white-space: nowrap;
    text-align: center;
    width: 1%;
  }

  .app-chip-cell {
    white-space: nowrap;
  }

  .public-link {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    color: #7c3aed;
    border: 1px solid #ddd6fe;
    background: #f5f3ff;
    border-radius: 6px;
    text-decoration: none;
    transition: background 0.15s, color 0.15s, border-color 0.15s, transform 0.1s;
  }

  .public-link svg {
    width: 16px;
    height: 16px;
  }

  .public-link:hover {
    background: #ede9fe;
    color: #6d28d9;
    border-color: #c4b5fd;
    transform: scale(1.05);
  }

  .app-link {
    display: inline-flex;
    align-items: center;
    gap: 4px;
    font-size: 0.82rem;
    font-weight: 500;
    color: #0d9488;
    text-decoration: none;
    padding: 4px 12px;
    border: 1px solid #5eead4;
    border-radius: 20px;
    background: #f0fdfa;
    transition: background 0.15s, color 0.15s, border-color 0.15s;
  }

  .app-link:hover {
    background: #ccfbf1;
    color: #0f766e;
    border-color: #2dd4bf;
  }

  .muted {
    color: #cbd5e1;
    font-size: 0.9rem;
  }

  .port-cell {
    white-space: nowrap;
  }

  .port {
    font-family: 'Consolas', 'Courier New', monospace;
    font-size: 0.82rem;
    color: #475569;
    background: #f1f5f9;
    padding: 3px 10px;
    border-radius: 6px;
    border: 1px solid #e2e8f0;
  }

  .webhook-cell {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .deploy-btn {
    flex-shrink: 0;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    padding: 0;
    border: 1px solid #93c5fd;
    background: #eff6ff;
    border-radius: 8px;
    cursor: pointer;
    font-size: 1rem;
    line-height: 1;
    transition: background 0.15s, border-color 0.15s, transform 0.1s;
  }

  .deploy-btn:hover:not(:disabled) {
    background: #dbeafe;
    border-color: #2563eb;
    transform: scale(1.05);
  }

  .deploy-btn:active:not(:disabled) {
    transform: scale(0.95);
  }

  .deploy-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .webhook-link {
    font-family: 'Consolas', 'Courier New', monospace;
    font-size: 0.78rem;
    color: #2563eb;
    text-decoration: none;
    word-break: break-all;
    border-bottom: 1px dashed #93c5fd;
    transition: color 0.15s, border-color 0.15s;
  }

  .webhook-link:hover {
    color: #1d4ed8;
    border-bottom-color: #1d4ed8;
  }

  .status-cell {
    white-space: nowrap;
    min-width: 130px;
  }

  .status {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    font-size: 0.82rem;
    font-weight: 500;
    padding: 4px 10px;
    border-radius: 20px;
  }

  .status.idle {
    color: #94a3b8;
  }

  .status.loading {
    color: #2563eb;
    background: #eff6ff;
    border: 1px solid #bfdbfe;
  }

  .status.success {
    color: #15803d;
    background: #f0fdf4;
    border: 1px solid #bbf7d0;
  }

  .status.error {
    color: #b91c1c;
    background: #fef2f2;
    border: 1px solid #fecaca;
  }

  .spinner {
    width: 13px;
    height: 13px;
    border: 2px solid #bfdbfe;
    border-top-color: #2563eb;
    border-radius: 50%;
    animation: spin 0.7s linear infinite;
    flex-shrink: 0;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  footer {
    text-align: center;
    padding: 0 40px;
    color: #64748b;
    font-size: 0.85rem;
  }

  .flow {
    font-family: 'Consolas', 'Courier New', monospace;
    background: rgba(37, 99, 235, 0.08);
    padding: 2px 8px;
    border-radius: 4px;
    color: #1e40af;
  }
</style>
