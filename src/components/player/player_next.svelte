<script>
    import { MediaPlayer } from "~/stores/elements.js";
    import { CurrentMedia, IsMobile, NowPlayingQueue } from "~/stores/state.js";
    import MaterialSymbol from "~/components/materialSymbol.svelte";
    import GenericCard from "~/components/cards/genericCard.svelte";

    let dropdown;
    let nextItem;
    let timeout;

    $: $CurrentMedia, update();
    $: $NowPlayingQueue, update();

    function update() {
        nextItem = $MediaPlayer?.findViableItem("next");
    }

    function handleOver() {
        timeout = setTimeout(() => {
            dropdown.show();
        }, 600);
    }

    function handleOut() {
        clearTimeout(timeout);
        dropdown.hide();
    }
</script>

<sl-dropdown bind:this={dropdown} on:pointerleave={handleOut} placement="top">
    <sl-button
        disabled={!nextItem}
        on:click={() => $MediaPlayer.next()}
        on:pointerover={handleOver}
        slot="trigger"
    >
        <MaterialSymbol name="skip_next" />
    </sl-button>

    {#if nextItem && !$IsMobile}
        <sl-card>
            <GenericCard
                type={nextItem.object_type}
                data={nextItem}
                showActions={false}
            />
        </sl-card>
    {/if}
</sl-dropdown>

<style>
    sl-button :global(*) {
        pointer-events: none;
    }

    sl-card {
        max-width: 400px;
    }

    sl-card::part(body) {
        padding: var(--spacing-sm);
        padding-inline-end: var(--spacing-md);
    }
</style>
