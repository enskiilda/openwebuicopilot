<script lang="ts">
        import { toast } from 'svelte-sonner';
        import { v4 as uuidv4 } from 'uuid';

        import { goto } from '$app/navigation';
        import {
                user,
                chats,
                settings,
                showSettings,
                chatId,
                tags,
                folders as _folders,
                showSidebar,
                showSearch,
                mobile,
                showArchivedChats,
                pinnedChats,
                scrollPaginationEnabled,
                currentChatPage,
                temporaryChatEnabled,
                channels,
                socket,
                config,
                isApp,
                models,
                selectedFolder,
                WEBUI_NAME
        } from '$lib/stores';
        import { onMount, getContext, tick, onDestroy } from 'svelte';

        const i18n = getContext('i18n');

        import {
                getChatList,
                getAllTags,
                getPinnedChatList,
                toggleChatPinnedStatusById,
                getChatById,
                updateChatFolderIdById,
                importChats
        } from '$lib/apis/chats';
        import { createNewFolder, getFolders, updateFolderParentIdById } from '$lib/apis/folders';
        import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';

        import ArchivedChatsModal from './ArchivedChatsModal.svelte';
        import UserMenu from './Sidebar/UserMenu.svelte';
        import ChatItem from './Sidebar/ChatItem.svelte';
        import Spinner from '../common/Spinner.svelte';
        import Loader from '../common/Loader.svelte';
        import Folder from '../common/Folder.svelte';
        import Tooltip from '../common/Tooltip.svelte';
        import Folders from './Sidebar/Folders.svelte';
        import { getChannels, createNewChannel } from '$lib/apis/channels';
        import ChannelModal from './Sidebar/ChannelModal.svelte';
        import ChannelItem from './Sidebar/ChannelItem.svelte';
        import Settings from '../icons/Settings.svelte';
        import SearchModal from './SearchModal.svelte';
        import FolderModal from './Sidebar/Folders/FolderModal.svelte';
        import Sidebar from '../icons/Sidebar.svelte';
        import PinnedModelList from './Sidebar/PinnedModelList.svelte';
        import Note from '../icons/Note.svelte';
        import { slide } from 'svelte/transition';
        import HotkeyHint from '../common/HotkeyHint.svelte';

        const BREAKPOINT = 768;

        let scrollTop = 0;

        let navElement;
        let shiftKey = false;

        let selectedChatId = null;
        let showCreateChannel = false;

        // Pagination variables
        let chatListLoading = false;
        let allChatsLoaded = false;

        let showCreateFolderModal = false;

        let folders = {};
        let folderRegistry = {};

        let newFolderId = null;

        $: if ($selectedFolder) {
                initFolders();
        }

        const initFolders = async () => {
                const folderList = await getFolders(localStorage.token).catch((error) => {
                        toast.error(`${error}`);
                        return [];
                });
                _folders.set(folderList.sort((a, b) => b.updated_at - a.updated_at));

                folders = {};

                // First pass: Initialize all folder entries
                for (const folder of folderList) {
                        // Ensure folder is added to folders with its data
                        folders[folder.id] = { ...(folders[folder.id] || {}), ...folder };

                        if (newFolderId && folder.id === newFolderId) {
                                folders[folder.id].new = true;
                                newFolderId = null;
                        }
                }

                // Second pass: Tie child folders to their parents
                for (const folder of folderList) {
                        if (folder.parent_id) {
                                // Ensure the parent folder is initialized if it doesn't exist
                                if (!folders[folder.parent_id]) {
                                        folders[folder.parent_id] = {}; // Create a placeholder if not already present
                                }

                                // Initialize childrenIds array if it doesn't exist and add the current folder id
                                folders[folder.parent_id].childrenIds = folders[folder.parent_id].childrenIds
                                        ? [...folders[folder.parent_id].childrenIds, folder.id]
                                        : [folder.id];

                                // Sort the children by updated_at field
                                folders[folder.parent_id].childrenIds.sort((a, b) => {
                                        return folders[b].updated_at - folders[a].updated_at;
                                });
                        }
                }
        };

        const createFolder = async ({ name, data }) => {
                name = name?.trim();
                if (!name) {
                        toast.error($i18n.t('Folder name cannot be empty.'));
                        return;
                }

                const rootFolders = Object.values(folders).filter((folder) => folder.parent_id === null);
                if (rootFolders.find((folder) => folder.name.toLowerCase() === name.toLowerCase())) {
                        // If a folder with the same name already exists, append a number to the name
                        let i = 1;
                        while (
                                rootFolders.find((folder) => folder.name.toLowerCase() === `${name} ${i}`.toLowerCase())
                        ) {
                                i++;
                        }

                        name = `${name} ${i}`;
                }

                // Add a dummy folder to the list to show the user that the folder is being created
                const tempId = uuidv4();
                folders = {
                        ...folders,
                        tempId: {
                                id: tempId,
                                name: name,
                                created_at: Date.now(),
                                updated_at: Date.now()
                        }
                };

                const res = await createNewFolder(localStorage.token, {
                        name,
                        data
                }).catch((error) => {
                        toast.error(`${error}`);
                        return null;
                });

                if (res) {
                        // newFolderId = res.id;
                        await initFolders();
                }
        };

        const initChannels = async () => {
                await channels.set(await getChannels(localStorage.token));
        };

        const initChatList = async () => {
                // Reset pagination variables
                console.log('initChatList');
                currentChatPage.set(1);
                allChatsLoaded = false;
                scrollPaginationEnabled.set(false);

                initFolders();
                await Promise.all([
                        await (async () => {
                                console.log('Init tags');
                                const _tags = await getAllTags(localStorage.token);
                                tags.set(_tags);
                        })(),
                        await (async () => {
                                console.log('Init pinned chats');
                                const _pinnedChats = await getPinnedChatList(localStorage.token);
                                pinnedChats.set(_pinnedChats);
                        })(),
                        await (async () => {
                                console.log('Init chat list');
                                const _chats = await getChatList(localStorage.token, $currentChatPage);
                                await chats.set(_chats);
                        })()
                ]);

                // Enable pagination
                scrollPaginationEnabled.set(true);
        };

        const loadMoreChats = async () => {
                chatListLoading = true;

                currentChatPage.set($currentChatPage + 1);

                let newChatList = [];

                newChatList = await getChatList(localStorage.token, $currentChatPage);

                // once the bottom of the list has been reached (no results) there is no need to continue querying
                allChatsLoaded = newChatList.length === 0;
                await chats.set([...($chats ? $chats : []), ...newChatList]);

                chatListLoading = false;
        };

        const importChatHandler = async (items, pinned = false, folderId = null) => {
                console.log('importChatHandler', items, pinned, folderId);
                for (const item of items) {
                        console.log(item);
                        if (item.chat) {
                                await importChats(localStorage.token, [
                                        {
                                                chat: item.chat,
                                                meta: item?.meta ?? {},
                                                pinned: pinned,
                                                folder_id: folderId,
                                                created_at: item?.created_at ?? null,
                                                updated_at: item?.updated_at ?? null
                                        }
                                ]);
                        }
                }

                initChatList();
        };

        const inputFilesHandler = async (files) => {
                console.log(files);

                for (const file of files) {
                        const reader = new FileReader();
                        reader.onload = async (e) => {
                                const content = e.target.result;

                                try {
                                        const chatItems = JSON.parse(content);
                                        importChatHandler(chatItems);
                                } catch {
                                        toast.error($i18n.t(`Invalid file format.`));
                                }
                        };

                        reader.readAsText(file);
                }
        };

        const tagEventHandler = async (type, tagName, chatId) => {
                console.log(type, tagName, chatId);
                if (type === 'delete') {
                        initChatList();
                } else if (type === 'add') {
                        initChatList();
                }
        };

        let draggedOver = false;

        const onDragOver = (e) => {
                e.preventDefault();

                // Check if a file is being draggedOver.
                if (e.dataTransfer?.types?.includes('Files')) {
                        draggedOver = true;
                } else {
                        draggedOver = false;
                }
        };

        const onDragLeave = () => {
                draggedOver = false;
        };

        const onDrop = async (e) => {
                e.preventDefault();
                console.log(e); // Log the drop event

                // Perform file drop check and handle it accordingly
                if (e.dataTransfer?.files) {
                        const inputFiles = Array.from(e.dataTransfer?.files);

                        if (inputFiles && inputFiles.length > 0) {
                                console.log(inputFiles); // Log the dropped files
                                inputFilesHandler(inputFiles); // Handle the dropped files
                        }
                }

                draggedOver = false; // Reset draggedOver status after drop
        };

        let touchstart;
        let touchend;

        function checkDirection() {
                const screenWidth = window.innerWidth;
                const swipeDistance = Math.abs(touchend.screenX - touchstart.screenX);
                if (touchstart.clientX < 40 && swipeDistance >= screenWidth / 8) {
                        if (touchend.screenX < touchstart.screenX) {
                                showSidebar.set(false);
                        }
                        if (touchend.screenX > touchstart.screenX) {
                                showSidebar.set(true);
                        }
                }
        }

        const onTouchStart = (e) => {
                touchstart = e.changedTouches[0];
                console.log(touchstart.clientX);
        };

        const onTouchEnd = (e) => {
                touchend = e.changedTouches[0];
                checkDirection();
        };

        const onKeyDown = (e) => {
                if (e.key === 'Shift') {
                        shiftKey = true;
                }
        };

        const onKeyUp = (e) => {
                if (e.key === 'Shift') {
                        shiftKey = false;
                }
        };

        const onFocus = () => {};

        const onBlur = () => {
                shiftKey = false;
                selectedChatId = null;
        };

        let unsubscribers = [];
        onMount(async () => {
                await showSidebar.set(!$mobile ? localStorage.sidebar === 'true' : false);

                unsubscribers = [
                        mobile.subscribe((value) => {
                                if ($showSidebar && value) {
                                        showSidebar.set(false);
                                }

                                if ($showSidebar && !value) {
                                        const navElement = document.getElementsByTagName('nav')[0];
                                        if (navElement) {
                                                navElement.style['-webkit-app-region'] = 'drag';
                                        }
                                }
                        }),
                        showSidebar.subscribe(async (value) => {
                                localStorage.sidebar = value;

                                // nav element is not available on the first render
                                const navElement = document.getElementsByTagName('nav')[0];

                                if (navElement) {
                                        if ($mobile) {
                                                if (!value) {
                                                        navElement.style['-webkit-app-region'] = 'drag';
                                                } else {
                                                        navElement.style['-webkit-app-region'] = 'no-drag';
                                                }
                                        } else {
                                                navElement.style['-webkit-app-region'] = 'drag';
                                        }
                                }

                                if (value) {
                                        await initChannels();
                                        await initChatList();
                                }
                        })
                ];

                window.addEventListener('keydown', onKeyDown);
                window.addEventListener('keyup', onKeyUp);

                window.addEventListener('touchstart', onTouchStart);
                window.addEventListener('touchend', onTouchEnd);

                window.addEventListener('focus', onFocus);
                window.addEventListener('blur', onBlur);

                const dropZone = document.getElementById('sidebar');

                dropZone?.addEventListener('dragover', onDragOver);
                dropZone?.addEventListener('drop', onDrop);
                dropZone?.addEventListener('dragleave', onDragLeave);
        });

        onDestroy(() => {
                if (unsubscribers && unsubscribers.length > 0) {
                        unsubscribers.forEach((unsubscriber) => {
                                if (unsubscriber) {
                                        unsubscriber();
                                }
                        });
                }

                window.removeEventListener('keydown', onKeyDown);
                window.removeEventListener('keyup', onKeyUp);

                window.removeEventListener('touchstart', onTouchStart);
                window.removeEventListener('touchend', onTouchEnd);

                window.removeEventListener('focus', onFocus);
                window.removeEventListener('blur', onBlur);

                const dropZone = document.getElementById('sidebar');

                dropZone?.removeEventListener('dragover', onDragOver);
                dropZone?.removeEventListener('drop', onDrop);
                dropZone?.removeEventListener('dragleave', onDragLeave);
        });

        const newChatHandler = async () => {
                selectedChatId = null;
                selectedFolder.set(null);

                if ($user?.role !== 'admin' && $user?.permissions?.chat?.temporary_enforced) {
                        await temporaryChatEnabled.set(true);
                } else {
                        await temporaryChatEnabled.set(false);
                }

                setTimeout(() => {
                        if ($mobile) {
                                showSidebar.set(false);
                        }
                }, 0);
        };

        const itemClickHandler = async () => {
                selectedChatId = null;
                chatId.set('');

                if ($mobile) {
                        showSidebar.set(false);
                }

                await tick();
        };

        const isWindows = /Windows/i.test(navigator.userAgent);
