<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket, each with the ticket it now has of its own.
  // Counted the way the desk counts (by tenant, as CauseTable does). `shift` is the desk's own read
  // (desk-notes.json, oneDayMove): those reports stay counted where they're filed, and get a row too.
  type Ticket = { url: string; state: string };
  let { ticket, tickets = {} }: { ticket: number; tickets?: Record<string, Ticket> } = $props();

  const rows = $derived((reports as Report[]).filter((r) => r.mergedInto === ticket));
  const merged = $derived(uniqueById(rows));
  const groups = $derived(groupByCause(merged, tenantOf));
  const shifted = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const problems = $derived([
    ...groups.map((g) => ({ name: g.cause as string, reports: g.reports, taggedAs: g.taggedAs, also: '' })),
    {
      name: 'shift',
      reports: shifted,
      taggedAs: shifted.reduce<Record<string, number>>((n, r) => ({ ...n, [r.meta.intakeTag]: (n[r.meta.intakeTag] ?? 0) + 1 }), {}),
      also: `also counted under ${[...new Set(shifted.map((r) => groups.find((g) => g.reports.includes(r))?.cause))].join(', ')}`,
    },
  ]);
  const tags = (taggedAs: Record<string, number>) =>
    Object.entries(taggedAs)
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
  const days = (rs: Report[]) => rs.map((r) => r.receivedAt.slice(0, 10)).sort();
  const issue = (url: string) => url.replace(/^.*\/issues\//, '#');
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports in #{ticket} ·
  <strong>{problems.length}</strong>
  {problems.length === 1 ? 'problem' : 'problems'}
  <span class="raw">({rows.length} rows in the export; {rows.length - merged.length} came twice and count once)</span>
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>intake tagged as</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each problems as p (p.name)}
      {@const t = tickets[p.name]}
      {@const seen = days(p.reports)}
      <tr class:open={!t}>
        <td><code>{p.name}</code></td>
        <td>{p.reports.length}{#if p.also}<span class="also">{p.also}</span>{/if}</td>
        <td class="tags">{tags(p.taggedAs)}</td>
        <td>{new Set(p.reports.map((r) => r.tenant)).size}</td>
        <td>{seen[0]} → {seen[seen.length - 1]}</td>
        <td>
          {#if t}
            <a href={t.url}>{issue(t.url)}</a> <span class="state">{t.state}</span>
          {:else}
            no ticket yet
          {/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .raw, .also, .state { color: var(--muted); font-size: 0.85em; }
  .also { display: block; }
  .tags { color: var(--muted); font-size: 0.85em; }
  .open td { background: var(--warn-bg); }
  .open td:last-child { color: var(--warn); font-weight: 600; }
</style>
