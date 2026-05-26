<script lang="ts">
        import { toast } from '$lib/notification';
        import { onMount, tick } from 'svelte';
        import { settings, hfEnabled, searchEnabled, githubEnabled, kernelEnabled, kernelBrowserUrl, kernelSessionId, desktopEnabled, desktopLoading, desktopBrowserUrl, desktopSessionId } from '$lib/stores';
        import { blobToFile, findWordIndices } from '$lib/utils';

        import Suggestions from './MessageInput/Suggestions.svelte';
        import AddFilesPlaceholder from '../AddFilesPlaceholder.svelte';
        import { SUPPORTED_FILE_TYPE, SUPPORTED_FILE_EXTENSIONS } from '$lib/constants';
        import Documents from './MessageInput/Documents.svelte';
        import Models from './MessageInput/Models.svelte';
        import BottomDrawer from '../common/BottomDrawer.svelte';

        let drawerOpen = false;
        let dropdownOpen = false;
        let dropdownDirection: 'up' | 'down' = 'down';

        const closeMenu = () => {
                drawerOpen = false;
                dropdownOpen = false;
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
                mediaRecorder.onstart = () => {
                        isRecording = true;

                };
                mediaRecorder.ondataavailable = (event) => audioChunks.push(event.data);
                mediaRecorder.onstop = async () => {
                        isRecording = false;

                        // Create a blob from the audio chunks
                        const audioBlob = new Blob(audioChunks, { type: 'audio/wav' });

                        const file = blobToFile(audioBlob, 'recording.wav');

                        const res = null;

                        if (res) {
                                prompt = res.text;
                                await tick();
                                chatTextAreaElement?.focus();

                                if (prompt !== '' && $settings?.speechAutoSend === true) {
                                        submitPrompt(prompt, user);
                                }
                        }

                        audioChunks = [];
                };

                // Start recording
                mediaRecorder.start();

                // Monitor silence
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

                        if (domainData.some((value) => value > 0)) {
                                lastSoundTime = Date.now();
                        }

                        if (isRecording && Date.now() - lastSoundTime > 3000) {
                                mediaRecorder.stop();
                                audioContext.close();
                                return;
                        }

                        window.requestAnimationFrame(detectSound);
                };

                window.requestAnimationFrame(detectSound);
        };

        const speechRecognitionHandler = () => {
                // Check if SpeechRecognition is supported

                if (isRecording) {
                        if (speechRecognition) {
                                speechRecognition.stop();
                        }

                        if (mediaRecorder) {
                                mediaRecorder.stop();
                        }
                } else {
                        isRecording = true;

                        if ($settings?.audio?.STTEngine ?? '' !== '') {
                                startRecording();
                        } else {
                                if ('SpeechRecognition' in window || 'webkitSpeechRecognition' in window) {
                                        // Create a SpeechRecognition object
                                        speechRecognition = new (window.SpeechRecognition || window.webkitSpeechRecognition)();

                                        // Set continuous to true for continuous recognition
                                        speechRecognition.continuous = true;

                                        // Set the timeout for turning off the recognition after inactivity (in milliseconds)
                                        const inactivityTimeout = 3000; // 3 seconds

                                        let timeoutId;
                                        // Start recognition
                                        speechRecognition.start();

                                        // Event triggered when speech is recognized
                                        speechRecognition.onresult = async (event) => {
                                                // Clear the inactivity timeout
                                                clearTimeout(timeoutId);

                                                // Handle recognized speech

                                                const transcript = event.results[Object.keys(event.results).length - 1][0].transcript;

                                                prompt = `${prompt}${transcript}`;

                                                await tick();
                                                chatTextAreaElement?.focus();

                                                // Restart the inactivity timeout
                                                timeoutId = setTimeout(() => {

                                                        speechRecognition.stop();
                                                }, inactivityTimeout);
                                        };

                                        // Event triggered when recognition is ended
                                        speechRecognition.onend = function () {
                                                // Restart recognition after it ends

                                                isRecording = false;
                                                if (prompt !== '' && $settings?.speechAutoSend === true) {
                                                        submitPrompt(prompt, user);
                                                }
                                        };

                                        // Event triggered when an error occurs
                                        speechRecognition.onerror = function (event) {

                                                toast.error(`Błąd rozpoznawania mowy: ${event.error}`);
                                                isRecording = false;
                                        };
                                } else {
                                        toast.error('API Rozpoznawania Mowy nie jest obsługiwane w tej przeglądarce.');
                                }
                        }
                }
        };

        const uploadDoc = async (file) => {

                const doc = {
                        type: 'doc',
                        name: file.name,
                        collection_name: '',
                        upload_status: false,
                        error: ''
                };

                try {
                        files = [...files, doc];

                        if (['audio/mpeg', 'audio/wav'].includes(file['type'])) {
                                const res = null;

                                if (res) {

                                        const blob = new Blob([res.text], { type: 'text/plain' });
                                        file = blobToFile(blob, `${file.name}.txt`);
                                }
                        }

                        const res = null;

                        if (res) {
                                doc.upload_status = true;
                                doc.collection_name = res.collection_name;
                                files = files;
                        }
                } catch (e) {
                        // Remove the failed doc from the files array
                        files = files.filter((f) => f.name !== file.name);
                        toast.error(e);
                }
        };

        const uploadWeb = async (url) => {

                const doc = {
                        type: 'doc',
                        name: url,
                        collection_name: '',
                        upload_status: false,
                        url: url,
                        error: ''
                };

                try {
                        files = [...files, doc];
                        const res = null;

                        if (res) {
                                doc.upload_status = true;
                                doc.collection_name = res.collection_name;
                                files = files;
                        }
                } catch (e) {
                        // Remove the failed doc from the files array
                        files = files.filter((f) => f.name !== url);
                        toast.error(e);
                }
        };

        onMount(() => {
                window.setTimeout(() => chatTextAreaElement?.focus(), 0);

                const dropZone = document.querySelector('body');

                const handleKeyDown = (event: KeyboardEvent) => {
                        if (event.key === 'Escape') {

                                dragged = false;
                        }
                };

                const onDragOver = (e) => {
                        e.preventDefault();
                        dragged = true;
                };

                const onDragLeave = () => {
                        dragged = false;
                };

                const onDrop = async (e) => {
                        e.preventDefault();

                        if (e.dataTransfer?.files) {
                                const inputFiles = Array.from(e.dataTransfer?.files);

                                if (inputFiles && inputFiles.length > 0) {
                                        inputFiles.forEach((file) => {

                                                if (['image/gif', 'image/jpeg', 'image/png'].includes(file['type'])) {
                                                        let reader = new FileReader();
                                                        reader.onload = (event) => {
                                                                files = [
                                                                        ...files,
                                                                        {
                                                                                type: 'image',
                                                                                url: `${event.target.result}`
                                                                        }
                                                                ];
                                                        };
                                                        reader.readAsDataURL(file);
                                                } else if (
                                                        SUPPORTED_FILE_TYPE.includes(file['type']) ||
                                                        SUPPORTED_FILE_EXTENSIONS.includes(file.name.split('.').at(-1))
                                                ) {
                                                        uploadDoc(file);
                                                } else {
                                                        toast.error(
                                                                `Unknown File Type '{{file_type}}', but accepting and treating as plain text`,
                                                                        { file_type: file['type'] }
                                                        );
                                                        uploadDoc(file);
                                                }
                                        });
                                } else {
                                        toast.error('Plik nie został znaleziony.');
                                }
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
        <div
                class="fixed top-0 left-0 w-full h-full flex justify-center items-center z-[9999] bg-gray-50/60 dark:bg-gray-900/60 backdrop-blur-sm binary-transparency"
        >
                <div class="m-auto flex flex-col pointer-events-none">
                        <div class="m-auto mb-4">
                                <svg
                                        xmlns="http://www.w3.org/2000/svg"
                                        viewBox="0 0 24 24"
                                        fill="currentColor"
                                        class="w-12 h-12"
                                >
                                        <path
                                                fill-rule="evenodd"
                                                d="M10.5 3.75a6 6 0 0 0-5.98 6.496A5.25 5.25 0 0 0 6.75 20.25H18a4.5 4.5 0 0 0 1.106-8.866 9 9 0 0 0-8.606-7.634ZM11.25 10.5V15h1.5v-4.5h2.25L12 7.5l-3 3h2.25Z"
                                                clip-rule="evenodd"
                                        />
                                </svg>
                        </div>
                        <div class="m-auto text-2xl font-medium">Upuść pliki tutaj</div>
                </div>
        </div>
{/if}

<div class="w-full font-primary">
        <div class=" mx-auto inset-x-0 bg-transparent flex justify-center">
                <div class="flex flex-col px-3 max-w-6xl w-full">
                        <div class="relative">
                                {#if autoScroll === false && messages.length > 0}
                                        <div class=" absolute -top-12 left-0 right-0 flex justify-center z-30">
                                                <button
                                                        class=" bg-white border border-gray-100 dark:border-none dark:bg-white/20 p-1.5 rounded-full"
                                                        on:click={() => {
                                                                autoScroll = true;
                                                                scrollToBottom();
                                                        }}
                                                >
                                                        <svg
                                                                xmlns="http://www.w3.org/2000/svg"
                                                                viewBox="0 0 20 20"
                                                                fill="currentColor"
                                                                class="w-5 h-5"
                                                        >
                                                                <path
                                                                        fill-rule="evenodd"
                                                                        d="M10 3a.75.75 0 01.75.75v10.638l3.96-4.158a.75.75 0 111.08 1.04l-5.25 5.5a.75.75 0 01-1.08 0l-5.25-5.5a.75.75 0 111.08-1.04l3.96 4.158V3.75A.75.75 0 0110 3z"
                                                                        clip-rule="evenodd"
                                                                />
                                                        </svg>
                                                </button>
                                        </div>
                                {/if}
                        </div>
                </div>
        </div>

        <div class="bg-transparent">
                <div class="max-w-6xl px-2.5 mx-auto inset-x-0">
                        <div class="">
                                <input
                                        bind:this={filesInputElement}
                                        bind:files={inputFiles}
                                        type="file"
                                        hidden
                                        multiple
                                        on:change={async () => {
                                                if (inputFiles && inputFiles.length > 0) {
                                                        const _inputFiles = Array.from(inputFiles);
                                                        _inputFiles.forEach((file) => {
                                                                if (['image/gif', 'image/jpeg', 'image/png'].includes(file['type'])) {
                                                                        let reader = new FileReader();
                                                                        reader.onload = (event) => {
                                                                                files = [
                                                                                        ...files,
                                                                                        {
                                                                                                type: 'image',
                                                                                                url: `${event.target.result}`
                                                                                        }
                                                                                ];
                                                                        };
                                                                        reader.readAsDataURL(file);
                                                                } else {
                                                                        uploadDoc(file);
                                                                }
                                                        });
                                                } else {
                                                        toast.error('Plik nie został znaleziony.');
                                                }

                                                filesInputElement.value = '';
                                        }}
                                />

                                <form
                                        class="w-full flex flex-col gap-1.5"
                                        on:submit|preventDefault={() => {
                                                submitPrompt(prompt, user);
                                                prompt = '';
                                        }}
                                >
                                        <div
                                                class="flex-1 flex flex-col relative w-full rounded-3xl border border-gray-200 dark:border-none bg-white dark:bg-[#222] px-2 pt-2 pb-2.5 shadow-[0_0_40px_rgba(0,0,0,0.04)] dark:shadow-none transition"
                                        >
                                                {#if files.length > 0}
                                                        <div class="mx-2 mt-2 mb-1 flex flex-wrap gap-2.5">
                                                                {#each files as file, fileIdx}
                                                                <div class=" relative group">
                                                                        {#if file.type === 'image'}
                                                                                <img src={file.url} alt="input" class=" h-16 w-16 rounded-xl object-cover" />
                                                                        {:else if file.type === 'doc'}
                                                                                <div
                                                                                        class="h-16 w-[15rem] flex items-center space-x-3 px-2.5 dark:bg-gray-600 rounded-xl border border-gray-200 dark:border-none"
                                                                                >
                                                                                        <div class="p-2.5 bg-red-400 text-white rounded-lg">
                                                                                                {#if file.upload_status}
                                                                                                        <svg
                                                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                                                viewBox="0 0 24 24"
                                                                                                                fill="currentColor"
                                                                                                                class="w-6 h-6"
                                                                                                        >
                                                                                                                <path
                                                                                                                        fill-rule="evenodd"
                                                                                                                        d="M5.625 1.5c-1.036 0-1.875.84-1.875 1.875v17.25c0 1.035.84 1.875 1.875 1.875h12.75c1.035 0 1.875-.84 1.875-1.875V12.75A3.75 3.75 0 0 0 16.5 9h-1.875a1.875 1.875 0 0 1-1.875-1.875V5.25A3.75 3.75 0 0 0 9 1.5H5.625ZM7.5 15a.75.75 0 0 1 .75-.75h7.5a.75.75 0 0 1 0 1.5h-7.5A.75.75 0 0 1 7.5 15Zm.75 2.25a.75.75 0 0 0 0 1.5H12a.75.75 0 0 0 0-1.5H8.25Z"
                                                                                                                        clip-rule="evenodd"
                                                                                                                />
                                                                                                                <path
                                                                                                                        d="M12.971 1.816A5.23 5.23 0 0 1 14.25 5.25v1.875c0 .207.168.375.375.375H16.5a5.23 5.23 0 0 1 3.434 1.279 9.768 9.768 0 0 0-6.963-6.963Z"
                                                                                                                />
                                                                                                        </svg>
                                                                                                {:else}
                                                                                                        <svg
                                                                                                                class=" w-6 h-6 translate-y-[0.5px]"
                                                                                                                fill="currentColor"
                                                                                                                viewBox="0 0 24 24"
                                                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                                                ><style>
                                                                                                                        .spinner_qM83 {
                                                                                                                                animation: spinner_8HQG 1.05s infinite;
                                                                                                                        }
                                                                                                                        .spinner_oXPr {
                                                                                                                                animation-delay: 0.1s;
                                                                                                                        }
                                                                                                                        .spinner_ZTLf {
                                                                                                                                animation-delay: 0.2s;
                                                                                                                        }
                                                                                                                        @keyframes spinner_8HQG {
                                                                                                                                0%,
                                                                                                                                57.14% {
                                                                                                                                        animation-timing-function: cubic-bezier(0.33, 0.66, 0.66, 1);
                                                                                                                                        transform: translate(0);
                                                                                                                                }
                                                                                                                                28.57% {
                                                                                                                                        animation-timing-function: cubic-bezier(0.33, 0, 0.66, 0.33);
                                                                                                                                        transform: translateY(-6px);
                                                                                                                                }
                                                                                                                                100% {
                                                                                                                                        transform: translate(0);
                                                                                                                                }
                                                                                                                        }
                                                                                                                </style><circle class="spinner_qM83" cx="4" cy="12" r="2.5" /><circle
                                                                                                                        class="spinner_qM83 spinner_oXPr"
                                                                                                                        cx="12"
                                                                                                                        cy="12"
                                                                                                                        r="2.5"
                                                                                                                /><circle
                                                                                                                        class="spinner_qM83 spinner_ZTLf"
                                                                                                                        cx="20"
                                                                                                                        cy="12"
                                                                                                                        r="2.5"
                                                                                                                /></svg
                                                                                                        >
                                                                                                {/if}
                                                                                        </div>

                                                                                        <div class="flex flex-col justify-center -space-y-0.5">
                                                                                                <div class=" dark:text-gray-100 text-sm font-medium line-clamp-1">
                                                                                                        {file.name}
                                                                                                </div>

                                                                                                <div class=" text-gray-500 text-sm">{'Dokument'}</div>
                                                                                        </div>
                                                                                </div>
                                                                        {:else if file.type === 'collection'}
                                                                                <div
                                                                                        class="h-16 w-[15rem] flex items-center space-x-3 px-2.5 dark:bg-gray-600 rounded-xl border border-gray-200 dark:border-none"
                                                                                >
                                                                                        <div class="p-2.5 bg-red-400 text-white rounded-lg">
                                                                                                <svg
                                                                                                        xmlns="http://www.w3.org/2000/svg"
                                                                                                        viewBox="0 0 24 24"
                                                                                                        fill="currentColor"
                                                                                                        class="w-6 h-6"
                                                                                                >
                                                                                                        <path
                                                                                                                d="M7.5 3.375c0-1.036.84-1.875 1.875-1.875h.375a3.75 3.75 0 0 1 3.75 3.75v1.875C13.5 8.161 14.34 9 15.375 9h1.875A3.75 3.75 0 0 1 21 12.75v3.375C21 17.16 20.16 18 19.125 18h-9.75A1.875 1.875 0 0 1 7.5 16.125V3.375Z"
                                                                                                        />
                                                                                                        <path
                                                                                                                d="M15 5.25a5.23 5.23 0 0 0-1.279-3.434 9.768 9.768 0 0 1 6.963 6.963A5.23 5.23 0 0 0 17.25 7.5h-1.875A.375.375 0 0 1 15 7.125V5.25ZM4.875 6H6v10.125A3.375 3.375 0 0 0 9.375 19.5H16.5v1.125c0 1.035-.84 1.875-1.875 1.875h-9.75A1.875 1.875 0 0 1 3 20.625V7.875C3 6.839 3.84 6 4.875 6Z"
                                                                                                        />
                                                                                                </svg>
                                                                                        </div>

                                                                                        <div class="flex flex-col justify-center -space-y-0.5">
                                                                                                <div class=" dark:text-gray-100 text-sm font-medium line-clamp-1">
                                                                                                        {file?.title ?? `#${file.name}`}
                                                                                                </div>

                                                                                                <div class=" text-gray-500 text-sm">{'Kolekcja'}</div>
                                                                                        </div>
                                                                                </div>
                                                                        {/if}

                                                                        <div class=" absolute -top-1 -right-1">
                                                                                <button
                                                                                        class=" bg-gray-400 text-white border border-white rounded-full group-hover:visible invisible transition"
                                                                                        type="button"
                                                                                        on:click={() => {
                                                                                                files.splice(fileIdx, 1);
                                                                                                files = files;
                                                                                        }}
                                                                                >
                                                                                        <svg
                                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                                viewBox="0 0 20 20"
                                                                                                fill="currentColor"
                                                                                                class="w-4 h-4"
                                                                                        >
                                                                                                <path
                                                                                                        d="M6.28 5.22a.75.75 0 00-1.06 1.06L8.94 10l-3.72 3.72a.75.75 0 101.06 1.06L10 11.06l3.72 3.72a.75.75 0 101.06-1.06L11.06 10l3.72-3.72a.75.75 0 00-1.06-1.06L10 8.94 6.28 5.22z"
                                                                                                />
                                                                                        </svg>
                                                                                </button>
                                                                        </div>
                                                                </div>
                                                                {/each}
                                                        </div>
                                                {/if}

                                                <div class="flex">
                                                        <div class="flex-1 flex flex-col relative w-full px-1">
                                                                <Documents
                                                                        bind:this={documentsElement}
                                                                        on:select={(e) => {
                                                                                uploadDoc(e.detail);
                                                                                chatTextAreaElement.focus();
                                                                        }}
                                                                />

                                                                <Models
                                                                        bind:this={modelsElement}
                                                                        on:select={(e) => {
                                                                                prompt = `${prompt}${e.detail} `;
                                                                                chatTextAreaElement.focus();
                                                                        }}
                                                                />

                                                                <textarea
                                                                        bind:this={chatTextAreaElement}
                                                                        class="scrollbar-hidden bg-transparent dark:text-gray-100 outline-none w-full py-1.5 px-2.5 resize-none"
                                                                        placeholder={chatInputPlaceholder !== ''
                                                                                ? chatInputPlaceholder
                                                                                : 'Zapytaj o cokolwiek...'}
                                                                        rows="1"
                                                                        bind:value={prompt}
                                                                        on:keypress={(e) => {
                                                                                if (e.keyCode === 13 && !e.shiftKey) {
                                                                                        e.preventDefault();
                                                                                }
                                                                                if (prompt !== '' && e.keyCode === 13 && !e.shiftKey) {
                                                                                        submitPrompt(prompt, user);
                                                                                        prompt = '';
                                                                                }
                                                                        }}
                                                                        on:keydown={(e) => {
                                                                                if (e.key === 'ArrowUp' || e.key === 'ArrowDown') {
                                                                                        if (documentsElement.getVisible()) {
                                                                                                e.preventDefault();
                                                                                                documentsElement.handleKeyDown(e);
                                                                                        } else if (modelsElement.getVisible()) {
                                                                                                e.preventDefault();
                                                                                                modelsElement.handleKeyDown(e);
                                                                                        }
                                                                                }

                                                                                if (e.key === 'Tab') {
                                                                                        if (documentsElement.getVisible()) {
                                                                                                e.preventDefault();
                                                                                                documentsElement.handleKeyDown(e);
                                                                                        } else if (modelsElement.getVisible()) {
                                                                                                e.preventDefault();
                                                                                                modelsElement.handleKeyDown(e);
                                                                                        }
                                                                                }

                                                                                if (e.key === 'Escape') {
                                                                                        documentsElement.setVisible(false);
                                                                                        modelsElement.setVisible(false);
                                                                                }
                                                                        }}
                                                                        on:input={() => {
                                                                                const words = findWordIndices(prompt);
                                                                                const word = words.find((word) => word.end === chatTextAreaElement.selectionStart);

                                                                                if (word) {
                                                                                        const query = word.word;

                                                                                        if (query.startsWith('#')) {
                                                                                                documentsElement.setQuery(query.slice(1));
                                                                                                documentsElement.setVisible(true);
                                                                                        } else {
                                                                                                documentsElement.setVisible(false);
                                                                                        }

                                                                                        if (query.startsWith('@')) {
                                                                                                modelsElement.setQuery(query.slice(1));
                                                                                                modelsElement.setVisible(true);
                                                                                        } else {
                                                                                                modelsElement.setVisible(false);
                                                                                        }
                                                                                }
                                                                        }}
                                                                />
                                                        </div>

                                                        <div class="flex flex-col justify-end mb-0.5">
                                                                {#if prompt === ''}
                                                                        <div class="flex items-center space-x-1 px-1">
                                                                                <button
                                                                                        class=" text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200 transition p-2"
                                                                                        type="button"
                                                                                        on:click={() => {
                                                                                                drawerOpen = true;
                                                                                        }}
                                                                                >
                                                                                        <svg
                                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                                viewBox="0 0 20 20"
                                                                                                fill="currentColor"
                                                                                                class="w-5 h-5"
                                                                                        >
                                                                                                <path
                                                                                                        fill-rule="evenodd"
                                                                                                        d="M15.621 4.379a3 3 0 00-4.242 0l-7 7a3 3 0 004.242 4.242l7-7a3 3 0 00-4.242-4.242l-5.25 5.25a1 1 0 001.414 1.414l5.25-5.25a1 1 0 011.414 1.414l-7 7a1 1 0 01-1.414-1.414l7-7a1 1 0 011.414 0z"
                                                                                                        clip-rule="evenodd"
                                                                                                />
                                                                                        </svg>
                                                                                </button>

                                                                                {#if speechRecognitionEnabled}
                                                                                        <button
                                                                                                class=" text-gray-500 hover:text-gray-700 dark:text-gray-400 dark:hover:text-gray-200 transition p-2"
                                                                                                type="button"
                                                                                                on:click={speechRecognitionHandler}
                                                                                        >
                                                                                                {#if isRecording}
                                                                                                        <svg
                                                                                                                class=" w-5 h-5 text-red-500"
                                                                                                                fill="currentColor"
                                                                                                                viewBox="0 0 24 24"
                                                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                                                ><style>
                                                                                                                        .spinner_S117 {
                                                                                                                                animation: spinner_H6oV 1s linear infinite;
                                                                                                                        }
                                                                                                                        @keyframes spinner_H6oV {
                                                                                                                                to {
                                                                                                                                        transform: rotate(360deg);
                                                                                                                                }
                                                                                                                        }
                                                                                                                </style><path
                                                                                                                        class="spinner_S117"
                                                                                                                        d="M12,1A11,11,0,1,0,23,12,11,11,0,0,0,12,1Zm0,19a8,8,0,1,1,8-8A8,8,0,0,1,12,20Z"
                                                                                                                        opacity=".25"
                                                                                                                /><path
                                                                                                                        class="spinner_S117"
                                                                                                                        d="M10.14,1.16a11,11,0,0,0-9,8.92A1.59,1.59,0,0,0,2.46,12,1.52,1.52,0,0,0,4.11,10.7a8,8,0,0,1,6.66-6.61A1.42,1.42,0,0,0,12,2.69h0A1.57,1.57,0,0,0,10.14,1.16Z"
                                                                                                                /></svg
                                                                                                        >
                                                                                                {:else}
                                                                                                        <svg
                                                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                                                viewBox="0 0 20 20"
                                                                                                                fill="currentColor"
                                                                                                                class="w-5 h-5"
                                                                                                        >
                                                                                                                <path
                                                                                                                        d="M7 4a3 3 0 016 0v6a3 3 0 11-6 0V4z"
                                                                                                                />
                                                                                                                <path
                                                                                                                        d="M5.5 9.643a.75.75 0 00-1.5 0V10c0 3.06 2.29 5.585 5.25 5.954V17.5h-1.5a.75.75 0 000 1.5h4.5a.75.75 0 000-1.5h-1.5v-1.546A6.001 6.001 0 0016 10v-.357a.75.75 0 00-1.5 0V10a4.5 4.5 0 01-9 0v-.357z"
                                                                                                                />
                                                                                                        </svg>
                                                                                                {/if}
                                                                                        </button>
                                                                                {/if}
                                                                        </div>
                                                                {:else}
                                                                        <div class="flex items-center px-1">
                                                                                <button
                                                                                        class=" bg-black text-white dark:bg-white dark:text-black transition p-2 rounded-2xl"
                                                                                        type="submit"
                                                                                >
                                                                                        <svg
                                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                                viewBox="0 0 20 20"
                                                                                                fill="currentColor"
                                                                                                class="w-5 h-5"
                                                                                        >
                                                                                                <path
                                                                                                        fill-rule="evenodd"
                                                                                                        d="M10 17a.75.75 0 01-.75-.75V5.612L5.29 9.77a.75.75 0 01-1.08-1.04l5.25-5.5a.75.75 0 011.08 0l5.25 5.5a.75.75 0 11-1.08 1.04l-3.96-4.158V16.25A.75.75 0 0110 17z"
                                                                                                        clip-rule="evenodd"
                                                                                                />
                                                                                        </svg>
                                                                                </button>
                                                                        </div>
                                                                {/if}
                                                        </div>
                                                </div>
                                        </div>
                                </form>

                                <div class="mt-2.5">
                                        <Suggestions {suggestionPrompts} {submitPrompt} />
                                </div>
                        </div>
                </div>
        </div>
</div>

<BottomDrawer bind:open={drawerOpen}>
        <div class="flex flex-col gap-2 p-3">
                <div class="flex gap-2">
                <button
                        class="flex-1 flex flex-col items-center justify-center gap-1 py-2 rounded-2xl bg-gray-100 dark:bg-[#2a2a2a] hover:bg-gray-200 dark:hover:bg-[#333] transition text-gray-800 dark:text-gray-100"
                        on:click={() => {
                                closeMenu();
                                filesInputElement.click();
                        }}
                >
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor"><path fill-rule="evenodd" d="M7.161 3h9.678c.527 0 .982 0 1.356.03.395.033.789.104 1.167.297a3 3 0 0 1 1.311 1.311c.193.378.264.772.296 1.167.031.375.031.83.031 1.356v9.678c0 .527 0 .982-.03 1.356-.033.395-.104.789-.297 1.167a3 3 0 0 1-1.311 1.311c-.378.193-.772.264-1.167.296-.375.031-.83.031-1.356.031H7.16c-.527 0-.981 0-1.356-.03-.395-.033-.789-.104-1.167-.297a3 3 0 0 1-1.311-1.311c-.193-.378-.264-.772-.296-1.167A17.9 17.9 0 0 1 3 16.838V7.162c0-.527 0-.981.03-1.356.033-.395.104-.789.297-1.167a3 3 0 0 1 1.311-1.311c.378-.193.772-.264 1.167-.296C6.18 3 6.635 3 7.161 3ZM5.968 5.024c-.272.022-.373.06-.422.085a1 1 0 0 0-.437.437c-.025.05-.063.15-.085.422C5 6.25 5 6.623 5 7.2v6.386l.879-.879a3 3 0 0 1 4.242 0L16.414 19h.386c.577 0 .949 0 1.232-.024.272-.022.372-.06.422-.085a1 1 0 0 0 .437-.437c.025-.05.063-.15.085-.422C19 17.75 19 17.377 19 16.8V7.2c0-.577 0-.949-.024-1.232-.022-.272-.06-.373-.085-.422a1 1 0 0 0-.437-.437c-.05-.025-.15-.063-.422-.085C17.75 5 17.377 5 16.8 5H7.2c-.577 0-.949 0-1.232.024ZM13.586 19l-4.879-4.879a1 1 0 0 0-1.414 0L5 16.414v.386c0 .577 0 .949.024 1.232.022.272.06.372.085.422a1 1 0 0 0 .437.437c.05.025.15.063.422.085C6.25 19 6.623 19 7.2 19h6.386ZM14.5 8.5a1 1 0 1 0 0 2 1 1 0 0 0 0-2Zm-3 1a3 3 0 1 1 6 0 3 3 0 0 1-6 0Z" clip-rule="evenodd" /></svg>




                        <span class="text-xs font-semibold">Zdjęcia</span>
                </button>

                <button
                        class="flex-1 flex flex-col items-center justify-center gap-1 py-2 rounded-2xl bg-gray-100 dark:bg-[#2a2a2a] hover:bg-gray-200 dark:hover:bg-[#333] transition text-gray-800 dark:text-gray-100"
                        on:click={() => {
                                closeMenu();
                                filesInputElement.click();
                        }}
                >
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor"><path fill-rule="evenodd" d="M9 7a5 5 0 0 1 10 0v8a7 7 0 1 1-14 0V9a1 1 0 0 1 2 0v6a5 5 0 0 0 10 0V7a3 3 0 1 0-6 0v8a1 1 0 1 0 2 0V9a1 1 0 1 1 2 0v6a3 3 0 1 1-6 0V7Z" clip-rule="evenodd" /></svg>


                        <span class="text-xs font-semibold">Pliki</span>
                </button>
        </div>



        <button
        	class="flex items-center gap-3 w-full px-1 py-2.5 rounded-xl hover:bg-gray-100 dark:hover:bg-[#2a2a2a] transition text-gray-900 dark:text-gray-100"
        	on:click={() => {
        		setSingleMode($searchEnabled ? null : 'search');
        		closeMenu();
        	}}
        >
        	<svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor" class="shrink-0 text-gray-700 dark:text-gray-300"><path fill-rule="evenodd" clip-rule="evenodd" d="M4.06189 11H7.52048C7.61065 8.81122 7.99451 6.81287 8.59312 5.27359C8.67816 5.05493 8.76873 4.84242 8.86496 4.63763C6.29788 5.73214 4.41978 8.13001 4.06189 11ZM12 2C6.47715 2 2 6.47715 2 12C2 17.5228 6.47715 22 12 22C17.5228 22 22 17.5228 22 12C22 6.47715 17.5228 2 12 2ZM12 4C11.8911 4 11.6784 4.0528 11.3738 4.3841C11.0684 4.71624 10.7474 5.25207 10.4571 5.99849C9.96096 7.27434 9.61141 9.01803 9.52232 11H14.4777C14.3886 9.01803 14.039 7.27434 13.5429 5.99849C13.2526 5.25207 12.9316 4.71624 12.6262 4.3841C12.3216 4.0528 12.1089 4 12 4ZM16.4795 11C16.3893 8.81122 16.0055 6.81287 15.4069 5.27359C15.3218 5.05493 15.2313 4.84242 15.135 4.63763C17.7021 5.73214 19.5802 8.13001 19.9381 11H16.4795ZM14.4777 13H9.52232C9.61141 14.982 9.96096 16.7257 10.4571 18.0015C10.7474 18.7479 11.0684 19.2838 11.3738 19.6159C11.6784 19.9472 11.8911 20 12 20C12.1089 20 12.3216 19.9472 12.6262 19.6159C12.9316 19.2838 13.2526 18.7479 13.5429 18.0015C14.039 16.7257 14.3886 14.982 14.4777 13ZM15.135 19.3624C15.2313 19.1576 15.3218 18.9451 15.4069 18.7264C16.0055 17.1871 16.3893 15.1888 16.4795 13H19.9381C19.5802 15.87 17.7021 18.2679 15.135 19.3624ZM8.86496 19.3624C8.76873 19.1576 8.67816 18.9451 8.59312 18.7264C7.99451 17.1871 7.61065 15.1888 7.52048 13H4.06189C4.41978 15.87 6.29788 18.2679 8.86496 19.3624Z" fill="currentColor" /></svg>
        	<div class="flex flex-col text-left flex-1 min-w-0">
        		<span class="text-sm font-semibold leading-tight">Szukaj w internecie</span>
        		<span class="text-xs text-gray-500 dark:text-gray-400 truncate">Brave Search — aktualne wyniki</span>
        	</div>
        	<div class="w-5 h-5 rounded-full border-2 flex items-center justify-center shrink-0 {$searchEnabled ? 'bg-gray-800 border-gray-800 dark:bg-white dark:border-white' : 'border-gray-300 dark:border-gray-600'}">
        		{#if $searchEnabled}
        			<svg width="10" height="8" viewBox="0 0 10 8" fill="none"><path d="M1 4l3 3 5-6" stroke="white" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
        		{/if}
        	</div>
        </button>

        <button
                class="flex items-center gap-3 w-full px-1 py-2.5 rounded-xl hover:bg-gray-100 dark:hover:bg-[#2a2a2a] transition text-gray-900 dark:text-gray-100"
                        on:click={() => {
                                setSingleMode($hfEnabled ? null : 'hf');
                                closeMenu();
                        }}
        >
                <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor" class="shrink-0 text-gray-700 dark:text-gray-300"><path fill-rule="evenodd" d="M12.5 3.444a1 1 0 0 0-1 0l-6.253 3.61 6.768 3.807 6.955-3.682-6.47-3.735Zm7.16 5.632L13 12.602v7.666l6.16-3.556a1 1 0 0 0 .5-.867V9.076ZM11 20.268v-7.683L4.34 8.839v7.006a1 1 0 0 0 .5.867L11 20.268Zm-.5-18.557a3 3 0 0 1 3 0l6.66 3.846a3 3 0 0 1 1.5 2.598v7.69a3 3 0 0 1-1.5 2.598L13.5 22.29a3 3 0 0 1-3 0l-6.66-3.846a3 3 0 0 1-1.5-2.598v-7.69a3 3 0 0 1 1.5-2.598L10.5 1.71Z" clip-rule="evenodd" /></svg>




                <div class="flex flex-col text-left flex-1 min-w-0">
                        <span class="text-sm font-semibold leading-tight">HuggingFace Hub</span>
                        <span class="text-xs text-gray-500 dark:text-gray-400 truncate">Modele, datasety, spaces</span>
                </div>
                <div class="w-5 h-5 rounded-full border-2 flex items-center justify-center shrink-0 {$hfEnabled ? 'bg-gray-800 border-gray-800 dark:bg-white dark:border-white' : 'border-gray-300 dark:border-gray-600'}">
                        {#if $hfEnabled}
                                <svg width="10" height="8" viewBox="0 0 10 8" fill="none"><path d="M1 4l3 3 5-6" stroke="white" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
                        {/if}
                </div>
        </button>

        <button
                class="flex items-center gap-3 w-full px-1 py-2.5 rounded-xl hover:bg-gray-100 dark:hover:bg-[#2a2a2a] transition text-gray-900 dark:text-gray-100"
                on:click={() => {
                        setSingleMode($githubEnabled ? null : 'github');
                        closeMenu();
                }}
        >
                <svg viewBox="0 0 24 24" fill="currentColor" class="w-5 h-5 shrink-0 text-gray-700 dark:text-gray-300">
                        <path d="M12 2C6.477 2 2 6.484 2 12.021c0 4.428 2.865 8.185 6.839 9.504.5.092.682-.217.682-.482 0-.237-.009-.868-.014-1.703-2.782.605-3.369-1.342-3.369-1.342-.454-1.155-1.11-1.463-1.11-1.463-.908-.62.069-.608.069-.608 1.003.07 1.531 1.032 1.531 1.032.892 1.53 2.341 1.088 2.91.832.092-.647.35-1.088.636-1.338-2.22-.253-4.555-1.113-4.555-4.951 0-1.093.39-1.988 1.029-2.688-.103-.253-.446-1.272.098-2.65 0 0 .84-.27 2.75 1.026A9.564 9.564 0 0112 6.844c.85.004 1.705.115 2.504.337 1.909-1.296 2.747-1.027 2.747-1.027.546 1.379.202 2.398.1 2.651.64.7 1.028 1.595 1.028 2.688 0 3.848-2.339 4.695-4.566 4.943.359.309.678.92.678 1.855 0 1.338-.012 2.419-.012 2.747 0 .268.18.58.688.482A10.019 10.019 0 0022 12.021C22 6.484 17.522 2 12 2z"/>
                </svg>
                <div class="flex flex-col text-left flex-1 min-w-0">
                        <span class="text-sm font-semibold leading-tight">GitHub Agent</span>
                        <span class="text-xs text-gray-500 dark:text-gray-400 truncate">Repozytoria, pliki, commity, gałęzie</span>
                </div>
                <div class="w-5 h-5 rounded-full border-2 flex items-center justify-center shrink-0 {$githubEnabled ? 'bg-gray-800 border-gray-800 dark:bg-white dark:border-white' : 'border-gray-300 dark:border-gray-600'}">
                        {#if $githubEnabled}
                                <svg width="10" height="8" viewBox="0 0 10 8" fill="none"><path d="M1 4l3 3 5-6" stroke="white" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
                        {/if}
                </div>
        </button>

        <button
                class="flex items-center gap-3 w-full px-1 py-2.5 rounded-xl hover:bg-gray-100 dark:hover:bg-[#2a2a2a] transition text-gray-900 dark:text-gray-100"
                on:click={async () => {
                        const enabling = !$kernelEnabled;
                        setSingleMode(enabling ? 'kernel' : null);
                        closeMenu();
                        if (enabling && !$kernelBrowserUrl) {
                                try {
                                        const res = await fetch('/api/kernel/session', { method: 'POST' });
                                        const data = await res.json();
                                        kernelBrowserUrl.set(data.live_view_url);
                                        kernelSessionId.set(data.session_id);
                                } catch (e) {
                                        console.error('Kernel session error', e);
                                }
                        }
                }}
        >
                <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor" class="shrink-0 text-gray-700 dark:text-gray-300"><path fill-rule="evenodd" clip-rule="evenodd" d="M11.4053 13.3184C10.986 12.1299 12.1299 10.986 13.3184 11.4053L21.1787 14.1797C22.4453 14.6267 22.5315 16.385 21.3145 16.9531L18.3408 18.3408L16.9531 21.3145C16.385 22.5315 14.6267 22.4453 14.1797 21.1787L11.4053 13.3184ZM15.6572 19.3594L16.6055 17.3301L16.666 17.2139C16.8193 16.9487 17.0506 16.7359 17.3301 16.6055L19.3594 15.6572L13.6387 13.6387L15.6572 19.3594Z" fill="currentColor" /><path d="M17 3C19.2091 3 21 4.79086 21 7V9C21 9.55228 20.5523 10 20 10C19.4477 10 19 9.55228 19 9V7C19 5.89543 18.1046 5 17 5H7C5.89543 5 5 5.89543 5 7V17C5 18.1046 5.89543 19 7 19H9C9.55228 19 10 19.4477 10 20C10 20.5523 9.55228 21 9 21H7C4.79086 21 3 19.2091 3 17V7C3 4.79086 4.79086 3 7 3H17Z" fill="currentColor" /><path d="M8.50195 12.2051C9.00644 12.2561 9.40039 12.6823 9.40039 13.2002C9.40027 13.718 9.00637 14.1443 8.50195 14.1953L8.40039 14.2002H7.2002C6.64799 14.2002 6.20033 13.7524 6.2002 13.2002C6.2002 12.6479 6.64791 12.2002 7.2002 12.2002H8.40039L8.50195 12.2051Z" fill="currentColor" /><path d="M7.99316 7.99316C8.38369 7.60264 9.0167 7.60264 9.40723 7.99316L10.3076 8.89355L10.376 8.96875C10.6963 9.36152 10.6737 9.9415 10.3076 10.3076C9.94148 10.6736 9.36148 10.6963 8.96875 10.376L8.89355 10.3076L7.99316 9.40723C7.60268 9.01674 7.60276 8.3837 7.99316 7.99316Z" fill="currentColor" /><path d="M13.2002 6.2002C13.7525 6.2002 14.2002 6.64791 14.2002 7.2002V8.40039C14.2 8.95247 13.7523 9.40039 13.2002 9.40039C12.6481 9.40039 12.2004 8.95247 12.2002 8.40039V7.2002C12.2002 6.64791 12.6479 6.2002 13.2002 6.2002Z" fill="currentColor" /></svg>




                <div class="flex flex-col text-left flex-1 min-w-0">
                        <span class="text-sm font-semibold leading-tight">Browser Agent</span>
                        <span class="text-xs text-gray-500 dark:text-gray-400 truncate">Autonomiczna przeglądarka</span>
                </div>
                <div class="w-5 h-5 rounded-full border-2 flex items-center justify-center shrink-0 {$kernelEnabled ? 'bg-gray-800 border-gray-800 dark:bg-white dark:border-white' : 'border-gray-300 dark:border-gray-600'}">
                        {#if $kernelEnabled}
                                <svg width="10" height="8" viewBox="0 0 10 8" fill="none"><path d="M1 4l3 3 5-6" stroke="white" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
                        {/if}
                </div>
        </button>

        <button
                class="flex items-center gap-3 w-full px-1 py-2.5 rounded-xl hover:bg-gray-100 dark:hover:bg-[#2a2a2a] transition text-gray-900 dark:text-gray-100 disabled:opacity-50"
                disabled={$desktopLoading}
                on:click={async () => {
                        if ($desktopEnabled) {
                                setSingleMode(null);
                                desktopLoading.set(false);
                                desktopBrowserUrl.set('');
                                desktopSessionId.set('');
                                closeMenu();
                        } else {
                                desktopLoading.set(true);
                                setSingleMode('desktop');
                                closeMenu();
                                try {
                                        const res = await fetch('/api/e2b/session', { method: 'POST' });
                                        const data = await res.json();
                                        desktopBrowserUrl.set(data.live_view_url);
                                        desktopSessionId.set(data.session_id);
                                        desktopEnabled.set(true);
                                } catch (e) {
                                        console.error('Desktop session error', e);
                                } finally {
                                        desktopLoading.set(false);
                                }
                        }
                }}
        >
                {#if $desktopLoading}
                        <svg class="shrink-0 animate-spin text-gray-700 dark:text-gray-300" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="10" stroke-opacity="0.25"/><path d="M12 2a10 10 0 0 1 10 10" stroke-linecap="round"/></svg>
                {:else}
                        <svg width="20" height="20" viewBox="0 0 24 24" fill="currentColor" class="shrink-0 text-gray-700 dark:text-gray-300"><path d="M6.16146 3L17.8385 3C18.3657 2.99998 18.8205 2.99997 19.195 3.03057C19.5904 3.06287 19.9836 3.13419 20.362 3.32698C20.9265 3.6146 21.3854 4.07354 21.673 4.63803C21.8658 5.01641 21.9371 5.40963 21.9694 5.80497C22 6.17954 22 6.6343 22 7.16144V13.3386C22 13.8657 22 14.3205 21.9694 14.695C21.9371 15.0904 21.8658 15.4836 21.673 15.862C21.3854 16.4265 20.9265 16.8854 20.362 17.173C19.9836 17.3658 19.5904 17.4371 19.195 17.4694C18.8205 17.5 18.3657 17.5 17.8385 17.5H16.5V20C16.5 20.5523 16.5523 21 15.5 21H8.5C7.94772 21 7.5 20.5523 7.5 20V17.5H6.16148C5.63432 17.5 5.17955 17.5 4.80497 17.4694C4.40963 17.4371 4.01641 17.3658 3.63803 17.173C3.07354 16.8854 2.6146 16.4265 2.32698 15.862C2.13419 15.4836 2.06287 15.0904 2.03057 14.695C1.99997 14.3205 1.99998 13.8657 2 13.3385V7.16146C1.99998 6.63431 1.99997 6.17955 2.03057 5.80497C2.06287 5.40963 2.13419 5.01641 2.32698 4.63803C2.6146 4.07354 3.07354 3.6146 3.63803 3.32698C4.01641 3.13419 4.40963 3.06287 4.80497 3.03057C5.17955 2.99997 5.63431 2.99998 6.16146 3ZM16.5 15.5H17.8385C18.3657 15.5 18.721 15.5 18.9886 15.4782C19.2452 15.4573 19.3458 15.4243 19.4072 15.3931C19.5511 15.3198 19.6802 15.1906 19.7535 15.0468C19.7847 14.9854 19.8177 14.8848 19.8386 14.6282C19.8604 14.3606 19.8604 14.0053 19.8604 13.4781V7.02187C19.8604 6.4947 19.8604 6.13941 19.8386 5.87183C19.8177 5.61522 19.7847 5.5146 19.7535 5.45322C19.6802 5.30935 19.5511 5.18018 19.4072 5.10688C19.3458 5.07567 19.2452 5.04269 18.9886 5.02179C18.721 5 18.3657 5 17.8385 5L6.16146 5C5.6343 5 5.27898 5 5.01141 5.02179C4.7548 5.04269 4.65418 5.07567 4.5928 5.10688C4.44893 5.18018 4.31976 5.30935 4.24646 5.45322C4.21525 5.5146 4.18227 5.61522 4.16137 5.87183C4.13958 6.13941 4.13958 6.4947 4.13958 7.02187V13.4781C4.13958 14.0053 4.13958 14.3606 4.16137 14.6282C4.18227 14.8848 4.21525 14.9854 4.24646 15.0468C4.31976 15.1906 4.44893 15.3198 4.5928 15.3931C4.65418 15.4243 4.7548 15.4573 5.01141 15.4782C5.27898 15.5 5.6343 15.5 6.16146 15.5H7.5V13C7.5 12.4477 7.94772 12 8.5 12H15.5C16.0523 12 16.5 12.4477 16.5 13V15.5ZM15 15.5V13.5H9V15.5H15Z" fill-rule="evenodd" clip-rule="evenodd" /></svg>
                {/if}
                <div class="flex flex-col text-left flex-1 min-w-0">
                        <span class="text-sm font-semibold leading-tight">Desktop Agent</span>
                        <span class="text-xs text-gray-500 dark:text-gray-400 truncate">Zdalny pulpit — pełna moc</span>
                </div>
                <div class="w-5 h-5 rounded-full border-2 flex items-center justify-center shrink-0 {$desktopEnabled ? 'bg-gray-800 border-gray-800 dark:bg-white dark:border-white' : 'border-gray-300 dark:border-gray-600'}">
                        {#if $desktopEnabled}
                                <svg width="10" height="8" viewBox="0 0 10 8" fill="none"><path d="M1 4l3 3 5-6" stroke="white" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
                        {/if}
                </div>
        </button>
</div>
</BottomDrawer>
