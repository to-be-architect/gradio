<script lang="ts">
	import { createEventDispatcher } from "svelte";
	import type { SelectData } from "@gradio/utils";

	export let value: boolean;
	export let disabled: boolean = false;
	export let label: string;

	const dispatch = createEventDispatcher<{
		change: boolean;
		select: SelectData;
	}>();

	function handle_change(value: boolean) {
		dispatch("change", value);
	}

	$: handle_change(value);
</script>

<!-- svelte-ignore a11y-label-has-associated-control -->
<label class:disabled>
	<input
		bind:checked={value}
		on:input={(evt) =>
			dispatch("select", {
				index: 0,
				value: label,
				selected: evt.currentTarget.checked
			})}
		{disabled}
		type="checkbox"
		name="test"
		data-testid="checkbox"
	/>
	<span class="ml-2">{label}</span>
</label>

<style>
	label {
		display: flex;
		align-items: center;
		cursor: pointer;
		color: var(--body-text-color);
		font-weight: var(--checkbox-label-text-weight);
		font-size: var(--checkbox-label-text-size);
		line-height: var(--line-md);
	}

	label > * + * {
		margin-left: var(--size-2);
	}

	input {
		--ring-color: transparent;
		position: relative;
		box-shadow: var(--input-shadow);
		border: 1px solid var(--checkbox-border-color);
		border-radius: var(--checkbox-border-radius);
		background-image: var(--checkbox-check);
		background-color: var(--checkbox-background-color);
		line-height: var(--line-sm);
	}

	input:checked,
	input:checked:hover,
	input:checked:focus {
		border-color: var(--checkbox-border-color-selected);
		background-color: var(--checkbox-background-color-selected);
	}

	input:hover {
		border-color: var(--checkbox-border-color-hover);
		background-color: var(--checkbox-background-color-hover);
	}

	input:focus {
		border-color: var(--checkbox-border-color-focus);
		background-color: var(--checkbox-background-color-focus);
	}

	input[disabled],
	.disabled {
		cursor: not-allowed;
	}
</style>
