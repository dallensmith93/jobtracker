<script lang="ts">
  import { onMount } from "svelte";

  type Status = "Applied" | "Interviewing" | "Offer" | "Rejected";

  type JobApp = {
    id: string;
    company: string;
    role: string;
    status: Status;
    date: string; // YYYY-MM-DD
    link: string;
    notes: string;
    createdAt: number;
  };

  const STORAGE_KEY = "job-tracker-svelte:v1";

  let company = "";
  let role = "";
  let status: Status = "Applied";
  let date = new Date().toISOString().slice(0, 10);
  let link = "";
  let notes = "";

  let apps: JobApp[] = [];
  let filter: "All" | Status = "All";
  let query = "";

  function uid() {
    return crypto.randomUUID ? crypto.randomUUID() : String(Math.random()).slice(2);
  }

  function save() {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(apps));
  }

  function load(): JobApp[] {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return [];
    try {
      return JSON.parse(raw) as JobApp[];
    } catch {
      return [];
    }
  }

  onMount(() => {
    apps = load().sort((a, b) => b.createdAt - a.createdAt);
  });

  function addApp() {
    const cleanCompany = company.trim();
    const cleanRole = role.trim();

    if (!cleanCompany || !cleanRole) return;

    const item: JobApp = {
      id: uid(),
      company: cleanCompany,
      role: cleanRole,
      status,
      date,
      link: link.trim(),
      notes: notes.trim(),
      createdAt: Date.now()
    };

    apps = [item, ...apps];
    save();

    company = "";
    role = "";
    status = "Applied";
    link = "";
    notes = "";
  }

  function removeApp(id: string) {
    apps = apps.filter((a) => a.id !== id);
    save();
  }

  function setStatus(id: string, next: Status) {
    apps = apps.map((a) => (a.id === id ? { ...a, status: next } : a));
    save();
  }

  $: filtered = apps
    .filter((a) => (filter === "All" ? true : a.status === filter))
    .filter((a) => {
      const q = query.trim().toLowerCase();
      if (!q) return true;
      return (
        a.company.toLowerCase().includes(q) ||
        a.role.toLowerCase().includes(q) ||
        a.notes.toLowerCase().includes(q)
      );
    });

  $: counts = {
    All: apps.length,
    Applied: apps.filter((a) => a.status === "Applied").length,
    Interviewing: apps.filter((a) => a.status === "Interviewing").length,
    Offer: apps.filter((a) => a.status === "Offer").length,
    Rejected: apps.filter((a) => a.status === "Rejected").length
  };
</script>

<svelte:head>
  <title>Job Tracker (Svelte)</title>
</svelte:head>

