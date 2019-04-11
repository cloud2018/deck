<!--
  - @copyright Copyright (c) 2018 John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @author John Molakvoæ <skjnldsv@protonmail.com>
  -
  - @license GNU AGPL version 3 or any later version
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
	<router-link :id="`board-${board.id}`"
		:title="board.title" :class="[{'icon-loading-small': loading, deleted: deleted, editing: editing }, classes]"
		:to="routeTo" tag="li">
		<div :style="{ backgroundColor: `#${board.color}` }" class="app-navigation-entry-bullet" />
		<a href="#">
			{{ board.title }}
		</a>
		<div v-if="actions.length > 0" class="app-navigation-entry-utils">
			<ul>
				<li class="app-navigation-entry-utils-menu-button">
					<button v-click-outside="hideMenu" @click="showMenu" />
				</li>
			</ul>
		</div>
		<div :class="{ 'open': menuOpen }" class="app-navigation-entry-menu">
			<popover-menu :menu="actions" />
		</div>

		<!-- undo action -->
		<div v-if="deleted" class="app-navigation-entry-deleted">
			<div class="app-navigation-entry-deleted-description">
				{{ undoText }}
			</div>
			<button
				:title="t('settings', 'Undo')"
				class="app-navigation-entry-deleted-button icon-history"
				@click="unDelete" />
		</div>

		<!-- edit entry -->
		<div v-if="editing" class="app-navigation-entry-edit">
			<form @submit.prevent.stop="applyEdit">
				<input v-model="editTitle" type="text">
				<input type="submit" value="" class="icon-confirm">
				<input type="submit" value="" class="icon-close"
					@click.stop.prevent="cancelEdit">
			</form>
		</div>
	</router-link>
</template>

<script>
import { PopoverMenu } from 'nextcloud-vue'
import ClickOutside from 'vue-click-outside'

export default {
	name: 'AppNavigationBoard',
	components: {
		PopoverMenu
	},
	directives: {
		ClickOutside
	},
	props: {
		board: {
			type: Object,
			required: true
		}
	},
	data() {
		return {
			classes: [],
			deleted: false,
			loading: false,
			editing: false,
			menuOpen: false,
			undoTimeoutHandle: null,
			editTitle: ''
		}
	},
	computed: {
		undoText: function() {
			// todo translation
			return 'deleted ' + this.board.title
		},
		routeTo: function() {
			return {
				name: 'board',
				params: { id: this.board.id }
			}
		},
		actions: function() {
			/* eslint-disable vue/no-side-effects-in-computed-properties */
			/* eslint-disable vue/no-async-in-computed-properties */
			const actions = []

			// do not show actions while the item is loading
			if (this.loading === false) {

				actions.push({
					action: () => {
						this.hideMenu()
						this.editTitle = this.board.title
						this.editing = true
					},
					icon: 'icon-edit',
					text: t('deck', 'Edit board')
				})

				if (!this.board.archived) {
					actions.push({
						action: () => {
							this.hideMenu()
							this.loading = true
							this.$store.dispatch('archiveBoard', this.board)
						},
						icon: 'icon-archive',
						text: t('deck', 'Archive board')
					})
				}

				actions.push({
					action: () => {
						this.hideMenu()
						this.loading = true
						this.boardApi.deleteBoard(this.board)
							.then(() => {
								this.loading = false
								this.deleted = true
								this.undoTimeoutHandle = setTimeout(() => {
									this.$store.dispatch('removeBoard', this.board)
								}, 7000)
							})
					},
					icon: 'icon-delete',
					text: t('deck', 'Delete board')
				})

				actions.push({
					action: () => {},
					icon: 'icon-settings',
					text: t('deck', 'Board details')
				})

			}

			return actions
		}
	},
	watch: {},
	mounted() {
		// prevent click outside event with popupItem.
		this.popupItem = this.$el
	},
	methods: {
		unDelete() {
			clearTimeout(this.undoTimeoutHandle)
			this.boardApi.unDeleteBoard(this.board)
				.then(() => {
					this.deleted = false
				})
		},
		showMenu() {
			this.menuOpen = true
		},
		hideMenu() {
			this.menuOpen = false
		},
		applyEdit(e) {
			this.editing = false
			if (this.editTitle) {
				this.loading = true
				const copy = JSON.parse(JSON.stringify(this.board))
				copy.title = this.editTitle
				this.$store.dispatch('updateBoard', copy)
					.then(() => {
						this.loading = false
					})
			}
		},
		cancelEdit(e) {
			this.editing = false
		}
	},
	inject: [
		'boardApi'
	]
}
</script>

<style lang="scss" scoped>
	#app-navigation #deck-navigation .editing {
		height: auto !important;
	}
</style>