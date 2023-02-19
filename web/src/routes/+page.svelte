<script lang="ts">
    import { nip19 } from 'nostr-tools'

    let profile: Profile = {
        display_name: '',
        name: '',
        nip05: '',
        picture: '',
        website: '',
    };
    let npub = '';
    let pubkey = '';
    let nprofile = '';
    let nprofileJson = '';
    let relays: Relay[] = [];

    const defaultRelays = [
        'wss://relay.damus.io',
        'wss://relay-jp.nostr.wirednet.jp',
        'wss://nostr-relay.nokotaro.com',
        'wss://nostr.h3z.jp',
        'wss://nostr.holybea.com',
        'wss://relay.nostr.or.jp',
    ];

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
                        'kinds': [ 0, 3, 10002 ],
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

    async function showProfile() {
        console.log(npub);
        const { type, data } = nip19.decode(npub);
        pubkey = data as string;
        console.log(type, pubkey);
        if (typeof pubkey !== 'string') {
            return;
        }
        console.log(nip19.nprofileEncode({ pubkey, relays: defaultRelays }));
        const messages = await getMessagesFromRelays(pubkey);

        // Profile
        const profileMessages = messages.filter(x => x.kind === 0);
        profileMessages.sort((x, y) => x.created_at - y.created_at);
        const latestProfileMessage = profileMessages.find(_ => true);
        if (latestProfileMessage !== undefined) {
            console.log(latestProfileMessage);
            profile = JSON.parse(latestProfileMessage.content);
            console.log(profile);
        }

        // Relays
        const regacyRelaysMessages = messages.filter(x => x?.kind === 3);
        regacyRelaysMessages.sort((x, y) => x.created_at - y.created_at);
        const latestRegacyRelaysMessage = regacyRelaysMessages.find(_ => true);
        if (latestRegacyRelaysMessage !== undefined) {
            const regacyRelaysData = JSON.parse(latestRegacyRelaysMessage.content);
            console.log(regacyRelaysData);
            const regacyRelays: [string, RelayMeta][] = Object.entries(regacyRelaysData);
            relays = regacyRelays.map(([ url, meta ]) => { return { url: new URL(url), read: meta.read, write: meta.write }; })
        }

        const nprofileData = {
            pubkey,
            relays: relays.map(x => x.url.toString()),
        };
        nprofile = nip19.nprofileEncode(nprofileData);
        nprofileJson = JSON.stringify(nprofileData, null, 2);
    };

    interface Event {
        id: string,
        pubkey: string,
        created_at: number,
        kind: number,
        tags: any[],
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
</script>

<main>
    <h1>nprofile</h1>

    <form on:submit|preventDefault={showProfile}>
        <input type="text" bind:value={npub} placeholder="npub..." size="100" required>
        <input type="submit" value="Show">
    </form>
    
    {#if pubkey}
    <section class="profile">
        <div>
            <img src="{profile.picture}" alt="">
        </div>
        <div>{profile.display_name}</div>
        <div>{profile.name}</div>
        <div>{profile.nip05}</div>
        <div>{profile.website}</div>
        <div>{npub}</div>
        <div>{pubkey}</div>
        <div class="nprofile">{nprofile}</div>
        <pre class="nprofile">{nprofileJson}</pre>
    </section>
    {/if}

    {#if relays}
    <section class="relays">
        <ul>
            {#each relays as relay}
                <li>
                    <article>
                        <div>{relay.url}</div>
                        <div>{relay.read}</div>
                        <div>{relay.write}</div>
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
