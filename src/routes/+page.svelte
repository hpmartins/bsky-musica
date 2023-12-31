<script lang="ts">
    import { applyAction, deserialize, enhance } from '$app/forms';
    import type { ActionResult, error } from '@sveltejs/kit';
    import type { ActionData, PageServerData } from './$types';
    import { invalidateAll } from '$app/navigation';
    import { dev } from '$app/environment';
    import { PUBLIC_DEVEL_LOGIN, PUBLIC_DEVEL_PWD } from '$env/static/public';
    import Rect from './Rect.svelte';
    import type { CirclesOptionsType } from '$lib/types';
    import Preview from './Preview.svelte';

    let imageOptions: CirclesOptionsType = {
        orbits: 2,
        add_watermark: false,
        add_date: true,
        bg_color: '#1D428A',
        add_border: true,
        border_color: '#FFC72C'
    };

    export let form: ActionData;

    export let data: PageServerData;
    $: user = data.user;
    $: lastfm = data.lastfm;
    $: spotify = data.spotify;

    let loginError = false;

    async function handleLogin(event: { currentTarget: EventTarget & HTMLFormElement }) {
        const action = event.currentTarget.action;
        const data = new FormData(event.currentTarget);
        let identifier = data.get('identifier');
        let password = data.get('password');

        loginError = false;

        if (dev) {
            identifier = PUBLIC_DEVEL_LOGIN;
            password = PUBLIC_DEVEL_PWD;
        }

        const auth = await fetch('https://bsky.social/xrpc/com.atproto.server.createSession', {
            method: 'POST',
            body: JSON.stringify({
                identifier: identifier,
                password: password
            }),
            headers: {
                'Content-Type': 'application/json'
            }
        }).then((res) => {
            if (res.ok) {
                return res.json();
            } else {
                loginError = true;
            }
        });

        const response = await fetch(action, {
            method: 'POST',
            body: JSON.stringify({
                did: auth.did,
                handle: auth.handle,
                accessJwt: auth.accessJwt,
                refreshJwt: auth.refreshJwt,
                didDoc: auth.didDoc
            })
        });

        const result: ActionResult = deserialize(await response.text());

        if (result.type === 'success') {
            // rerun all `load` functions, following the successful update
            await invalidateAll();
        }

        applyAction(result);
    }
</script>

