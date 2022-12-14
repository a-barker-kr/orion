<template>
	<div class="container">
		<div class="filter-container">
			<FiltersComponent :options="filterOptions" :show-info="true">
				<template #info>
					<IconComponent
						name="question-circle"
						fill="white"
						v-tippy="{ placement: 'bottom' }"
						:content="'Once you have unlocked each mastery camouflage, a mastery challenge is unlocked that requires you to get a certain amount of kills while using that specific camouflage to complete it.'" />
					<div class="mobile">
						<IconComponent name="question-circle" fill="white"></IconComponent>
						<p>
							Once you have unlocked each mastery camouflage, a mastery challenge is unlocked that
							requires you to get a certain amount of kills while using that specific camouflage to
							complete it.
						</p>
					</div>
				</template>
			</FiltersComponent>
			<div class="toggles">
				<FavoritesToggleComponent />
				<LayoutToggleComponent />
			</div>
		</div>

		<WeaponsComponent :weapons="filteredWeapons" :favorites="favorites" :mastery="true" />

		<ProgressComponent
			:progress="masteryProgress"
			data-camo="orion"
			label="Mastery progress"
			tooltip="Progress towards completing all mastery challenges">
			<template #modal-header>Mastery challenges completed 👏🥳</template>
			<template #modal-body>
				<p>
					Congratulations on finishing all mastery challenges! That's quite the feat! You first
					started tracking your grind here
					<b>{{ daysSinceStart }} days ago</b> on
					{{ new Date(getBeganGrind).toLocaleDateString('en-US') }}.
				</p>
			</template>
		</ProgressComponent>
	</div>
</template>

<script>
import { mapState } from 'pinia'
import { useStore } from '@/stores/store'
import { groupBy, daysBetweenDates, roundToTwoDecimals } from '@/utils/utils'

import FiltersComponent from '@/components/FiltersComponent.vue'
import WeaponsComponent from '@/components/WeaponsComponent.vue'
import ProgressComponent from '@/components/ProgressComponent.vue'
import LayoutToggleComponent from '@/components/LayoutToggleComponent.vue'
import FavoritesToggleComponent from '@/components/FavoritesToggleComponent.vue'

export default {
	components: {
		FiltersComponent,
		WeaponsComponent,
		ProgressComponent,
		LayoutToggleComponent,
		FavoritesToggleComponent,
	},

	data() {
		return {
			store: useStore(),
		}
	},

	computed: {
		...mapState(useStore, ['weapons', 'filters', 'weaponCategories', 'beganGrind']),

		getBeganGrind() {
			return this.beganGrind ?? new Date()
		},

		daysSinceStart() {
			const days = daysBetweenDates(this.getBeganGrind, new Date())
			return days ? days : 1
		},

		filterOptions() {
			return [
				{
					label: 'Category',
					key: 'weaponCategory',
					type: 'select',
					options: this.weaponCategories,
				},
				{
					label: 'Hide Gold',
					key: 'hideGold',
					type: 'checkbox',
				},
				{
					label: 'Hide Platinum',
					key: 'hidePlatinum',
					type: 'checkbox',
				},
				{
					label: 'Hide Polyatomic',
					key: 'hidePolyatomic',
					type: 'checkbox',
				},
				{
					label: 'Hide Orion',
					key: 'hideOrion',
					type: 'checkbox',
				},
			]
		},

		filteredWeapons() {
			let filteredWeapons = this.weapons
			const { hideGold, hidePlatinum, hidePolyatomic, hideOrion, weaponCategory } = this.filters

			if (hideGold) {
				filteredWeapons = filteredWeapons.filter((weapon) => !weapon.masteryProgress['Gold'])
			}

			if (hidePlatinum) {
				filteredWeapons = filteredWeapons.filter((weapon) => !weapon.masteryProgress['Platinum'])
			}

			if (hidePolyatomic) {
				filteredWeapons = filteredWeapons.filter((weapon) => !weapon.masteryProgress['Polyatomic'])
			}

			if (hideOrion) {
				filteredWeapons = filteredWeapons.filter((weapon) => !weapon.masteryProgress['Orion'])
			}

			if (weaponCategory && weaponCategory !== 'null') {
				filteredWeapons = filteredWeapons.filter((weapon) => weapon.category === weaponCategory)
			}

			return groupBy(filteredWeapons, (weapon) => weapon.category)
		},

		favorites() {
			if (!this.store) return []
			const favorites = this.store.getFavorites('mastery')
			return this.weapons.filter((weapon) => favorites.includes(weapon.name))
		},

		masteryProgress() {
			const weapons = this.weapons.filter((weapon) => !weapon.comingSoon)
			const total = weapons.length * 4
			const completed = weapons.reduce(
				(acc, weapon) => acc + Object.values(weapon.masteryProgress).reduce((a, b) => a + b, 0),
				0
			)

			return roundToTwoDecimals((completed / total) * 100)
		},
	},
}
</script>

<style lang="scss" scoped>
.container {
	text-align: center;
}

h1 {
	margin-top: 75px;
}

h2 {
	margin: 30px auto 0;
	max-width: 450px;
}

.filter-container {
	align-items: center;
	display: flex;
	width: 100%;

	@media (max-width: $tablet) {
		flex-direction: column;

		::v-deep .filters-component {
			margin-bottom: 20px;
			margin-right: 0;
			width: 100%;
		}
	}
}

::v-deep .filters-component {
	flex-grow: 1;
	margin-right: 15px;
}
</style>
