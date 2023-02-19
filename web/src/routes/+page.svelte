<script lang="ts">
    import { nip19 } from 'nostr-tools';
    import { page } from '$app/stores';
	import { afterNavigate } from '$app/navigation';

    let input = '';
    let profile: Profile = {
        display_name: '',
        name: '',
        nip05: '',
        picture: '',
        website: '',
    };
    let npubProfile = '';
    let pubkeyProfile = '';
    let nprofile = '';
    let nprofileJson = '';
    let recommendedRelay = '';
    let relaysOf10002: string[] = [];
    let relaysOf3: Relay[] = [];
    let contacts: Contact[] = [];

    const defaultRelays = [
        'wss://relay.damus.io',
        'wss://relay-jp.nostr.wirednet.jp',
        'wss://nostr-relay.nokotaro.com',
        'wss://nostr.h3z.jp',
        'wss://nostr.holybea.com',
        'wss://relay.nostr.or.jp',
    ];

    afterNavigate(async () => {
        const p = $page.url.searchParams.get('p');
        if (p !== null) {
            console.log('[p]', p);
            input = p;
            await show();
        }
    });

    async function getMessages(relay: string, pubkey: string): Promise<Event[]> {
        return new Promise(async (resolve, reject) => {
            let messages: Event[] = [];

            // Timeout in 3 seconds
            setTimeout(() => {
                reject();
            }, 3000);

            const ws = new WebSocket(relay);
            ws.onerror = () => {
                console.error('Error');
                reject();
            };
            ws.onopen = () => {
                console.log('Opened');
                ws.send(JSON.stringify([
                    'REQ',
                    Math.floor(Math.random() * 99999).toString(),
                    {
                        'kinds': [ 0, 2, 3, 10002 ],
                        'authors': [ pubkey ],
                    },
                ]));
            };
            ws.onclose = () => {
                console.log('Closed');
            };
            ws.onmessage = event => {
                console.log('Message');
                const data = JSON.parse(event.data);
                console.log(data);
                switch (data[0]) {
                    case 'EOSE':
                        ws.close();
                        resolve(messages);
                        break;
                    case 'EVENT':
                        const e: Event = data[2];
                        messages.push(e);
                        break;
                }
            };
        });
    }

    async function getMessagesFromRelays(pubkey: string) {
        const messages = await Promise.all(defaultRelays.map(async relay => {
            try {
                return await getMessages(relay, pubkey);
            } catch (error) {
                console.error(error);
                return [];
            }
        }));
        return messages.flat();
    };

    async function show() {
        console.log('[input]', input);
        const npub = input.startsWith('npub') ? input : nip19.npubEncode(input);
        console.log(npub);
        const { type, data } = nip19.decode(npub);
        const pubkey = data as string;
        console.log(type, pubkey);
        if (typeof pubkey !== 'string') {
            return;
        }
        console.log(nip19.nprofileEncode({ pubkey, relays: defaultRelays }));
        const messages = await getMessagesFromRelays(pubkey);

        if (messages.length === 0) {
            console.warn('Not found');
            return;
        }

        npubProfile = npub;
        pubkeyProfile = pubkey;

        // Profile
        const profileMessages = messages.filter(x => x.kind === 0);
        profileMessages.sort((x, y) => x.created_at - y.created_at);
        const latestProfileMessage = profileMessages.find(_ => true);
        if (latestProfileMessage !== undefined) {
            console.log(latestProfileMessage);
            profile = JSON.parse(latestProfileMessage.content);
            console.log(profile);
        } else {
            console.error('Profile not found');
        }

        // Relays
        const recommendedRelayMessages = messages.filter(x => x.kind === 2);
        recommendedRelayMessages.sort((x, y) => x.created_at - y.created_at);
        const latestRecommendedRelay = recommendedRelayMessages.find(_ => true);
        if (latestRecommendedRelay !== undefined) {
            recommendedRelay = latestRecommendedRelay.content;
        } else {
            recommendedRelay = '';
        }

        const relaysMessagesOf10002 = messages.filter(x => x.kind === 10002);
        relaysMessagesOf10002.sort((x, y) => x.created_at - y.created_at);
        const latestRelaysMessageOf10002 = relaysMessagesOf10002.find(_ => true);
        if (latestRelaysMessageOf10002 !== undefined) {
            relaysOf10002 = latestRelaysMessageOf10002.tags.map(x => x[1]);
        } else {
            relaysOf10002 = [];
        }

        const relaysMessagesOf3 = messages.filter(x => x.kind === 3);
        relaysMessagesOf3.sort((x, y) => x.created_at - y.created_at);
        const latestRelaysMessageOf3 = relaysMessagesOf3.find(_ => true);
        if (latestRelaysMessageOf3 !== undefined) {
            const relaysData = JSON.parse(latestRelaysMessageOf3.content);
            console.log(relaysData);
            const relays: [string, RelayMeta][] = Object.entries(relaysData);
            relaysOf3 = relays.map(([ url, meta ]) => { return { url: new URL(url), read: meta.read, write: meta.write }; })
        } else {
            relaysOf3 = [];
        }

        const nprofileData = {
            pubkey,
            relays: relaysOf10002.length > 0 ? relaysOf10002 : relaysOf3.map(x => x.url.toString()),
        };
        nprofile = nip19.nprofileEncode(nprofileData);
        nprofileJson = JSON.stringify(nprofileData, null, 2);

        // Contact
        if (latestRelaysMessageOf3 !== undefined) {
            contacts = latestRelaysMessageOf3.tags.map(([ tag, pubkey, relay, petname ]) => {
                return {
                    npub: nip19.npubEncode(pubkey),
                    pubkey,
                    relay,
                    petname,
                };
            });
        } else {
            contacts = [];
        }
    };

    interface Event {
        id: string,
        pubkey: string,
        created_at: number,
        kind: number,
        tags: string[][],
        content: string,
        sig: string,
    }

    interface Profile {
        display_name: string,
        name: string,
        nip05: string,
        picture: string,
        website: string,
    }

    interface RelayMeta {
        read: boolean,
        write: boolean,
    }

    interface Relay {
        url: URL,
        read: boolean,
        write: boolean,
    }

    interface Contact {
        npub: string,
        pubkey: string,
        relay: string,
        petname: string,
    }
