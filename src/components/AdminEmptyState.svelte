<script lang="ts">
	import { RotateCcw } from 'lucide-svelte';
	import { t } from '$lib/i18n';

	export let onReset: () => void;
</script>

<div class="p-14 text-center">
	<p class="font-display text-xl font-bold text-ink">{$t.admin.emptyTitle}</p>
	<p class="mt-1 text-sm text-slate-soft">{$t.admin.emptyDesc}</p>
	<button
		type="button"
		on:click={onReset}
		class="mt-5 inline-flex items-center gap-2 border border-line bg-canvas px-4 py-2 text-xs font-bold text-ink transition hover:border-ink"
	>
		<RotateCcw size={13} />
		{$t.admin.resetFilter}
	</button>
</div>

