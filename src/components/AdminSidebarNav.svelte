<script lang="ts">
	import { Compass, ArrowUpRight, LogOut, X } from 'lucide-svelte';
	import { t } from '$lib/i18n';

	type TabKey = 'dashboard' | 'places' | 'categories' | 'reviews' | 'users';

	export let tab: TabKey;
	export let tabConfig: [TabKey, any][];
	export let tabCounts: Record<TabKey, number | null>;
	export let user: { name?: string; email?: string } | null = null;
	export let mobile = false;
	export let onSelectTab: (key: TabKey) => void;
	export let onCloseMobile: (() => void) | undefined = undefined;
	export let onLogout: (() => void) | undefined = undefined;

	function initials(name: string) {
		return (name || 'A')
			.split(' ')
			.filter(Boolean)
			.slice(0, 2)
			.map((s) => s[0]?.toUpperCase() || '')
			.join('');
	}
</script>

<div class="flex h-full flex-col justify-between bg-forest text-white">
	<div>
		<!-- Brand Header -->
		<div class="flex h-16 items-center justify-between border-b border-white/10 px-6">
			<a href="/" class="flex items-center gap-2 font-display text-2xl font-extrabold tracking-tight">
				<Compass size={21} class="text-emerald-300" />
				Jelajah
			</a>
			{#if mobile && onCloseMobile}
				<button
					type="button"
					aria-label="Close menu"
					on:click={onCloseMobile}
					class="grid h-8 w-8 place-items-center border border-white/20 text-white/80 hover:bg-white hover:text-ink"
				>
					<X size={16} />
				</button>
			{:else}
				<span class="mono border border-emerald-300/30 bg-emerald-300/10 px-2 py-0.5 text-emerald-300">
					Admin
				</span>
			{/if}
		</div>

		<!-- Navigation -->
		<div class="px-4 pt-6">
			<p class="mono px-3 pb-3 text-white/45">{$t.admin.title}</p>
			<nav class="space-y-1">
				{#each tabConfig as [key, Icon]}
					{@const active = tab === key}
					<button
						type="button"
						on:click={() => onSelectTab(key)}
						class="flex w-full items-center justify-between border-l-4 px-3 py-3 text-left text-sm transition {active
							? 'border-brand bg-white font-bold text-ink'
							: 'border-transparent font-medium text-white/75 hover:bg-white/10 hover:text-white'}"
					>
						<span class="flex items-center gap-3">
							<svelte:component
								this={Icon}
								size={17}
								class={active ? 'text-brand' : 'text-emerald-300/80'}
							/>
							<span>{$t.admin.tabs[key]}</span>
						</span>
						{#if tabCounts[key] !== null}
							<span
								class="mono px-1.5 py-0.5 {active
									? 'bg-brand-soft text-emerald-deep'
									: 'bg-white/10 text-white/70'}"
							>
								{tabCounts[key]}
							</span>
						{/if}
					</button>
				{/each}
			</nav>
		</div>
	</div>

	<!-- Sidebar Footer -->
	<div class="space-y-4 border-t border-white/10 p-5">
		<a
			href="/"
			class="flex items-center justify-between border border-white/20 px-3.5 py-2.5 text-xs font-bold text-white/90 transition hover:border-emerald-300 hover:bg-white/10"
		>
			<span>{$t.admin.backToLanding}</span>
			<ArrowUpRight size={15} class="text-emerald-300" />
		</a>

		{#if mobile && onLogout}
			<button
				type="button"
				on:click={onLogout}
				class="flex w-full items-center justify-center gap-2 border border-red-400/40 bg-red-500/10 py-2.5 text-xs font-bold text-red-200"
			>
				<LogOut size={14} />
				{$t.nav.signOut}
			</button>
		{:else}
			<div class="flex items-center gap-3 pt-1">
				<div
					class="grid h-9 w-9 shrink-0 place-items-center border border-emerald-300/30 bg-emerald-300/15 font-display text-sm font-extrabold text-emerald-300"
				>
					{initials(user?.name || 'Admin')}
				</div>
				<div class="min-w-0 flex-1">
					<p class="truncate text-sm font-bold text-white">{user?.name || 'Admin Jelajah'}</p>
					<p class="truncate text-xs text-white/55">{user?.email || 'admin@jelajah.id'}</p>
				</div>
			</div>
		{/if}
	</div>
</div>

