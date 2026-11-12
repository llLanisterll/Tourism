<script lang="ts">
	import { onMount } from 'svelte';
	import { goto } from '$app/navigation';
	import {
		MapPinned,
		Tags,
		MessageSquare,
		Users,
		LayoutDashboard,
		Search,
		Plus,
		Pencil,
		Trash2,
		LogOut,
		X,
		EyeOff,
		Eye,
		Star,
		Menu,
		ArrowUpRight,
		ExternalLink,
		Check,
		CircleAlert,
		TrendingUp,
		Shield
	} from 'lucide-svelte';
	import {
		getPlaces,
		savePlaces,
		getCategories,
		saveCategories,
		getReviews,
		saveReviews
	} from '$lib/data';
	import { priceRange } from '$lib/scoring';
	import { userStore } from '$lib/userStore';
	import LangSwitcher from '$components/LangSwitcher.svelte';
	import AdminPlaceModal from '$components/AdminPlaceModal.svelte';
	import AdminEmptyState from '$components/AdminEmptyState.svelte';
	import AdminSidebarNav from '$components/AdminSidebarNav.svelte';
	import { t, locale } from '$lib/i18n';
	import type { Place, Review } from '$lib/types';

	type TabKey = 'dashboard' | 'places' | 'categories' | 'reviews' | 'users';
	type ReviewFilter = 'all' | 'visible' | 'hidden';
	type DeleteTarget = {
		type: 'place' | 'review' | 'user';
		id: string;
		label: string;
	};

	let tab: TabKey = 'dashboard';
	let mobileNavOpen = false;

	const tabConfig: [TabKey, any][] = [
		['dashboard', LayoutDashboard],
		['places', MapPinned],
		['categories', Tags],
		['reviews', MessageSquare],
		['users', Users]
	];

	interface Acct {
		name: string;
		email: string;
		role: 'admin' | 'user';
		locked?: boolean;
	}

	const seedUsers: Acct[] = [
		{ name: 'Admin Jelajah', email: 'admin@jelajah.id', role: 'admin', locked: true },
		{ name: 'Rani Putri', email: 'user@jelajah.id', role: 'user', locked: true }
	];

	let q = '';
	let placeCategoryFilter = 'all';
	let reviewStatusFilter: ReviewFilter = 'all';

	let places: Place[] = [];
	let cats: string[] = [];
	let reviews: Review[] = [];
	let accts: Acct[] = [];
	let user: { name?: string; email?: string; role?: string } | null = null;

	let editing: Place | null = null;
	let editingIsNew = false;
	let newCat = '';
	let renaming: [string, string] | null = null;
	let pendingDelete: DeleteTarget | null = null;
	let notice: { tone: 'error' | 'success'; text: string } | null = null;

	onMount(async () => {
		places = getPlaces();
		cats = getCategories();
		reviews = getReviews();

		try {
			accts = JSON.parse(localStorage.getItem('accounts') || '[]');
		} catch {
			accts = [];
		}

		if ($userStore) {
			user = $userStore;
			return;
		}

		try {
			const { supabase } = await import('$lib/supabaseClient');
			const {
				data: { session }
			} = await supabase.auth.getSession();
			if (session?.user) {
				user = {
					name: session.user.user_metadata?.name || session.user.email,
					email: session.user.email,
					role: session.user.user_metadata?.role || 'user'
				};
				userStore.init(user);
				return;
			}
		} catch {}

		try {
			const sess = JSON.parse(localStorage.getItem('session') || 'null');
			if (sess) {
				user = sess;
				userStore.init(sess);
			}
		} catch {}
	});

	function showNotice(tone: 'error' | 'success', text: string) {
		notice = { tone, text };
	}

	function switchTab(next: TabKey) {
		tab = next;
		mobileNavOpen = false;
		notice = null;
	}

	function commitPlaces(v: Place[]) {
		places = v;
		savePlaces(v);
	}

	function commitCats(v: string[]) {
		cats = v;
		saveCategories(v);
	}

	function commitReviews(v: Review[]) {
		reviews = v;
		saveReviews(v);
	}

	function commitAccts(v: Acct[]) {
		accts = v;
		localStorage.setItem('accounts', JSON.stringify(v));
	}

	async function logout() {
		try {
			const { supabase } = await import('$lib/supabaseClient');
			await supabase.auth.signOut();
		} catch {}
		userStore.logout();
		await goto('/login', { invalidateAll: true });
	}

	function initials(name: string) {
		return (name || 'A')
			.split(' ')
			.filter(Boolean)
			.slice(0, 2)
			.map((s) => s[0]?.toUpperCase() || '')
			.join('');
	}

	function placeName(id: string) {
		return places.find((p) => p.id === id)?.name ?? 'Deleted place';
	}

	function resetFilters() {
		q = '';
		placeCategoryFilter = 'all';
		reviewStatusFilter = 'all';
	}

	// Reactive filtered datasets
	$: term = q.trim().toLowerCase();
	$: allUsers = [...seedUsers, ...accts];

	$: filteredPlaces = places.filter(
		(p) =>
			(placeCategoryFilter === 'all' || p.category === placeCategoryFilter) &&
			(!term ||
				[p.name, p.area, p.address, p.category, ...p.tags]
					.join(' ')
					.toLowerCase()
					.includes(term))
	);

	$: filteredCats = cats.filter((c) => !term || c.toLowerCase().includes(term));

	$: filteredReviews = reviews.filter((r) => {
		const statusOk =
			reviewStatusFilter === 'all' ||
			(reviewStatusFilter === 'visible' && !r.hidden) ||
			(reviewStatusFilter === 'hidden' && r.hidden);
		const textOk =
			!term ||
			(r.reviewerName + ' ' + r.reviewText + ' ' + placeName(r.placeId))
				.toLowerCase()
				.includes(term);
		return statusOk && textOk;
	});

	$: filteredUsers = allUsers.filter(
		(u) => !term || (u.name + ' ' + u.email + ' ' + u.role).toLowerCase().includes(term)
	);

	$: tabCounts = {
		dashboard: null,
		places: places.length,
		categories: cats.length,
		reviews: reviews.length,
		users: allUsers.length
	} as Record<TabKey, number | null>;

	// Dashboard Analytics & Health Metrics
	$: avgRating = places.length
		? (places.reduce((sum, p) => sum + (p.rating || 0), 0) / places.length).toFixed(1)
		: '0.0';
	$: hiddenReviewsCount = reviews.filter((r) => r.hidden).length;
	$: visibleReviewsCount = reviews.length - hiddenReviewsCount;
	$: adminCount = allUsers.filter((u) => u.role === 'admin').length;

	$: reviewFilters = [
		{ val: 'all' as ReviewFilter, label: $t.admin.filterStatus.all, count: reviews.length },
		{
			val: 'visible' as ReviewFilter,
			label: $t.admin.filterStatus.visible,
			count: visibleReviewsCount
		},
		{
			val: 'hidden' as ReviewFilter,
			label: $t.admin.filterStatus.hidden,
			count: hiddenReviewsCount
		}
	];

	$: report = (() => {
		const baseline: Record<string, number> = {
			Culinary: 420,
			Cafe: 310,
			Beach: 280,
			Nature: 190,
			Culture: 160,
			Resort: 90
		};
		for (const c of cats) {
			if (!(c in baseline)) {
				baseline[c] = Math.max(40, places.filter((p) => p.category === c).length * 45);
			}
		}
		const totalCatSearches = Math.max(
			1,
			Object.values(baseline).reduce((a, b) => a + b, 0)
		);
		const topCats = Object.entries(baseline)
			.filter(([c]) => !term || c.toLowerCase().includes(term))
			.sort((a, b) => b[1] - a[1])
			.slice(0, 6)
			.map(([c, count]) => ({
				name: c,
				count,
				share: Math.round((count / totalCatSearches) * 100)
			}));

		const topPlaces = places
			.filter(
				(p) =>
					!term ||
					[p.name, p.category, p.area].join(' ').toLowerCase().includes(term)
			)
			.map((p) => ({
				place: p,
				volume: p.reviewCount + reviews.filter((r) => r.placeId === p.id).length
			}))
			.sort((a, b) => b.volume - a.volume)
			.slice(0, 5);

		const recentReviews = [...reviews]
			.filter(
				(r) =>
					!term ||
					(r.reviewerName + ' ' + r.reviewText + ' ' + placeName(r.placeId))
						.toLowerCase()
						.includes(term)
			)
			.sort((a, b) => b.createdAt.localeCompare(a.createdAt))
			.slice(0, 4);

		return { topCats, topPlaces, recentReviews, searches: 1840 };
	})();

	$: kpiCards = [
		{
			key: 'places' as TabKey,
			label: $t.admin.stats.places,
			value: String(places.length),
			sub: `${cats.length} ${$t.admin.activeCategories.toLowerCase()}`,
			icon: MapPinned
		},
		{
			key: 'reviews' as TabKey,
			label: $t.admin.stats.reviews,
			value: String(reviews.length),
			sub: `${visibleReviewsCount} ${$t.admin.table.visible.toLowerCase()} · ${hiddenReviewsCount} ${$t.admin.table.hidden.toLowerCase()}`,
			icon: MessageSquare
		},
		{
			key: 'users' as TabKey,
			label: $t.admin.stats.users,
			value: String(allUsers.length),
			sub: `${adminCount} ${$t.admin.adminAccounts.toLowerCase()}`,
			icon: Users
		},
		{
			key: 'dashboard' as TabKey,
			label: $t.admin.stats.searches,
			value: report.searches.toLocaleString($locale === 'id' ? 'id-ID' : 'en-US'),
			sub: `★ ${avgRating} ${$t.admin.avgRating.toLowerCase()}`,
			icon: TrendingUp
		}
	];

	function openAddPlace() {
		editingIsNew = true;
		editing = {
			id: `place-${Date.now()}`,
			name: '',
			category: cats[0] || 'Culinary',
			area: '',
			address: '',
			lat: -8.5,
			lng: 115.2,
			priceMin: 25000,
			priceMax: 100000,
			rating: 4.5,
			reviewCount: 0,
			image:
				'https://images.unsplash.com/photo-1539755530862-00f623c00f52?w=900&fit=crop&auto=format&q=80',
			blurb: '',
			hours: '08:00 – 22:00',
			tags: []
		};
	}

	function openEditPlace(place: Place) {
		editingIsNew = false;
		editing = { ...place, tags: [...(place.tags || [])] };
	}

	function handleSavePlace(updated: Place) {
		const exists = places.some((r) => r.id === updated.id);
		commitPlaces(
			exists ? places.map((r) => (r.id === updated.id ? updated : r)) : [updated, ...places]
		);
		editing = null;
	}

	function addCategory() {
		const n = newCat.trim();
		if (!n || cats.some((c) => c.toLowerCase() === n.toLowerCase())) {
			showNotice('error', $t.admin.categoryDuplicate);
			return;
		}
		commitCats([...cats, n]);
		newCat = '';
		notice = null;
	}

	function saveRename(original: string) {
		if (!renaming) return;
		const next = renaming[1].trim();
		if (!next) {
			renaming = null;
			return;
		}
		if (next !== original && cats.some((c) => c.toLowerCase() === next.toLowerCase())) {
			showNotice('error', $t.admin.categoryDuplicate);
			return;
		}
		commitCats(cats.map((x) => (x === original ? next : x)));
		commitPlaces(places.map((p) => (p.category === original ? { ...p, category: next } : p)));
		renaming = null;
		notice = null;
	}

	function handleDeleteCategory(c: string) {
		if (places.some((p) => p.category === c)) {
			showNotice('error', $t.admin.categoryInUse);
			return;
		}
		commitCats(cats.filter((x) => x !== c));
		notice = null;
	}

	function toggleReviewVisibility(id: string) {
		commitReviews(reviews.map((x) => (x.id === id ? { ...x, hidden: !x.hidden } : x)));
	}

	function toggleUserRole(email: string) {
		commitAccts(
			accts.map((a) =>
				a.email === email ? { ...a, role: a.role === 'admin' ? 'user' : 'admin' } : a
			)
		);
	}

	function executePendingDelete() {
		if (!pendingDelete) return;
		const { type, id } = pendingDelete;
		if (type === 'place') {
			commitPlaces(places.filter((x) => x.id !== id));
		} else if (type === 'review') {
			commitReviews(reviews.filter((x) => x.id !== id));
		} else if (type === 'user') {
			commitAccts(accts.filter((a) => a.email !== id));
		}
		pendingDelete = null;
	}

	function handleWindowKeydown(e: KeyboardEvent) {
		if (e.key === 'Escape') {
			if (pendingDelete) pendingDelete = null;
			if (mobileNavOpen) mobileNavOpen = false;
			if (renaming) renaming = null;
		}
	}

	const thCls = 'px-5 py-3.5 font-normal mono text-slate-soft';
	const inputCls =
		'w-full border border-line bg-white px-3.5 py-2.5 text-sm text-ink outline-none transition focus:border-brand';
