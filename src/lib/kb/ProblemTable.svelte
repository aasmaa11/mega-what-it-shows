<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // `tickets`: problem → the ticket that now holds it.
  let { ticket, tickets }: { ticket: number; tickets: Record<string, string> } = $props();

  // The desk's view: each report counted where it is filed (by cause), and the desk-noted
  // one-day moves counted again as their own problem, as CauseTable footnotes them.
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const oneDay = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const rows = $derived([
    ...groups.map((g) => ({ problem: g.cause, reports: g.reports, filedUnder: null as string | null })),
    { problem: 'one-day', reports: oneDay, filedUnder: groups.find((g) => readAs(g, notes.oneDayMove))?.cause ?? null },
  ]);
  const seen = (rs: Report[]) => {
    const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
    return `${days[0]} → ${days[days.length - 1]}`;
  };
  const ticketNo = (url: string) => `#${url.split('/').pop()}`;
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>channels</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as row (row.problem)}
      <tr>
        <td><code>{row.problem}</code></td>
        <td>{row.reports.length}</td>
        <td>{new Set(row.reports.map((r) => r.tenant)).size}</td>
        <td>{[...new Set(row.reports.map((r) => r.channel))].sort().join(' · ')}</td>
        <td>{seen(row.reports)}</td>
        <td>
          {#if tickets[row.problem]}<a href={tickets[row.problem]}>{ticketNo(tickets[row.problem])}</a>{:else}<span class="none">no ticket</span>{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

{#each rows as row (row.problem)}
  {#if row.filedUnder}
    <p class="note">
      The {row.reports.length} <code>{row.problem}</code> reports are filed under <code>{row.filedUnder}</code> by tenant, and
      counted there too.
    </p>
  {/if}
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .none { color: var(--warn); font-weight: 600; }
</style>
