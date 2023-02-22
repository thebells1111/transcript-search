<script>
	import parseSRT from 'parse-srt';
	import VirtualList from 'svelte-tiny-virtual-list';

	export let episode;
	export let searchQuery = '';
	export let player;
	export let transcriptIndex = 0;
	export let episodeTranscript;

	let scrollToIndex = 0;
	let currentIndex = 0;

	let listHeight = 0;
	let scrollToAlignment = 'start';

	let scrollStatus = 'Disabled';

	let filteredIndices = [];

	$: console.log(episode);

	$: if (episode && episode.fetchedTranscript) {
		console.log(episode['podcast:transcript']);
		const transcriptSRT = episode?.['podcast:transcript']?.['@_type'] === 'application/srt';

		if (transcriptSRT) {
			let transcript = parseSRT(episode.fetchedTranscript);
			let t = transcript
				.map((v) => v.text)
				.join(' ')
				.replace(/(<|&lt;)br\s*\/*(>|&gt;)/g, ' ');

			episodeTranscript = [...transcript];
			transcript.full = t.split('|-|').join(' ');
			searchTranscript(searchQuery);
		} else {
			episodeTranscript = null;
		}
	} else {
		episodeTranscript = null;
	}

	function jumpToSection(section, index) {
		if (section.start) {
			player.currentTime = section.start;
			transcriptIndex = index;
		}
	}

	$: if (transcriptIndex && scrollStatus === 'Enabled') {
		scrollToIndex = transcriptIndex;
	}

	function searchTranscript(transcriptSearchQuery) {
		if (transcriptSearchQuery) {
			scrollStatus = 'Disabled';
			currentIndex = 0;
			filteredIndices = getAllIndexes(episodeTranscript, transcriptSearchQuery);
			if (filteredIndices.length > 0) {
				setTimeout(() => {
					scrollToIndex = filteredIndices[0];
					jumpToSection(episodeTranscript?.[scrollToIndex], scrollToIndex);
				}, 10);

				scrollToAlignment = 'center';
			}
		} else {
			currentIndex = 0;
			scrollToIndex = transcriptIndex || 0;
			filteredIndices = [];
			scrollToAlignment = 'start';
		}

		function getAllIndexes(arr, val) {
			var indexes = [],
				i;
			for (i = 0; i < arr.length; i++)
				if (arr[i].text.toLowerCase().includes(val.toLowerCase())) {
					indexes.push(i);
				}
			return indexes;
		}
	}

	function handleScrollStatus() {
		scrollStatus = scrollStatus === 'Enabled' ? 'Disabled' : 'Enabled';
	}

	function getNextIndex() {
		currentIndex++;
		if (currentIndex === filteredIndices.length) {
			currentIndex = 0;
		}
		scrollToIndex = undefined;
		setTimeout(() => {
			(scrollToIndex = filteredIndices[currentIndex]), 1;
			jumpToSection(episodeTranscript?.[scrollToIndex], scrollToIndex);
		});
	}
	function getPreviousIndex() {
		currentIndex--;
		if (currentIndex === -1) {
			currentIndex = filteredIndices.length - 1;
		}
		scrollToIndex = undefined;
		setTimeout(() => {
			(scrollToIndex = filteredIndices[currentIndex]), 1;
			jumpToSection(episodeTranscript?.[scrollToIndex], scrollToIndex);
		});
	}

	function convertTime(time) {
		try {
			let formatTime = new Date(time * 1000).toISOString().substr(11, 8);
			if (formatTime.charAt(0) === '0') {
				return formatTime.substring(1);
			}

			return formatTime;
		} catch {
			(err) => console.log(err);
		}
	}
</script>

{#if episodeTranscript?.length}
	<div class="bar-2">
		<label>
			<input type="checkbox" checked={scrollStatus === 'Enabled'} on:change={handleScrollStatus} />
			Scrolling {scrollStatus}
		</label>
		{#if filteredIndices.length > 0}
			<div class="index-select">
				<button class="previous" on:click={getPreviousIndex}>&#9664</button>
				<span>
					{currentIndex + 1} of {filteredIndices.length}
				</span>
				<button class="next" on:click={getNextIndex}>&#9654</button>
			</div>
		{/if}
	</div>

	<div class="list-height" bind:clientHeight={listHeight}>
		<VirtualList
			height={listHeight}
			width="100%"
			itemCount={episodeTranscript.length}
			itemSize={50}
			overscanCount={5}
			{scrollToIndex}
			{scrollToAlignment}
		>
			<div
				slot="item"
				let:index
				let:style
				{style}
				class="row"
				class:active={index === transcriptIndex}
				class:searched={index === filteredIndices[currentIndex]}
				on:click={jumpToSection.bind(this, episodeTranscript?.[index], index)}
			>
				<p class="transcript-text">
					{@html episodeTranscript?.[index].text || ''}
				</p>
				<p class="transcript-time">
					{convertTime(episodeTranscript?.[index].start) ?? ''}
				</p>
			</div>
		</VirtualList>
	</div>
{/if}

<style>
	.list-height {
		height: calc(100% - 53px);
		padding: 0 0 0 8px;
	}

	div {
		display: flex;
	}
	div.searched p {
		font-weight: 700;
		color: purple;
	}
	div.active p {
		font-weight: 700;
		color: red;
	}

	.transcript-text {
		flex: 1 1 auto;
	}

	.transcript-time {
		padding-right: 8px;
	}

	.bar-2 {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 8px 0 8px 8px;
		border-bottom: 1px solid lightgray;
	}

	.index-select > span {
		width: 100px;
		text-align: center;
		display: inline-flex;
		align-items: center;
		justify-content: center;
		color: var(--primary-text-color);
	}

	button {
		background-color: transparent;
		border: none;
		padding: 0;
		margin: 0;
		color: var(--primary-text-color);
		height: 36px;
		width: 36px;
		text-align: center;
		cursor: pointer;
	}
	label {
		cursor: pointer;
	}

	.next {
		text-align: start;
	}
	.previous {
		text-align: end;
	}

	.row {
		cursor: pointer;
	}
</style>
