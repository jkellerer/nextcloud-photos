<!--
 - @copyright Copyright (c) 2022 Louis Chemineau <louis@chmn.me>
 - @copyright Copyright (c) 2022 Marcel Klehr <mklehr@gmx.net>
 -
 - @author Louis Chemineau <louis@chmn.me>
 - @author Marcel Klehr <mklehr@gmx.net>
 -
 - @license AGPL-3.0-or-later
 -
 - This program is free software: you can redistribute it and/or modify
 - it under the terms of the GNU Affero General Public License as
 - published by the Free Software Foundation, either version 3 of the
 - License, or (at your option) any later version.
 -
 - This program is distributed in the hope that it will be useful,
 - but WITHOUT ANY WARRANTY; without even the implied warranty of
 - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 - GNU Affero General Public License for more details.
 -
 - You should have received a copy of the GNU Affero General Public License
 - along with this program. If not, see <http://www.gnu.org/licenses/>.
 -
 -->
<template>
	<!-- Errors handlers-->
	<NcEmptyContent v-if="face === undefined && !loadingFiles && !loadingFaces" class="empty-content-with-illustration">
		<template #icon>
			<AccountBoxMultipleOutline />
		</template>
		{{ t('photos', 'This person could not be found') }}
	</NcEmptyContent>
	<NcEmptyContent v-else-if="errorFetchingFiles || errorFetchingFaces">
		<template #icon>
			<AlertCircle />
		</template>
		{{ t('photos', 'An error occurred') }}
	</NcEmptyContent>

	<div v-else class="face">
		<div class="face__header">
			<div class="face__header__left">
				<NcActions>
					<NcActionButton @click="$router.push('/faces/')">
						<template #icon>
							<ArrowLeft />
						</template>{{ t('photos', 'Back') }}
					</NcActionButton>
				</NcActions>
				<div class="face__header__title">
					<h2 v-if="face !== undefined" :class="{'face-name': true, 'hidden-visually': face.basename.match(/^[0-9]+$/)}">
						{{ face.basename }}
					</h2>
				</div>

				<NcLoadingIcon v-if="loadingCount > 0 || loadingFaces" />
			</div>
			<div v-if="face !== undefined" class="face__header__actions">
				<NcActions>
					<NcActionButton :close-after-click="true"
						:aria-label="t('photos', 'Rename person')"
						@click="showRenameModal = true">
						<template #icon>
							<Pencil />
						</template>
						{{ t('photos', 'Rename person') }}
					</NcActionButton>
				</NcActions>
				<NcActions :force-menu="true">
					<NcActionButton v-if="Object.keys(faces).length > 1"
						:close-after-click="true"
						:aria-label="t('photos', 'Merge with different person')"
						@click="showMergeModal = true">
						<template #icon>
							<Merge />
						</template>
						{{ t('photos', 'Merge with different person') }}
					</NcActionButton>
					<template v-if="selectedFileIds.length">
						<NcActionButton :close-after-click="true"
							:aria-label="t('photos', 'Download selected files')"
							@click="downloadSelection">
							<Download slot="icon" />
							{{ t('photos', 'Download selected photos') }}
						</NcActionButton>
						<NcActionButton v-if="shouldFavoriteSelection"
							:close-after-click="true"
							:aria-label="t('photos', 'Mark selection as favorite')"
							@click="favoriteSelection">
							<Star slot="icon" />
							{{ t('photos', 'Favorite') }}
						</NcActionButton>
						<NcActionButton v-else
							:close-after-click="true"
							:aria-label="t('photos', 'Remove selection from favorites')"
							@click="unFavoriteSelection">
							<Star slot="icon" />
							{{ t('photos', 'Remove from favorites') }}
						</NcActionButton>
						<NcActionButton :close-after-click="true"
							@click="handleRemoveFilesFromFace(selectedFileIds)">
							<template #icon>
								<Close />
							</template>
							{{ n('photos', 'Remove photo from person', 'Remove photos from person', selectedFileIds.length) }}
						</NcActionButton>
					</template>
					<NcActionButton :close-after-click="true"
						@click="handleDeleteFace">
						<template #icon>
							<Close />
						</template>
						{{ t('photos', 'Remove person') }}
					</NcActionButton>
				</NcActions>
			</div>
		</div>

		<FilesListViewer v-if="face !== undefined"
			class="face__photos"
			:container-element="appContent"
			:file-ids="faceFileIds"
			:loading="loadingFiles || loadingFaces">
			<File slot-scope="{file, visibility}"
				:file="files[file.id]"
				:allow-selection="true"
				:selected="selection[file.id] === true"
				:visibility="visibility"
				:semaphore="semaphore"
				@click="openViewer"
				@select-toggled="onFileSelectToggle" />
		</FilesListViewer>

		<NcModal v-if="showRenameModal"
			:title="t('photos', 'Rename person')"
			@close="showRenameModal = false">
			<div class="rename-form">
				<input ref="nameInput"
					v-focus
					:value="faceName"
					type="text"
					name="name"
					required
					:placeholder="t('photos', 'Name of this person')"
					@keydown.enter="handleRenameFace($refs.nameInput.value)">
				<NcButton :aria-label="t('photos', 'Save.')"
					type="primary"
					:disabled="$refs.nameInput && $refs.nameInput.value.trim() === ''"
					@click="handleRenameFace($refs.nameInput.value)">
					<template #icon>
						<NcLoadingIcon v-if="loadingCount" />
						<Send v-else />
					</template>
					{{ t('photos', 'Save') }}
				</NcButton>
			</div>
		</NcModal>

		<NcModal v-if="showMergeModal"
			:title="t('photos', 'Merge person')"
			@close="showMergeModal = false">
			<FaceMergeForm :first-face="faceName" @select="handleMerge($event)" />
		</NcModal>
	</div>
