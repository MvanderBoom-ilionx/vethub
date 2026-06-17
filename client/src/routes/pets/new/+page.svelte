<script lang="ts">
	import { goto } from '$app/navigation';
	import { createPet } from '$lib/api/pet/PetController';
	import { getPetTypes } from '$lib/api/pet-type/PetTypeController';
	import { getOwners } from '$lib/api/owner/OwnerController';
	import type { PetTypeResponse, OwnerResponse } from '$lib/api/models';
	import { Button } from '$lib/components/ui/button';
	import PetForm from '$lib/components/pets/PetForm.svelte';
	import { ArrowLeft, PawPrint } from 'lucide-svelte';
	import { toast } from 'svelte-sonner';

	let petTypes = $state<PetTypeResponse[]>([]);
	let owners = $state<OwnerResponse[]>([]);
	let loading = $state(true);

	async function loadFormData() {
		loading = true;
		try {
			[petTypes, owners] = await Promise.all([getPetTypes(), getOwners()]);
		} catch (err) {
			toast.error('Failed to load form data');
			console.error('Error loading form data:', err);
		} finally {
			loading = false;
		}
	}

	async function handleSubmit(data: {
		name: string;
		birthDate: string;
		typeId: number;
		ownerId?: number;
	}) {
		if (!data.ownerId) {
			toast.error('Please select an owner');
			return;
		}
		try {
			const pet = await createPet({
				name: data.name,
				birthDate: data.birthDate,
				typeId: data.typeId,
				ownerId: data.ownerId
			});
			toast.success(`${pet.name} added successfully`);
			goto(`/pets/${pet.id}`);
		} catch (err) {
			toast.error('Failed to add pet');
			console.error('Error creating pet:', err);
			throw err;
		}
	}

	$effect(() => {
		loadFormData();
	});
</script>

<svelte:head>
	<title>Add Pet | VetHub</title>
</svelte:head>

<div class="container mx-auto px-4 py-8">
	<!-- Back Button -->
	<Button variant="ghost" href="/pets" class="mb-6 gap-2">
		<ArrowLeft class="h-4 w-4" />
		Back to Pets
	</Button>

	<div class="mx-auto max-w-lg">
		<!-- Page Header -->
		<div class="mb-8 flex items-center gap-3">
			<div class="flex h-12 w-12 items-center justify-center rounded-lg bg-primary/10">
				<PawPrint class="h-6 w-6 text-primary" />
			</div>
			<div>
				<h1 class="text-2xl font-bold text-foreground">Add New Pet</h1>
				<p class="text-sm text-muted-foreground">Register a new pet in the system</p>
			</div>
		</div>

		{#if loading}
			<div class="card p-12 text-center">
				<div class="mx-auto mb-4 h-8 w-8 animate-spin rounded-full border-4 border-primary border-t-transparent"></div>
				<p class="text-muted-foreground">Loading...</p>
			</div>
		{:else}
			<div class="card p-6">
				<PetForm
					{petTypes}
					{owners}
					onSubmit={handleSubmit}
					submitLabel="Save Pet"
				/>
			</div>
		{/if}
	</div>
</div>
