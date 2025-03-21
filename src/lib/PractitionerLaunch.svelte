<script lang="ts">
	import { onMount } from 'svelte';
	import PatientBanner from './PatientBanner.svelte';
	import ObservationViewer from './ObservationViewer.svelte';
	import axios from 'axios';

	let launch: string;
	let clientId = 'b1e03012-2af4-490e-96cc-891b6e22504d';
	let redirectUri = 'http://localhost:5173';
	let baseUrl: string;

	interface TokenResponse {
		access_token: string;
		patient: string;
		scope: string;
		need_patient_banner: boolean;
		id_token: string;
		smart_style_url: string;
		expires_in: number;
	}

	let token: TokenResponse | undefined = undefined;

	const LOCALSTORAGE_TOKEN_ENDPOINT = 'tokenEndpoint';
	const LOCALSTORAGE_ISS = 'iss';
	const LOCALSTORAGE_TOKEN_JSON = 'token';

	let observationPosting = false;

	const constructAuthUrl = (authorizationEndpoint: string, launch: string, iss: string) => {
		const url = new URL(authorizationEndpoint);
		url.searchParams.set('client_id', clientId);
		url.searchParams.set('redirect_uri', redirectUri);
		url.searchParams.set(
			'scope',
			'openid fhirUser launch user/Patient.read user/Observation.read user/Observation.write'
		);
		url.searchParams.set('response_type', 'code');
		url.searchParams.set('aud', iss);
		url.searchParams.set('launch', launch);
		return url.href;
	};

	const makeTokenRequest = async (code: string) => {
		const tokenEndpoint = localStorage.getItem(LOCALSTORAGE_TOKEN_ENDPOINT);

		if (!tokenEndpoint) {
			throw new Error('Token endpoint could not be found in localstorage');
		}

		// make the token request
		const params = new URLSearchParams();
		params.set('grant_type', 'authorization_code');
		params.set('code', code);
		params.set('redirect_uri', redirectUri);
		params.set('client_id', clientId);
		const tokenResponse = await axios.post(tokenEndpoint, params, {
			headers: {
				'Content-Type': 'application/x-www-form-urlencoded'
			}
		});
		localStorage.removeItem(LOCALSTORAGE_TOKEN_ENDPOINT);
		return tokenResponse.data;
	};

	onMount(async () => {
		const launchUrl = new URL(window.location.href);
		const issParam = launchUrl.searchParams.get('iss') || localStorage.getItem(LOCALSTORAGE_ISS);
		if (!issParam) {
			throw new Error('Missing iis parameter');
		}
		localStorage.setItem(LOCALSTORAGE_ISS, issParam);
		baseUrl = issParam;
		const iss = issParam;

		const launchParam = launchUrl.searchParams.get('launch');

		const code = launchUrl.searchParams.get('code');
		const tokenJSON = localStorage.getItem(LOCALSTORAGE_TOKEN_JSON);
		const issLocalStorage = localStorage.getItem(LOCALSTORAGE_ISS);

		if (token) {
			// TODO validate token expiry
			return;
		}

		if (code) {
			token = await makeTokenRequest(code);
			console.log('token ' + token);
			return;
		}

		if (!issParam || !launchParam) {
			throw new Error('iss or launch parameters not found');
		}
		// iss stuff was here
		const launch = launchParam;
		const smartConfigResponse = await axios.get(`${iss}/.well-known/smart-configuration`);
		const smartConfiguration = smartConfigResponse.data;
		const authorizationEndpoint = smartConfiguration.authorization_endpoint as string;
		const tokenEndpoint = smartConfiguration.token_endpoint as string;

		// store tokenEndpoint for future use
		localStorage.setItem(LOCALSTORAGE_TOKEN_ENDPOINT, tokenEndpoint);

		const redirectUrl = constructAuthUrl(authorizationEndpoint, launch, iss);
		window.location.href = redirectUrl;
	});
</script>

{#if !token}
	Loading...
{:else if token?.need_patient_banner}
	<PatientBanner {baseUrl} accessToken={token.access_token} patientId={token.patient}
	></PatientBanner>
	<ObservationViewer {baseUrl} accessToken={token.access_token} patientId={token.patient}
	></ObservationViewer>
{/if}
