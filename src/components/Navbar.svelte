<script lang="ts">
  import { page } from "$app/stores";
  import { goto } from "$app/navigation";
  import { ArrowLeft, Compass, LogOut, Menu, X } from "lucide-svelte";
  import LangSwitcher from "$components/LangSwitcher.svelte";
  import { userStore } from "$lib/userStore";
  import { t } from "$lib/i18n";
  import { onMount } from "svelte";

  export let user: { name: string; role: string; email?: string } | null = null;

  let scrolled = false;
  let isHome = false;
  let mobileMenuOpen = false;
  let loggedOut = false;

  $: if ($userStore) loggedOut = false;
  $: activeUser = loggedOut ? null : $userStore || user;
  $: isAdminUser = activeUser?.role === "admin";
  $: isHome = $page.url.pathname === "/" || $page.url.pathname === "/home";

  // Track scroll on the client
  function handleScroll() {
    scrolled = window.scrollY > 40;
  }

  onMount(() => {
    userStore.init(user);
    handleScroll();
    window.addEventListener("scroll", handleScroll, { passive: true });
    return () => window.removeEventListener("scroll", handleScroll);
  });

  // Close mobile menu on page navigation
  $: if ($page.url.pathname) {
    mobileMenuOpen = false;
  }

  $: clear = isHome && !scrolled && !mobileMenuOpen;

  async function logout() {
    loggedOut = true;
    try {
      const { supabase } = await import("$lib/supabaseClient");
      await supabase.auth.signOut();
    } catch {}
    userStore.logout();
    mobileMenuOpen = false;
    await goto("/", { invalidateAll: true });
  }
</script>

<header
  class={`${isHome ? "fixed" : "sticky"} inset-x-0 top-0 z-30 border-b transition-colors duration-300 ${clear ? "border-transparent bg-transparent text-white" : "border-line bg-canvas/90 text-ink backdrop-blur"}`}
