<script lang="ts">
	import axios from 'axios';
	import type { Observation, Bundle, BundleEntry, OperationOutcome } from 'fhir/r4';
	import { formatRelative } from 'date-fns';

	export let accessToken: string;
	export let patientId: string;
	export let baseUrl: string;
	export let title: string = 'Observations';

	let temperature: number;
	let observationPosting = false;

	const getObservations = async () => {
		console.log(`patient: ${patientId}`);
		const observationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(
			`${baseUrl}/Observation`,
			{
				params: { patient: patientId, sort: '-date', category: 'vital-signs' },
				headers: { Authorization: `Bearer ${accessToken}` }
			}
		);

		return observationResponse.data;
	};

	let observationResults = getObservations();

	const postTemperature = async (temperature: number) => {
		observationPosting = true;
		const temperatureResource = {
			resourceType: 'Observation',
			status: 'final',
			category: [
				{
					coding: [
						{
							system: 'http://terminology.hl7.org/CodeSystem/observation-category',
							code: 'vital-signs',
							display: 'Vital Signs'
						}
					],
					text: 'Vital Signs'
				}
			],
			code: {
				coding: [
					{
						system: 'http://loinc.org',
						code: '8331-1'
					}
				],
				text: 'Temperature Oral'
			},
			subject: {
				reference: `Patient/${patientId}`
			},
			effectiveDateTime: new Date().toISOString(),
			valueQuantity: {
				value: temperature,
				unit: 'degC',
				system: 'http://unitsofmeasure.org',
				code: 'Cel'
			}
		};

		console.log(temperatureResource);
		const tempObservationResponse = await axios.post(
			`${baseUrl}/Observation`,
			temperatureResource,
			{ headers: { Authorization: `Bearer ${accessToken}` } }
		);
		console.log(tempObservationResponse);
		observationPosting = false;
		observationResults = getObservations();
	};

	const getObservationDisplay = (observation: Observation | undefined) => {
		if (!observation) return '';

		const codeableConcept = observation?.valueCodeableConcept;
		if (codeableConcept) {
			return observation?.valueCodeableConcept?.text;
		}
		const isBp = observation.code?.coding?.find((a) => a.code === '85354-9');

		if (isBp) {
			const systolicComponent = observation.component?.find((a) =>
				a.code?.coding?.find((b) => b.code === '8480-6')
			);
			const systolic = systolicComponent?.valueQuantity?.value;

			const diastolicComponent = observation.component?.find((a) =>
				a.code?.coding?.find((b) => b.code === '8462-4')
			);
			const diastolic = diastolicComponent?.valueQuantity?.value;

			return `${systolic}/${diastolic}`;
		}

		if (!observation?.valueQuantity?.unit) {
			return `${observation?.valueQuantity?.value}`;
		}
		return `${observation?.valueQuantity?.value} ${observation?.valueQuantity?.unit}`;
	};

	const getObservationEntries = (bundle: Bundle<Observation | OperationOutcome>) => {
		if (!bundle?.entry) return [];
		const results = bundle.entry?.filter(
			(entry) => entry.resource?.resourceType === 'Observation'
		) as BundleEntry<Observation>[];

		const nonPanelResults = results.filter((entry) => entry.resource?.hasMember == undefined);
		return nonPanelResults.sort((a, b) => {
			const dateA = new Date(a.resource?.effectiveDateTime || 0).getTime();
			const dateB = new Date(b.resource?.effectiveDateTime || 0).getTime();
			return dateB - dateA; // Sort descending (most recent first)
		});
	};
</script>

<div class="mx-auto max-w-2xl rounded-xl border border-gray-200 bg-white p-6 shadow-md">
	{#await observationResults}
		<p class="text-sm text-gray-500">Loading observations...</p>
	{:then observations}
		<h2 class="mb-4 text-2xl font-semibold text-gray-800">{title}</h2>

		<div class="space-y-6">
			<div class="rounded-md border border-gray-200 bg-gray-50 p-4">
				<p class="mb-2 text-sm font-medium text-gray-700">Record new temperature (°C)</p>
				<form
					on:submit|preventDefault={() => {
						postTemperature(temperature);
					}}
					class="flex items-center gap-3"
				>
					<input
						step=".1"
						min="10"
						max="50"
						bind:value={temperature}
						type="number"
						placeholder="Enter temp"
						class="w-36 rounded-md border border-gray-300 px-3 py-2 text-sm shadow-sm focus:border-blue-500 focus:ring-1 focus:ring-blue-500 focus:outline-none"
					/>
					{#if observationPosting}
						<span class="text-sm text-gray-500">Creating observation...</span>
					{:else}
						<button
							type="submit"
							class="rounded-md bg-blue-600 px-4 py-2 text-sm text-white transition hover:bg-blue-700"
						>
							Submit
						</button>
					{/if}
				</form>
			</div>

			<div class="space-y-4">
				{#each getObservationEntries(observations) as observation, i}
					<div class="rounded-md border border-gray-100 bg-gray-50 p-4 shadow-sm">
						<p class="text-sm font-semibold text-gray-700">{observation.resource?.code?.text}</p>
						<p class="text-base text-gray-900">{getObservationDisplay(observation.resource)}</p>
						{#if observation?.resource?.effectiveDateTime}
							<p class="mt-1 text-xs text-gray-500">
								{formatRelative(new Date(observation.resource.effectiveDateTime), new Date())}
							</p>
						{/if}
					</div>
				{/each}
			</div>
		</div>
	{/await}
</div>
