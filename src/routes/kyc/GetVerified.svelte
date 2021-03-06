<script>
	// external components
	import { RSA, Crypt } from 'hybrid-crypto-js';
	import { push } from 'svelte-spa-router'
	import axios from 'axios'

	// internal components
	import appConfig from '@@Config/app';
	import GoBack from '@@Components/GoBack.svelte';
	import Submit from '@@Components/Submit.svelte';
	import Info from '@@Components/Info.svelte';
	import Header from '@@Components/Header.svelte';
	import { citizen } from "@@Stores/citizen.js";
	import { users } from "@@Stores/users.js";

	const username = $users.loggedInUser;
	const wallet = $users.users.filter(u => u.username === username)[0].wallet;
	const keypair = $users.users.filter(u => u.username === username)[0].keypair;
	const id = $users.users.filter(u => u.username === username)[0].id;

	var rsa = new RSA();
	var crypt = new Crypt();
	$: infoType = '';
	$: infoValue = '';
	$: infoLoading = false;

	$: submitIsDisabled = 
		$citizen.firstname.length === 0
		|| $citizen.lastname.length === 0
		|| $citizen.nation.length === 0
		|| $citizen.nationalId.length === 0;
		 

	const generateKeyPair = () => {
		return new Promise((resolve) => {
			rsa.generateKeyPair(resolve);
		})
	}
	
	const createIdentity = async (e) => {
		e.preventDefault();

		infoLoading = true;

		// create keypair (to be shared later)
		const keypairToShare = await generateKeyPair();

		// encrypt the identity with the publicKey
		const encryptedIdentity = crypt.encrypt(keypairToShare.publicKey, JSON.stringify($citizen));
		
		// post the identity to the nerves
		const resIdentity = await axios
			.post(appConfig.nerves.identity.url, {
				encryptedIdentity,
				user: {username, wallet}
			})
			.catch(e => {
				infoType = 'error';
				infoValue = e.message;
				infoLoading = false;
			})

		if (!resIdentity) return;

		infoType = 'info';
		infoValue = resIdentity.data.message;
		
		// create verification jobs
		const resJob = await axios
			.post(appConfig.nerves.job.url, {
				type: 'confirmation',
				data: encryptedIdentity,
				chaincode: 'identity',
				key: id,
				user: {username, wallet}
			})
			.catch(e => {
				infoType = 'error';
				infoValue = e.message;
				infoLoading = false;
			})

		if (!resJob) return;

		infoType = 'info';
		infoValue = resJob.data.message;

		const { workersIds, jobId } = resJob.data;

		// share the keypair with the workers
		const myEncryptedKeyPair = crypt.encrypt(keypair.publicKey, JSON.stringify(keypairToShare));
		const decryptedKeypairRaw = crypt.decrypt(keypair.privateKey, myEncryptedKeyPair);
		const decryptedKeypair = JSON.parse(decryptedKeypairRaw.message)
		console.log({keypairToShare, myEncryptedKeyPair, decryptedKeypairRaw, decryptedKeypair})
		const sharedWith = workersIds.reduce((acc, worker) => {
			return {
				...acc,
				[worker._id]: {keyPair: crypt.encrypt(worker.publicKey, JSON.stringify(keypairToShare))}
			}
		}, {});

		const resKeypair = await axios
			.post(appConfig.nerves.user.keypair.url, {
				sharedWith,
				groupId: jobId,
				myEncryptedKeyPair,
				type: 'job',
				user: {username, wallet}
			})
			.catch(e => {
				infoType = 'error';
				infoValue = e.message
			})

		if (!resKeypair) return;

		$users.users.filter(u => u.username === username)[0].identity = {...$citizen};
		infoType = 'info';
		infoValue = 'Your identity have been successfully created. Wait for confirmations. You will be redirected.';
		setTimeout(() => push('/'), 1500);
	}
</script>

<Header title="Get verified" />

<form class="content">
	<Info type={infoType} value={infoValue} loading={infoLoading} />
	<input type="text" bind:value={$citizen.firstname} placeholder="Firstname" />
	<input type="text" bind:value={$citizen.lastname} placeholder="Lastname" />
	<input type="text" bind:value={$citizen.nation} placeholder="Nation" />
	<input type="text" bind:value={$citizen.nationalId} placeholder="Nation ID" />
	<Submit onclick={createIdentity} disabled={submitIsDisabled} />
</form>

<GoBack />

<style>
  .content {
		padding-bottom: 3vw;
		width: 100%;
		display: flex;
		flex-direction: column;
		align-items: center;
  }
</style>
