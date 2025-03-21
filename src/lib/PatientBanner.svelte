<script lang="ts">
	import axios from 'axios';
	import type { Patient } from 'fhir/r2';
	import { onMount } from 'svelte';

	export let accessToken: string;
	export let baseUrl: string;
	export let patientId: string;
	let patientResource: Patient | undefined = undefined;

	onMount(async () => {
		const patientResourceResponse = await axios.get<Patient>(`${baseUrl}/Patient/${patientId}`, {
			headers: {
				Authorization: `Bearer ${accessToken}`
			}
		});
		console.log(baseUrl);
		patientResource = patientResourceResponse.data;
	});
</script>

{#if patientResource}
	<div class="mb-6 rounded-xl border border-gray-200 bg-white p-4 shadow-sm">
		<h3 class="mb-3 text-lg font-semibold text-gray-800">Patient Info</h3>
		<div class="flex flex-wrap gap-x-8 gap-y-2 text-sm text-gray-700">
			<p><span class="font-medium text-gray-600">Name:</span> {patientResource.name?.[0]?.text}</p>
			<p><span class="font-medium text-gray-600">DoB:</span> {patientResource.birthDate}</p>
			<p><span class="font-medium text-gray-600">Gender:</span> {patientResource.gender}</p>
		</div>
	</div>
{:else}
	Loading patient...
{/if}
