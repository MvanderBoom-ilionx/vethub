<script lang="ts">
	import { getPets, deletePet } from '$lib/api/pet/PetController';
	import type { PetResponse } from '$lib/api/models';
	import { Button } from '$lib/components/ui/button';
	import { Input } from '$lib/components/ui/input';
	import { Badge } from '$lib/components/ui/badge';
	import * as Table from '$lib/components/ui/table';
	import * as Dialog from '$lib/components/ui/dialog';
	import { PawPrint, Plus, Search, Trash2 } from 'lucide-svelte';
	import { toast } from 'svelte-sonner';

	let pets = $state<PetResponse[]>([]);
	let loading = $state(true);
	let searchQuery = $state('');
	let deleteDialogOpen = $state(false);
	let deletingPet = $state<PetResponse | null>(null);
	let deleting = $state(false);

	let filteredPets = $derived(() => {
		if (!searchQuery.trim()) return pets;
		const query = searchQuery.toLowerCase();
		return pets.filter(
			(pet) =>
				pet.name?.toLowerCase().includes(query) ||
				`${pet.owner?.firstName ?? ''} ${pet.owner?.lastName ?? ''}`.toLowerCase().includes(query) ||
				pet.birthDate?.toString().includes(query)
		);
	});

	async function loadPets() {
		loading = true;
		try {
			pets = await getPets();
		} catch (err) {
			toast.error('Failed to load pets');
			console.error('Error loading pets:', err);
		} finally {
			loading = false;
		}
	}

	function openDeleteDialog(pet: PetResponse) {
		deletingPet = pet;
		deleteDialogOpen = true;
	}

	async function confirmDelete() {
		if (!deletingPet?.id) return;
		deleting = true;
		try {
			await deletePet(deletingPet.id);
			toast.success(`${deletingPet.name} deleted successfully`);
			pets = pets.filter((p) => p.id !== deletingPet!.id);
		} catch (err) {
			toast.error('Failed to delete pet');
			console.error('Error deleting pet:', err);
		} finally {
			deleting = false;
			deleteDialogOpen = false;
			deletingPet = null;
		}
	}

	$effect(() => {
		loadPets();
	});
</script>

<svelte:head>
	<title>Pets | VetHub</title>
</svelte:head>

<div class="container mx-auto px-4 py-8">
	<!-- Header -->
	<div class="mb-8 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
		<div class="flex items-center gap-3">
			<div class="flex h-12 w-12 items-center justify-center rounded-lg bg-primary/10">
				<PawPrint class="h-6 w-6 text-primary" />
			</div>
			<div>
				<h1 class="text-2xl font-bold text-foreground">Pets</h1>
				<p class="text-sm text-muted-foreground">Browse and manage all pets</p>
			</div>
		</div>
		<Button href="/pets/new" class="gap-2">
			<Plus class="h-4 w-4" />
			Add Pet
		</Button>
	</div>

	<!-- Search -->
	<div class="mb-6">
		<div class="relative max-w-md">
			<Search class="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
			<Input
				type="search"
				placeholder="Search by pet name, owner, or birth date..."
				bind:value={searchQuery}
				class="pl-10"
			/>
		</div>
	</div>

	<!-- Table -->
	{#if loading}
		<div class="card p-12 text-center">
			<div class="mx-auto mb-4 h-8 w-8 animate-spin rounded-full border-4 border-primary border-t-transparent"></div>
			<p class="text-muted-foreground">Loading pets...</p>
		</div>
	{:else if filteredPets().length === 0}
		<div class="card p-12 text-center">
			<PawPrint class="mx-auto mb-4 h-12 w-12 text-muted-foreground/50" />
			{#if searchQuery}
				<p class="text-muted-foreground">No pets found matching "{searchQuery}"</p>
			{:else}
				<p class="text-muted-foreground">No pets registered yet</p>
			{/if}
		</div>
	{:else}
		<div class="card overflow-hidden">
			<Table.Root>
				<Table.Header>
					<Table.Row>
						<Table.Head>Name</Table.Head>
						<Table.Head>Type</Table.Head>
						<Table.Head>Owner</Table.Head>
						<Table.Head>Birth Date</Table.Head>
						<Table.Head class="w-[80px]"></Table.Head>
					</Table.Row>
				</Table.Header>
				<Table.Body>
					{#each filteredPets() as pet (pet.id)}
						<Table.Row class="hover:bg-muted/50">
							<Table.Cell>
								<a href="/pets/{pet.id}" class="font-medium text-foreground hover:text-primary">
									{pet.name}
								</a>
							</Table.Cell>
							<Table.Cell>
								<Badge variant="secondary">{pet.type?.name ?? '—'}</Badge>
							</Table.Cell>
							<Table.Cell class="text-muted-foreground">
								{pet.owner ? `${pet.owner.firstName} ${pet.owner.lastName}` : '—'}
							</Table.Cell>
							<Table.Cell class="text-muted-foreground">
								{pet.birthDate ?? '—'}
							</Table.Cell>
							<Table.Cell>
								<Button
									variant="ghost"
									size="icon"
									class="text-destructive hover:text-destructive"
									onclick={() => openDeleteDialog(pet)}
									aria-label="Delete {pet.name}"
								>
									<Trash2 class="h-4 w-4" />
								</Button>
							</Table.Cell>
						</Table.Row>
					{/each}
				</Table.Body>
			</Table.Root>
		</div>
		<p class="mt-4 text-sm text-muted-foreground">
			Showing {filteredPets().length} of {pets.length} pets
		</p>
	{/if}
</div>

<!-- Delete Confirmation Dialog -->
<Dialog.Root bind:open={deleteDialogOpen}>
	<Dialog.Content>
		<Dialog.Header>
			<Dialog.Title>Delete Pet</Dialog.Title>
			<Dialog.Description>
				Are you sure you want to delete {deletingPet?.name}? This action cannot be undone.
			</Dialog.Description>
		</Dialog.Header>
		<Dialog.Footer>
			<Button
				variant="outline"
				onclick={() => {
					deleteDialogOpen = false;
					deletingPet = null;
				}}
				disabled={deleting}
			>
				Cancel
			</Button>
			<Button variant="destructive" onclick={confirmDelete} disabled={deleting}>
				{#if deleting}
					Deleting...
				{:else}
					Delete
				{/if}
			</Button>
		</Dialog.Footer>
	</Dialog.Content>
</Dialog.Root>
