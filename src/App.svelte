<script>
  const projects = [
    {
      name: 'mide-chatbot',
      stack: 'Python',
      url: 'http://172.10.30.15:8090/hooks/despliegue-mide-chatbot',
    },
    {
      name: 'document_ai',
      stack: 'Python',
      url: 'http://172.10.30.15:8090/hooks/despliegue-document-ai',
    },
    {
      name: 'notificaciones_twilio',
      stack: 'Python',
      url: 'http://172.10.30.15:8090/hooks/despliegue-notificaciones-twilio',
    },
    {
      name: 'svelte_mide',
      stack: 'Svelte',
      url: 'http://172.10.30.15:8090/hooks/despliegue-svelte-mide',
    },
  ];

  let hookStates = $state({});

  async function triggerWebhook(event, project) {
    event.preventDefault();
    if (hookStates[project.name] === 'loading') return;
    hookStates[project.name] = 'loading';
    try {
      await fetch(project.url, { mode: 'no-cors' });
      hookStates[project.name] = 'success';
    } catch (_) {
      hookStates[project.name] = 'error';
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
      <table>
        <thead>
          <tr>
            <th>Proyecto</th>
            <th>Stack</th>
            <th>URL Webhook</th>
            <th>Estado</th>
          </tr>
        </thead>
        <tbody>
          {#each projects as project}
            <tr>
              <td class="project-name">{project.name}</td>
              <td>
                <span class="badge {project.stack.startsWith('Svelte') ? 'badge-svelte' : 'badge-python'}">
                  {project.stack}
                </span>
              </td>
              <td>
                <a
                  href={project.url}
                  class="webhook-link"
                  onclick={(e) => triggerWebhook(e, project)}
                >
                  {project.url}
                </a>
              </td>
              <td class="status-cell">
                {#if hookStates[project.name] === 'loading'}
                  <span class="status loading">
                    <span class="spinner"></span> Desplegando...
                  </span>
                {:else if hookStates[project.name] === 'success'}
                  <span class="status success">✓ Desplegado</span>
                {:else if hookStates[project.name] === 'error'}
                  <span class="status error">✗ Error</span>
                {:else}
                  <span class="status idle">— Listo</span>
                {/if}
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
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
