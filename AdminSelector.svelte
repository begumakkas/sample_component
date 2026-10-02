<script lang="ts">
    import * as NativeSelect from "$lib/components/ui/native-select/index.js";
    import { selectionsStore } from '$lib/stores/selections.js';
    import ChevronDownIcon from '@lucide/svelte/icons/chevron-down';
    import { get } from 'svelte/store';
    import { onMount } from 'svelte';

    /// when multiple=true, renders a multi-select checkbox dropdown
    let { multiple = false } = $props();

    let adminUnits = $state([]);
    let selectedAdminUnit = $state('');
    let selectedAdminUnits: number[] = $state([]);
    let previousCountry = $state(null);
    let open = $state(false);
    let loading = $state(false);
    let error = $state(null);
    let dropdownRef: HTMLDivElement;

    // fetch admin units for a given country
    async function fetchAdminUnits(countryId: number) {
        loading = true;
        error = null;
        try {
            const res = await fetch(`/api/adminunit/?country_id=${countryId}`);
            if (!res.ok) throw new Error('Failed to load admin units');
            adminUnits = await res.json();
        } catch (e) {
            error = e.message;
            console.error(e);
            adminUnits = [];
        } finally {
            loading = false;
        }
    }

    // re-fetch admin units when country changes and clear current selection
    $effect(() => {
        const countryId = $selectionsStore.country_id;
    if (countryId && countryId !== previousCountry) {
        fetchAdminUnits(countryId);
        selectedAdminUnit = ''; // clear admin unit when country changes
        selectedAdminUnits = [];
        previousCountry = countryId;
    }
    });

    function handleClickOutside(e: MouseEvent) {
        if (dropdownRef && !dropdownRef.contains(e.target as Node)) {
            open = false;
        }
    }

    // seed selector from store on mount (e.g. when navigating from saved analyses) or survey
    onMount(() => {
        const stored = get(selectionsStore);
        if (stored.country_id) {
            fetchAdminUnits(stored.country_id);
            previousCountry = stored.country_id;

            if (multiple) {
                // if coming from survey, seed the multi-select with the single selection
                if (stored.admin_unit_ids?.length > 0) {
                    selectedAdminUnits = stored.admin_unit_ids;
                } else if (stored.admin_unit_id) {
                    selectedAdminUnits = [Number(stored.admin_unit_id)];
                }
            } else {
                if (stored.admin_unit_id) selectedAdminUnit = String(stored.admin_unit_id);
            }
        }
        document.addEventListener('click', handleClickOutside);
        return () => document.removeEventListener('click', handleClickOutside);
    });

    // write selection changes back to the store
    $effect(() => {
    if (multiple) {
        if (selectedAdminUnits.length > 0) {
            selectionsStore.update(s => ({...s, admin_unit_ids: selectedAdminUnits }));
        }
    } else {
        if (selectedAdminUnit) {
            const name = adminUnits.find(u => u.id == selectedAdminUnit)?.name;
            selectionsStore.update(s => ({
                ...s,
                admin_unit_id: parseInt(selectedAdminUnit),
                admin_unit_name: name,
                results: null
            }));
        }
    }
    });

    // reset local state when store is cleared
    $effect(() => {
        if (!$selectionsStore.country_id) {
            if (selectedAdminUnits.length > 0) {
                selectedAdminUnits = [];
            }
            selectedAdminUnit = '';
            previousCountry = null;
        }
        if ($selectionsStore.admin_unit_ids.length === 0 && selectedAdminUnits.length > 0) selectedAdminUnits = [];
    })

    // toggle an admin unit in the multi-select list
    function toggleUnit(id: number) {
        if (selectedAdminUnits.includes(id)) {
            selectedAdminUnits = selectedAdminUnits.filter(u => u !== id);
        } else {
            selectedAdminUnits = [...selectedAdminUnits, id];
        }
    }

    // display label for the multi-select button
    function getLabel() {
        if (selectedAdminUnits.length === 0) return 'Select Regions';
        if (selectedAdminUnits.length === 1) {
            return adminUnits.find(u => u.id === selectedAdminUnits[0])?.name ?? '';
        }
        return `${selectedAdminUnits.length} regions selected`;
    }

</script>

{#if multiple}
<div class="relative w-48" bind:this={dropdownRef}>
    <button class="bg-input/50 placeholder:text-muted-foreground focus-visible:border-ring focus-visible:ring-ring/30 h-9 w-full min-w-0 appearance-none rounded-3xl border border-transparent py-1 pr-8 pl-3 text-sm transition-[color,box-shadow,background-color] select-none focus-visible:ring-3 outline-none cursor-pointer text-left"
    disabled={loading}
    onclick={() => open = !open}>
    {loading ? 'Loading...' : getLabel()}
    <ChevronDownIcon class="text-muted-foreground top-1/2 right-2.5 size-4 -translate-y-1/2 pointer-events-none absolute select-none" aria-hidden />
    </button>

    {#if open}
    <div class = "absolute z-10 mt-1 w-48 border border-border rounded-md bg-background shadow-md max-h-60 overflow-y-auto">
        {#each adminUnits as unit}
        <label class = "flex items-center gap-2 px-3 py-2 hover:bg-muted cursor-pointer text-sm">
            <input
                type="checkbox"
                checked = {selectedAdminUnits.includes(unit.id)}
                onchange={() => toggleUnit(unit.id)}
            />
            {unit.name}
        </label>
        {/each}
        {#if adminUnits.length === 0}
        <p class ="px-3 py-2 text-sm text-muted-foreground">Select a Country First</p>
        {/if}
    </div>
    {/if}

</div>

{:else}
<NativeSelect.Root class = "" bind:value={selectedAdminUnit} disabled={loading}>
    <NativeSelect.Option value="" selected disabled>
        {loading ? 'Loading...' : 'Select a Region'}
    </NativeSelect.Option>
    {#each adminUnits as unit}
        <NativeSelect.Option value={String(unit.id)}>{unit.name}</NativeSelect.Option>
    {/each}
</NativeSelect.Root>
{/if}
{#if error}
    <p class="text-sm text-destructive">{error}</p>
{/if}
