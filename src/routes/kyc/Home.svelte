<script>
	// external components
	import { push } from 'svelte-spa-router'
	
	// internal components
	import appConfig from '@@Config/app';
	import Button from '@@Components/Button.svelte';
	import Header from '@@Components/Header.svelte';
	import { users } from "@@Stores/users.js";

	$: loggedUser = $users.users.filter(u => u.username === $users.loggedInUser)[0];
</script>

<Header title="KYC" />
<Button label="BLOCKOTUS Home" onclick={() => push('/home')} />
<Button label="My identity" onclick={() => push('/kyc/identity')} disabled={!loggedUser.identity} />
<Button label="Create identity" onclick={() => push('/kyc/getverified')} disabled={loggedUser.identity} />
<Button label="Verify" onclick={() => push('/kyc/jobs')} />

<style>

</style>
