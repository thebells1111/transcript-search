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
	let searchResults = [];
	let podcastIndexSearchResults = [];
	let searchQuery;
	let searchInput = '';
	let player;
	let transcriptIndex = 0;
	let episodeTranscript;
	let isLoading = false;
	let isLoaded = false;

	let indexQuery = '';
	indexQuery = 'podcasting 2.0';

	function handleInput(e, cb) {
		if (e.key === 'Enter') {
			cb();
		}
	}

	async function searchPodcastIndex() {
		searchResults = [];
		feed = {};
		selectedEpisode = {};
		searchQuery = '';
		searchInput = '';
		let url = `api/queryindex?q=${encodeURIComponent(`/search/byterm?q=${indexQuery}`)}`;

		const res = await fetch(url);
		let data = await res.json();

		try {
			data = JSON.parse(data);
			data.feeds = data.feeds || [data.feed];
			podcastIndexSearchResults = data.feeds;
		} catch (error) {}
	}

	async function fetchTranscript(feedUrl) {
		podcastIndexSearchResults = [];
		isLoading = true;
		isLoaded = false;
		searchResults = [];
		feed = {};
		selectedEpisode = {};
		searchQuery = '';
		searchInput = '';
		searchInput = '"rad"';
		let res = await fetch('/api/proxy?q=' + encodeURIComponent(feedUrl));
		let data = await res.text();
		let xml2Json = parse(data, parserOptions);

		let xmlJson = xml2Json;

		feed = xmlJson.rss.channel;
		let _item = [].concat(feed.item);
		for (const [i, v] of _item.entries()) {
			if (v['podcast:transcript']) {
				let res = await fetch(
					'/api/proxy?q=' + encodeURIComponent(v['podcast:transcript']['@_url'])
				);
				let data = await res.text();
				_item[i].fetchedTranscript = data;
			}
		}

		feed.item = _item;
		isLoading = false;
		isLoaded = true;
	}

	function searchTranscripts(recall) {
		if (!recall) {
			selectedEpisode = {};
		}
		searchResults = [];
		searchQuery = searchInput || '';

		const numItems = feed.item.length;

		function search(string, searchQuery) {
			let regex;
			if (searchQuery.startsWith('"') && searchQuery.endsWith('"')) {
				const word = searchQuery.slice(1, -1);
				regex = new RegExp(`\\b${word}\\b[\\p{P}-]*`, 'gi');
			} else {
				regex = new RegExp(searchQuery, 'gi');
			}
			return string.match(regex) || [];
		}

		for (let i = 0; i < numItems; i++) {
			const v = feed.item[i];
			const results = search(v.fetchedTranscript, searchQuery);
			if (results.length) {
				searchResults.push([i, results]);
			}
		}
		if (isLoading) {
			setTimeout(searchTranscripts.bind(this, true), 500);
		}
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
			<input
				bind:value={indexQuery}
				placeholder="search for podcast"
				on:keypress={(e) => handleInput(e, searchPodcastIndex)}
			/>
			<button on:click={searchPodcastIndex}>Search Directory</button>
		</fetch-feed>
		<p class="alby-address">⚡ transcriptsearchtool@getalby.com</p>
	</header>

	{#if podcastIndexSearchResults.length}
		<ul>
			{#each podcastIndexSearchResults as feed}
				<li class="pi-result" on:click={fetchTranscript.bind(this, feed?.originalUrl)}>
					<img src={feed?.artwork || feed?.image} alt={feed?.title} width="40" height="40" />
					{feed?.title}
				</li>
			{/each}
		</ul>
	{/if}

	{#if feed?.title}
		<h1>{feed?.title || ''}</h1>
		<search-transcripts>
			<input
				bind:value={searchInput}
				placeholder={`search term  (put exact matches inside of double quotes, i.e. "rad"`}
				on:keypress={(e) => handleInput(e, searchTranscripts)}
			/>
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
					<cashapp>
						<div>
							<h3>Do you like this service?</h3>
							<h4>Consider using CashApp to <br /> help pay for development and hosting.</h4>
						</div>
						<img src="/$curiocaster.png" />
					</cashapp>
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
		{:else if !isLoading}
			<p>No Search Results Found</p>{/if}
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

	h3 {
		padding: 0 16px;
		margin: 8px 0;
	}

	input {
		width: 50%;
	}

	pane-container {
		display: flex;
		justify-content: space-between;
		width: 100%;
		height: calc(100vh - 250px);
		overflow: hidden;
		margin: 16px 0;
		border: 2px solid lightgray;
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

	right-pane {
		border-left: 1px solid lightgray;
	}
	left-pane {
		border-right: 1px solid lightgray;
	}
	left-pane ul {
		overflow: auto;
		height: calc(100% - 308px);
		border-bottom: 2px solid lightgray;
		margin: 8px 0 0 0;
	}

	cashapp {
		display: flex;
		height: 268px;
		justify-content: space-evenly;
	}

	cashapp h4 {
		margin: 0 0 12px 0;
		text-align: center;
	}
	cashapp h3 {
		text-align: center;
	}

	cashapp img {
		margin: 8px;
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

	.pi-result {
		display: flex;
		align-items: center;
	}

	.pi-result img {
		padding-right: 8px;
	}
</style>
