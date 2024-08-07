<script>
    import { _ } from "svelte-i18n";
    import { getContext, onMount, setContext } from "svelte";
    import { v4 as uuidv4 } from "uuid";
    import { API } from "~/stores/state.js";
    import { sampleSize } from "lodash-es";
    import { addAlert } from "~/logic/alert.js";
    import {
        getSomeSongsByGenre,
        getSomeSongsFromAlbumsByGenre,
        getSomeSongsFromArtistsByGenre,
        getSongsByYear,
        getSongsFromAlbums,
        getSongsFromAlbumsStartingWith,
        getSongsFromArtist,
        getSongsFromArtists,
        getSongsFromArtistsStartingWith,
        getSongsFromPlaylist,
        getSongsFromPlaylists,
    } from "~/logic/song.js";
    import { errorHandler, prepareForQueue } from "~/logic/helper.js";
    import { writable } from "svelte/store";
    import ActionPlay from "./items/actionPlay.svelte";
    import ActionPlayNext from "./items/actionPlayNext.svelte";
    import ActionPlayLast from "./items/actionPlayLast.svelte";
    import ActionShuffle from "./items/actionShuffle.svelte";
    import ActionShuffleNext from "./items/actionShuffleNext.svelte";
    import ActionShuffleLast from "./items/actionShuffleLast.svelte";
    import ActionArtistMix from "./items/actionArtistMix.svelte";
    import ActionAddToPlaylist from "./items/actionPlaylistAddTo.svelte";
    import ActionEditPlaylist from "./items/actionPlaylistEdit.svelte";
    import ActionDeletePlaylist from "./items/actionPlaylistDelete.svelte";
    import ActionUpdateFromTags from "./items/actionUpdateFromTags.svelte";
    import ActionUpdateArt from "./items/actionUpdateArt.svelte";
    import ActionDownload from "./items/actionDownload.svelte";
    import ActionFindDuplicates from "./items/actionFindDuplicates.svelte";
    import ActionShare from "./items/actionShare.svelte";
    import MaterialSymbol from "~/components/materialSymbol.svelte";
    import Portal from "~/components/portal.svelte";
    import { PlaySongsByOtherArtists } from "~/stores/settings.js";

    export let item = null;
    export let type; // artist, album, playlist, song etc
    export let displayMode; // menu, miniButtons or fullButtons; the UI elements to be rendered
    export let showShuffle = false;
    export let showLinks = false;
    export let hidden = false;
    export let data = {}; // any extra data needed, passed as an object key e.g. data.playlists

    const contextKey = uuidv4(); // unique key for each instance of actions

    let playLimit = 5000;
    let actionsBind;
    let dialogBind;
    let dropdownBind;

    // underscore prefixed items are accessor aliases of exported params
    let _item = writable(item);
    let _type = writable(type);
    let _displayMode = writable(displayMode);
    let _showShuffle = writable(showShuffle);
    let _data = writable(data);

    setContext(contextKey, {
        _item,
        _type,
        _displayMode,
        _data,
        _showShuffle,
        getSongs: () => doFetch(),
    });

    let filterToArtistID = getContext("filterToArtistID");

    /**
     * Determine which method should be used to get songs
     * @returns function
     */
    async function determineFetchURL() {
        let fetchURL = null;

        switch (type) {
            case "artist":
                fetchURL = getSongsFromArtist(item?.id);
                break;
            case "artists":
                fetchURL = getSongsFromArtists(sampleSize(data.artists, 100));
                break;
            case "artistAlpha":
                fetchURL = getSongsFromArtistsStartingWith(data.char);
                break;
            case "artistGenre":
            case "genre":
                fetchURL = getSomeSongsFromArtistsByGenre(data.id);
                break;
            case "album":
                fetchURL = $API.albumSongs({ filter: item?.id });
                break;
            case "albums":
                fetchURL = getSongsFromAlbums(sampleSize(data.albums, 100));
                break;
            case "albumGenre":
                fetchURL = getSomeSongsFromAlbumsByGenre(data.id);
                break;
            case "albumAlpha":
                fetchURL = getSongsFromAlbumsStartingWith(data.char);
                break;
            case "song":
            case "playlist_songs":
                fetchURL = $API.song({ filter: item?.id });
                break;
            case "songGenre":
                fetchURL = getSomeSongsByGenre(data.id);
                break;
            case "playlist":
            case "smartlist":
                let playlistInfo = await $API.playlist({ filter: item?.id });
                let playlistLimit =
                    playlistInfo?.items > playLimit ? playLimit : 0;

                if (playlistLimit > 0) {
                    addAlert({
                        title: $_("text.limitedItems", {
                            values: { n: playlistLimit },
                        }),
                        style: "info",
                    });
                }

                fetchURL = $API.playlistSongs({
                    filter: item?.id,
                    limit: playlistLimit,
                });
                break;
            case "playlists":
                fetchURL = getSongsFromPlaylists(data.playlists);
                break;
            case "year":
                fetchURL = getSongsByYear(data.from, data.to);
                break;
            case "artistMix":
                fetchURL = getSongsFromPlaylist({
                    id: item?.id,
                    type: "artist_mix",
                });
                break;
            case "live_stream":
                fetchURL = $API.liveStream({ filter: item?.id });
                break;
            case "podcast":
                fetchURL = $API.podcastEpisodes({ filter: item?.id });
                break;
            case "podcast_episode":
                fetchURL = $API.podcastEpisode({ filter: item?.id });
                break;
            default:
                break;
        }

        return fetchURL;
    }

    /**
     * Gets the songs based on action type
     * @returns Promise<array>
     */
    async function doFetch() {
        let songSubset = data.songs;

        if (songSubset?.length > playLimit) {
            addAlert({
                title: $_("text.limitedItems", { values: { n: playLimit } }),
                style: "info",
            });
            songSubset = sampleSize(data.songs, playLimit);
        }

        let result = data.songs ? songSubset : await determineFetchURL();

        if (result.error) {
            errorHandler("determining fetch URL", result.error);
            return [];
        }

        // make sure we are working with an array
        result = Array.isArray(result) ? result : [result];

        // filter out pending podcast episodes...
        result = result.filter(
            (item) => !item.state || item.state === "completed",
        );

        // filter out songs not by artist if that is set, but allow a single item through with the assumption its the only one we want to play
        if (
            result.length > 1 &&
            $PlaySongsByOtherArtists === "exclude" &&
            filterToArtistID
        ) {
            result = result.filter((item) =>
                item.artists.find((artist) => artist.id === filterToArtistID),
            );
        }

        return prepareForQueue(result);
    }
