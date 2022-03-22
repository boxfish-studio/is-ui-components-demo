<script lang="ts">
	import type { IdentityJson } from 'boxfish-studio--iota-is-sdk';
	import { UserType } from 'boxfish-studio--iota-is-sdk';
	import type {
		ActionButton,
		ExtendedUser,
		IdentityTemplate,
		TableData
	} from 'boxfish-studio--is-ui-components';
	import {
		addIdentityToSearchResults,
		ChannelMessages,
		CreateCredentialModal,
		CreateIdentityModal,
		FieldType,
		getVerifiableCredentials,
		Icon,
		IdentityDetails,
		isAsyncLoadingIdentities,
		ListManager,
		NotificationManager,
		searchAllIdentities,
		searchIdentitiesResults,
		searchIdentityByDID,
		selectedChannelData,
		selectedIdentity,
		startReadingChannel,
		stopIdentitiesSearch,
		stopReadingChannel,
		updateIdentityInSearchResults,
		USER_ICONS,
		WELCOME_LIST_RESULTS_NUMBER
	} from 'boxfish-studio--is-ui-components';
	import { onDestroy, onMount } from 'svelte';
	import { Container, Row } from 'sveltestrap';

	const SOFTWARE_VERSIONS = [
		{
			label: 'v1.0.0',
			value: 'v1.0.0'
		},
		{
			label: 'v1.1.0',
			value: 'v1.1.0'
		},
		{
			label: 'v1.2.0',
			value: 'v1.2.0'
		},
		{
			label: 'v1.3.0',
			value: 'v1.3.0'
		}
	];
	const DEVICE_TEMPLATE: IdentityTemplate[] = [
		{
			type: UserType.Device,
			fields: [
				{
					id: 'username',
					name: 'username',
					type: FieldType.Text,
					required: true
				},
				{
					id: 'softwareVersion',
					name: 'Software Version',
					type: FieldType.Selector,
					options: SOFTWARE_VERSIONS,
					required: true
				},
				{
					id: 'channel',
					name: 'Streams Channel',
					type: FieldType.Text,
					required: true
				}
			]
		}
	];
	const IDENTITY_LIST_BUTTONS: ActionButton[] = [
		{
			label: 'Add device',
			onClick: openCreateDeviceModal,
			icon: 'plus',
			color: 'dark'
		},
		{
			label: 'Create an identity',
			onClick: openCreateIdentityModal,
			icon: 'plus',
			color: 'dark'
		}
	];
	const IDENTITY_DETAILS_BUTTONS: ActionButton[] = [
		{
			label: 'Add credential',
			onClick: openCreateCredentialModal,
			icon: 'plus',
			color: 'dark'
		}
	];

	const TARGET_CHANNEL_ADDRESS =
		'db9660525ebf0970884da7d515f3337acd1eab48d7e1d37c1fe304bd192343c00000000000000000:bd1cb41465914401880f28e4';

	enum State {
		ListIdentities = 'listIdentities',
		IdentityDetail = 'identityDetail'
	}

	let state: State = State.ListIdentities;
	let loading: boolean = false;
	let query: string = '';
	let message: string;
	let isCreateIdentityModalOpen = false;
	let isCreateCredentialModalOpen = false;
	let isCreateDeviceModalOpen = false;

	$: $selectedIdentity, updateState();
	$: state, loadVCsOnSelectedIdentity();
	$: message =
		$isAsyncLoadingIdentities || loading || $searchIdentitiesResults?.length
			? null
			: 'No identities found';
	$: tableData = {
		headings: ['Identity', 'Type', 'Date created', 'Credentials'],
		rows: $searchIdentitiesResults.map((identity) => ({
			onClick: () => handleSelectIdentity(identity),
			content: [
				{
					icon: USER_ICONS[identity.claim?.type]?.icon,
					boxColor: USER_ICONS[identity.claim?.type]?.boxColor,
					value: identity?.username
				},
				{ value: identity?.claim?.type },
				{ value: identity?.registrationDate },
				{ value: identity?.numberOfCredentials ?? 0 }
			]
		}))
	} as TableData;

	onMount(async () => {
		searchAllIdentities('', { limit: WELCOME_LIST_RESULTS_NUMBER });
	});

	onDestroy(() => {
		stopIdentitiesSearch();
	});

	async function onSearch(): Promise<void> {
		await searchAllIdentities(query);
	}

	async function updateState(): Promise<void> {
		if ($selectedIdentity) {
			state = State.IdentityDetail;
			// custom: start reading channel data
			startReadingChannel(TARGET_CHANNEL_ADDRESS);
		} else {
			state = State.ListIdentities;
			stopReadingChannel();
		}
	}

	async function loadVCsOnSelectedIdentity(): Promise<void> {
		if (state === State.IdentityDetail) {
			loading = true;
			const vc = await getVerifiableCredentials($selectedIdentity?.id);
			selectedIdentity.update((identity) => ({ ...identity, vc }));
			loading = false;
		}
	}

	function handleSelectIdentity(identity: ExtendedUser): void {
		selectedIdentity.set(identity);
	}

	function handleBackClick(): void {
		selectedIdentity.set(null);
	}

	// Add the newly created identity to the search results
	async function onCreateIdentitySuccess(identity: IdentityJson): Promise<void> {
		loading = true;
		// If query is not empty, we need to search again to get the match results
		if (query?.length) {
			await onSearch();
		} else {
			// Add the identity to the search results directly, no need to search again
			await addIdentityToSearchResults(identity?.doc?.id);
		}
		loading = false;
	}

	// Add the newly created credential to the selected identity
	async function onCreateCredentialSuccess(): Promise<void> {
		loading = true;
		let identity = await searchIdentityByDID($selectedIdentity?.id);
		identity = { ...identity, numberOfCredentials: identity?.numberOfCredentials ?? 0 };
		if (identity) {
			updateIdentityInSearchResults(identity);
		}
		const vc = await getVerifiableCredentials($selectedIdentity?.id);
		selectedIdentity.update((identity) => ({ ...identity, vc }));
		loading = false;
	}

	function openCreateIdentityModal(): void {
		isCreateIdentityModalOpen = true;
	}
	function closeCreateIdentityModal(): void {
		isCreateIdentityModalOpen = false;
	}
	function openCreateCredentialModal(): void {
		isCreateCredentialModalOpen = true;
	}
	function closeCreateCredentialModal(): void {
		isCreateCredentialModalOpen = false;
	}
	function openCreateDeviceModal(): void {
		isCreateCredentialModalOpen = true;
	}
	function closeCreateDeviceModal(): void {
		isCreateCredentialModalOpen = false;
	}
