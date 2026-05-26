<script lang="ts">
	import { toast } from '$lib/notification';
	import { onMount } from 'svelte';
	import { settings, desktopLoading } from '$lib/stores';
	import { blobToFile } from '$lib/utils';
	import AddFilesPlaceholder from '../AddFilesPlaceholder.svelte';
	import BottomDrawer from '../common/BottomDrawer.svelte';

	export let submitPrompt: Function;
	export let stopResponse: Function;
	export let autoScroll = true;
	export let files = [];
	export let fileUploadEnabled = true;
	export let prompt = '';
	export let messages = [];

	let drawerOpen = false;
	let chatTextAreaElement: HTMLTextAreaElement;
	let filesInputElement;
	let dragged = false;

	$: if (prompt && chatTextAreaElement) {
		chatTextAreaElement.style.height = '';
		chatTextAreaElement.style.height = Math.min(chatTextAreaElement.scrollHeight, 200) + 'px';
	}

	const scrollToBottom = () => {
		const element = document.getElementById('messages-container');
		if (element) element.scrollTop = element.scrollHeight;
	};

	const uploadFile = (file) => {
		files = [...files, { type: 'doc', name: file.name, upload_status: true }];
	};

	onMount(() => {
		window.setTimeout(() => chatTextAreaElement?.focus(), 0);
		
		const onDrop = (e) => {
			e.preventDefault();
			if (e.dataTransfer?.files) {
				Array.from(e.dataTransfer.files).forEach(uploadFile);
			}
			dragged = false;
		};

		window.addEventListener('dragover', (e) => { e.preventDefault(); dragged = true; });
		window.addEventListener('dragleave', () => { dragged = false; });
		window.addEventListener('drop', onDrop);

		return () => {
			window.removeEventListener('dragover', (e) => e.preventDefault());
			window.removeEventListener('dragleave', () => dragged = false);
			window.removeEventListener('drop', onDrop);
		};
	});
</script>

{#if dragged}
	<div class="fixed w-full h-full flex z-50 pointer-events-none">
		<div class="absolute w-full h-full backdrop-blur bg-gray-800/40 flex justify-center">
			<div class="m-auto"><AddFilesPlaceholder /></div>
		</div>
	</div>
{/if}

<div class="w-full">
	<div class="bg-white dark:bg-gray-900">
		<div class="max-w-3xl px-2.5 mx-auto">
			<div class="pb-2">
				<input bind:this={filesInputElement} type="file" hidden multiple on:change={(e) => {
					if (e.target.files) Array.from(e.target.files).forEach(uploadFile);
				}} />
				
				<form class="flex flex-col relative w-full rounded-3xl px-1.5 border border-gray-100 dark:border-gray-850 bg-white dark:bg-[#303030]" 
					on:submit|preventDefault={() => { if (prompt.trim()) submitPrompt(prompt, null); }}>
					
					<div class="flex">
						{#if fileUploadEnabled}
							<div class="self-end mb-2 ml-1">
								<button class="bg-gray-50 hover:bg-gray-100 text-gray-800 dark:bg-gray-850 dark:text-white dark:hover:bg-gray-800 transition rounded-full p-1.5" 
									type="button" on:click={() => window.innerWidth >= 768 ? filesInputElement.click() : (drawerOpen = true)}>
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="w-[1.2rem] h-[1.2rem]">
										<path d="M8.75 3.75a.75.75 0 0 0-1.5 0v3.5h-3.5a.75.75 0 0 0 0 1.5h3.5v3.5a.75.75 0 0 0 1.5 0v-3.5h3.5a.75.75 0 0 0 0-1.5h-3.5v-3.5Z" />
									</svg>
								</button>
							</div>
						{/if}

						<textarea id="chat-textarea" bind:this={chatTextAreaElement} 
							class="dark:bg-[#303030] dark:text-gray-100 outline-none w-full py-3 px-3 rounded-xl resize-none h-[48px]" 
							placeholder="Wyślij wiadomość" bind:value={prompt} rows="1" />

						<div class="self-end mb-2 flex space-x-1 mr-1">
							{#if messages.length == 0 || messages.at(-1).done == true}
								<button class="bg-black text-white hover:bg-gray-900 dark:bg-white dark:text-black rounded-full p-1.5" type="submit">
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="w-5 h-5">
										<path fill-rule="evenodd" d="M8 14a.75.75 0 0 1-.75-.75V4.56L4.03 7.78a.75.75 0 0 1-1.06-1.06l4.5-4.5a.75.75 0 0 1 1.06 0l4.5 4.5a.75.75 0 0 1-1.06 1.06L8.75 4.56v8.69A.75.75 0 0 1 8 14Z" clip-rule="evenodd" />
									</svg>
								</button>
							{:else}
								<button class="bg-black text-white hover:bg-gray-900 dark:bg-white dark:text-black rounded-full p-1.5" type="button" on:click={stopResponse}>
									<svg width="24" height="24" viewBox="0 0 24 24" fill="currentColor" class="w-5 h-5"><path d="M6 8C6 6.89543 6.89543 6 8 6H16C17.1046 6 18 6.89543 18 8V16C18 17.1046 17.1046 18 16 18H8C6.89543 18 6 17.1046 6 16V8Z" /></svg>
								</button>
							{/if}
						</div>
					</div>
				</form>
			</div>
		</div>
	</div>
</div>

<BottomDrawer bind:open={drawerOpen}>
	<button class="w-full p-4 text-left" on:click={() => { drawerOpen = false; filesInputElement.click(); }}>Wybierz pliki</button>
</BottomDrawer>