</script>

<!-- Dynamic context menu -->
{#if displayMode === "menu"}
    <sl-menu>
        {#if showLinks && item}
            <sl-menu-item>
                {$_("text.goTo")}
                <sl-menu slot="submenu">
                    {#if type === "live_stream"}
                        <a href="#/radio-station/{item.id}">
                            <sl-menu-item>
                                <MaterialSymbol name="radio" slot="prefix" />
                                {item.name}
                            </sl-menu-item>
                        </a>
                    {:else if type === "podcast_episode"}
                        <a href="#/podcast-episode/{item.id}">
                            <sl-menu-item>
                                <MaterialSymbol name="podcast" slot="prefix" />
                                {item.name}
                            </sl-menu-item>
                        </a>
                    {:else if type === "song"}
                        <a href="#/song/{item.id}">
                            <sl-menu-item>
                                <MaterialSymbol
                                    name="music_note"
                                    slot="prefix"
                                />
                                {item.name}
                            </sl-menu-item>
                        </a>
                    {:else if type === "album"}
                        <a href="#/album/{item.id}">
                            <sl-menu-item>
                                <MaterialSymbol name="album" slot="prefix" />
                                {item.name}
                            </sl-menu-item>
                        </a>
                    {:else if type === "playlist"}
                        <a href="#/playlist/{item.id}">
                            <sl-menu-item>
                                <MaterialSymbol
                                    name="queue_music"
                                    slot="prefix"
                                />
                                {item.name}
                            </sl-menu-item>
                        </a>
                    {:else if type === "artistMix"}
                        <a href="#/mix/artist/{item.id}">
                            <sl-menu-item>
                                <MaterialSymbol
                                    name="queue_music"
                                    slot="prefix"
                                />
                                {item.name}
                            </sl-menu-item>
                        </a>
                    {/if}

                    <!-- PARENT ITEMS -->

                    {#if item.album?.id}
                        <a href="#/album/{item.album.id}">
                            <sl-menu-item>
                                <MaterialSymbol name="album" slot="prefix" />
                                {item.album.name}
                            </sl-menu-item>
                        </a>
                    {/if}

                    {#if item.podcast?.id}
                        <a href="#/podcast/{item.podcast.id}">
                            <sl-menu-item>
                                <MaterialSymbol
                                    name="swap_calls"
                                    slot="prefix"
                                />
                                {item.podcast.name}
                            </sl-menu-item>
                        </a>
                    {/if}

                    {#if item.artists?.length > 0}
                        {#each item.artists as artist}
                            <a href="#/artist/{artist.id}">
                                <sl-menu-item>
                                    <MaterialSymbol
                                        name="person"
                                        slot="prefix"
                                    />
                                    {artist.name}
                                </sl-menu-item>
                            </a>
                        {/each}
                    {/if}

                    <!-- add album artist if not already in artists -->
                    {#if item.albumArtist?.id && item.artists?.length > 0 && !item.artists.find((artist) => artist.id === item.albumArtist.id)}
                        <a href="#/artist/{item.albumArtist.id}">
                            <sl-menu-item>
                                <MaterialSymbol name="person" slot="prefix" />
                                {item.albumArtist.name}
                            </sl-menu-item>
                        </a>
                    {/if}
                </sl-menu>
            </sl-menu-item>
        {/if}

        {#if type === "playlist"}
            <ActionEditPlaylist {contextKey} />
            <ActionDeletePlaylist {contextKey} />
        {/if}

        {#if type === "song"}
            <ActionArtistMix {contextKey} />
        {/if}

        {#if type === "song"}
            <ActionShare {contextKey} />
        {/if}

        <ActionAddToPlaylist {contextKey} />
        <ActionShuffleNext {contextKey} />
        <ActionShuffleLast {contextKey} />

        {#if type === "artist" || type === "album" || type === "song"}
            <ActionUpdateFromTags {contextKey} />
        {/if}

        {#if type === "artist" || type === "album"}
            <ActionUpdateArt {contextKey} />
        {/if}

        {#if type === "song"}
            <ActionDownload {contextKey} />
        {/if}

        {#if type === "song"}
            <ActionFindDuplicates {item} />
        {/if}
    </sl-menu>
{/if}

<!-- Card/lister actions which are always the same -->
{#if displayMode === "miniButtons" || displayMode === "fullButtons"}
    <div class="c-actions {displayMode}" bind:this={actionsBind}>
        <sl-button-group>
            {#if !hidden}<ActionPlay {contextKey} />{/if}
            {#if !hidden}<ActionShuffle {contextKey} />{/if}
            {#if !hidden}<ActionPlayNext {contextKey} />{/if}
            {#if !hidden}<ActionPlayLast {contextKey} />{/if}

            <sl-dropdown hoist bind:this={dropdownBind}>
                <!-- button not shown, only used for positioning the dropdown -->
                <sl-button
                    slot="trigger"
                    size={displayMode === "miniButtons" ? "small" : "medium"}
                    style="visibility: hidden; pointer-events: none; width: 0;"
                >
                    &nbsp;
                </sl-button>
                <svelte:self {...$$props} displayMode="menu" />
            </sl-dropdown>

            <sl-button
                size={displayMode === "miniButtons" ? "small" : "medium"}
                on:click={(e) => {
                    let tabulator = actionsBind?.closest(".tabulator");
                    if (tabulator?.offsetHeight < 250) {
                        dialogBind?.show();
                    } else {
                        dropdownBind?.show();
                    }
                }}
            >
                <MaterialSymbol name="more_horiz" />
            </sl-button>
        </sl-button-group>
    </div>

    <!-- used when Tabulator is too short to show dropdown -->
    <Portal>
        <sl-dialog
            bind:this={dialogBind}
            on:click={() => {
                dialogBind?.hide();
            }}
        >
            <svelte:self {...$$props} displayMode="menu" />

            <sl-divider></sl-divider>

            <sl-alert open>
                <MaterialSymbol name="help" slot="icon" />
                <strong>Why was this not a dropdown menu?</strong>
                <p>
                    There was not enough room to display the dropdown, due to
                    technical reasons (CSS containing block) nothing can appear
                    outside of the table area.
                </p>

                <p>
                    This is a temporary workaround until the UI library can
                    implement the new Popover API.
                </p>
            </sl-alert>

            <sl-button
                slot="footer"
                variant="primary"
                on:click={() => {
                    dialogBind?.hide();
                }}
            >
                Close
            </sl-button>
        </sl-dialog>
    </Portal>
{/if}

<style>
    /* Hide the label when miniButtons */
    .c-actions.miniButtons :global(.label) {
        display: none;
    }

    sl-button-group::part(base) {
        flex-wrap: wrap;
    }
</style>
