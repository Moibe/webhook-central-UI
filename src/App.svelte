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
                <td>
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
