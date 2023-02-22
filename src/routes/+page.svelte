<script>
	import { parse } from 'fast-xml-parser';
	import { decode } from 'html-entities';

	import Player from './Player.svelte';
	import Transcripts from './Transcripts.svelte';

	const parserOptions = {
		attributeNamePrefix: '@_',
		//attrNodeName: false,
		//textNodeName : "#text",
		ignoreAttributes: false,
		ignoreNameSpace: false,
		attrValueProcessor: (val, attrName) => decode(val), //default is a=>a
		tagValueProcessor: (val, tagName) => decode(val) //default is a=>a
	};

	let feed = {};
	let selectedEpisode = {};
	let item = [];
	let searchResults = [];
	let searchQuery;
	let searchInput = '';
	let player;
	let transcriptIndex = 0;
	let episodeTranscript;

	let feedUrl = '';
	feedUrl = 'https://feeds.noagendaassets.com/noagenda.xml';
	searchInput = 'curio caster';

	function searchTranscripts() {
		searchResults = [];
		selectedEpisode = {};
		searchQuery = searchInput;
		for (const [i, v] of feed.item.entries()) {
			let results = [...v?.fetchedTranscript.matchAll(new RegExp(searchQuery, 'gi'))].map(
				(a) => a.index
			);
			if (results.length) {
				searchResults.push([i, results]);
			}
		}
	}

	async function fetchTranscript() {
		let res = await fetch(feedUrl);
		let data = await res.text();
		let xml2Json = parse(data, parserOptions);

		let xmlJson = xml2Json;

		feed = xmlJson.rss.channel;
		let _item = [].concat(feed.item);
		for (const [i, v] of _item.entries()) {
			if (v['podcast:transcript']) {
				let res = await fetch(v['podcast:transcript']['@_url']);
				let data = await res.text();
				_item[i].fetchedTranscript = data;
			}
		}

		feed.item = _item;
	}
</script>

<main>
	<header>
		<support>
			<p>Support Transcript Search Tool</p>
			<p>
				Install
				<a
					href="https://chrome.google.com/webstore/detail/alby-bitcoin-lightning-wa/iokeahhehimjnekafflcihljlcjccdbe"
				>
					<img src="/alby-small.png" alt="alby logo" class="alby-logo" />
					Alby
				</a>
				today
			</p>
		</support>

		<fetch-feed>
			<input bind:value={feedUrl} />
			<button on:click={fetchTranscript}>Get Feed</button>
		</fetch-feed>
		<p class="alby-address">⚡ transciptsearchtool@getalby.com</p>
	</header>

	<h1>{feed?.title || ''}</h1>
	{#if feed?.title}
		<search-transcripts>
			<input bind:value={searchInput} />
			<button on:click={searchTranscripts}>Search Transcripts</button>
		</search-transcripts>
		{#if searchResults?.length}
			<pane-container>
				<left-pane>
					<h3>Episodes</h3>
					<ul>
						{#each searchResults || [] as result}
							<li
								on:click={() => {
									selectedEpisode = feed.item[result[0]];
									player.src = selectedEpisode.enclosure['@_url'];
								}}
							>
								{feed.item[result[0]].title} - {result[1].length} occurrence{`${
									result[1].length > 1 ? 's' : ''
								}`}
							</li>
						{/each}
					</ul>
				</left-pane>
				<right-pane>
					<Transcripts
						episode={selectedEpisode}
						{searchQuery}
						{player}
						bind:transcriptIndex
						bind:episodeTranscript
					/>
				</right-pane>
			</pane-container>
			<Player bind:player bind:transcriptIndex bind:episodeTranscript />
		{/if}
	{/if}
</main>

<style>
	main {
		display: flex;
		flex-direction: column;
		justify-content: center;
	}

	header {
		width: 100%;
		display: flex;
		align-items: center;
		justify-content: space-between;
	}

	fetch-feed,
	search-transcripts {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 100%;
	}

	h1 {
		text-align: center;
	}

	input {
		width: 50%;
	}
	pane-container {
		display: flex;
		justify-content: space-between;
		width: 100%;
		height: calc(100vh - 230px);
		overflow: hidden;
		margin-bottom: 16px;
	}

	li {
		list-style: none;
		padding: 8px 0;
		color: blue;
		text-decoration: underline;
		cursor: pointer;
	}
	left-pane,
	right-pane {
		width: 50%;
		overflow: auto;
		height: 100%;
	}
	support {
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	p {
		margin: 0;
		white-space: nowrap;
	}
	.alby-logo {
		width: 20px;
	}
</style>