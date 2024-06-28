<script lang="ts">
	import { DropdownMenu } from 'bits-ui';
	import { marked } from 'marked';

	import { flyAndScale } from '$lib/utils/transitions';
	import { createEventDispatcher, onMount, getContext, tick } from 'svelte';

	import ChevronDown from '$lib/components/icons/ChevronDown.svelte';
	import Check from '$lib/components/icons/Check.svelte';
	import Search from '$lib/components/icons/Search.svelte';

	import { deleteModel, getOllamaVersion, pullModel } from '$lib/apis/ollama';

	import { user, MODEL_DOWNLOAD_POOL, models, mobile } from '$lib/stores';
	import { toast } from 'svelte-sonner';
	import { capitalizeFirstLetter, sanitizeResponseContent, splitStream } from '$lib/utils';
	import { getModels } from '$lib/apis';

	import Tooltip from '$lib/components/common/Tooltip.svelte';

	const i18n = getContext('i18n');
	const dispatch = createEventDispatcher();

	export let values: string[] = [];
	export let placeholder = 'Select models';
	export let searchEnabled = true;
	export let searchPlaceholder = $i18n.t('Search models');
	export let items: {
		label: string;
		value: string;
		// eslint-disable-next-line @typescript-eslint/no-explicit-any
		[key: string]: any;
	} = [];

	export let className = 'w-[30rem]';

	let show = false;

	$: selectedModelsx = items.filter(item => values.includes(item.value));

	let searchValue = '';
	let ollamaVersion = null;

	$: filteredItems = items.filter(
		(item) =>
			(searchValue
				? item.value.toLowerCase().includes(searchValue.toLowerCase()) ||
				  item.label.toLowerCase().includes(searchValue.toLowerCase()) ||
				  (item.model?.info?.meta?.tags ?? []).some((tag) =>
						tag.name.toLowerCase().includes(searchValue.toLowerCase())
				  )
				: true) && !(item.model?.info?.meta?.hidden ?? false)
	);

	/*const pullModelHandler = async () => {
		const sanitizedModelTag = searchValue.trim().replace(/^ollama\s+(run|pull)\s+/, '');

		if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag]) {
			toast.error(
				$i18n.t(`Model '{{modelTag}}' is already in queue for downloading.`, {
					modelTag: sanitizedModelTag
				})
			);
			return;
		}
		if (Object.keys($MODEL_DOWNLOAD_POOL).length === 3) {
			toast.error(
				$i18n.t('Maximum of 3 models can be downloaded simultaneously. Please try again later.')
			);
			return;
		}

		const [res, controller] = await pullModel(localStorage.token, sanitizedModelTag, '0').catch(
			(error) => {
				toast.error(error);
				return null;
			}
		);

		if (res) {
			const reader = res.body
				.pipeThrough(new TextDecoderStream())
				.pipeThrough(splitStream('\n'))
				.getReader();

			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL,
				[sanitizedModelTag]: {
					...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
					abortController: controller,
					reader,
					done: false
				}
			});

			while (true) {
				try {
					const { value, done } = await reader.read();
					if (done) break;

					let lines = value.split('\n');

					for (const line of lines) {
						if (line !== '') {
							let data = JSON.parse(line);
							if (data.error) {
								throw data.error;
							}
							if (data.detail) {
								throw data.detail;
							}

							if (data.status) {
								if (data.digest) {
									let downloadProgress = 0;
									if (data.completed) {
										downloadProgress = Math.round((data.completed / data.total) * 1000) / 10;
									} else {
										downloadProgress = 100;
									}

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											pullProgress: downloadProgress,
											digest: data.digest
										}
									});
								} else {
									toast.success(data.status);

									MODEL_DOWNLOAD_POOL.set({
										...$MODEL_DOWNLOAD_POOL,
										[sanitizedModelTag]: {
											...$MODEL_DOWNLOAD_POOL[sanitizedModelTag],
											done: data.status === 'success'
										}
									});
								}
							}
						}
					}
				} catch (error) {
					if (typeof error !== 'string') {
						error = error.message;
					}

					toast.error(error);
					break;
				}
			}

			if ($MODEL_DOWNLOAD_POOL[sanitizedModelTag].done) {
				toast.success(
					$i18n.t(`Model '{{modelName}}' has been successfully downloaded.`, {
						modelName: sanitizedModelTag
					})
				);

				models.set(await getModels(localStorage.token));
			} else {
				toast.error($i18n.t('Download canceled'));
			}

			delete $MODEL_DOWNLOAD_POOL[sanitizedModelTag];

			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
		}
	};
*/
	onMount(async () => {
		ollamaVersion = await getOllamaVersion(localStorage.token).catch((error) => false);
	});

	/*const cancelModelPullHandler = async (model: string) => {
		const { reader, abortController } = $MODEL_DOWNLOAD_POOL[model];
		if (abortController) {
			abortController.abort();
		}
		if (reader) {
			await reader.cancel();
			delete $MODEL_DOWNLOAD_POOL[model];
			MODEL_DOWNLOAD_POOL.set({
				...$MODEL_DOWNLOAD_POOL
			});
			await deleteModel(localStorage.token, model);
			toast.success(`${model} download has been canceled`);
		}
	};*/

	const toggleModelSelection = (item) => {
		if (values.includes(item.value)) {
			values = values.filter((v) => v !== item.value);
		} else {
			values = [...values, item.value];
		}
		dispatch('change', values);
		console.log(values)
		console.log(selectedModelsx)
	};
