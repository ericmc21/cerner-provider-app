<script lang="ts">
	import axios from 'axios';
	import type { Patient } from 'fhir/r2';
	import { onMount } from 'svelte';

	export let accessToken: string;
	export let baseUrl: string;
	export let patient: string;
	let patientResource: Patient | undefined = undefined;

	onMount(async () => {
		const patientResourceResponse = await axios.get<Patient>(`${baseUrl}/Patient/${patient}`, {
			headers: {
				Authorization: `Bearer ${accessToken}`
			}
		});
		console.log(baseUrl);
		patientResource = patientResourceResponse.data;
	});
</script>

{#if patientResource}
	<div class="flex gap-5 bg-slate-500 text-white">
		<p>Name: {patientResource.name?.[0]?.text}</p>
		<p>DoB: {patientResource.birthDate}</p>
		<p>Gender: {patientResource.gender}</p>
	</div>
{:else}
	Loading patient...
{/if}