</template>

<script>
import { mapActions, mapGetters } from 'vuex'
import Pencil from 'vue-material-design-icons/Pencil'
import Close from 'vue-material-design-icons/Close'
import AlertCircle from 'vue-material-design-icons/AlertCircle'
import Star from 'vue-material-design-icons/Star'
import Download from 'vue-material-design-icons/Download'
import Send from 'vue-material-design-icons/Send'
import Merge from 'vue-material-design-icons/Merge'
import ArrowLeft from 'vue-material-design-icons/ArrowLeft'
import AccountBoxMultipleOutline from 'vue-material-design-icons/AccountBoxMultipleOutline'

import { NcActions, NcActionButton, NcModal, NcEmptyContent, NcButton, NcLoadingIcon } from '@nextcloud/vue'

import FetchFilesMixin from '../mixins/FetchFilesMixin.js'
import FilesSelectionMixin from '../mixins/FilesSelectionMixin.js'
import FilesListViewer from '../components/FilesListViewer.vue'
import File from '../components/File.vue'
import logger from '../services/logger.js'
import FetchFacesMixin from '../mixins/FetchFacesMixin.js'
import Vue from 'vue'
import FaceMergeForm from '../components/FaceMergeForm.vue'

export default {
	name: 'FaceContent',
	components: {
		Pencil,
		Star,
		Download,
		Close,
		AlertCircle,
		Send,
		Merge,
		ArrowLeft,
		AccountBoxMultipleOutline,
		FaceMergeForm,
		FilesListViewer,
		File,
		NcLoadingIcon,
		NcEmptyContent,
		NcActions,
		NcActionButton,
		NcModal,
		NcButton,
	},

	directives: {
		focus(el) {
			Vue.nextTick(() => el.focus())
		},
	},

	mixins: [
		FetchFacesMixin,
		FetchFilesMixin,
		FilesSelectionMixin,
	],

	props: {
		faceName: {
			type: String,
			default: '/',
		},
	},

	data() {
		return {
			showMergeModal: false,
			showRenameModal: false,
			loadingCount: 0,
			appContent: document.getElementById('app-content-vue'),
		}
	},

	computed: {
		...mapGetters([
			'files',
			'facesFiles',
		]),

		/**
		 * @return {string[]} The face information for the current faceName.
		 */
		face() {
			return this.faces[this.faceName]
		},

		/**
		 * @return {string[]} The list of files for the current faceName.
		 */
		faceFileIds() {
			return this.facesFiles[this.faceName] || []
		},

		/** @type {boolean} */
		shouldFavoriteSelection() {
			// Favorite all selection if at least one file is not on the favorites.
			return this.selectedFileIds.some((fileId) => this.$store.state.files.files[fileId].favorite === 0)
		},
	},

	watch: {
		face() {
			if (this.face) {
				this.fetchFaceContent(this.faceName)
			}
		},
	},

	methods: {
		...mapActions([
			'appendFiles',
			'deleteFace',
			'renameFace',
			'downloadFiles',
			'toggleFavoriteForFiles',
			'removeFilesFromFace',
			'moveFilesToFace',
		]),

		openViewer(fileId) {
			const file = this.files[fileId]
			OCA.Viewer.open({
				path: file.filename,
				list: this.faceFileIds.map(fileId => ({
					...this.files[fileId],
					basename: this.files[fileId].basename.split('-').slice(1).join('-'),
				})).filter(file => !file.sectionHeader),
				loadMore: file.loadMore ? async () => await file.loadMore(true) : () => [],
				canLoop: file.canLoop,
			})
		},

		async handleRemoveFilesFromFace(fileIds) {
			try {
				this.loadingCount++
				await this.removeFilesFromFace({ faceName: this.faceName, fileIdsToRemove: fileIds })
				this.resetSelection()
			} catch (error) {
				logger.error(error)
			} finally {
				this.loadingCount--
			}
		},

		async handleDeleteFace() {
			try {
				this.loadingCount++
				await this.deleteFace({ faceName: this.faceName })
				this.$router.push('/faces')
			} catch (error) {
				logger.error(error)
			} finally {
				this.loadingCount--
			}
		},

		async handleRenameFace(faceName) {
			try {
				this.loadingCount++
				this.showRenameModal = false
				const oldName = this.faceName
				await this.renameFace({ oldName, faceName })
				this.$router.push({ name: 'facecontent', params: { faceName } })
			} catch (error) {
				logger.error(error)
			} finally {
				this.loadingCount--
			}
		},

		async handleMerge(faceName) {
			try {
				this.loadingCount++
				await this.moveFilesToFace({ oldFace: this.faceName, faceName, fileIdsToMove: this.facesFiles[this.faceName] })
				await this.deleteFace({ faceName: this.faceName })
				this.showMergeModal = false
				this.$router.push({ name: 'facecontent', params: { faceName } })
			} catch (error) {
				logger.error(error)
			} finally {
				this.loadingCount--
			}
		},

		async favoriteSelection() {
			try {
				this.loadingCount++
				await this.toggleFavoriteForFiles({ fileIds: this.selectedFileIds, favoriteState: true })
			} catch (error) {
				logger.error(error)
			} finally {
				this.loadingCount--
			}
		},

		async unFavoriteSelection() {
			try {
				this.loadingCount++
				await this.toggleFavoriteForFiles({ fileIds: this.selectedFileIds, favoriteState: false })
			} catch (error) {
				logger.error(error)
			} finally {
				this.loadingCount--
			}
		},

		async downloadSelection() {
			try {
				this.loadingCount++
				await this.downloadFiles(this.selectedFileIds)
			} catch (error) {
				logger.error(error)
			} finally {
				this.loadingCount--
			}
		},
	},
}
</script>
<style lang="scss" scoped>
.face {
	display: flex;
	flex-direction: column;

	&__empty {
		display: flex;
		flex-direction: column;
		align-items: center;

		&__button {
			margin-top: 32px;
		}

	}

	&__header {
		display: flex;
		min-height: 60px;
		align-items: center;
		justify-content: space-between;
		position: sticky;
		z-index: 3;
		background: var(--color-main-background);
		padding: 0 64px;

		@media only screen and (max-width: 1020px) {
			padding: 0;
			padding-left: 64px;
		}

		&__left {
			height: 100%;
			display: flex;
			align-items: center;
		}

		&__title {
			margin-left: 10px;
			h2 {
				margin-bottom: 0;
			}
		}

		&__loader {
			margin-left: 32px;
		}

		&__actions {
			display: flex;
			align-items: center;

			button {
				margin-left: 16px;
			}
		}
	}

	&__photos {
		margin-top: 16px;
		height: 100%;
		min-height: 0; // Prevent it from overflowing in a flex context.
		padding: 0 64px;

		@media only screen and (max-width: 1020px) {
			padding: 0;
		}
	}
}

.empty-content-with-illustration :deep .empty-content__icon {
	width: 200px;
	height: 200px;

	svg {
		width: 200px;
		height: 200px;
	}
}

.rename-form {
	display: flex;
	flex-direction: row;
	align-items: center;
	height: 70px;
	padding: 16px;

	input {
		width: 80%;
	}
}
</style>