<main class="wrap">
  <header class="header">
    <div>
      <h1>Job Tracker</h1>
      <p class="sub">SvelteKit + TypeScript • CRUD, filters, and localStorage persistence.</p>
    </div>

    <a class="chip" href="https://svelte.dev" target="_blank" rel="noreferrer">Built with Svelte</a>
  </header>

  <section class="card">
    <h2>Add application</h2>

    <form class="form" on:submit|preventDefault={addApp}>
      <label>
        <span>Company *</span>
        <input bind:value={company} placeholder="Company" autocomplete="organization" />
      </label>

      <label>
        <span>Role *</span>
        <input bind:value={role} placeholder="Software Engineer" autocomplete="organization-title" />
      </label>

      <label>
        <span>Status</span>
        <select bind:value={status}>
          <option>Applied</option>
          <option>Interviewing</option>
          <option>Offer</option>
          <option>Rejected</option>
        </select>
      </label>

      <label>
        <span>Date</span>
        <input type="date" bind:value={date} />
      </label>

      <label class="full">
        <span>Link</span>
        <input bind:value={link} placeholder="https://..." />
      </label>

      <label class="full">
        <span>Notes</span>
        <textarea bind:value={notes} rows="3" placeholder="Recruiter name, follow-up date, take-home, etc." />
      </label>

      <div class="actions full">
        <button type="submit">Add</button>
        <button
          type="button"
          class="ghost"
          on:click={() => {
            company = "";
            role = "";
            status = "Applied";
            link = "";
            notes = "";
          }}
        >
          Clear
        </button>
      </div>

      <p class="hint full">* Required fields</p>
    </form>
  </section>

  <section class="toolbar">
    <div class="filters">
      <button class:active={filter === "All"} on:click={() => (filter = "All")}>
        All <span>{counts.All}</span>
      </button>
      <button class:active={filter === "Applied"} on:click={() => (filter = "Applied")}>
        Applied <span>{counts.Applied}</span>
      </button>
      <button class:active={filter === "Interviewing"} on:click={() => (filter = "Interviewing")}>
        Interviewing <span>{counts.Interviewing}</span>
      </button>
      <button class:active={filter === "Offer"} on:click={() => (filter = "Offer")}>
        Offer <span>{counts.Offer}</span>
      </button>
      <button class:active={filter === "Rejected"} on:click={() => (filter = "Rejected")}>
        Rejected <span>{counts.Rejected}</span>
      </button>
    </div>

    <input class="search" bind:value={query} placeholder="Search company, role, notes..." />
  </section>

  <section class="list">
    {#if filtered.length === 0}
      <div class="empty">
        <h3>No results</h3>
        <p>Try changing the filter or adding your first application.</p>
      </div>
    {:else}
      {#each filtered as app (app.id)}
        <article class="item">
          <div class="top">
            <div>
              <h3>{app.company}</h3>
              <p class="meta"><strong>{app.role}</strong> • {app.date}</p>
            </div>

            <div class="right">
              <select
                class="status"
                value={app.status}
                on:change={(e) =>
                  setStatus(app.id, (e.currentTarget as HTMLSelectElement).value as Status)
                }
              >
                <option>Applied</option>
                <option>Interviewing</option>
                <option>Offer</option>
                <option>Rejected</option>
              </select>

              <button class="danger" on:click={() => removeApp(app.id)}>Delete</button>
            </div>
          </div>

          {#if app.link}
            <p class="link"><a href={app.link} target="_blank" rel="noreferrer">{app.link}</a></p>
          {/if}

          {#if app.notes}
            <p class="notes">{app.notes}</p>
          {/if}
        </article>
      {/each}
    {/if}
  </section>

  <footer class="footer">
    <p>Tip: Uses <code>localStorage</code>, so your data stays after refresh (on this browser).</p>
  </footer>
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji", "Segoe UI Emoji";
    background: #0b1020;
    color: #e8ebff;
  }

  .wrap {
    max-width: 980px;
    margin: 0 auto;
    padding: 28px 16px 48px;
  }

  .header {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 18px;
  }

  h1 {
    margin: 0;
    font-size: 34px;
    letter-spacing: -0.02em;
  }

  .sub {
    margin: 6px 0 0;
    opacity: 0.8;
  }

  .chip {
    display: inline-flex;
    padding: 10px 12px;
    border: 1px solid rgba(232, 235, 255, 0.18);
    border-radius: 999px;
    color: #e8ebff;
    text-decoration: none;
    background: rgba(255, 255, 255, 0.04);
  }

  .card {
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid rgba(232, 235, 255, 0.12);
    border-radius: 16px;
    padding: 16px;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.25);
  }

  h2 {
    margin: 0 0 12px;
    font-size: 18px;
  }

  .form {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 12px;
  }

  label span {
    display: block;
    font-size: 12px;
    opacity: 0.85;
    margin-bottom: 6px;
  }

  input, select, textarea {
    width: 100%;
    box-sizing: border-box;
    padding: 10px 12px;
    border-radius: 12px;
    border: 1px solid rgba(232, 235, 255, 0.16);
    background: rgba(255, 255, 255, 0.03);
    color: #e8ebff;
    outline: none;
  }

  input::placeholder, textarea::placeholder {
    color: rgba(232, 235, 255, 0.55);
  }

  input:focus, select:focus, textarea:focus {
    border-color: rgba(232, 235, 255, 0.38);
  }

  .full {
    grid-column: 1 / -1;
  }

  .actions {
    display: flex;
    gap: 10px;
    align-items: center;
  }

  button {
    border-radius: 12px;
    border: 1px solid rgba(232, 235, 255, 0.18);
    background: rgba(232, 235, 255, 0.14);
    color: #e8ebff;
    padding: 10px 12px;
    cursor: pointer;
  }

  button:hover {
    background: rgba(232, 235, 255, 0.2);
  }

  .ghost {
    background: transparent;
  }

  .hint {
    margin: 0;
    font-size: 12px;
    opacity: 0.75;
  }

  .toolbar {
    display: flex;
    gap: 12px;
    align-items: center;
    justify-content: space-between;
    margin: 16px 0;
    flex-wrap: wrap;
  }

  .filters {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
  }

  .filters button {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    background: rgba(255, 255, 255, 0.03);
  }

  .filters button span {
    font-size: 12px;
    padding: 2px 8px;
    border-radius: 999px;
    background: rgba(232, 235, 255, 0.12);
  }

  .filters button.active {
    background: rgba(232, 235, 255, 0.18);
  }

  .search {
    min-width: 240px;
    flex: 1;
  }

  .list {
    display: grid;
    gap: 12px;
  }

  .item {
    background: rgba(255, 255, 255, 0.03);
    border: 1px solid rgba(232, 235, 255, 0.12);
    border-radius: 16px;
    padding: 14px;
  }

  .top {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    align-items: flex-start;
  }

  .meta {
    margin: 6px 0 0;
    opacity: 0.85;
  }

  .right {
    display: flex;
    gap: 10px;
    align-items: center;
  }

  .status {
    padding: 8px 10px;
  }

  .danger {
    border-color: rgba(255, 120, 120, 0.35);
    background: rgba(255, 120, 120, 0.18);
  }

  .danger:hover {
    background: rgba(255, 120, 120, 0.26);
  }

  .link, .notes {
    margin: 10px 0 0;
    opacity: 0.9;
    word-break: break-word;
  }

  a {
    color: #b9c2ff;
  }

  .empty {
    padding: 18px;
    border-radius: 16px;
    border: 1px dashed rgba(232, 235, 255, 0.18);
    opacity: 0.9;
  }

  .footer {
    margin-top: 22px;
    opacity: 0.75;
    font-size: 12px;
  }

  @media (max-width: 680px) {
    .form {
      grid-template-columns: 1fr;
    }
  }
</style>
