<script lang="ts">
	import { onMount } from 'svelte';
	import axios from 'axios';

	let iss: string;
	let launch: string;
	let authorizationEndpoint: string;
	onMount(async () => {
		const launchUrl = new URL(window.location.href);
		const issParam = launchUrl.searchParams.get('iss');
		const launchParam = launchUrl.searchParams.get('launch');

		if (!issParam || !launchParam) {
			throw new Error('iss or launch parameters not found');
		}
		iss = issParam;
		launch = launchParam;
		const smartConfigResponse = await axios.get(`${iss}/.well-known/smart-configuration`);
		const smartConfiguration = smartConfigResponse.data;
		authorizationEndpoint = smartConfiguration.authorization_endpoint as string;
		console.log(iss, launch, smartConfiguration);
	});
</script>

<pre>
	{JSON.stringify({ iss, launch, authorizationEndpoint }, null, 2)}
</pre>