</script>

<Container class="py-5">
	<Row class="mb-4">
		<h1>Demo 1: Authentication</h1>
	</Row>
	{#if state === State.ListIdentities}
		<ListManager
			showSearch
			{onSearch}
			{tableData}
			{message}
			loading={loading || $isAsyncLoadingIdentities}
			actionButtons={IDENTITY_LIST_BUTTONS}
			bind:searchQuery={query}
		/>
	{:else if state === State.IdentityDetail}
		<div class="mb-4 align-self-start">
			<button on:click={handleBackClick} class="btn d-flex align-items-center">
				<Icon type="arrow-left" size={16} />
				<span class="ms-2">Back</span>
			</button>
		</div>
		<IdentityDetails
			{loading}
			actionButtons={IDENTITY_DETAILS_BUTTONS}
			onRevokeSuccess={updateIdentityInSearchResults}
			identity={$selectedIdentity}
		/>
		<div class="mb-4">
			<ChannelMessages channelData={$selectedChannelData} />
		</div>
	{/if}
</Container>
<CreateIdentityModal
	isOpen={isCreateIdentityModalOpen}
	onModalClose={closeCreateIdentityModal}
	onSuccess={onCreateIdentitySuccess}
/>
<CreateCredentialModal
	isOpen={isCreateCredentialModalOpen}
	onModalClose={closeCreateCredentialModal}
	targetDid={$selectedIdentity?.id}
	onSuccess={onCreateCredentialSuccess}
/>
<!-- Custom modal for devices -->
<CreateIdentityModal
	isOpen={isCreateDeviceModalOpen}
	onModalClose={closeCreateDeviceModal}
	identitiesTemplate={DEVICE_TEMPLATE}
/>
<NotificationManager />
