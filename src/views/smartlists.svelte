<script>
    import { _ } from "svelte-i18n";
    import { API, PageTitle, User } from "~/stores/state";
    import Tabulator from "~/components/lister/Tabulator.svelte";
    import { createQuery } from "@tanstack/svelte-query";
    import { errorHandler } from "~/logic/helper.js";
    import { smartlistsPreset } from "~/components/lister/columns.js";

    let title = $_("text.smartlists");
    $PageTitle = title;

    let tabulator = null;

    $: query = createQuery({
        queryKey: ["smartlists"],
        queryFn: async () => {
            let result = await $API.smartlists();

            if (result.error) {
                errorHandler("getting smartlists", result.error);
                return [];
            }

            return result;
        },
        enabled: $User.isLoggedIn,
    });

    // alias of returned data
    $: smartlists = $query.data || [];
</script>

<div class="page-header">
    <h1 class="page-title">{title}</h1>
</div>

{#if $query.isLoading}
    <p>{$_("text.loading")}</p>
{:else if $query.isError}
    <p>Error: {$query.error.message}</p>
{:else if $query.isSuccess}
    {#if smartlists.length === 0}
        <p>{$_("text.noItemsFound")}</p>
    {:else}
        <div class="lister-tabulator">
            <Tabulator
                bind:tabulator
                data={smartlists}
                columns={smartlistsPreset}
                options={{ persistenceID: "smartlists" }}
            ></Tabulator>
        </div>
    {/if}
{/if}
