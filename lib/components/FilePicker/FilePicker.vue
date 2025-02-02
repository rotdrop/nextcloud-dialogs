<template>
	<NcDialog v-bind="dialogProps" @update:open="handleClose">
		<template #navigation="{ isCollapsed }">
			<FilePickerNavigation :is-collapsed="isCollapsed" :current-view.sync="currentView" :filter-string.sync="filterString" />
		</template>

		<div class="file-picker__main">
			<!-- Header title / file list breadcrumbs -->
			<FilePickerBreadcrumbs v-if="currentView === 'files'"
				:path.sync="currentPath"
				:show-menu="allowPickDirectory"
				@create-node="onCreateFolder" />
			<div v-else class="file-picker__view">
				<h3>{{ viewHeadline }}</h3>
			</div>

			<!-- File list -->
			<!-- If loading or files found show file list, otherwise show empty content-->
			<FileList v-if="isLoading || filteredFiles.length > 0"
				:allow-pick-directory="allowPickDirectory"
				:current-view="currentView"
				:files="filteredFiles"
				:multiselect="multiselect"
				:loading="isLoading"
				:path.sync="currentPath"
				:selected-files.sync="selectedFiles"
				:name="viewHeadline"
				@update:path="currentView = 'files'" />
			<NcEmptyContent v-else-if="filterString"
				:name="t('No matching files')"
				:description="t('No files matching your filter were found.')">
				<template #icon>
					<IconFile />
				</template>
			</NcEmptyContent>
			<NcEmptyContent v-else
				:name="t('No files in here')"
				:description="noFilesDescription">
				<template #icon>
					<IconFile />
				</template>
			</NcEmptyContent>
		</div>
	</NcDialog>
</template>

<script setup lang="ts">
import type { IFilePickerButton, IFilePickerButtonFactory, IFilePickerFilter } from '../types'
import type { Node } from '@nextcloud/files'

import IconFile from 'vue-material-design-icons/File.vue'
import FileList from './FileList.vue'
import FilePickerBreadcrumbs from './FilePickerBreadcrumbs.vue'
import FilePickerNavigation from './FilePickerNavigation.vue'

import { emit as emitOnEventBus } from '@nextcloud/event-bus'
import { davRootPath } from '@nextcloud/files'
import { NcDialog, NcEmptyContent } from '@nextcloud/vue'
import { join } from 'path'
import { computed, onMounted, ref, toRef } from 'vue'
import { showError } from '../../toast'
import { useDAVFiles } from '../../composables/dav'
import { useMimeFilter } from '../../composables/mime'
import { t } from '../../utils/l10n'
import { useFilesSettings } from '../../composables/filesSettings'

const props = withDefaults(defineProps<{
	/** Buttons to be displayed */
	buttons: IFilePickerButton[] | IFilePickerButtonFactory

	/** The name of file picker dialog (heading) */
	name: string

	/**
	 * Can directories be picked
	 * @default false
	 */
	allowPickDirectory?: boolean

	/**
	 * Where to mount the dialog
	 * @default 'body'
	 */
	container?: string

	/**
	 * Custom filter function used to filter pickable files
	 */
	filterFn?: IFilePickerFilter

	/**
	 * List of allowed mime types
	 * You can use placeholders for e.g. allowing all subtypes of images `['image/*']`.
	 * Note that if unset all files are allowed, which is the same as passing `['*∕*']`
	 * @default []
	 */
	mimetypeFilter?: string[]

	/**
	 * Is it allowed to pick multiple files
	 * @default true
	 */
	multiselect?: boolean

	/**
	 * The initial path of the file picker
	 * @default '/'
	 */
	 path?: string
}>(), {
	allowPickDirectory: false,
	container: 'body',
	filterFn: undefined,
	mimetypeFilter: () => [],
	multiselect: true,
	path: undefined,
})

const emit = defineEmits<{
	(e: 'close', v?: Node[]): void
}>()

/**
 * Props to be passed to the underlying Dialog component
 */
const dialogProps = computed(() => ({
	container: props.container,
	name: props.name,
	buttons: dialogButtons.value,
	size: 'large',
	contentClasses: ['file-picker__content'],
	dialogClasses: ['file-picker'],
	navigationClasses: ['file-picker__navigation'],
}))

/**
 * Map buttons to Dialog buttons by wrapping the callback function to pass the selected files
 */
const dialogButtons = computed(() => {
	const buttons = typeof props.buttons === 'function'
		? props.buttons(selectedFiles.value as Node[], currentPath.value, currentView.value)
		: props.buttons

	return buttons.map((button) => ({
		...button,
		callback: async () => {
			const nodes = selectedFiles.value.length === 0 && props.allowPickDirectory ? [await getFile(currentPath.value)] : selectedFiles.value as Node[]
			button.callback(nodes)
			emit('close', nodes)
		},
	} as IFilePickerButton))
})

/**
 * Name of the currently active view
 */
