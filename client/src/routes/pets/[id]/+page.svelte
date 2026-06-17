<script lang="ts">
	import { page } from '$app/stores';
	import { goto } from '$app/navigation';
	import { getPetById, deletePet } from '$lib/api/pet/PetController';
	import { getOwnerById } from '$lib/api/owner/OwnerController';
	import type { PetResponse, OwnerResponse } from '$lib/api/models';
	import { Button } from '$lib/components/ui/button';
	import { Badge } from '$lib/components/ui/badge';
	import * as Card from '$lib/components/ui/card';
	import * as Dialog from '$lib/components/ui/dialog';
	import { PawPrint, Calendar, ArrowLeft, Trash2, Stethoscope, Plus, User } from 'lucide-svelte';
	import { toast } from 'svelte-sonner';

	let pet = $state<PetResponse | null>(null);
	let loading = $state(true);
	let deleteDialogOpen = $state(false);
	let deleting = $state(false);
	let ownerModalOpen = $state(false);
	let owner = $state<OwnerResponse | null>(null);
	let ownerLoading = $state(false);

	const petId = $derived(Number($page.params.id));

	async function loadPet() {
		loading = true;
		try {
			pet = await getPetById(petId);
		} catch (err) {
			toast.error('Failed to load pet');
			console.error('Error:', err);
		} finally {
			loading = false;
		}
	}

	async function openOwnerModal() {
		if (!pet?.owner?.id) return;
		ownerModalOpen = true;
		if (!owner || owner.id !== pet.owner.id) {
			ownerLoading = true;
			try {
				owner = await getOwnerById(pet.owner.id);
			} catch (err) {
				toast.error('Failed to load owner details');
				console.error('Error:', err);
			} finally {
				ownerLoading = false;
			}
		}
	}

	async function confirmDelete() {
		if (!pet?.id) return;
		deleting = true;
		try {
			await deletePet(pet.id);
			toast.success(`${pet.name} deleted successfully`);
			goto('/pets');
		} catch (err) {
			toast.error('Failed to delete pet');
			console.error('Error:', err);
		} finally {
			deleting = false;
			deleteDialogOpen = false;
		}
	}

	function formatDate(dateStr: string | undefined): string {
		if (!dateStr) return 'Unknown';
		return new Date(dateStr).toLocaleDateString('en-US', {
			year: 'numeric',
			month: 'long',
			day: 'numeric'
		});
	}

	function calculateAge(birthDate: string | undefined): string {
		if (!birthDate) return 'Unknown age';
		const birth = new Date(birthDate);
		const now = new Date();
		const years = Math.floor((now.getTime() - birth.getTime()) / (365.25 * 24 * 60 * 60 * 1000));
		if (years === 0) {
			const months = Math.floor(
				(now.getTime() - birth.getTime()) / (30.44 * 24 * 60 * 60 * 1000)
			);
			return months <= 1 ? '< 1 month old' : `${months} months old`;
		}
		return years === 1 ? '1 year old' : `${years} years old`;
	}

	$effect(() => {
		if (petId) {
			loadPet();
		}
	});
</script>

<svelte:head>
	<title>{pet ? pet.name : 'Pet'} | VetHub</title>
</svelte:head>

