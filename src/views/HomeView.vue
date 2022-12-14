<template>
	<div class="container">
		<AlertComponent style="margin-bottom: 30px">
			This tracker is currently under development and more content will be added continuously during
			the coming weeks. Please report any bugs or issues. Thanks and good luck
			with the grind! ✌
		</AlertComponent>

		<div class="filter-container">
			<FiltersComponent :options="filterOptions" :show-info="true">
				<template #info>
					<IconComponent name="question-circle" fill="white" v-tippy="{ placement: 'bottom' }"
						:content="'You only need to complete the number of base guns there are for each category to earn the Platinum camouflage challenge. For example, the Assault Rifles requires a total of 8 Gold camouflages to unlock the Platinum camouflage challenge for all weapons in that category. Press this icon to read more.'"
						@click="$router.push('/requirements')" />
					<div class="mobile">
						<IconComponent name="question-circle" fill="white"></IconComponent>
						<p>
							You only need to complete the number of base guns there are for each category to earn
							the Platinum camouflage challenge. For example, the Assault Rifles requires a total of
							8 Gold camouflages to unlock the Platinum camouflage challenge for all weapons in that
							category. Read more
							<router-link to="/requirements">here</router-link>.
						</p>
					</div>
				</template>
			</FiltersComponent>
			<div class="toggles">
				<FavoritesToggleComponent />
				<LayoutToggleComponent />
			</div>
		</div>

		<WeaponsComponent :weapons="filteredWeapons" :favorites="favorites" />

		<ProgressComponent :progress="polyProgress" data-camo="poly" label="Polyatomic progress"
			tooltip="Progress towards the Polyatomic camouflage">
			<template #modal-header>Polyatomic unlocked! 👏🥳</template>
			<template #modal-body>
				<p>
					Congratulations on finishing the Polyatomic camouflage grind! It's been a long ride! You first
					started tracking your grind here
					<b>{{ daysSinceStart }} days ago</b> on
					{{ new Date(getBeganGrind).toLocaleDateString('en-US') }}.
				</p>
			</template>
		</ProgressComponent>
		<ProgressComponent :progress="orionProgress" data-camo="orion" label="Orion progress"
			tooltip="Progress towards the Orion camouflage">
			<template #modal-header>Orion unlocked! 👏🥳</template>
			<template #modal-body>
				<p>
					Congratulations on finishing the Orion camouflage grind! It's been a long ride! You first
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
import firebase from 'firebase/app';
import 'firebase/auth';

import AlertComponent from '@/components/AlertComponent.vue'
import FiltersComponent from '@/components/FiltersComponent.vue'
import WeaponsComponent from '@/components/WeaponsComponent.vue'
import ProgressComponent from '@/components/ProgressComponent.vue'
import LayoutToggleComponent from '@/components/LayoutToggleComponent.vue'
import FavoritesToggleComponent from '@/components/FavoritesToggleComponent.vue'

export default {
	components: {
		AlertComponent,
		FiltersComponent,
		WeaponsComponent,
		ProgressComponent,
		LayoutToggleComponent,
		FavoritesToggleComponent,
	},

	created() {
		firebase.auth().onAuthStateChanged(user => {
			if (user) {
				// user is authenticated
				// console.log('signed in user', user)
			} else {
				// user is not authenticated
			}
		});
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
			]
		},

		filteredWeapons() {
			let filteredWeapons = this.weapons
			const { hideGold, hidePlatinum, hidePolyatomic, weaponCategory } = this.filters

			if (hideGold) {
				filteredWeapons = filteredWeapons.filter((weapon) => !weapon.progress['Gold'])
			}

			if (hidePlatinum) {
				filteredWeapons = filteredWeapons.filter((weapon) => !weapon.progress['Platinum'])
			}

			if (hidePolyatomic) {
				filteredWeapons = filteredWeapons.filter((weapon) => !weapon.progress['Polyatomic'])
			}

			if (weaponCategory && weaponCategory !== 'null') {
				filteredWeapons = filteredWeapons.filter((weapon) => weapon.category === weaponCategory)
			}

			return groupBy(filteredWeapons, (weapon) => weapon.category)
		},

		favorites() {
			if (!this.store) return []
			const favorites = this.store.getFavorites('weapons')
			return this.weapons.filter((weapon) => favorites.includes(weapon.name))
		},

		orionProgress() {
			// Set the amount of required weapons to complete the Orion camouflage
			const requiredWeapons = this.weapons.filter((weapon) => !weapon.dlc).length

			// Sort and filter out the weapons with the most progress
			const mostProgressedWeapons = this.weapons
				.map((weapon) => {
					let totalCamouflages = Object.keys(weapon.progress).length
					let completedCamouflages = Object.values(weapon.progress).reduce((a, b) => a + b, 0)

					return {
						...weapon,
						completed: Object.values(weapon.progress).reduce((a, b) => a + b, 0),
						completedPercentage: completedCamouflages / totalCamouflages,
					}
				})
				.sort((a, b) => b.completedPercentage - a.completedPercentage)
				.splice(0, requiredWeapons)

			// Count the amount of camouflages completed for the most progress weapons
			const totalCamouflagesCompleted = mostProgressedWeapons.reduce((a, b) => a + b.completed, 0)

			// Count the required amount of camouflages to complete the Orion camouflage
			const requiredCamouflages = mostProgressedWeapons.reduce((a, b) => {
				return a + Object.keys(b.progress).length
			}, 0)

			return roundToTwoDecimals((totalCamouflagesCompleted / requiredCamouflages) * 100)
		},
		polyProgress() {
			// Set the amount of required weapons to complete the Orion camouflage
			const requiredWeapons = this.weapons.filter((weapon) => !weapon.dlc).length



			// Sort and filter out the weapons with the most progress
			const mostProgressedWeapons = this.weapons
				.map((weapon) => {
					const mutableWeaponJson = JSON.stringify(weapon)
					const mutableWeapon = JSON.parse(mutableWeaponJson)
					delete mutableWeapon.progress.Polyatomic
					let totalCamouflages = Object.keys(mutableWeapon.progress).length
					let completedCamouflages = Object.values(mutableWeapon.progress).reduce((a, b) => a + b, 0)

					return {
						...mutableWeapon,
						completed: Object.values(mutableWeapon.progress).reduce((a, b) => a + b, 0),
						completedPercentage: completedCamouflages / totalCamouflages,
					}
				})
				.sort((a, b) => b.completedPercentage - a.completedPercentage)
				.splice(0, requiredWeapons)

			// const updatedWeaponProgress = mostProgressedWeapons.map((weapon))

			// Count the amount of camouflages completed for the most progress weapons
			const totalCamouflagesCompleted = mostProgressedWeapons.reduce((a, b) => a + b.completed, 0)

			// Count the required amount of camouflages to complete the Orion camouflage
			const requiredCamouflages = mostProgressedWeapons.reduce((a, b) => {
				return a + Object.keys(b.progress).length
			}, 0)

			return roundToTwoDecimals((totalCamouflagesCompleted / requiredCamouflages) * 100)
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

		.toggles {
			display: flex;
			justify-content: space-between;
			width: 100%;

			> :first-child {
				margin-right: 20px;
			}
		}
	}
}

::v-deep .filters-component {
	flex-grow: 1;
	margin-right: 15px;
}
</style>
