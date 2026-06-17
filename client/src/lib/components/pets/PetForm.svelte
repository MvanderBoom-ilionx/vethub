<script lang="ts">
	import type { components } from '$lib/types/api';
	import { Button } from '$lib/components/ui/button';
	import { Input } from '$lib/components/ui/input';
	import { Label } from '$lib/components/ui/label';
	import * as Select from '$lib/components/ui/select';
	import { Loader2 } from 'lucide-svelte';

	type PetType = components['schemas']['PetTypeResponse'];
	type OwnerResponse = components['schemas']['OwnerResponse'];

	interface Props {
		name?: string;
		birthDate?: string;
		typeId?: number;
		petTypes: PetType[];
		owners?: OwnerResponse[];
		selectedOwnerId?: number;
		onSubmit: (data: { name: string; birthDate: string; typeId: number; ownerId?: number }) => Promise<void>;
		submitLabel?: string;
	}

	let {
		name: initialName = '',
		birthDate: initialBirthDate = '',
		typeId: initialTypeId,
		petTypes,
		owners,
		selectedOwnerId: initialOwnerId,
		onSubmit,
		submitLabel = 'Save'
	}: Props = $props();

	let name = $state(initialName);
	let birthDate = $state(initialBirthDate);
	let selectedTypeId = $state<number | undefined>(initialTypeId);
	let selectedOwnerId = $state<number | undefined>(initialOwnerId);
	let submitting = $state(false);

	let selectedType = $derived(petTypes.find((t) => t.id === selectedTypeId));
	let selectedOwner = $derived(owners?.find((o) => o.id === selectedOwnerId));

	const isValid = $derived(
		!!selectedTypeId && (owners === undefined || !!selectedOwnerId)
	);

	async function handleSubmit(e: Event) {
		e.preventDefault();
		if (!isValid) return;
		submitting = true;
		try {
			await onSubmit({
				name: name.trim(),
				birthDate,
				typeId: selectedTypeId!,
				ownerId: selectedOwnerId
			});
		} finally {
			submitting = false;
		}
	}
</script>

<form onsubmit={handleSubmit} class="space-y-6">
	<div class="space-y-2">
		<Label for="name">Pet Name</Label>
		<Input
			id="name"
			bind:value={name}
			placeholder="Enter pet name"
			required
			disabled={submitting}
		/>
	</div>

	<div class="space-y-2">
		<Label for="birthDate">Birth Date</Label>
		<Input
			id="birthDate"
			type="date"
			bind:value={birthDate}
			required
			disabled={submitting}
		/>
	</div>

	<div class="space-y-2">
		<Label for="petType">Pet Type</Label>
		<Select.Root
			type="single"
			value={selectedTypeId?.toString()}
			onValueChange={(value) => (selectedTypeId = value ? Number(value) : undefined)}
		>
			<Select.Trigger id="petType" class="w-full" disabled={submitting}>
				{selectedType?.name || 'Select a pet type'}
			</Select.Trigger>
			<Select.Content>
				{#each petTypes as petType (petType.id)}
					<Select.Item value={petType.id.toString()}>
						{petType.name}
					</Select.Item>
				{/each}
			</Select.Content>
		</Select.Root>
	</div>

	{#if owners !== undefined}
		<div class="space-y-2">
			<Label for="owner">Owner</Label>
			<Select.Root
				type="single"
				value={selectedOwnerId?.toString()}
				onValueChange={(value) => (selectedOwnerId = value ? Number(value) : undefined)}
			>
				<Select.Trigger id="owner" class="w-full" disabled={submitting}>
					{selectedOwner ? `${selectedOwner.firstName} ${selectedOwner.lastName}` : 'Search owners...'}
				</Select.Trigger>
				<Select.Content>
					{#each owners as owner (owner.id)}
						<Select.Item value={owner.id.toString()}>
							{owner.firstName} {owner.lastName}
						</Select.Item>
					{/each}
				</Select.Content>
			</Select.Root>
		</div>
	{/if}

	<div class="flex justify-end gap-3">
		<Button type="button" variant="outline" onclick={() => history.back()} disabled={submitting}>
			Cancel
		</Button>
		<Button type="submit" disabled={submitting || !isValid}>
			{#if submitting}
				<Loader2 class="mr-2 h-4 w-4 animate-spin" />
			{/if}
			{submitLabel}
		</Button>
	</div>
</form>
