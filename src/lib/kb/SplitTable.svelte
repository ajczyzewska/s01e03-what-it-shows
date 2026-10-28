<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  type Problem = 'parser' | 'hint' | 'one-day shift' | 'unplaced';

  let { ticket, issues }: { ticket: number; issues: Partial<Record<Problem, string>> } = $props();

  const exported = (reports as Report[]).length;
  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  // Counted the desk's way (CauseTable): every report under its cause. The one-day reports stay
  // filed under their cause too, and get a row of their own because they are a different bug.
  const rows = $derived.by(() => {
    const groups = groupByCause(merged, tenantOf);
    const byCause = groups.map((g) => ({
      problem: g.cause as Problem,
      count: g.reports.length,
      tenants: g.tenants,
      firstSeen: g.firstSeen,
      lastSeen: g.lastSeen,
      alsoIn: null as string | null,
    }));
    const shift = merged.filter((r) => notes.oneDayMove.includes(r.id));
    const days = shift.map((r) => r.receivedAt.slice(0, 10)).sort();
    const filedUnder = groups.filter((g) => readAs(g, notes.oneDayMove)).map((g) => g.cause).join(', ');
    return shift.length
      ? [
          ...byCause,
          {
            problem: 'one-day shift' as Problem,
            count: shift.length,
            tenants: new Set(shift.map((r) => r.tenant)).size,
            firstSeen: days[0],
            lastSeen: days[days.length - 1],
            alsoIn: filedUnder,
          },
        ]
      : byCause;
  });
  const causes = $derived(rows.filter((r) => !r.alsoIn).length);
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} (<strong>{exported - merged.length}</strong>
  duplicate rows in the export dropped) ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows as g (g.problem)}
      <tr class:missing={!issues[g.problem]}>
        <td><code>{g.problem}</code></td>
        <td>{g.count}{#if g.alsoIn}<sup>*</sup>{/if}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>
          {#if issues[g.problem]}<a href={issues[g.problem]}>{issues[g.problem]?.replace(/^.*\/issues\//, '#')}</a>{:else}no ticket{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

{#each rows.filter((r) => r.alsoIn) as g (g.problem)}
  <p class="note">
    <sup>*</sup> The desk files these {g.count} by tenant, so they are also counted under <code>{g.alsoIn}</code>.
    The {causes} cause rows add up to {merged.length}.
  </p>
{/each}

<style>
  .sum { font-size: 1.1rem; }
  .note { color: var(--muted); font-size: 0.85em; }
  .missing td { background: var(--warn-bg); }
  .missing td:last-child { color: var(--warn); font-weight: 600; }
</style>
