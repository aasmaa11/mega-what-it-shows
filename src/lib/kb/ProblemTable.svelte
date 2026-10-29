<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a merged ticket. causes.ts places a report by tenant; the reports the desk
  // read as a one-day move (desk-notes.json) are a problem of their own, so they get their own row.
  let { ticket, issues, repo }: { ticket: number; issues: Record<string, number>; repo: string } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const moved = (r: Report) => notes.oneDayMove.includes(r.id);

  const rows = $derived.by(() => {
    const byCause = groupByCause(merged.filter((r) => !moved(r)), tenantOf).map((g) => ({ problem: g.cause, reports: g.reports }));
    const oneDay = merged.filter(moved);
    return [...byCause, ...(oneDay.length ? [{ problem: 'one-day move', reports: oneDay }] : [])].map((row) => {
      const days = row.reports.map((r) => r.receivedAt.slice(0, 10)).sort();
      return { ...row, tenants: new Set(row.reports.map((r) => r.tenant)).size, firstSeen: days[0], lastSeen: days[days.length - 1] };
    });
  });
  const total = $derived(rows.reduce((n, r) => n + r.reports.length, 0));
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} · <strong>{rows.length}</strong> problems
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as row (row.problem)}
      <tr>
        <td><code>{row.problem}</code></td>
        <td>{row.reports.length}</td>
        <td>{row.tenants}</td>
        <td>{row.firstSeen} → {row.lastSeen}</td>
        <td>
          {#if issues[row.problem]}
            <a href="{repo}/issues/{issues[row.problem]}">#{issues[row.problem]}</a>
          {:else}
            <span class="none">no ticket</span>
          {/if}
        </td>
      </tr>
    {/each}
  </tbody>
  <tfoot>
    <tr><td>total</td><td class:off={total !== merged.length}>{total}</td><td colspan="3"></td></tr>
  </tfoot>
</table>

<style>
  .sum { font-size: 1.1rem; }
  tfoot td { font-weight: 600; }
  .off, .none { color: var(--warn); font-weight: 600; }
</style>
