<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Problem = 'parser' | 'hint' | 'one-day shift' | 'unplaced';

  let { ticket, issues }: { ticket: number; issues: Partial<Record<Problem, string>> } = $props();

  // causes.ts only knows month/day flips. The desk's oneDayMove reports are not flips at all:
  // they get their own problem here instead of a footnote under `parser`.
  const problemOf = (r: Report): Problem =>
    notes.oneDayMove.includes(r.id) ? 'one-day shift' : (causeOf(r, tenantOf(r)) ?? 'unplaced');

  const rows = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived.by(() => {
    const by = new Map<Problem, Report[]>();
    for (const r of rows) by.set(problemOf(r), [...(by.get(problemOf(r)) ?? []), r]);
    return [...by.entries()]
      .map(([problem, rs]) => {
        const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
        return {
          problem,
          count: rs.length,
          tenants: new Set(rs.map((r) => r.tenant)).size,
          firstSeen: days[0],
          lastSeen: days[days.length - 1],
        };
      })
      .sort((a, b) => b.count - a.count);
  });
  const exported = (reports as Report[]).length;
</script>

<p class="sum">
  <strong>{rows.length}</strong> reports merged into #{ticket} (<strong>{exported - rows.length}</strong>
  duplicate rows in the export dropped) ·
  <strong>{groups.length}</strong>
  {groups.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each groups as g (g.problem)}
      <tr class:missing={!issues[g.problem]}>
        <td><code>{g.problem}</code></td>
        <td>{g.count}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>
          {#if issues[g.problem]}<a href={issues[g.problem]}>{issues[g.problem]?.replace(/^.*\/issues\//, '#')}</a>{:else}no ticket{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .missing td { background: var(--warn-bg); }
  .missing td:last-child { color: var(--warn); font-weight: 600; }
</style>