</script>

<DropdownMenu.Root
	bind:open={show}
	onOpenChange={async () => {
		searchValue = '';
		window.setTimeout(() => document.getElementById('model-search-input')?.focus(), 0);
	}}
>
	<DropdownMenu.Trigger class="relative w-full" aria-label={placeholder}>
		<div
			class="flex w-full text-left px-0.5 outline-none bg-transparent truncate text-lg font-semibold placeholder-gray-400 focus:outline-none"
		>
			{#if selectedModelsx.length > 0}
				{selectedModelsx.map((model) => model.label).join(', ')}
			{:else}
				{placeholder}
			{/if}
			<ChevronDown className=" self-center ml-2 size-3" strokeWidth="2.5" />
		</div>
	</DropdownMenu.Trigger>

	<DropdownMenu.Content
		class=" z-[9999] {$mobile
			? `w-full`
			: `${className}`} max-w-[calc(100vw-1rem)] justify-start rounded-xl  bg-white dark:bg-gray-850 dark:text-white shadow-lg border border-gray-300/30 dark:border-gray-850/50  outline-none "
		transition={flyAndScale}
		side={$mobile ? 'bottom' : 'bottom-start'}
		sideOffset={4}
	>
		<slot>
			{#if searchEnabled}
				<div class="flex items-center gap-2.5 px-5 mt-3.5 mb-3">
					<Search className="size-4" strokeWidth="2.5" />

					<input
						id="model-search-input"
						bind:value={searchValue}
						class="w-full text-sm bg-transparent outline-none"
						placeholder={searchPlaceholder}
						autocomplete="off"
					/>
				</div>

				<hr class="border-gray-100 dark:border-gray-800" />
			{/if}

			<div class="px-3 my-2 max-h-64 overflow-y-auto scrollbar-hidden">
				{#each filteredItems as item}
					<button
						aria-label="model-item"
						class="flex w-full text-left font-medium line-clamp-1 select-none items-center rounded-button py-2 pl-3 pr-1.5 text-sm text-gray-700 dark:text-gray-100 outline-none transition-all duration-75 hover:bg-gray-100 dark:hover:bg-gray-800 rounded-lg cursor-pointer data-[highlighted]:bg-muted"
						on:click={() => toggleModelSelection(item)}
					>
						<div class="flex flex-col">
							{#if $mobile && (item?.model?.info?.meta?.tags ?? []).length > 0}
								<div class="flex gap-0.5 self-start h-full mb-0.5 -translate-x-1">
									{#each item.model?.info?.meta?.tags as tag (tag)}
										<span
											class="capitalize px-1.5 py-0.5 rounded-full bg-black/5 dark:bg-white/10 text-[0.625rem] h-full text-black/60 dark:text-white/80 tracking-wide"
										>
											{tag.name}
										</span>
									{/each}
								</div>
							{/if}

							<div class="flex items-center gap-1.5">
								<span>{item.label}</span>
							</div>

							{#if item?.model?.info?.meta?.description}
								<div class="text-xs text-gray-500 dark:text-gray-400 line-clamp-2 mt-0.5">
									{@html sanitizeResponseContent(marked(item?.model?.info?.meta?.description))}
								</div>
							{/if}
						</div>

						{#if values.includes(item.value)}
							<Check class="size-4 text-gray-800 dark:text-gray-200" />
						{/if}
					</button>
				{/each}

				<!--{#if ollamaVersion && ollamaVersion !== false}
					<div class="mt-2 text-sm dark:text-white px-2">
						<div class="flex flex-col items-center">
							<p class="text-center text-xs dark:text-white text-gray-800 w-full">
								{ollamaVersion}
							</p>
							<button
								class="w-full text-left p-2.5 rounded-md border border-gray-500/50 hover:bg-gray-100 dark:hover:bg-gray-700"
								on:click={pullModelHandler}
							>
								Pull Model
							</button>
						</div>
					</div>
				{/if}-->
			</div>
		</slot>
	</DropdownMenu.Content>
</DropdownMenu.Root>

<style>
	/* Add custom styles here if needed */
</style>