>
  <div
    class="mx-auto flex h-16 max-w-[1400px] items-center justify-between px-5 lg:px-10"
  >
    <!-- Logo -->
    <a
      href={activeUser ? "/home" : "/"}
      class="flex items-center gap-2 font-display text-2xl font-extrabold tracking-tight"
    >
      <Compass size={22} class={clear ? "text-white" : "text-brand"} />
      Jelajah
    </a>

    <!-- Desktop nav -->
    <nav class="hidden items-center gap-5 md:flex lg:gap-8">
      {#if activeUser}
        <a
          href="/home"
          class={`text-sm font-semibold transition lg:text-base ${clear ? "text-white/85 hover:text-white" : "text-slate-600 hover:text-brand"} ${$page.url.pathname === "/home" ? (clear ? "!text-white font-bold" : "!text-brand font-bold") : ""}`}
        >
          {$t.nav.home}
        </a>
        <a
          href="/find"
          class={`text-sm font-semibold transition lg:text-base ${clear ? "text-white/85 hover:text-white" : "text-slate-600 hover:text-brand"} ${$page.url.pathname === "/find" ? (clear ? "!text-white font-bold" : "!text-brand font-bold") : ""}`}
        >
          {$t.nav.preferences}
        </a>
        <a
          href="/results"
          class={`text-sm font-semibold transition lg:text-base ${clear ? "text-white/85 hover:text-white" : "text-slate-600 hover:text-brand"} ${$page.url.pathname === "/results" ? (clear ? "!text-white font-bold" : "!text-brand font-bold") : ""}`}
        >
          {$t.nav.explore}
        </a>
        <a
          href="/wishlist"
          class={`text-sm font-semibold transition lg:text-base ${clear ? "text-white/85 hover:text-white" : "text-slate-600 hover:text-brand"} ${$page.url.pathname === "/wishlist" ? (clear ? "!text-white font-bold" : "!text-brand font-bold") : ""}`}
        >
          {$t.nav.wishlist}
        </a>
      {:else}
        <a
          href="/#how"
          class={`text-sm font-semibold transition lg:text-base ${clear ? "text-white/85 hover:text-white" : "text-slate-600 hover:text-brand"}`}
        >
          {$t.nav.howItWorks}
        </a>
        <a
          href="/find"
          class={`text-sm font-semibold transition lg:text-base ${clear ? "text-white/85 hover:text-white" : "text-slate-600 hover:text-brand"} ${$page.url.pathname === "/find" ? (clear ? "!text-white font-bold" : "!text-brand font-bold") : ""}`}
        >
          {$t.nav.preferences}
        </a>
        <a
          href="/results"
          class={`text-sm font-semibold transition lg:text-base ${clear ? "text-white/85 hover:text-white" : "text-slate-600 hover:text-brand"} ${$page.url.pathname === "/results" ? (clear ? "!text-white font-bold" : "!text-brand font-bold") : ""}`}
        >
          {$t.nav.explore}
        </a>
      {/if}
    </nav>

    <!-- Right actions -->
    <div class="flex items-center gap-2.5 sm:gap-3">
      {#if isAdminUser}
        <a
          href="/admin"
          class={`hidden h-9 items-center gap-1.5 border px-3.5 text-xs font-bold transition sm:inline-flex ${
            clear
              ? "border-white bg-white text-ink hover:border-brand hover:bg-brand hover:text-white"
              : "border-forest bg-forest text-emerald-300 hover:border-brand hover:bg-brand hover:text-white"
          }`}
        >
          <ArrowLeft size={14} />
          <span>{$t.nav.backToDashboard}</span>
        </a>
      {/if}

      <LangSwitcher {clear} />

      {#if activeUser}
        <span
          class={`mono text-xs hidden max-w-[130px] truncate font-bold ${
            isAdminUser ? "xl:inline" : "sm:inline"
          }`}
          >{activeUser.name}</span
        >
        <button
          on:click={logout}
          aria-label={$t.nav.signOut}
          class={`grid h-9 w-9 place-items-center border transition ${clear ? "border-white/40 hover:bg-white hover:text-ink" : "border-line hover:bg-ink hover:text-white"}`}
        >
          <LogOut size={16} />
        </button>
      {:else}
        <a
          href="/login"
          class={`mono hidden sm:inline-block px-5 py-2.5 text-xs sm:text-[13px] uppercase tracking-wider font-bold transition ${clear ? "bg-white text-ink hover:bg-brand hover:text-white" : "bg-ink text-white hover:bg-brand"}`}
        >
          Login/Sign In
        </a>
      {/if}

      <!-- Mobile menu toggle button -->
      <button
        type="button"
        on:click={() => (mobileMenuOpen = !mobileMenuOpen)}
        aria-label="Toggle navigation menu"
        class={`grid h-9 w-9 place-items-center border md:hidden transition ${clear ? "border-white/40 text-white" : "border-line text-ink"}`}
      >
        {#if mobileMenuOpen}
          <X size={18} />
        {:else}
          <Menu size={18} />
        {/if}
      </button>
    </div>
  </div>

  <!-- Mobile dropdown menu -->
  {#if mobileMenuOpen}
    <div
      class="border-b border-line bg-canvas/98 px-5 py-6 text-ink shadow-xl backdrop-blur md:hidden"
    >
      <div class="flex flex-col space-y-4">
        {#if activeUser}
          <div class="border-b border-line pb-3">
            <p class="text-xs text-slate-soft uppercase tracking-wider">
              Masuk sebagai
            </p>
            <p class="font-bold text-ink text-base">{activeUser.name}</p>
          </div>
          <a
            href="/home"
            class="text-base font-semibold py-1.5 hover:text-brand"
            >{$t.nav.home}</a
          >
          <a
            href="/find"
            class="text-base font-semibold py-1.5 hover:text-brand"
            >{$t.nav.preferences}</a
          >
          <a
            href="/results"
            class="text-base font-semibold py-1.5 hover:text-brand"
            >{$t.nav.explore}</a
          >
          <a
            href="/wishlist"
            class="text-base font-semibold py-1.5 hover:text-brand"
            >{$t.nav.wishlist}</a
          >
          {#if isAdminUser}
            <a
              href="/admin"
              class="mt-1 flex items-center justify-center gap-2 bg-forest py-3 text-center text-sm font-bold text-emerald-300 shadow-sm transition hover:bg-brand hover:text-white"
            >
              <ArrowLeft size={16} />
              {$t.nav.backToDashboard}
            </a>
          {/if}
          <button
            type="button"
            on:click={logout}
            class="flex items-center gap-2 border-t border-line pt-4 text-left font-bold text-red-600"
          >
            <LogOut size={16} />
            {$t.nav.signOut}
          </button>
        {:else}
          <a
            href="/#how"
            class="text-base font-semibold py-1.5 hover:text-brand"
            >{$t.nav.howItWorks}</a
          >
          <a
            href="/find"
            class="text-base font-semibold py-1.5 hover:text-brand"
            >{$t.nav.preferences}</a
          >
          <a
            href="/results"
            class="text-base font-semibold py-1.5 hover:text-brand"
            >{$t.nav.explore}</a
          >
          <a
            href="/login"
            class="mt-2 block bg-brand py-3 text-center text-base font-bold text-white shadow-sm"
          >
            Login/Sign In
          </a>
        {/if}
      </div>
    </div>
  {/if}
</header>

{#if isAdminUser && !mobileMenuOpen && $page.url.pathname !== "/login"}
  <a
    href="/admin"
    class="fixed bottom-5 right-5 z-40 inline-flex items-center gap-2 border border-emerald-300/30 bg-forest px-4 py-3 text-xs font-bold text-emerald-300 shadow-xl transition hover:bg-brand hover:text-white sm:hidden"
  >
    <ArrowLeft size={15} />
    <span>{$t.nav.backToDashboard}</span>
  </a>
{/if}