<div class="container mx-auto px-4 py-8">
	<!-- Back Button -->
	<div class="mb-6">
		<Button variant="ghost" href="/pets" class="gap-2">
			<ArrowLeft class="h-4 w-4" />
			Back to Pets
		</Button>
	</div>

	{#if loading}
		<div class="card p-12 text-center">
			<div class="mx-auto mb-4 h-8 w-8 animate-spin rounded-full border-4 border-primary border-t-transparent"></div>
			<p class="text-muted-foreground">Loading pet...</p>
		</div>
	{:else if !pet}
		<div class="card p-12 text-center">
			<PawPrint class="mx-auto mb-4 h-12 w-12 text-muted-foreground/50" />
			<p class="text-muted-foreground">Pet not found</p>
		</div>
	{:else}
		<!-- Pet Info Card -->
		<Card.Root class="mb-8">
			<Card.Header>
				<div class="flex flex-col gap-4 sm:flex-row sm:items-start sm:justify-between">
					<div class="flex items-center gap-4">
						<div class="flex h-16 w-16 items-center justify-center rounded-full bg-accent/10">
							<PawPrint class="h-8 w-8 text-accent" />
						</div>
						<div>
							<Card.Title class="text-2xl">{pet.name}</Card.Title>
							<div class="mt-1 flex items-center gap-2">
								<Badge variant="secondary">{pet.type?.name ?? 'Unknown type'}</Badge>
								<span class="text-muted-foreground">•</span>
								<span class="text-muted-foreground">{calculateAge(pet.birthDate)}</span>
							</div>
						</div>
					</div>
					<div class="flex gap-2">
						<Button variant="destructive" onclick={() => (deleteDialogOpen = true)} class="gap-2">
							<Trash2 class="h-4 w-4" />
							Delete
						</Button>
					</div>
				</div>
			</Card.Header>
			<Card.Content>
				<div class="space-y-3">
					<div class="flex items-center gap-3 text-muted-foreground">
						<Calendar class="h-5 w-5" />
						<span>Born: {formatDate(pet.birthDate)}</span>
					</div>
					{#if pet.owner}
						<div class="flex items-center gap-3">
							<User class="h-5 w-5 text-muted-foreground" />
							<button
								class="font-medium text-primary underline-offset-4 hover:underline"
								onclick={openOwnerModal}
							>
								{pet.owner.firstName} {pet.owner.lastName}
							</button>
						</div>
					{/if}
				</div>
			</Card.Content>
		</Card.Root>

		<!-- Visits Section -->
		<div class="mb-6 flex items-center justify-between">
			<h2 class="text-xl font-semibold text-foreground">Visit History</h2>
		</div>

		{#if !pet.visits?.length}
			<div class="card p-8 text-center">
				<Stethoscope class="mx-auto mb-4 h-12 w-12 text-muted-foreground/50" />
				<p class="text-muted-foreground">No visits recorded for this pet</p>
			</div>
		{:else}
			<div class="card overflow-hidden">
				<table class="w-full text-sm">
					<thead class="border-b border-border bg-muted/50">
						<tr>
							<th class="px-4 py-3 text-left font-medium text-muted-foreground">Date</th>
							<th class="px-4 py-3 text-left font-medium text-muted-foreground">Description</th>
						</tr>
					</thead>
					<tbody>
						{#each pet.visits.sort((a, b) => new Date(b.date ?? '').getTime() - new Date(a.date ?? '').getTime()) as visit (visit.id)}
							<tr class="border-b border-border last:border-0 hover:bg-muted/30">
								<td class="px-4 py-3 text-muted-foreground">{formatDate(visit.date)}</td>
								<td class="px-4 py-3 text-foreground">{visit.description}</td>
							</tr>
						{/each}
					</tbody>
				</table>
			</div>
		{/if}
	{/if}
</div>

<!-- Delete Confirmation Dialog -->
<Dialog.Root bind:open={deleteDialogOpen}>
	<Dialog.Content>
		<Dialog.Header>
			<Dialog.Title>Delete Pet</Dialog.Title>
			<Dialog.Description>
				Are you sure you want to delete {pet?.name}? This action cannot be undone.
			</Dialog.Description>
		</Dialog.Header>
		<Dialog.Footer>
			<Button variant="outline" onclick={() => (deleteDialogOpen = false)} disabled={deleting}>
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

<!-- Owner Details Modal -->
<Dialog.Root bind:open={ownerModalOpen}>
	<Dialog.Content>
		<Dialog.Header>
			<Dialog.Title>Owner Details</Dialog.Title>
		</Dialog.Header>
		{#if ownerLoading}
			<div class="py-8 text-center">
				<div class="mx-auto h-6 w-6 animate-spin rounded-full border-4 border-primary border-t-transparent"></div>
			</div>
		{:else if owner}
			<div class="space-y-3 py-2">
				<div class="grid grid-cols-[120px_1fr] gap-2 text-sm">
					<span class="font-medium text-muted-foreground">First Name:</span>
					<span>{owner.firstName}</span>
					<span class="font-medium text-muted-foreground">Last Name:</span>
					<span>{owner.lastName}</span>
					<span class="font-medium text-muted-foreground">Address:</span>
					<span>{owner.address ?? '—'}</span>
					<span class="font-medium text-muted-foreground">City:</span>
					<span>{owner.city ?? '—'}</span>
					<span class="font-medium text-muted-foreground">Telephone:</span>
					<span>{owner.telephone ?? '—'}</span>
				</div>
			</div>
		{/if}
		<Dialog.Footer>
			<Button variant="outline" onclick={() => (ownerModalOpen = false)}>Close</Button>
		</Dialog.Footer>
	</Dialog.Content>
</Dialog.Root>