const currentView = ref<'files' | 'favorites' | 'recent'>('files')

/**
 * Headline to be used on the current view
 */
const viewHeadline = computed(() => currentView.value === 'favorites' ? t('Favorites') : (currentView.value === 'recent' ? t('Recent') : ''))

/**
 * All currently selected files
 */
const selectedFiles = ref<Node[]>([])

/**
 * Last path navigated to using the file picker
 * (required as sessionStorage is not reactive)
 */
const savedPath = ref(window?.sessionStorage.getItem('NC.FilePicker.LastPath') || '/')

/**
 * The path the user manually navigated to using this filepicker instance
 */
const navigatedPath = ref<string>()

/**
 * The current path that should be picked from
 */
const currentPath = computed({
	// Only use the path for the files view as favorites and recent only works on the root
	get: () => currentView.value === 'files' ? navigatedPath.value || props.path || savedPath.value : '/',
	/**
	 * Navigate to the new path and save it to the session storage
	 *
	 * @param path The new path
	 */
	set: (path: string) => {
		if (props.path === undefined) {
			window.sessionStorage.setItem('NC.FilePicker.LastPath', path)
		}
		navigatedPath.value = path
		// Reset selected files
		selectedFiles.value = []
	},
})

/**
 * A string used to filter files in current view
 */
const filterString = ref('')

const { isSupportedMimeType } = useMimeFilter(toRef(props, 'mimetypeFilter')) // vue 3.3 will allow cleaner syntax of toRef(() => props.mimetypeFilter)

const { files, isLoading, loadFiles, getFile, client } = useDAVFiles(currentView, currentPath)

onMounted(() => loadFiles())

const { showHiddenFiles } = useFilesSettings()

/**
 * The files list filtered by the current value of the filter input
 */
const filteredFiles = computed(() => {
	let filtered = files.value

	if (!showHiddenFiles.value) {
		// Hide hidden files if not configured otherwise
		filtered = filtered.filter((file) => !file.basename.startsWith('.'))
	}
	if (props.mimetypeFilter.length > 0) {
		// filter by mime type but always include folders to navigate
		filtered = filtered.filter(file => file.type === 'folder' || (file.mime && isSupportedMimeType(file.mime)))
	}
	if (filterString.value) {
		filtered = filtered.filter((file) => file.basename.toLowerCase().includes(filterString.value.toLowerCase()))
	}
	if (props.filterFn) {
		filtered = filtered.filter((f) => props.filterFn(f as Node))
	}
	return filtered
})

/**
 * If no files are found in the current view this message will be shown in the EmptyContent
 */
const noFilesDescription = computed(() => {
	if (currentView.value === 'files') {
		return t('Upload some content or sync with your devices!')
	} else if (currentView.value === 'recent') {
		return t('Files and folders you recently modified will show up here.')
	} else {
		return t('Files and folders you mark as favorite will show up here.')
	}
})

/**
 * Handle creating new folder (breadcrumb menu)
 * This will create the folder using WebDAV, reload the directory content and signal the directory creation to fhe files app
 *
 * @param name The new folder name
 */
const onCreateFolder = async (name: string) => {
	try {
		await client.createDirectory(join(davRootPath, currentPath.value, name))
		// reload file list
		await loadFiles()
		// emit event bus to force files app to reload that file if needed
		emitOnEventBus('files:node:created', files.value.filter((file) => file.basename === name)[0])
	} catch (error) {
		console.warn('Could not create new folder', { name, error })
		// show error to user
		showError(t('Could not create the new folder'))
	}
}

/**
 * Handle closing the file picker
 * @param open If the dialog is open
 */
const handleClose = (open: boolean) => {
	if (!open) {
		emit('close')
	}
}
</script>

<script lang="ts">
export default {
	name: 'FilePicker',
}
</script>

<style scoped lang="scss">
.file-picker {
	&__view {
		height: 50px; // align with breadcrumbs
		display: flex;
		justify-content: start;
		align-items: center;

		h3 {
			font-weight: bold;
			height: fit-content;
			margin: 0;
		}
	}

	&__main {
		box-sizing: border-box;
		width: 100%;
		display: flex;
		flex-direction: column;
		// Auto fit height
		min-height: 0;
		flex: 1;
		// align with navigation on smaller screens
		padding-inline: 2px;

		* {
			box-sizing: border-box;
		}
	}
}

:deep(.file-picker) {
	// Dialog is max. 900px wide so the best looking height seems to be 800px
	height: min(80vh, 800px)!important;
}

@media (max-width: 512px) {
	:deep(.file-picker) {
		// below 512px the modal is fullscreen so we use 100% height - margin of heading (4px + 12px) - height of heading (default-clickable-area)
		height: calc(100% - 16px - var(--default-clickable-area))!important;
	}
}

:deep(.file-picker__content) {
	display: flex;
	flex-direction: column;
	overflow: hidden;
}
</style>
