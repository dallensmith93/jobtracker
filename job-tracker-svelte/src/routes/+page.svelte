<script lang="ts">
  import { onMount } from "svelte";
  import { fly, fade, scale } from "svelte/transition";

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

  function statusClass(s: Status) {
    return s === "Applied"
      ? "applied"
      : s === "Interviewing"
      ? "interviewing"
      : s === "Offer"
      ? "offer"
      : "rejected";
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
        <input bind:value={company} placeholder="company" autocomplete="organization" />
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
      <div class="empty" in:scale={{ duration: 180 }} out:fade={{ duration: 120 }}>
        <div class="emptyIcon">✨</div>
        <h3>No results</h3>
        <p>Try changing the filter, clearing search, or add your first application.</p>
      </div>
    {:else}
      {#each filtered as app (app.id)}
        <article class="item" in:fly={{ y: 10, duration: 180 }} out:fade={{ duration: 120 }}>
          <div class="top">
            <div>
              <h3>{app.company}</h3>
              <p class="meta"><strong>{app.role}</strong> • {app.date}</p>
              <div class={"pill " + statusClass(app.status)}>{app.status}</div>
            </div>

            <div class="right">
              <select
                class={"status " + statusClass(app.status)}
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
    font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Helvetica, Arial, "Apple Color Emoji",
      "Segoe UI Emoji";
    color: rgba(255, 255, 255, 0.92);
    background: radial-gradient(1000px 700px at 15% 15%, rgba(124, 92, 255, 0.35), transparent 60%),
      radial-gradient(900px 650px at 85% 25%, rgba(61, 220, 255, 0.28), transparent 55%),
      radial-gradient(1000px 700px at 55% 90%, rgba(255, 115, 180, 0.18), transparent 60%),
      #070a14;
  }

  :global(*) {
    box-sizing: border-box;
  }

  .wrap {
    max-width: 980px;
    margin: 0 auto;
    padding: 34px 16px 56px;
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
    font-size: 36px;
    letter-spacing: -0.03em;
    line-height: 1.05;
  }

  .sub {
    margin: 8px 0 0;
    opacity: 0.82;
  }

  .chip {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    padding: 10px 14px;
    border-radius: 999px;
    color: rgba(255, 255, 255, 0.9);
    text-decoration: none;
    border: 1px solid rgba(255, 255, 255, 0.14);
    background: rgba(255, 255, 255, 0.06);
    backdrop-filter: blur(10px);
    box-shadow: 0 14px 30px rgba(0, 0, 0, 0.22);
    transition: transform 120ms ease, background 120ms ease, border 120ms ease;
  }

  .chip:hover {
    transform: translateY(-1px);
    background: rgba(255, 255, 255, 0.09);
    border-color: rgba(255, 255, 255, 0.22);
  }

  .card {
    background: rgba(255, 255, 255, 0.06);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 18px;
    padding: 16px;
    backdrop-filter: blur(14px);
    box-shadow: 0 18px 50px rgba(0, 0, 0, 0.35);
  }

  h2 {
    margin: 0 0 12px;
    font-size: 18px;
    letter-spacing: -0.01em;
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

  input,
  select,
  textarea {
    width: 100%;
    padding: 11px 12px;
    border-radius: 14px;
    border: 1px solid rgba(255, 255, 255, 0.14);
    background: rgba(10, 14, 30, 0.45);
    color: rgba(255, 255, 255, 0.92);
    outline: none;
    transition: border 120ms ease, transform 120ms ease, background 120ms ease;
  }

  input::placeholder,
  textarea::placeholder {
    color: rgba(255, 255, 255, 0.5);
  }

  input:focus,
  select:focus,
  textarea:focus {
    border-color: rgba(160, 160, 255, 0.55);
    background: rgba(10, 14, 30, 0.62);
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
    border-radius: 14px;
    border: 1px solid rgba(255, 255, 255, 0.16);
    background: linear-gradient(180deg, rgba(255, 255, 255, 0.14), rgba(255, 255, 255, 0.08));
    color: rgba(255, 255, 255, 0.92);
    padding: 11px 14px;
    cursor: pointer;
    backdrop-filter: blur(10px);
    box-shadow: 0 12px 24px rgba(0, 0, 0, 0.22);
    transition: transform 120ms ease, background 120ms ease, border 120ms ease;
  }

  button:hover {
    transform: translateY(-1px);
    background: linear-gradient(180deg, rgba(255, 255, 255, 0.18), rgba(255, 255, 255, 0.1));
    border-color: rgba(255, 255, 255, 0.22);
  }

  button:active {
    transform: translateY(0px);
  }

  .ghost {
    background: rgba(255, 255, 255, 0.06);
    box-shadow: none;
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
    background: rgba(255, 255, 255, 0.06);
    box-shadow: none;
  }

  .filters button span {
    font-size: 12px;
    padding: 2px 9px;
    border-radius: 999px;
    background: rgba(255, 255, 255, 0.12);
  }

  .filters button.active {
    border-color: rgba(180, 180, 255, 0.45);
    background: rgba(160, 160, 255, 0.14);
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
    background: rgba(255, 255, 255, 0.055);
    border: 1px solid rgba(255, 255, 255, 0.12);
    border-radius: 18px;
    padding: 14px;
    backdrop-filter: blur(14px);
    box-shadow: 0 16px 40px rgba(0, 0, 0, 0.28);
    transition: transform 140ms ease, border 140ms ease, background 140ms ease;
  }

  .item:hover {
    transform: translateY(-2px);
    border-color: rgba(255, 255, 255, 0.18);
    background: rgba(255, 255, 255, 0.075);
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

  .pill {
    display: inline-flex;
    align-items: center;
    padding: 6px 10px;
    border-radius: 999px;
    font-size: 12px;
    margin-top: 10px;
    border: 1px solid rgba(255, 255, 255, 0.14);
    background: rgba(255, 255, 255, 0.06);
    width: fit-content;
  }

  .applied {
    border-color: rgba(120, 170, 255, 0.35);
    background: rgba(120, 170, 255, 0.14);
  }
  .interviewing {
    border-color: rgba(255, 210, 120, 0.35);
    background: rgba(255, 210, 120, 0.14);
  }
  .offer {
    border-color: rgba(120, 255, 190, 0.35);
    background: rgba(120, 255, 190, 0.14);
  }
  .rejected {
    border-color: rgba(255, 120, 160, 0.35);
    background: rgba(255, 120, 160, 0.14);
  }

  .status {
    padding: 9px 10px;
  }

  .danger {
    border-color: rgba(255, 120, 160, 0.35);
    background: rgba(255, 120, 160, 0.16);
    box-shadow: none;
  }

  .danger:hover {
    background: rgba(255, 120, 160, 0.24);
  }

  .link,
  .notes {
    margin: 10px 0 0;
    opacity: 0.92;
    word-break: break-word;
  }

  a {
    color: rgba(190, 200, 255, 0.95);
  }

  .empty {
    padding: 22px;
    border-radius: 18px;
    border: 1px dashed rgba(255, 255, 255, 0.16);
    background: rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(14px);
    text-align: center;
  }

  .emptyIcon {
    font-size: 34px;
    margin-bottom: 6px;
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
    .right {
      flex-direction: column;
      align-items: stretch;
    }
  }
</style>
