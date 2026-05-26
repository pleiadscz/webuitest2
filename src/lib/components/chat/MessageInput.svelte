<script lang="ts">
	import { toast } from '$lib/notification';
	import { onMount, tick } from 'svelte';
	import { settings, hfEnabled, searchEnabled, githubEnabled, kernelEnabled, kernelBrowserUrl, kernelSessionId, desktopEnabled, desktopLoading, desktopBrowserUrl, desktopSessionId } from '$lib/stores';
	import { blobToFile, calculateSHA256, findWordIndices } from '$lib/utils';

	import Prompts from './MessageInput/PromptCommands.svelte';
	import Suggestions from './MessageInput/Suggestions.svelte';
	import AddFilesPlaceholder from '../AddFilesPlaceholder.svelte';
	import { SUPPORTED_FILE_TYPE, SUPPORTED_FILE_EXTENSIONS } from '$lib/constants';
	import Documents from './MessageInput/Documents.svelte';
	import Models from './MessageInput/Models.svelte';
	import BottomDrawer from '../common/BottomDrawer.svelte';

	let drawerOpen = false;

	const closeMenu = () => {
		drawerOpen = false;
	};

	const setSingleMode = (mode) => {
		hfEnabled.set(mode === 'hf');
		searchEnabled.set(mode === 'search');
		githubEnabled.set(mode === 'github');
		kernelEnabled.set(mode === 'kernel');
		desktopEnabled.set(mode === 'desktop');
	};

	export let submitPrompt: Function;
	export let stopResponse: Function;
	export let suggestionPrompts = [];
	export let autoScroll = true;
	let chatTextAreaElement: HTMLTextAreaElement;
	let filesInputElement;

	let promptsElement;
	let documentsElement;
	let modelsElement;

	let inputFiles;
	let dragged = false;

	let user = null;
	let chatInputPlaceholder = '';

	export let files = [];
	export let fileUploadEnabled = true;
	export let speechRecognitionEnabled = true;
	export let prompt = '';
	export let messages = [];

	let speechRecognition;

	$: if (prompt) {
		if (chatTextAreaElement) {
			chatTextAreaElement.style.height = '';
			chatTextAreaElement.style.height = Math.min(chatTextAreaElement.scrollHeight, 200) + 'px';
		}
	}

	let mediaRecorder;
	let audioChunks = [];
	let isRecording = false;
	const MIN_DECIBELS = -45;

	const scrollToBottom = () => {
		const element = document.getElementById('messages-container');
		element.scrollTop = element.scrollHeight;
	};

	const startRecording = async () => {
		const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
		mediaRecorder = new MediaRecorder(stream);
		mediaRecorder.onstart = () => { isRecording = true; };
		mediaRecorder.ondataavailable = (event) => audioChunks.push(event.data);
		mediaRecorder.onstop = async () => {
			isRecording = false;
			const audioBlob = new Blob(audioChunks, { type: 'audio/wav' });
			const file = blobToFile(audioBlob, 'recording.wav');
			audioChunks = [];
		};
		mediaRecorder.start();
		monitorSilence(stream);
	};

	const monitorSilence = (stream) => {
		const audioContext = new AudioContext();
		const audioStreamSource = audioContext.createMediaStreamSource(stream);
		const analyser = audioContext.createAnalyser();
		analyser.minDecibels = MIN_DECIBELS;
		audioStreamSource.connect(analyser);
		const bufferLength = analyser.frequencyBinCount;
		const domainData = new Uint8Array(bufferLength);
		let lastSoundTime = Date.now();
		const detectSound = () => {
			analyser.getByteFrequencyData(domainData);
			if (domainData.some((value) => value > 0)) lastSoundTime = Date.now();
			if (isRecording && Date.now() - lastSoundTime > 3000) {
				mediaRecorder.stop();
				audioContext.close();
				return;
			}
			window.requestAnimationFrame(detectSound);
		};
		window.requestAnimationFrame(detectSound);
	};

	const uploadDoc = async (file) => {
		const doc = { type: 'doc', name: file.name, collection_name: '', upload_status: false, error: '' };
		try {
			files = [...files, doc];
			files = files;
		} catch (e) {
			files = files.filter((f) => f.name !== file.name);
			toast.error(e);
		}
	};

	const uploadWeb = async (url) => {
		const doc = { type: 'doc', name: url, collection_name: '', upload_status: false, url: url, error: '' };
		try {
			files = [...files, doc];
			files = files;
		} catch (e) {
			files = files.filter((f) => f.name !== url);
			toast.error(e);
		}
	};

	onMount(() => {
		window.setTimeout(() => chatTextAreaElement?.focus(), 0);
		const dropZone = document.querySelector('body');
		const handleKeyDown = (event: KeyboardEvent) => { if (event.key === 'Escape') dragged = false; };
		const onDragOver = (e) => { e.preventDefault(); dragged = true; };
		const onDragLeave = () => { dragged = false; };
		const onDrop = async (e) => {
			e.preventDefault();
			if (e.dataTransfer?.files) {
				const inputFiles = Array.from(e.dataTransfer?.files);
				inputFiles.forEach((file) => {
					if (['image/gif', 'image/jpeg', 'image/png'].includes(file['type'])) {
						let reader = new FileReader();
						reader.onload = (event) => { files = [...files, { type: 'image', url: `${event.target.result}` }]; };
						reader.readAsDataURL(file);
					} else {
						uploadDoc(file);
					}
				});
			}
			dragged = false;
		};
		window.addEventListener('keydown', handleKeyDown);
		dropZone?.addEventListener('dragover', onDragOver);
		dropZone?.addEventListener('drop', onDrop);
		dropZone?.addEventListener('dragleave', onDragLeave);
		return () => {
			window.removeEventListener('keydown', handleKeyDown);
			dropZone?.removeEventListener('dragover', onDragOver);
			dropZone?.removeEventListener('drop', onDrop);
			dropZone?.removeEventListener('dragleave', onDragLeave);
		};
	});
