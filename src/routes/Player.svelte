<script>
	import { onMount } from 'svelte';
	export let player;
	export let transcriptIndex;
	export let episodeTranscript;

	onMount(() => {
		player.ontimeupdate = () => {
			let playerTime = player?.currentTime || 0;

			while (playerTime >= episodeTranscript?.[transcriptIndex + 1]?.start) {
				transcriptIndex++;
			}

			while (playerTime < episodeTranscript?.[transcriptIndex]?.start) {
				transcriptIndex--;
			}
		};
	});
</script>

<audio bind:this={player} controls />

<style>
	audio {
		width: 100%;
		height: 54px;
	}
</style>