</script>

<main>
    <h1>nprofile</h1>

    <form>
        <input type="text" name="p" bind:value={input} placeholder="npub or hex" size="100" required>
        <input type="submit" value="Show">
    </form>
    
    {#if pubkeyProfile}
    <section class="profile">
        <h2>Profile</h2>
        <div>
            <img src="{profile.picture}" alt="">
        </div>
        <div>{profile.display_name}</div>
        <div>{profile.name}</div>
        <div>{profile.nip05}</div>
        <div>{profile.website}</div>
        <div>{npubProfile}</div>
        <div>{pubkeyProfile}</div>
        <div class="nprofile">{nprofile}</div>
        <pre class="nprofile">{nprofileJson}</pre>

        <div>Recommended relay: {recommendedRelay === '' ? '-' : recommendedRelay}</div>
    </section>

    <section class="relays">
        <h2>Relays (kind: 10002)</h2>
        <ul>
            {#each relaysOf10002 as relay}
                <li>
                    <article>
                        <div>{relay}</div>
                    </article>
                </li>
            {/each}
        </ul>
    </section>

    <section class="relays">
        <h2>Relays (kind: 3)</h2>
        <ul>
            {#each relaysOf3 as relay}
                <li>
                    <article>
                        <div>{relay.url}</div>
                        <div>Read: {relay.read}</div>
                        <div>Write: {relay.write}</div>
                    </article>
                </li>
            {/each}
        </ul>
    </section>

    <section class="contacts">
        <h2>Contacts</h2>
        <ul>
            {#each contacts as contact}
                <li>
                    <article>
                        <div><a href="?p={contact.npub}">{contact.npub}</a></div>
                        <div>{contact.pubkey}</div>
                        {#if contact.relay}
                        <div>{contact.relay}</div>
                        {/if}
                        {#if contact.petname}
                        <div>{contact.petname}</div>
                        {/if}
                    </article>
                </li>
            {/each}
        </ul>
    </section>
    {/if}
</main>

<style>
    main {
        max-width: 800px;
        margin: 0 auto;
    }

    img {
      width: 128px;
      height: 128px;
      border-radius: 50%;
      margin-right: 32px;
    }

    .profile div {
        margin: 0.2em auto;
    }

    .nprofile {
        overflow-wrap: break-word;
    }
</style>