</script>

{#if dragged}
	<div class="fixed lg:w-[calc(100%-260px)] w-full h-full flex z-50 touch-none pointer-events-none" id="dropzone" role="region" aria-label="Drag and Drop Container">
		<div class="absolute w-full h-full backdrop-blur bg-gray-800/40 flex justify-center">
			<div class="m-auto pt-64 flex flex-col justify-center"><div class="max-w-md"><AddFilesPlaceholder /></div></div>
		</div>
	</div>
{/if}

<div class="w-full">
	<div class="px-2.5 -mb-0.5 mx-auto inset-x-0 bg-transparent flex justify-center">
		<div class="flex flex-col max-w-3xl w-full">
			<div class="relative">
				{#if autoScroll === false && messages.length > 0}
					<div class=" absolute -top-12 left-0 right-0 flex justify-center">
						<button class=" bg-white border border-gray-100 dark:border-none dark:bg-white/20 p-1.5 rounded-full" on:click={() => { autoScroll = true; scrollToBottom(); }}>
							<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="w-5 h-5"><path fill-rule="evenodd" d="M10 3a.75.75 0 01.75.75v10.638l3.96-4.158a.75.75 0 111.08 1.04l-5.25 5.5a.75.75 0 01-1.08 0l-5.25-5.5a.75.75 0 111.08-1.04l3.96 4.158V3.75A.75.75 0 0110 3z" clip-rule="evenodd" /></svg>
						</button>
					</div>
				{/if}
			</div>
			<div class="w-full relative">
				{#if prompt.charAt(0) === '/'}<Prompts bind:this={promptsElement} bind:prompt />
				{:else if prompt.charAt(0) === '#'}<Documents bind:this={documentsElement} bind:prompt on:url={(e) => uploadWeb(e.detail)} />
				{:else if prompt.charAt(0) === '@'}<Models bind:this={modelsElement} bind:prompt bind:user bind:chatInputPlaceholder {messages} />{/if}
			</div>
		</div>
	</div>

	<div class="bg-white dark:bg-gray-900">
		<div class="max-w-3xl px-2.5 mx-auto inset-x-0">
			<div class="pb-2">
				<input bind:this={filesInputElement} bind:files={inputFiles} type="file" hidden multiple on:change={async () => { if(inputFiles) Array.from(inputFiles).forEach(uploadDoc); }} />
				<form class="flex flex-col relative w-full rounded-3xl px-1.5 border border-gray-100 dark:border-gray-850 bg-white dark:bg-[#303030]" on:submit|preventDefault={() => submitPrompt(prompt, user)}>
					<div class="flex">
						{#if fileUploadEnabled}
							<div class="self-end mb-2 ml-1">
								<button class="bg-gray-50 hover:bg-gray-100 text-gray-800 dark:bg-gray-850 dark:text-white dark:hover:bg-gray-800 transition rounded-full p-1.5" type="button" on:click={() => window.innerWidth >= 768 ? filesInputElement.click() : (drawerOpen = true)}>
									<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="w-[1.2rem] h-[1.2rem]"><path d="M8.75 3.75a.75.75 0 0 0-1.5 0v3.5h-3.5a.75.75 0 0 0 0 1.5h3.5v3.5a.75.75 0 0 0 1.5 0v-3.5h3.5a.75.75 0 0 0 0-1.5h-3.5v-3.5Z" /></svg>
								</button>
							</div>
						{/if}
						<textarea id="chat-textarea" bind:this={chatTextAreaElement} class="dark:bg-[#303030] dark:text-gray-100 outline-none w-full py-3 px-3 rounded-xl resize-none h-[48px]" placeholder="Wyślij wiadomość" bind:value={prompt} rows="1" />
						<div class="self-end mb-2 flex space-x-1 mr-1">
							<button class="bg-black text-white hover:bg-gray-900 dark:bg-white dark:text-black rounded-full p-1.5" type="submit"><svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 16 16" fill="currentColor" class="w-5 h-5"><path fill-rule="evenodd" d="M8 14a.75.75 0 0 1-.75-.75V4.56L4.03 7.78a.75.75 0 0 1-1.06-1.06l4.5-4.5a.75.75 0 0 1 1.06 0l4.5 4.5a.75.75 0 0 1-1.06 1.06L8.75 4.56v8.69A.75.75 0 0 1 8 14Z" clip-rule="evenodd" /></svg></button>
						</div>
					</div>
				</form>
			</div>
		</div>
	</div>
</div>

<BottomDrawer bind:open={drawerOpen}>
	<button class="w-full p-4 text-left" on:click={() => { closeMenu(); filesInputElement.click(); }}>Wybierz pliki</button>
</BottomDrawer>