</script>

<ArchivedChatsModal
        bind:show={$showArchivedChats}
        onUpdate={async () => {
                await initChatList();
        }}
/>

<ChannelModal
        bind:show={showCreateChannel}
        onSubmit={async ({ name, access_control }) => {
                name = name?.trim();
                if (!name) {
                        toast.error($i18n.t('Channel name cannot be empty.'));
                        return;
                }

                const res = await createNewChannel(localStorage.token, {
                        name: name,
                        access_control: access_control
                }).catch((error) => {
                        toast.error(`${error}`);
                        return null;
                });

                if (res) {
                        $socket.emit('join-channels', { auth: { token: $user?.token } });
                        await initChannels();
                        showCreateChannel = false;
                }
        }}
/>

<FolderModal
        bind:show={showCreateFolderModal}
        onSubmit={async (folder) => {
                await createFolder(folder);
                showCreateFolderModal = false;
        }}
/>

<!-- svelte-ignore a11y-no-static-element-interactions -->

{#if $showSidebar}
        <div
                class=" {$isApp
                        ? ' ml-[4.5rem] md:ml-0'
                        : ''} fixed md:hidden z-40 top-0 right-0 left-0 bottom-0 bg-black/60 w-full min-h-screen h-screen flex justify-center overflow-hidden overscroll-contain"
                on:mousedown={() => {
                        showSidebar.set(!$showSidebar);
                }}
        />
{/if}