</script>

<svelte:window on:keydown={handleWindowKeydown} />

<svelte:head>
	<title>{$t.admin.tabs[tab]} — {$t.admin.title} · Jelajah</title>
</svelte:head>

<div class="grid min-h-screen bg-canvas text-ink lg:grid-cols-[260px_1fr]">
	<!-- Desktop Sticky Sidebar (Editorial Forest Theme matching Landing Page) -->
	<aside class="hidden lg:sticky lg:top-0 lg:block lg:h-screen">
		<AdminSidebarNav {tab} {tabConfig} {tabCounts} {user} onSelectTab={switchTab} />
	</aside>

	<!-- Mobile Slide-over Drawer -->
	{#if mobileNavOpen}
		<div class="fixed inset-0 z-50 flex lg:hidden" role="dialog" aria-modal="true">
			<div
				class="fixed inset-0 bg-forest/70 backdrop-blur-sm"
				on:click={() => (mobileNavOpen = false)}
				role="presentation"
			/>
			<aside class="relative z-10 w-72 shadow-2xl">
				<AdminSidebarNav
					{tab}
					{tabConfig}
					{tabCounts}
					{user}
					mobile={true}
					onSelectTab={switchTab}
					onCloseMobile={() => (mobileNavOpen = false)}
					onLogout={logout}
				/>
			</aside>
		</div>
	{/if}

	<!-- Main Workspace -->
	<div class="min-w-0">
		<!-- Sticky Top Bar matching public Navbar.svelte -->
		<header
			class="sticky top-0 z-20 flex h-16 items-center justify-between gap-3 border-b border-line bg-canvas/90 px-4 backdrop-blur sm:px-8"
		>
			<div class="flex min-w-0 flex-1 items-center gap-3">
				<button
					type="button"
					on:click={() => (mobileNavOpen = true)}
					aria-label="Open navigation menu"
					class="grid h-9 w-9 shrink-0 place-items-center border border-line bg-white text-ink transition hover:border-ink lg:hidden"
				>
					<Menu size={18} />
				</button>

				<label
					class="flex w-full max-w-md items-center gap-2.5 border border-line bg-white px-3.5 py-2 transition focus-within:border-brand"
				>
					<Search size={15} class="shrink-0 text-slate-soft" />
					<input
						bind:value={q}
						placeholder={$t.admin.searchPlaceholder}
						class="w-full bg-transparent text-sm outline-none placeholder:text-slate-soft/60"
					/>
					{#if q}
						<button
							type="button"
							aria-label="Clear search"
							on:click={() => (q = '')}
							class="text-slate-soft hover:text-ink"
						>
							<X size={14} />
						</button>
					{/if}
				</label>
			</div>

			<div class="flex shrink-0 items-center gap-2.5 sm:gap-3">
				<a
					href="/"
					class="hidden items-center gap-1.5 border border-line bg-white px-3 py-2 text-xs font-bold text-ink transition hover:border-ink sm:inline-flex"
				>
					{$t.admin.viewSite}
					<ArrowUpRight size={14} class="text-brand" />
				</a>

				<LangSwitcher />

				<span class="mono hidden max-w-[130px] truncate font-bold text-ink xl:inline">
					{user?.name || 'Admin'}
				</span>

				<button
					type="button"
					on:click={logout}
					aria-label={$t.nav.signOut}
					class="inline-flex h-9 items-center gap-2 border border-line bg-white px-3 text-xs font-bold text-ink transition hover:bg-ink hover:text-white"
				>
					<LogOut size={14} />
					<span class="hidden sm:inline">{$t.nav.signOut}</span>
				</button>
			</div>
		</header>

		<main class="mx-auto max-w-[1320px] space-y-8 p-5 sm:p-8 lg:p-10">
			<!-- Mobile Horizontal Tab Strip -->
			<div class="flex gap-2 overflow-x-auto pb-1 lg:hidden">
				{#each tabConfig as [key, Icon]}
					{@const active = tab === key}
					<button
						type="button"
						on:click={() => switchTab(key)}
						class="inline-flex shrink-0 items-center gap-2 border px-3.5 py-2 text-xs font-bold transition {active
							? 'border-ink bg-ink text-white'
							: 'border-line bg-white text-slate-soft hover:border-ink hover:text-ink'}"
					>
						<svelte:component this={Icon} size={14} />
						{$t.admin.tabs[key]}
					</button>
				{/each}
			</div>

			<!-- Inline Notice Banner -->
			{#if notice}
				<div
					role="status"
					class="flex items-center justify-between gap-3 border-l-4 px-4 py-3 text-sm font-semibold {notice.tone ===
					'error'
						? 'border-red-600 bg-red-50 text-red-800'
						: 'border-brand bg-brand-soft text-emerald-deep'}"
				>
					<span class="flex items-center gap-2">
						{#if notice.tone === 'error'}
							<CircleAlert size={17} />
						{:else}
							<Check size={17} />
						{/if}
						{notice.text}
					</span>
					<button
						type="button"
						aria-label="Dismiss"
						on:click={() => (notice = null)}
						class="opacity-70 hover:opacity-100"
					>
						<X size={15} />
					</button>
				</div>
			{/if}

			<!-- Editorial Section Header (matches Landing Page section headers) -->
			<div class="flex flex-wrap items-end justify-between gap-4 border-b border-ink pb-5">
				<div>
					<p class="mono text-brand">{$t.admin.title}</p>
					<h1 class="mt-1.5 font-display text-3xl font-extrabold tracking-tight sm:text-4xl">
						{$t.admin.tabs[tab]}
					</h1>
					<p class="mt-1.5 max-w-2xl text-sm text-slate-soft">
						{$t.admin.tabSubtitles[tab]}
					</p>
				</div>

				{#if tab === 'dashboard' || tab === 'places'}
					<button
						type="button"
						on:click={openAddPlace}
						class="inline-flex items-center gap-2 bg-brand px-5 py-3 text-sm font-bold text-white transition hover:bg-emerald-deep"
					>
						<Plus size={16} />
						{$t.admin.addPlace}
					</button>
				{/if}
			</div>

			<!-- ==================== TAB: DASHBOARD ==================== -->
			{#if tab === 'dashboard'}
				<!-- Stats Strip (mirrors Landing Page Stats Bar) -->
				<section class="border border-line bg-line">
					<dl class="grid grid-cols-2 gap-px lg:grid-cols-4">
						{#each kpiCards as kpi}
							<button
								type="button"
								on:click={() => switchTab(kpi.key)}
								class="group bg-white px-5 py-6 text-left transition hover:bg-canvas sm:px-7 sm:py-7"
							>
								<div class="flex items-center justify-between gap-2">
									<dt class="mono text-slate-soft group-hover:text-brand">{kpi.label}</dt>
									<span
										class="grid h-7 w-7 place-items-center bg-brand-soft text-emerald-deep transition group-hover:bg-brand group-hover:text-white"
									>
										<svelte:component this={kpi.icon} size={14} />
									</span>
								</div>
								<dd class="mt-3 font-display text-4xl font-extrabold text-ink sm:text-5xl">
									{kpi.value}
								</dd>
								<p class="mono mt-2 text-slate-soft">{kpi.sub}</p>
							</button>
						{/each}
					</dl>
				</section>

				<!-- Two-Column Editorial Analytics (White Popular Places + Forest Method-style Categories) -->
				<div class="grid gap-6 xl:grid-cols-[1.08fr_1fr]">
					<!-- Top Places -->
					<section class="border border-line bg-white p-6 sm:p-8">
						<div class="flex items-end justify-between border-b border-ink pb-4">
							<div>
								<p class="mono text-brand">{$t.admin.popularityRank}</p>
								<h2 class="mt-1 font-display text-2xl font-extrabold">{$t.admin.topPlaces}</h2>
							</div>
							<button
								type="button"
								on:click={() => switchTab('places')}
								class="mono text-slate-soft transition hover:text-brand"
							>
								{$t.admin.viewAll}
							</button>
						</div>
						<p class="mono mb-6 mt-3 text-slate-soft">{$t.admin.topPlacesSub}</p>

						{#if report.topPlaces.length === 0}
							<p class="border border-dashed border-line py-10 text-center text-sm text-slate-soft">
								{$t.admin.emptyTitle}
							</p>
						{:else}
							{@const maxVol = Math.max(1, ...report.topPlaces.map((r) => r.volume))}
							<ul class="divide-y divide-line">
								{#each report.topPlaces as item, i}
									<li class="py-4 first:pt-0 last:pb-0">
										<div class="flex items-center gap-4">
											<span class="w-8 shrink-0 font-display text-2xl font-extrabold text-brand/35">
												0{i + 1}
											</span>
											<img
												src={item.place.image}
												alt={item.place.name}
												class="h-12 w-14 shrink-0 border border-line bg-canvas object-cover"
											/>
											<div class="min-w-0 flex-1">
												<div class="flex items-baseline justify-between gap-2">
													<p class="truncate font-display text-base font-bold text-ink">
														{item.place.name}
													</p>
													<span class="mono shrink-0 font-bold text-ink">
														{item.volume.toLocaleString()}
													</span>
												</div>
												<div class="mt-0.5 flex items-center justify-between text-xs text-slate-soft">
													<span>{item.place.category} · {item.place.area || 'Bali'}</span>
													<span class="inline-flex items-center gap-1 font-semibold text-ink">
														<Star size={11} class="fill-amber-400 text-amber-400" />
														{item.place.rating}
													</span>
												</div>
												<div class="mt-2 h-1.5 bg-canvas">
													<div
														class="h-1.5 bg-brand transition-all duration-500"
														style="width: {(item.volume / maxVol) * 100}%"
													/>
												</div>
											</div>
										</div>
									</li>
								{/each}
							</ul>
						{/if}
					</section>

					<!-- Top Categories (Mirrors Landing Page's Dark Forest Methodology Block) -->
					<section class="flex flex-col justify-between bg-forest p-6 text-white sm:p-8">
						<div>
							<p class="mono text-emerald-300">{$t.admin.methodMirror}</p>
							<h2 class="mt-1 font-display text-2xl font-extrabold sm:text-3xl">
								{$t.admin.topCats}
							</h2>
							<p class="mt-1.5 text-sm text-white/65">{$t.admin.topCatsSub}</p>

							<div class="mt-6 divide-y divide-white/15 border-y border-white/15">
								{#each report.topCats as catItem}
									<div class="grid grid-cols-[72px_1fr] items-center gap-x-4 gap-y-1.5 py-4">
										<span class="row-span-2 font-display text-3xl font-extrabold leading-none text-emerald-300">
											{catItem.share}<span class="text-lg">%</span>
										</span>
										<div class="flex items-baseline justify-between gap-2">
											<span class="font-display text-base font-bold text-white">
												{catItem.name}
											</span>
											<span class="mono text-white/60">
												{catItem.count.toLocaleString()}
											</span>
										</div>
										<div>
											<div class="h-px bg-white/20">
												<div
													class="h-px bg-emerald-300 transition-all duration-500"
													style="width: {Math.min(100, catItem.share * 2.5)}%"
												/>
											</div>
										</div>
									</div>
								{/each}
							</div>
						</div>

						<div class="mt-6 flex items-center justify-between pt-2">
							<span class="mono text-white/50">
								{cats.length} {$t.admin.activeCategories}
							</span>
							<button
								type="button"
								on:click={() => switchTab('categories')}
								class="mono text-emerald-300 transition hover:text-white"
							>
								{$t.admin.manageCategories}
							</button>
						</div>
					</section>
				</div>

				<!-- Bottom Overview Grid: Recent Reviews + Quick Actions & Content Health -->
				<div class="grid gap-6 xl:grid-cols-[1.35fr_1fr]">
					<!-- Recent Reviews -->
					<section class="border border-line bg-white p-6 sm:p-8">
						<div class="flex items-end justify-between border-b border-ink pb-4">
							<div>
								<p class="mono text-brand">{$t.admin.tabs.reviews}</p>
								<h2 class="mt-1 font-display text-2xl font-extrabold">
									{$t.admin.recentReviews}
								</h2>
							</div>
							<button
								type="button"
								on:click={() => switchTab('reviews')}
								class="mono text-slate-soft transition hover:text-brand"
							>
								{$t.admin.viewAll}
							</button>
						</div>
						<p class="mt-2 text-sm text-slate-soft">{$t.admin.recentReviewsSub}</p>

						<div class="mt-5 divide-y divide-line">
							{#each report.recentReviews as r (r.id)}
								<div class="flex items-start justify-between gap-4 py-4 first:pt-0 last:pb-0">
									<div class="min-w-0 flex-1">
										<div class="flex flex-wrap items-center gap-2">
											<span class="font-bold text-ink">{r.reviewerName}</span>
											<span class="mono text-slate-soft">· {placeName(r.placeId)}</span>
											<span class="inline-flex items-center gap-0.5 text-xs font-bold text-ink">
												<Star size={12} class="fill-amber-400 text-amber-400" />
												{r.rating}
											</span>
										</div>
										<p class="mt-1.5 line-clamp-2 text-sm text-slate-soft">{r.reviewText}</p>
									</div>
									<div class="flex shrink-0 items-center gap-2">
										<span
											class="mono px-2 py-1 {r.hidden
												? 'border border-line bg-canvas text-slate-soft'
												: 'bg-brand-soft text-emerald-deep'}"
										>
											{r.hidden ? $t.admin.table.hidden : $t.admin.table.visible}
										</span>
										<button
											type="button"
											aria-label={r.hidden ? 'Show review' : 'Hide review'}
											on:click={() => toggleReviewVisibility(r.id)}
											class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-ocean hover:bg-ocean-soft hover:text-ocean"
										>
											{#if r.hidden}
												<Eye size={15} />
											{:else}
												<EyeOff size={15} />
											{/if}
										</button>
									</div>
								</div>
							{/each}
						</div>
					</section>

					<!-- Quick Actions & Content Health -->
					<section class="flex flex-col justify-between border border-line bg-white p-6 sm:p-8">
						<div>
							<div class="border-b border-ink pb-4">
								<p class="mono text-brand">{$t.admin.overviewBadge}</p>
								<h2 class="mt-1 font-display text-2xl font-extrabold">
									{$t.admin.quickActions}
								</h2>
							</div>
							<p class="mt-2 text-sm text-slate-soft">{$t.admin.quickActionsSub}</p>

							<div class="mt-5 grid grid-cols-2 gap-px border border-line bg-line">
								<div class="bg-canvas p-4">
									<p class="mono text-slate-soft">{$t.admin.avgRating}</p>
									<p class="mt-1.5 flex items-center gap-1.5 font-display text-2xl font-extrabold">
										<Star size={18} class="fill-amber-400 text-amber-400" />
										{avgRating}
									</p>
								</div>
								<div class="bg-canvas p-4">
									<p class="mono text-slate-soft">{$t.admin.activeCategories}</p>
									<p class="mt-1.5 font-display text-2xl font-extrabold">{cats.length}</p>
								</div>
								<div class="bg-canvas p-4">
									<p class="mono text-slate-soft">{$t.admin.hiddenReviews}</p>
									<p class="mt-1.5 font-display text-2xl font-extrabold">{hiddenReviewsCount}</p>
								</div>
								<div class="bg-canvas p-4">
									<p class="mono text-slate-soft">{$t.admin.adminAccounts}</p>
									<p class="mt-1.5 font-display text-2xl font-extrabold">{adminCount}</p>
								</div>
							</div>
						</div>

						<div class="mt-6 grid gap-2.5 sm:grid-cols-2">
							<button
								type="button"
								on:click={openAddPlace}
								class="inline-flex items-center justify-center gap-2 bg-ink px-4 py-3 text-xs font-bold text-white transition hover:bg-brand"
							>
								<Plus size={15} />
								{$t.admin.addPlace}
							</button>
							<a
								href="/results"
								class="inline-flex items-center justify-center gap-2 border border-line bg-white px-4 py-3 text-xs font-bold text-ink transition hover:border-ink"
							>
								{$t.admin.viewSite}
								<ArrowUpRight size={14} class="text-brand" />
							</a>
						</div>
					</section>
				</div>
			{/if}

			<!-- ==================== TAB: PLACES ==================== -->
			{#if tab === 'places'}
				<section class="border border-line bg-white">
					<div class="flex flex-wrap items-center justify-between gap-4 border-b border-line p-5">
						<div class="flex flex-wrap items-center gap-2">
							<button
								type="button"
								on:click={() => (placeCategoryFilter = 'all')}
								class="border px-3 py-1.5 text-xs font-bold transition {placeCategoryFilter === 'all'
									? 'border-ink bg-ink text-white'
									: 'border-line bg-canvas text-slate-soft hover:border-ink hover:text-ink'}"
							>
								{$t.admin.allCategories} ({places.length})
							</button>
							{#each cats as c}
								{@const count = places.filter((p) => p.category === c).length}
								<button
									type="button"
									on:click={() => (placeCategoryFilter = c)}
									class="border px-3 py-1.5 text-xs font-bold transition {placeCategoryFilter === c
										? 'border-brand bg-brand text-white'
										: 'border-line bg-white text-slate-soft hover:border-ink hover:text-ink'}"
								>
									{c} ({count})
								</button>
							{/each}
						</div>

						<span class="mono text-slate-soft">
							{$t.admin.allPlaces}: {filteredPlaces.length}
						</span>
					</div>

					{#if filteredPlaces.length === 0}
						<AdminEmptyState onReset={resetFilters} />
					{:else}
						<div class="overflow-x-auto">
							<table class="w-full min-w-[820px] text-left text-sm">
								<thead class="border-b border-line bg-canvas">
									<tr>
										<th class={thCls}>{$t.admin.table.place}</th>
										<th class={thCls}>{$t.admin.table.category}</th>
										<th class={thCls}>{$t.admin.table.price}</th>
										<th class={thCls}>{$t.admin.table.rating}</th>
										<th class="{thCls} text-right">{$t.admin.table.actions}</th>
									</tr>
								</thead>
								<tbody class="divide-y divide-line">
									{#each filteredPlaces as r (r.id)}
										<tr class="transition hover:bg-canvas/60">
											<td class="px-5 py-4">
												<div class="flex items-center gap-4">
													<img
														src={r.image}
														alt={r.name}
														class="h-12 w-16 shrink-0 border border-line bg-canvas object-cover"
													/>
													<div class="min-w-0">
														<p class="font-display text-base font-bold text-ink">{r.name}</p>
														<p class="mt-0.5 truncate text-xs text-slate-soft">
															{r.area ? `${r.area} · ` : ''}{r.address}
														</p>
													</div>
												</div>
											</td>
											<td class="px-5 py-4">
												<span class="mono border border-line bg-canvas px-2.5 py-1 text-ink">
													{r.category}
												</span>
											</td>
											<td class="px-5 py-4">
												<p class="font-semibold text-ink">{priceRange(r, $locale === 'en')}</p>
												<p class="mono mt-0.5 text-slate-soft">{r.hours}</p>
											</td>
											<td class="px-5 py-4">
												<span class="inline-flex items-center gap-1 font-bold text-ink">
													<Star size={14} class="fill-amber-400 text-amber-400" />
													{r.rating || '–'}
												</span>
												<span class="ml-1 text-xs text-slate-soft">({r.reviewCount})</span>
											</td>
											<td class="px-5 py-4">
												<div class="flex justify-end gap-1.5">
													<a
														href="/places/{r.id}"
														title={$t.admin.preview}
														aria-label={$t.admin.preview}
														class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-ink hover:text-ink"
													>
														<ExternalLink size={15} />
													</a>
													<button
														type="button"
														title={$t.admin.editPlace}
														aria-label={$t.admin.editPlace}
														on:click={() => openEditPlace(r)}
														class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-ocean hover:bg-ocean-soft hover:text-ocean"
													>
														<Pencil size={15} />
													</button>
													<button
														type="button"
														title={$t.admin.delete}
														aria-label={$t.admin.delete}
														on:click={() =>
															(pendingDelete = { type: 'place', id: r.id, label: r.name })}
														class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-red-300 hover:bg-red-50 hover:text-red-600"
													>
														<Trash2 size={15} />
													</button>
												</div>
											</td>
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
					{/if}
				</section>
			{/if}

			<!-- ==================== TAB: CATEGORIES ==================== -->
			{#if tab === 'categories'}
				<div class="grid gap-6 lg:grid-cols-[340px_1fr]">
					<section class="h-fit border border-line bg-white p-6">
						<p class="mono text-brand">{$t.admin.tabs.categories}</p>
						<h2 class="mt-1 font-display text-xl font-extrabold">
							{$t.admin.addCategoryHeading}
						</h2>
						<p class="mt-1.5 text-sm leading-relaxed text-slate-soft">
							{$t.admin.addCategoryDesc}
						</p>

						<form on:submit|preventDefault={addCategory} class="mt-5 space-y-3">
							<label class="block">
								<span class="mono text-slate-soft">{$t.admin.table.category}</span>
								<input
									bind:value={newCat}
									placeholder={$t.admin.newCategoryName}
									class="{inputCls} mt-1.5"
								/>
							</label>
							<button
								type="submit"
								class="inline-flex w-full items-center justify-center gap-2 bg-brand px-4 py-2.5 text-sm font-bold text-white transition hover:bg-emerald-deep"
							>
								<Plus size={16} />
								{$t.admin.addCategoryBtn}
							</button>
						</form>
					</section>

					<section class="border border-line bg-white">
						{#if filteredCats.length === 0}
							<AdminEmptyState onReset={resetFilters} />
						{:else}
							<table class="w-full text-left text-sm">
								<thead class="border-b border-line bg-canvas">
									<tr>
										<th class={thCls}>{$t.admin.table.category}</th>
										<th class={thCls}>{$t.admin.stats.places}</th>
										<th class="{thCls} text-right">{$t.admin.table.actions}</th>
									</tr>
								</thead>
								<tbody class="divide-y divide-line">
									{#each filteredCats as c (c)}
										{@const count = places.filter((p) => p.category === c).length}
										{@const share = places.length ? Math.round((count / places.length) * 100) : 0}
										<tr class="transition hover:bg-canvas/60">
											<td class="px-5 py-4 font-display text-base font-bold">
												{#if renaming?.[0] === c}
													<input
														bind:value={renaming[1]}
														on:keydown={(e) => {
															if (e.key === 'Enter') {
																e.preventDefault();
																saveRename(c);
															}
														}}
														class="{inputCls} max-w-xs py-1.5"
													/>
												{:else}
													{c}
												{/if}
											</td>
											<td class="px-5 py-4">
												<div class="flex items-center gap-3">
													<span class="w-6 font-bold text-ink">{count}</span>
													<div class="hidden h-1.5 w-28 bg-canvas sm:block">
														<div class="h-1.5 bg-brand" style="width: {share}%" />
													</div>
													<span class="mono hidden text-slate-soft sm:inline">{share}%</span>
												</div>
											</td>
											<td class="px-5 py-4">
												<div class="flex justify-end gap-1.5">
													{#if renaming?.[0] === c}
														<button
															type="button"
															on:click={() => saveRename(c)}
															class="bg-brand px-3 py-1.5 text-xs font-bold text-white transition hover:bg-emerald-deep"
														>
															{$t.admin.table.save}
														</button>
														<button
															type="button"
															on:click={() => (renaming = null)}
															class="border border-line bg-white px-3 py-1.5 text-xs font-bold text-slate-soft hover:border-ink hover:text-ink"
														>
															{$t.admin.cancel}
														</button>
													{:else}
														<button
															type="button"
															title="Rename"
															aria-label="Rename"
															on:click={() => (renaming = [c, c])}
															class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-ocean hover:bg-ocean-soft hover:text-ocean"
														>
															<Pencil size={15} />
														</button>
													{/if}
													<button
														type="button"
														title={$t.admin.delete}
														aria-label={$t.admin.delete}
														on:click={() => handleDeleteCategory(c)}
														class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-red-300 hover:bg-red-50 hover:text-red-600"
													>
														<Trash2 size={15} />
													</button>
												</div>
											</td>
										</tr>
									{/each}
								</tbody>
							</table>
						{/if}
					</section>
				</div>
			{/if}

			<!-- ==================== TAB: REVIEWS ==================== -->
			{#if tab === 'reviews'}
				<section class="border border-line bg-white">
					<div class="flex flex-wrap items-center justify-between gap-4 border-b border-line p-5">
						<div>
							<h2 class="font-display text-xl font-extrabold">{$t.admin.moderateHeading}</h2>
							<p class="mt-0.5 text-sm text-slate-soft">{$t.admin.moderateSub}</p>
						</div>

						<div class="flex flex-wrap gap-2">
							{#each reviewFilters as rf}
								<button
									type="button"
									on:click={() => (reviewStatusFilter = rf.val)}
									class="border px-3 py-1.5 text-xs font-bold transition {reviewStatusFilter === rf.val
										? 'border-ink bg-ink text-white'
										: 'border-line bg-canvas text-slate-soft hover:border-ink hover:text-ink'}"
								>
									{rf.label} ({rf.count})
								</button>
							{/each}
						</div>
					</div>

					{#if filteredReviews.length === 0}
						<AdminEmptyState onReset={resetFilters} />
					{:else}
						<div class="overflow-x-auto">
							<table class="w-full min-w-[820px] text-left text-sm">
								<thead class="border-b border-line bg-canvas">
									<tr>
										<th class={thCls}>{$t.admin.table.reviewer}</th>
										<th class={thCls}>{$t.admin.table.place}</th>
										<th class={thCls}>{$t.admin.table.rating}</th>
										<th class={thCls}>{$t.admin.table.review}</th>
										<th class={thCls}>{$t.admin.table.status}</th>
										<th class="{thCls} text-right">{$t.admin.table.actions}</th>
									</tr>
								</thead>
								<tbody class="divide-y divide-line">
									{#each filteredReviews as r (r.id)}
										<tr class="transition {r.hidden ? 'bg-canvas/70 text-slate-soft' : 'hover:bg-canvas/60'}">
											<td class="px-5 py-4">
												<p class="font-bold {r.hidden ? 'text-slate-soft' : 'text-ink'}">
													{r.reviewerName}
												</p>
												<p class="mono mt-0.5 text-slate-soft">{r.createdAt}</p>
											</td>
											<td class="px-5 py-4">
												<a
													href="/places/{r.placeId}"
													class="font-semibold text-ink underline decoration-line underline-offset-4 hover:decoration-brand"
												>
													{placeName(r.placeId)}
												</a>
											</td>
											<td class="px-5 py-4">
												<span class="inline-flex items-center gap-1 font-bold text-ink">
													<Star size={13} class="fill-amber-400 text-amber-400" />
													{r.rating}
												</span>
											</td>
											<td class="max-w-sm px-5 py-4">
												<div class="flex items-start gap-3">
													{#if r.photoUrl}
														<img
															src={r.photoUrl}
															alt=""
															class="h-10 w-10 shrink-0 border border-line object-cover"
														/>
													{/if}
													<p class="line-clamp-2 text-sm leading-relaxed">{r.reviewText}</p>
												</div>
											</td>
											<td class="px-5 py-4">
												<span
													class="mono inline-block px-2.5 py-1 {r.hidden
														? 'border border-line bg-canvas text-slate-soft'
														: 'bg-brand-soft text-emerald-deep'}"
												>
													{r.hidden ? $t.admin.table.hidden : $t.admin.table.visible}
												</span>
											</td>
											<td class="px-5 py-4">
												<div class="flex justify-end gap-1.5">
													<button
														type="button"
														aria-label={r.hidden ? 'Show' : 'Hide'}
														title={r.hidden ? 'Show' : 'Hide'}
														on:click={() => toggleReviewVisibility(r.id)}
														class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-ocean hover:bg-ocean-soft hover:text-ocean"
													>
														{#if r.hidden}
															<Eye size={15} />
														{:else}
															<EyeOff size={15} />
														{/if}
													</button>
													<button
														type="button"
														aria-label={$t.admin.delete}
														title={$t.admin.delete}
														on:click={() =>
															(pendingDelete = {
																type: 'review',
																id: r.id,
																label: `${r.reviewerName} (${placeName(r.placeId)})`
															})}
														class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-red-300 hover:bg-red-50 hover:text-red-600"
													>
														<Trash2 size={15} />
													</button>
												</div>
											</td>
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
					{/if}
				</section>
			{/if}

			<!-- ==================== TAB: USERS ==================== -->
			{#if tab === 'users'}
				<section class="border border-line bg-white">
					<div class="flex flex-wrap items-center justify-between gap-4 border-b border-line p-5">
						<div class="flex items-center gap-3">
							<span class="grid h-9 w-9 place-items-center bg-brand-soft text-emerald-deep">
								<Shield size={17} />
							</span>
							<div>
								<h2 class="font-display text-lg font-extrabold">
									{allUsers.length} {$t.admin.stats.users}
								</h2>
								<p class="mono text-slate-soft">
									{adminCount} Admin · {allUsers.length - adminCount} User
								</p>
							</div>
						</div>
					</div>

					{#if filteredUsers.length === 0}
						<AdminEmptyState onReset={resetFilters} />
					{:else}
						<div class="overflow-x-auto">
							<table class="w-full min-w-[680px] text-left text-sm">
								<thead class="border-b border-line bg-canvas">
									<tr>
										<th class={thCls}>{$t.admin.table.name}</th>
										<th class={thCls}>{$t.admin.table.email}</th>
										<th class={thCls}>{$t.admin.table.role}</th>
										<th class="{thCls} text-right">{$t.admin.table.actions}</th>
									</tr>
								</thead>
								<tbody class="divide-y divide-line">
									{#each filteredUsers as u (u.email)}
										<tr class="transition hover:bg-canvas/60">
											<td class="px-5 py-4">
												<div class="flex items-center gap-3">
													<span
														class="grid h-9 w-9 shrink-0 place-items-center font-display text-xs font-extrabold {u.role ===
														'admin'
															? 'bg-forest text-emerald-300'
															: 'bg-brand-soft text-emerald-deep'}"
													>
														{initials(u.name)}
													</span>
													<span class="font-bold text-ink">{u.name}</span>
												</div>
											</td>
											<td class="px-5 py-4 text-slate-soft">{u.email}</td>
											<td class="px-5 py-4">
												<span
													class="mono inline-block px-2.5 py-1 {u.role === 'admin'
														? 'bg-forest text-emerald-300'
														: 'bg-brand-soft text-emerald-deep'}"
												>
													{u.role}
												</span>
											</td>
											<td class="px-5 py-4">
												<div class="flex justify-end gap-2">
													{#if u.locked}
														<span class="mono border border-line bg-canvas px-2.5 py-1.5 text-slate-soft">
															{$t.admin.table.builtIn}
														</span>
													{:else}
														<button
															type="button"
															on:click={() => toggleUserRole(u.email)}
															class="border border-line bg-white px-3 py-1.5 text-xs font-bold text-ocean transition hover:border-ocean hover:bg-ocean-soft"
														>
															{u.role === 'admin' ? $t.admin.makeUser : $t.admin.makeAdmin}
														</button>
														<button
															type="button"
															aria-label={$t.admin.delete}
															title={$t.admin.delete}
															on:click={() =>
																(pendingDelete = {
																	type: 'user',
																	id: u.email,
																	label: `${u.name} (${u.email})`
																})}
															class="grid h-8 w-8 place-items-center border border-line text-slate-soft transition hover:border-red-300 hover:bg-red-50 hover:text-red-600"
														>
															<Trash2 size={15} />
														</button>
													{/if}
												</div>
											</td>
										</tr>
									{/each}
								</tbody>
							</table>
						</div>
					{/if}
				</section>
			{/if}
		</main>
	</div>

	<!-- Add / Edit Place Modal -->
	{#if editing}
		<AdminPlaceModal
			place={editing}
			isNew={editingIsNew}
			categories={cats}
			onSave={handleSavePlace}
			onClose={() => (editing = null)}
		/>
	{/if}

	<!-- Confirm Delete Dialog -->
	{#if pendingDelete}
		<div
			class="fixed inset-0 z-50 grid place-items-center p-4"
			role="dialog"
			aria-modal="true"
		>
			<button
				type="button"
				tabindex="-1"
				aria-label={$t.admin.cancel}
				on:click={() => (pendingDelete = null)}
				class="fixed inset-0 cursor-default bg-forest/70 backdrop-blur-sm"
			/>
			<div class="rise relative z-10 w-full max-w-md border border-ink bg-white p-6 shadow-2xl">
				<p class="mono text-red-600">{$t.admin.confirmDeleteTitle}</p>
				<h2 class="mt-2 font-display text-2xl font-extrabold text-ink">
					{pendingDelete.label}
				</h2>
				<p class="mt-2 text-sm text-slate-soft">
					{pendingDelete.type === 'place'
						? $t.admin.confirmDeletePlace
						: pendingDelete.type === 'review'
							? $t.admin.confirmDeleteReview
							: $t.admin.confirmDeleteUser}
				</p>
				<div class="mt-6 flex justify-end gap-3">
					<button
						type="button"
						on:click={() => (pendingDelete = null)}
						class="border border-line bg-white px-5 py-2.5 text-sm font-bold text-ink transition hover:border-ink"
					>
						{$t.admin.cancel}
					</button>
					<button
						type="button"
						on:click={executePendingDelete}
						class="bg-red-600 px-5 py-2.5 text-sm font-bold text-white transition hover:bg-red-700"
					>
						{$t.admin.delete}
					</button>
				</div>
			</div>
		</div>
	{/if}
</div>
