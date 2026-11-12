<script lang="ts">
	import { X, ImageOff } from 'lucide-svelte';
	import { t } from '$lib/i18n';
	import type { Place } from '$lib/types';

	export let place: Place;
	export let isNew = false;
	export let categories: string[] = [];
	export let onSave: (updated: Place) => void;
	export let onClose: () => void;

	let form: Place = { ...place, tags: [...(place.tags || [])] };
	let tagsText = (place.tags || []).join(', ');
	let imgFailed = false;

	function handleKeydown(e: KeyboardEvent) {
		if (e.key === 'Escape') {
			onClose();
		}
	}

	function handleSubmit() {
		const tags = tagsText
			.split(',')
			.map((s) => s.trim())
			.filter(Boolean);
		const priceMin = Math.max(0, Number(form.priceMin) || 0);
		const priceMax = Math.max(priceMin, Number(form.priceMax) || priceMin);

		onSave({
			...form,
			name: form.name.trim(),
			area: form.area.trim(),
			address: form.address.trim(),
			hours: form.hours.trim() || '08:00 – 22:00',
			blurb: form.blurb.trim(),
			image:
				form.image.trim() ||
				'https://images.unsplash.com/photo-1539755530862-00f623c00f52?w=900&fit=crop&auto=format&q=80',
			priceMin,
			priceMax,
			lat: Number(form.lat) || -8.5,
			lng: Number(form.lng) || 115.2,
			tags
		});
	}

	const inputCls =
		'mt-1.5 w-full border border-line bg-white px-3.5 py-2.5 text-sm text-ink outline-none transition focus:border-brand';
</script>

<svelte:window on:keydown={handleKeydown} />

<div
	class="fixed inset-0 z-50 grid place-items-center overflow-y-auto p-4"
	role="dialog"
	aria-modal="true"
	aria-labelledby="place-modal-title"
>
	<button
		type="button"
		tabindex="-1"
		aria-label={$t.admin.cancel}
		on:click={onClose}
		class="fixed inset-0 cursor-default bg-forest/70 backdrop-blur-sm"
	/>
	<form
		on:submit|preventDefault={handleSubmit}
		class="rise relative z-10 w-full max-w-2xl border border-ink bg-white shadow-2xl"
	>
		<!-- Modal Header -->
		<div class="flex items-start justify-between border-b border-line bg-forest px-6 py-5 text-white">
			<div>
				<p class="mono text-emerald-300">{$t.admin.modal.badge}</p>
				<h2 id="place-modal-title" class="mt-1 font-display text-2xl font-extrabold">
					{isNew ? $t.admin.addPlace : $t.admin.editPlace}
				</h2>
			</div>
			<button
				type="button"
				aria-label={$t.admin.cancel}
				on:click={onClose}
				class="grid h-9 w-9 place-items-center border border-white/20 text-white/80 transition hover:bg-white hover:text-ink"
			>
				<X size={18} />
			</button>
		</div>

		<!-- Modal Body -->
		<div class="max-h-[72vh] space-y-5 overflow-y-auto p-6">
			<!-- Image Preview & URL -->
			<div class="flex flex-col gap-4 border border-line bg-canvas p-4 sm:flex-row sm:items-center">
				<div class="h-24 w-full shrink-0 overflow-hidden border border-line bg-white sm:w-36">
					{#if form.image && !imgFailed}
						<img
							src={form.image}
							alt={form.name || 'Preview'}
							on:error={() => (imgFailed = true)}
							class="h-full w-full object-cover"
						/>
					{:else}
						<div class="grid h-full w-full place-items-center text-slate-soft">
							<ImageOff size={22} />
						</div>
					{/if}
				</div>
				<label class="block flex-1">
					<span class="mono text-slate-soft">{$t.admin.modal.image}</span>
					<input
						type="url"
						bind:value={form.image}
						on:input={() => (imgFailed = false)}
						placeholder={$t.admin.modal.imagePlaceholder}
						class={inputCls}
					/>
				</label>
			</div>

			<!-- Name & Category -->
			<div class="grid gap-4 sm:grid-cols-2">
				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.name} *</span>
					<input
						required
						bind:value={form.name}
						placeholder={$t.admin.modal.namePlaceholder}
						class={inputCls}
					/>
				</label>

				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.category}</span>
					<select bind:value={form.category} class={inputCls}>
						{#each categories as c}
							<option value={c}>{c}</option>
						{/each}
					</select>
				</label>
			</div>

			<!-- Area & Hours -->
			<div class="grid gap-4 sm:grid-cols-2">
				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.area}</span>
					<input
						bind:value={form.area}
						placeholder={$t.admin.modal.areaPlaceholder}
						class={inputCls}
					/>
				</label>

				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.hours}</span>
					<input
						bind:value={form.hours}
						placeholder={$t.admin.modal.hoursPlaceholder}
						class={inputCls}
					/>
				</label>
			</div>

			<!-- Address -->
			<label class="block">
				<span class="mono text-slate-soft">{$t.admin.modal.address}</span>
				<input
					bind:value={form.address}
					placeholder={$t.admin.modal.addressPlaceholder}
					class={inputCls}
				/>
			</label>

			<!-- Pricing & Coordinates -->
			<div class="grid grid-cols-2 gap-4 sm:grid-cols-4">
				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.priceMin}</span>
					<input type="number" min="0" step="5000" bind:value={form.priceMin} class={inputCls} />
				</label>
				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.priceMax}</span>
					<input type="number" min="0" step="5000" bind:value={form.priceMax} class={inputCls} />
				</label>
				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.lat}</span>
					<input type="number" step="any" bind:value={form.lat} class={inputCls} />
				</label>
				<label class="block">
					<span class="mono text-slate-soft">{$t.admin.modal.lng}</span>
					<input type="number" step="any" bind:value={form.lng} class={inputCls} />
				</label>
			</div>

			<!-- Tags -->
			<label class="block">
				<span class="mono text-slate-soft">{$t.admin.modal.tags}</span>
				<input
					bind:value={tagsText}
					placeholder={$t.admin.modal.tagsPlaceholder}
					class={inputCls}
				/>
			</label>

			<!-- Description -->
			<label class="block">
				<span class="mono text-slate-soft">{$t.admin.modal.blurb}</span>
				<textarea
					bind:value={form.blurb}
					rows="3"
					placeholder={$t.admin.modal.blurbPlaceholder}
					class={inputCls}
				/>
			</label>
		</div>

		<!-- Modal Footer -->
		<div class="flex items-center justify-end gap-3 border-t border-line bg-canvas px-6 py-4">
			<button
				type="button"
				on:click={onClose}
				class="border border-line bg-white px-5 py-2.5 text-sm font-bold text-ink transition hover:border-ink"
			>
				{$t.admin.cancel}
			</button>
			<button
				type="submit"
				class="bg-brand px-6 py-2.5 text-sm font-bold text-white transition hover:bg-emerald-deep"
			>
				{isNew ? $t.admin.modal.createPlace : $t.admin.modal.saveChanges}
			</button>
		</div>
	</form>
</div>