<SearchModal
        bind:show={$showSearch}
        onClose={() => {
                if ($mobile) {
                        showSidebar.set(false);
                }
        }}
/>

<button
        id="sidebar-new-chat-button"
        class="hidden"
        on:click={() => {
                goto('/');
                newChatHandler();
        }}
/>

{#if !$mobile}
        <div
                class="h-full"
                id="sidebar-container"
        >
                {#if !$showSidebar}
                <div class="pt-2 pl-2">
                        <Tooltip
                                content={$i18n.t('Open Sidebar')}
                                placement="right"
                        >
                                <button
                                        class="flex rounded-xl size-9 justify-center items-center hover:bg-gray-100 dark:hover:bg-gray-850 transition cursor-pointer"
                                        on:click={() => {
                                                showSidebar.set(true);
                                        }}
                                        aria-label={$i18n.t('Open Sidebar')}
                                >
                                        <div class="self-center">
                                                <Sidebar />
                                        </div>
                                </button>
                        </Tooltip>
                </div>
                {/if}
                {#if $showSidebar}
                <div
                        bind:this={navElement}
                        id="sidebar-content"
                        class="h-screen max-h-[100dvh] min-h-screen select-none w-[260px] bg-gray-50/70 dark:bg-gray-950/70 shrink-0 text-gray-900 dark:text-gray-200 text-sm overflow-x-hidden transition-all duration-300"
                        transition:slide={{ duration: 250, axis: 'x' }}
                        data-state={$showSidebar}
                >
                        <div
                                class="my-auto flex flex-col justify-between h-screen max-h-[100dvh] w-[260px] overflow-x-hidden scrollbar-hidden z-50"
                        >
                                <div
                                        class="sidebar px-2 pt-2 pb-1.5 flex justify-between space-x-1 text-gray-600 dark:text-gray-400 sticky top-0 z-10 -mb-3"
                                >
                                        <a href="/" class="flex flex-1 px-1.5" on:click={newChatHandler}>
                                                <div
                                                        id="sidebar-webui-name"
                                                        class="self-center font-medium text-gray-850 dark:text-white font-primary"
                                                >
                                                        {$WEBUI_NAME}
                                                </div>
                                        </a>
                                        <Tooltip
                                                content={$i18n.t('Close Sidebar')}
                                                placement="bottom"
                                        >
                                                <button
                                                        class="flex rounded-xl size-8.5 justify-center items-center hover:bg-gray-100/50 dark:hover:bg-gray-850/50 transition {isWindows
                                                                ? 'cursor-pointer'
                                                                : 'cursor-[w-resize]'}"
                                                        on:click={() => {
                                                                showSidebar.set(false);
                                                        }}
                                                        aria-label={$i18n.t('Close Sidebar')}
                                                >
                                                        <div class="self-center p-1.5">
                                                                <Sidebar />
                                                        </div>
                                                </button>
                                        </Tooltip>

                                        <div
                                                class="{scrollTop > 0
                                                        ? 'visible'
                                                        : 'invisible'} sidebar-bg-gradient-to-b bg-linear-to-b from-gray-50 dark:from-gray-950 to-transparent from-50% pointer-events-none absolute inset-0 -z-10 -mb-6"
                                        ></div>
                                </div>

                                <div class="px-2 pt-2 flex space-x-1">
                                        <Tooltip content={$i18n.t('New Chat')} placement="bottom">
                                                <a
                                                        class="cursor-pointer flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition group"
                                                        href="/"
                                                        draggable="false"
                                                        on:click={async (e) => {
                                                                e.stopImmediatePropagation();
                                                                e.preventDefault();

                                                                goto('/');
                                                                newChatHandler();
                                                        }}
                                                        aria-label={$i18n.t('New Chat')}
                                                >
                                                        <div class="self-center flex items-center justify-center size-9">
                                                                <svg width="20" height="20" viewBox="0 0 20 20" fill="currentColor" xmlns="http://www.w3.org/2000/svg" class="icon" aria-hidden="true"><path d="M2.6687 11.333V8.66699C2.6687 7.74455 2.66841 7.01205 2.71655 6.42285C2.76533 5.82612 2.86699 5.31731 3.10425 4.85156L3.25854 4.57617C3.64272 3.94975 4.19392 3.43995 4.85229 3.10449L5.02905 3.02149C5.44666 2.84233 5.90133 2.75849 6.42358 2.71582C7.01272 2.66769 7.74445 2.66797 8.66675 2.66797H9.16675C9.53393 2.66797 9.83165 2.96586 9.83179 3.33301C9.83179 3.70028 9.53402 3.99805 9.16675 3.99805H8.66675C7.7226 3.99805 7.05438 3.99834 6.53198 4.04102C6.14611 4.07254 5.87277 4.12568 5.65601 4.20313L5.45581 4.28906C5.01645 4.51293 4.64872 4.85345 4.39233 5.27149L4.28979 5.45508C4.16388 5.7022 4.08381 6.01663 4.04175 6.53125C3.99906 7.05373 3.99878 7.7226 3.99878 8.66699V11.333C3.99878 12.2774 3.99906 12.9463 4.04175 13.4688C4.08381 13.9833 4.16389 14.2978 4.28979 14.5449L4.39233 14.7285C4.64871 15.1465 5.01648 15.4871 5.45581 15.7109L5.65601 15.7969C5.87276 15.8743 6.14614 15.9265 6.53198 15.958C7.05439 16.0007 7.72256 16.002 8.66675 16.002H11.3337C12.2779 16.002 12.9461 16.0007 13.4685 15.958C13.9829 15.916 14.2976 15.8367 14.5447 15.7109L14.7292 15.6074C15.147 15.3511 15.4879 14.9841 15.7117 14.5449L15.7976 14.3447C15.8751 14.128 15.9272 13.8546 15.9587 13.4688C16.0014 12.9463 16.0017 12.2774 16.0017 11.333V10.833C16.0018 10.466 16.2997 10.1681 16.6667 10.168C17.0339 10.168 17.3316 10.4659 17.3318 10.833V11.333C17.3318 12.2555 17.3331 12.9879 17.2849 13.5771C17.2422 14.0993 17.1584 14.5541 16.9792 14.9717L16.8962 15.1484C16.5609 15.8066 16.0507 16.3571 15.4246 16.7412L15.1492 16.8955C14.6833 17.1329 14.1739 17.2354 13.5769 17.2842C12.9878 17.3323 12.256 17.332 11.3337 17.332H8.66675C7.74446 17.332 7.01271 17.3323 6.42358 17.2842C5.90135 17.2415 5.44665 17.1577 5.02905 16.9785L4.85229 16.8955C4.19396 16.5601 3.64271 16.0502 3.25854 15.4238L3.10425 15.1484C2.86697 14.6827 2.76534 14.1739 2.71655 13.5771C2.66841 12.9879 2.6687 12.2555 2.6687 11.333ZM13.4646 3.11328C14.4201 2.334 15.8288 2.38969 16.7195 3.28027L16.8865 3.46485C17.6141 4.35685 17.6143 5.64423 16.8865 6.53613L16.7195 6.7207L11.6726 11.7686C11.1373 12.3039 10.4624 12.6746 9.72827 12.8408L9.41089 12.8994L7.59351 13.1582C7.38637 13.1877 7.17701 13.1187 7.02905 12.9707C6.88112 12.8227 6.81199 12.6134 6.84155 12.4063L7.10132 10.5898L7.15991 10.2715C7.3262 9.53749 7.69692 8.86241 8.23218 8.32715L13.2791 3.28027L13.4646 3.11328ZM15.7791 4.2207C15.3753 3.81702 14.7366 3.79124 14.3035 4.14453L14.2195 4.2207L9.17261 9.26856C8.81541 9.62578 8.56774 10.0756 8.45679 10.5654L8.41772 10.7773L8.28296 11.7158L9.22241 11.582L9.43433 11.543C9.92426 11.432 10.3749 11.1844 10.7322 10.8271L15.7791 5.78027L15.8552 5.69629C16.185 5.29194 16.1852 4.708 15.8552 4.30371L15.7791 4.2207Z"></path></svg>
                                                        </div>
                                                </a>
                                        </Tooltip>

                                        <Tooltip content={$i18n.t('Search')} placement="bottom">
                                                <button
                                                        class="cursor-pointer flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition group"
                                                        on:click={(e) => {
                                                                e.stopImmediatePropagation();
                                                                e.preventDefault();

                                                                showSearch.set(true);
                                                        }}
                                                        draggable="false"
                                                        aria-label={$i18n.t('Search')}
                                                >
                                                        <div class="self-center flex items-center justify-center size-9">
                                                                <svg width="20" height="20" viewBox="0 0 20 20" fill="currentColor" xmlns="http://www.w3.org/2000/svg" class="icon" aria-hidden="true"><path d="M14.0857 8.74999C14.0857 5.80355 11.6972 3.41503 8.75073 3.41503C5.80429 3.41503 3.41577 5.80355 3.41577 8.74999C3.41577 11.6964 5.80429 14.085 8.75073 14.085C11.6972 14.085 14.0857 11.6964 14.0857 8.74999ZM15.4158 8.74999C15.4158 10.3539 14.848 11.8245 13.9041 12.9746L13.9705 13.0303L16.9705 16.0303L17.0564 16.1338C17.2269 16.3919 17.1977 16.7434 16.9705 16.9707C16.7432 17.1975 16.3925 17.226 16.1345 17.0557L16.03 16.9707L13.03 13.9707L12.9753 13.9033C11.8253 14.8472 10.3547 15.415 8.75073 15.415C5.06975 15.415 2.08569 12.431 2.08569 8.74999C2.08569 5.06901 5.06975 2.08495 8.75073 2.08495C12.4317 2.08495 15.4158 5.06901 15.4158 8.74999Z"></path></svg>
                                                        </div>
                                                </button>
                                        </Tooltip>

                                        {#if ($config?.features?.enable_notes ?? false) && ($user?.role === 'admin' || ($user?.permissions?.features?.notes ?? true))}
                                                <Tooltip content={$i18n.t('Notes')} placement="bottom">
                                                        <a
                                                                class="cursor-pointer flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition group"
                                                                href="/notes"
                                                                on:click={async (e) => {
                                                                        e.stopImmediatePropagation();
                                                                        e.preventDefault();

                                                                        goto('/notes');
                                                                        itemClickHandler();
                                                                }}
                                                                draggable="false"
                                                                aria-label={$i18n.t('Notes')}
                                                        >
                                                                <div class="self-center flex items-center justify-center size-9">
                                                                        <Note className="size-4.5" />
                                                                </div>
                                                        </a>
                                                </Tooltip>
                                        {/if}

                                        {#if $user?.role === 'admin' || $user?.permissions?.workspace?.models || $user?.permissions?.workspace?.knowledge || $user?.permissions?.workspace?.prompts || $user?.permissions?.workspace?.tools}
                                                <Tooltip content={$i18n.t('Workspace')} placement="bottom">
                                                        <a
                                                                class="cursor-pointer flex rounded-xl hover:bg-gray-100 dark:hover:bg-gray-850 transition group"
                                                                href="/workspace"
                                                                on:click={async (e) => {
                                                                        e.stopImmediatePropagation();
                                                                        e.preventDefault();

                                                                        goto('/workspace');
                                                                        itemClickHandler();
                                                                }}
                                                                aria-label={$i18n.t('Workspace')}
                                                                draggable="false"
                                                        >
                                                                <div class="self-center flex items-center justify-center size-9">
                                                                        <svg
                                                                                xmlns="http://www.w3.org/2000/svg"
                                                                                fill="none"
                                                                                viewBox="0 0 24 24"
                                                                                stroke-width="1.5"
                                                                                stroke="currentColor"
                                                                                class="size-4.5"
                                                                        >
                                                                                <path
                                                                                        stroke-linecap="round"
                                                                                        stroke-linejoin="round"
                                                                                        d="M13.5 16.875h3.375m0 0h3.375m-3.375 0V13.5m0 3.375v3.375M6 10.5h2.25a2.25 2.25 0 0 0 2.25-2.25V6a2.25 2.25 0 0 0-2.25-2.25H6A2.25 2.25 0 0 0 3.75 6v2.25A2.25 2.25 0 0 0 6 10.5Zm0 9.75h2.25A2.25 2.25 0 0 0 10.5 18v-2.25a2.25 2.25 0 0 0-2.25-2.25H6a2.25 2.25 0 0 0-2.25 2.25V18A2.25 2.25 0 0 0 6 20.25Zm9.75-9.75H18a2.25 2.25 0 0 0 2.25-2.25V6A2.25 2.25 0 0 0 18 3.75h-2.25A2.25 2.25 0 0 0 13.5 6v2.25a2.25 2.25 0 0 0 2.25 2.25Z"
                                                                                />
                                                                        </svg>
                                                                </div>
                                                        </a>
                                                </Tooltip>
                                        {/if}
                                </div>

                        <div
                                class="relative flex flex-col flex-1 overflow-y-auto scrollbar-hidden pt-3 pb-3"
                                on:scroll={(e) => {
                                        if (e.target.scrollTop === 0) {
                                                scrollTop = 0;
                                        } else {
                                                scrollTop = e.target.scrollTop;
                                        }
                                }}
                        >
                                {#if ($models ?? []).length > 0 && (($settings?.pinnedModels ?? []).length > 0 || $config?.default_pinned_models)}
                                        <Folder
                                                id="sidebar-models"
                                                className="px-2 mt-0.5"
                                                name={$i18n.t('Models')}
                                                chevron={false}
                                                dragAndDrop={false}
                                        >
                                                <PinnedModelList bind:selectedChatId {shiftKey} />
                                        </Folder>
                                {/if}

                                {#if $config?.features?.enable_channels && ($user?.role === 'admin' || $channels.length > 0)}
                                        <Folder
                                                id="sidebar-channels"
                                                className="px-2 mt-0.5"
                                                name={$i18n.t('Channels')}
                                                chevron={false}
                                                dragAndDrop={false}
                                                onAdd={async () => {
                                                        if ($user?.role === 'admin') {
                                                                await tick();

                                                                setTimeout(() => {
                                                                        showCreateChannel = true;
                                                                }, 0);
                                                        }
                                                }}
                                                onAddLabel={$i18n.t('Create Channel')}
                                        >
                                                {#each $channels as channel}
                                                        <ChannelItem
                                                                {channel}
                                                                onUpdate={async () => {
                                                                        await initChannels();
                                                                }}
                                                        />
                                                {/each}
                                        </Folder>
                                {/if}

                                {#if folders}
                                        <Folder
                                                id="sidebar-folders"
                                                className="px-2 mt-0.5"
                                                name={$i18n.t('Folders')}
                                                chevron={false}
                                                onAdd={() => {
                                                        showCreateFolderModal = true;
                                                }}
                                                onAddLabel={$i18n.t('New Folder')}
                                                on:drop={async (e) => {
                                                        const { type, id, item } = e.detail;

                                                        if (type === 'folder') {
                                                                if (folders[id].parent_id === null) {
                                                                        return;
                                                                }

                                                                const res = await updateFolderParentIdById(localStorage.token, id, null).catch(
                                                                        (error) => {
                                                                                toast.error(`${error}`);
                                                                                return null;
                                                                        }
                                                                );

                                                                if (res) {
                                                                        await initFolders();
                                                                }
                                                        }
                                                }}
                                        >
                                                <Folders
                                                        bind:folderRegistry
                                                        {folders}
                                                        {shiftKey}
                                                        onDelete={(folderId) => {
                                                                selectedFolder.set(null);
                                                                initChatList();
                                                        }}
                                                        on:update={() => {
                                                                initChatList();
                                                        }}
                                                        on:import={(e) => {
                                                                const { folderId, items } = e.detail;
                                                                importChatHandler(items, false, folderId);
                                                        }}
                                                        on:change={async () => {
                                                                initChatList();
                                                        }}
                                                />
                                        </Folder>
                                {/if}

                                <Folder
                                        id="sidebar-chats"
                                        className="px-2 mt-0.5"
                                        name={$i18n.t('Chats')}
                                        chevron={false}
                                        on:change={async (e) => {
                                                selectedFolder.set(null);
                                        }}
                                        on:import={(e) => {
                                                importChatHandler(e.detail);
                                        }}
                                        on:drop={async (e) => {
                                                const { type, id, item } = e.detail;

                                                if (type === 'chat') {
                                                        let chat = await getChatById(localStorage.token, id).catch((error) => {
                                                                return null;
                                                        });
                                                        if (!chat && item) {
                                                                chat = await importChats(localStorage.token, [
                                                                        {
                                                                                chat: item.chat,
                                                                                meta: item?.meta ?? {},
                                                                                pinned: false,
                                                                                folder_id: null,
                                                                                created_at: item?.created_at ?? null,
                                                                                updated_at: item?.updated_at ?? null
                                                                        }
                                                                ]);
                                                        }

                                                        if (chat) {
                                                                console.log(chat);
                                                                if (chat.folder_id) {
                                                                        const res = await updateChatFolderIdById(localStorage.token, chat.id, null).catch(
                                                                                (error) => {
                                                                                        toast.error(`${error}`);
                                                                                        return null;
                                                                                }
                                                                        );

                                                                        folderRegistry[chat.folder_id]?.setFolderItems();
                                                                }

                                                                if (chat.pinned) {
                                                                        const res = await toggleChatPinnedStatusById(localStorage.token, chat.id);
                                                                }

                                                                initChatList();
                                                        }
                                                } else if (type === 'folder') {
                                                        if (folders[id].parent_id === null) {
                                                                return;
                                                        }

                                                        const res = await updateFolderParentIdById(localStorage.token, id, null).catch(
                                                                (error) => {
                                                                        toast.error(`${error}`);
                                                                        return null;
                                                                }
                                                        );

                                                        if (res) {
                                                                await initFolders();
                                                        }
                                                }
                                        }}
                                >
                                        {#if $pinnedChats.length > 0}
                                                <div class="mb-1">
                                                        <div class="flex flex-col space-y-1 rounded-xl">
                                                                <Folder
                                                                        id="sidebar-pinned-chats"
                                                                        buttonClassName=" text-gray-500"
                                                                        on:import={(e) => {
                                                                                importChatHandler(e.detail, true);
                                                                        }}
                                                                        on:drop={async (e) => {
                                                                                const { type, id, item } = e.detail;

                                                                                if (type === 'chat') {
                                                                                        let chat = await getChatById(localStorage.token, id).catch((error) => {
                                                                                                return null;
                                                                                        });
                                                                                        if (!chat && item) {
                                                                                                chat = await importChats(localStorage.token, [
                                                                                                        {
                                                                                                                chat: item.chat,
                                                                                                                meta: item?.meta ?? {},
                                                                                                                pinned: false,
                                                                                                                folder_id: null,
                                                                                                                created_at: item?.created_at ?? null,
                                                                                                                updated_at: item?.updated_at ?? null
                                                                                                        }
                                                                                                ]);
                                                                                        }

                                                                                        if (chat) {
                                                                                                console.log(chat);
                                                                                                if (chat.folder_id) {
                                                                                                        const res = await updateChatFolderIdById(
                                                                                                                localStorage.token,
                                                                                                                chat.id,
                                                                                                                null
                                                                                                        ).catch((error) => {
                                                                                                                toast.error(`${error}`);
                                                                                                                return null;
                                                                                                        });
                                                                                                }

                                                                                                if (!chat.pinned) {
                                                                                                        const res = await toggleChatPinnedStatusById(localStorage.token, chat.id);
                                                                                                }

                                                                                                initChatList();
                                                                                        }
                                                                                }
                                                                        }}
                                                                        name={$i18n.t('Pinned')}
                                                                >
                                                                        <div
                                                                                class="ml-3 pl-1 mt-[1px] flex flex-col overflow-y-auto scrollbar-hidden border-s border-gray-100 dark:border-gray-900 text-gray-900 dark:text-gray-200"
                                                                        >
                                                                                {#each $pinnedChats as chat, idx (`pinned-chat-${chat?.id ?? idx}`)}
                                                                                        <ChatItem
                                                                                                className=""
                                                                                                id={chat.id}
                                                                                                title={chat.title}
                                                                                                {shiftKey}
                                                                                                selected={selectedChatId === chat.id}
                                                                                                on:select={() => {
                                                                                                        selectedChatId = chat.id;
                                                                                                }}
                                                                                                on:unselect={() => {
                                                                                                        selectedChatId = null;
                                                                                                }}
                                                                                                on:change={async () => {
                                                                                                        initChatList();
                                                                                                }}
                                                                                                on:tag={(e) => {
                                                                                                        const { type, name } = e.detail;
                                                                                                        tagEventHandler(type, name, chat.id);
                                                                                                }}
                                                                                        />
                                                                                {/each}
                                                                        </div>
                                                                </Folder>
                                                        </div>
                                                </div>
                                        {/if}

                                        <div class=" flex-1 flex flex-col overflow-y-auto scrollbar-hidden">
                                                <div class="pt-1.5">
                                                        {#if $chats}
                                                                {#each $chats as chat, idx (`chat-${chat?.id ?? idx}`)}
                                                                        {#if idx === 0 || (idx > 0 && chat.time_range !== $chats[idx - 1].time_range)}
                                                                                <div
                                                                                        class="w-full pl-2.5 text-xs text-gray-500 dark:text-gray-500 font-medium {idx ===
                                                                                        0
                                                                                                ? ''
                                                                                                : 'pt-5'} pb-1.5"
                                                                                >
                                                                                        {$i18n.t(chat.time_range)}
                                                                                        <!-- localisation keys for time_range to be recognized from the i18next parser (so they don't get automatically removed):
                                                        {$i18n.t('Today')}
                                                        {$i18n.t('Yesterday')}
                                                        {$i18n.t('Previous 7 days')}
                                                        {$i18n.t('Previous 30 days')}
                                                        {$i18n.t('January')}
                                                        {$i18n.t('February')}
                                                        {$i18n.t('March')}
                                                        {$i18n.t('April')}
                                                        {$i18n.t('May')}
                                                        {$i18n.t('June')}
                                                        {$i18n.t('July')}
                                                        {$i18n.t('August')}
                                                        {$i18n.t('September')}
                                                        {$i18n.t('October')}
                                                        {$i18n.t('November')}
                                                        {$i18n.t('December')}
                                                        -->
                                                                                </div>
                                                                        {/if}

                                                                        <ChatItem
                                                                                className=""
                                                                                id={chat.id}
                                                                                title={chat.title}
                                                                                {shiftKey}
                                                                                selected={selectedChatId === chat.id}
                                                                                on:select={() => {
                                                                                        selectedChatId = chat.id;
                                                                                }}
                                                                                on:unselect={() => {
                                                                                        selectedChatId = null;
                                                                                }}
                                                                                on:change={async () => {
                                                                                        initChatList();
                                                                                }}
                                                                                on:tag={(e) => {
                                                                                        const { type, name } = e.detail;
                                                                                        tagEventHandler(type, name, chat.id);
                                                                                }}
                                                                        />
                                                                {/each}

                                                                {#if $scrollPaginationEnabled && !allChatsLoaded}
                                                                        <Loader
                                                                                on:visible={(e) => {
                                                                                        if (!chatListLoading) {
                                                                                                loadMoreChats();
                                                                                        }
                                                                                }}
                                                                        >
                                                                                <div
                                                                                        class="w-full flex justify-center py-1 text-xs animate-pulse items-center gap-2"
                                                                                >
                                                                                        <Spinner className=" size-4" />
                                                                                        <div class=" ">{$i18n.t('Loading...')}</div>
                                                                                </div>
                                                                        </Loader>
                                                                {/if}
                                                        {:else}
                                                                <div
                                                                        class="w-full flex justify-center py-1 text-xs animate-pulse items-center gap-2"
                                                                >
                                                                        <Spinner className=" size-4" />
                                                                        <div class=" ">{$i18n.t('Loading...')}</div>
                                                                </div>
                                                        {/if}
                                                </div>
                                        </div>
                                </Folder>
                        </div>

                        <div class="px-1.5 pt-1.5 pb-2 sticky bottom-0 z-10 -mt-3 sidebar">
                                <div
                                        class=" sidebar-bg-gradient-to-t bg-linear-to-t from-gray-50 dark:from-gray-950 to-transparent from-50% pointer-events-none absolute inset-0 -z-10 -mt-6"
                                ></div>
                                <div class="flex flex-col font-primary">
                                        {#if $user !== undefined && $user !== null}
                                                <UserMenu
                                                        role={$user?.role}
                                                        on:show={(e) => {
                                                                if (e.detail === 'archived-chat') {
                                                                        showArchivedChats.set(true);
                                                                }
                                                        }}
                                                >
                                                        <div
                                                                class=" flex items-center rounded-2xl py-2 px-1.5 w-full hover:bg-gray-100/50 dark:hover:bg-gray-900/50 transition"
                                                        >
                                                                <div class=" self-center mr-3">
                                                                        <Settings className="size-6" strokeWidth="1.5" />
                                                                </div>
                                                                <div class=" self-center font-medium">{$user?.name}</div>
                                                        </div>
                                                </UserMenu>
                                        {/if}
                                </div>
                        </div>
                </div>
        </div>
        {/if}
        </div>
{/if}