<div class="flex flex-col items-center gap-1 w-full max-w-4xl">
    <form method="POST">
        <div class="flex flex-col justify-center items-center gap-4">
            <div>
                {#if lastfm}
                    <select name="lastfmLimit" id="lastfmLimit" class="select select-bordered max-w-xs">
                        <option value={5}>5</option>
                        <option value={10}>10</option>
                        <option value={15}>15</option>
                        <option value={20}>20</option>
                        <option value={25}>25</option>
                    </select>
                    <select name="lastfmType" id="lastfmType" class="select select-bordered max-w-xs">
                        <!-- <option value="albums">Top albums</option> -->
                        <option value="artists">Top artists</option>
                        <!-- <option value="top_tracks">Top tracks</option> -->
                    </select>
                    <select name="lastfmTimeRange" id="lastfmTimeRange" class="select select-bordered max-w-xs">
                        <option value="7day">7 days</option>
                        <option value="1month">1 month</option>
                        <option value="3month">3 months</option>
                        <option value="6month">6 months</option>
                        <option value="12month">12 months</option>
                        <option value="overall">Overall</option>
                    </select>
                    <div class="join">
                        <button type="submit" class="btn btn-primary join-item" formaction="?/lastfmPreview"
                            >preview last.fm</button
                        >
                        <button class="btn btn-primary join-item" formaction="?/lastfmUnlink">x</button>
                    </div>
                {:else}
                    <button class="btn btn-primary" formaction="?/lastfmLink"> link last.fm </button>
                {/if}
            </div>
            <div>
                {#if spotify}
                    <select name="spotifyLimit" id="spotifyLimit" class="select select-bordered max-w-xs">
                        <option value={5}>5</option>
                        <option value={10}>10</option>
                        <option value={15}>15</option>
                        <option value={20}>20</option>
                        <option value={25}>25</option>
                    </select>
                    <select name="spotifyType" id="spotifyType" class="select select-bordered max-w-xs">
                        <option value="artists">Top artists</option>
                    </select>
                    <select name="spotifyTimeRange" id="spotifyTimeRange" class="select select-bordered max-w-xs">
                        <option value="short_term">Last 4 weeks</option>
                        <option value="medium_term">Last 6 months</option>
                        <option value="long_term">Overall</option>
                    </select>
                    <div class="join">
                        <button type="submit" class="btn btn-primary join-item" formaction="?/spotifyPreview"
                            >preview spotify</button
                        >
                        <button class="btn btn-primary join-item" formaction="?/spotifyUnlink">x</button>
                    </div>
                {:else}
                    <button type="submit" class="btn btn-primary" formaction="?/spotifyLink">link spotify</button>
                {/if}
            </div>
        </div>
    </form>
    {#if form?.query}
        <div class="flex flex-col justify-center items-center gap-4">
            <div class="grid grid-cols-1 md:grid-cols-1 gap-4 justify-center p-4 mt-3">
                <div>
                    Options:
                    <label class="label cursor-pointer gap-x-3 justify-start">
                        <input
                            class="checkbox checkbox-sm checkbox-secondary"
                            type="checkbox"
                            bind:checked={imageOptions.add_date}
                        />
                        <span class="label-text">Add date</span>
                    </label>
                    <label class="label cursor-pointer gap-x-3 justify-start">
                        <input
                            class="checkbox checkbox-sm checkbox-secondary"
                            type="checkbox"
                            bind:checked={imageOptions.add_watermark}
                            disabled
                        />
                        <span class="label-text">Add watermark</span>
                    </label>
                </div>
                <div>
                    <div>
                        Background color:
                        <input type="color" style="width:100%;" bind:value={imageOptions.bg_color} />
                    </div>
                    <div>
                        <label class="flex items-center space-x-2">
                            <input
                                class="checkbox checkbox-sm checkbox-secondary"
                                type="checkbox"
                                bind:checked={imageOptions.add_border}
                            />
                            <p>Border color:</p>
                        </label>
                        <input type="color" style="width:100%;" bind:value={imageOptions.border_color} />
                    </div>
                </div>
            </div>
        </div>
        <Preview query={form.query} />
        <div>
            {#key imageOptions}
                <Rect query={form.query} options={imageOptions} />
            {/key}
        </div>
    {/if}

    <hr class="w-1/2 h-1 mx-auto my-4 bg-secondary border-0 rounded md:my-4" />

    {#if user}
        <div>
            soon lmao

            <form method="POST" action="?/logout" use:enhance>
                <button class="btn btn-sm btn-primary text-secondary"> logout </button>
            </form>
        </div>
    {:else}
        <div>
            <p class="text-left text-primary">Post or schedule posting to Bluesky:</p>
            <form
                method="POST"
                action="?/login"
                class="mt-2 flex max-w-2xl mx-auto flex-col gap-0"
                on:submit|preventDefault={handleLogin}
            >
                <input
                    type="text"
                    name="identifier"
                    id="identifier"
                    class="input input-sm input-bordered border-primary hover:border-secondary w-full max-w-md"
                    placeholder="your bluesky identifier"
                />
                <input
                    type="password"
                    name="password"
                    id="password"
                    class="input input-sm input-bordered border-primary hover:border-secondary w-full max-w-md"
                    placeholder="app password"
                />
                <div class="text-left">
                    <span class="text-xs text-primary">
                        Create an
                        <a class="link" target="_blank" href="https://bsky.app/settings/app-passwords">
                            app password here
                        </a>
                    </span>
                </div>
                <div class="mx-auto flex">
                    <button class="btn btn-sm btn-primary text-secondary"> Login </button>
                </div>
                {#if loginError}
                    <div role="alert" class="alert alert-warning">
                        <span class="text-sm">Login failed</span>
                    </div>
                {/if}
            </form>
        </div>
    {/if}
</div>
