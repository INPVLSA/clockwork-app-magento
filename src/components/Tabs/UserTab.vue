<template>
	<div v-if="active">
		<div v-for="section, sectionIndex in userTab.sections" :key="`${$request.id}-${sectionIndex}`">
			<div v-if="section.showAs == 'counters'" class="counters-row">
				<div v-for="item, index in section.data" class="counter" :key="`${$request.id}-${index}`">
					<div class="counter-value">{{item.value}}</div>
					<div class="counter-title">{{item.key}}</div>
				</div>
			</div>

			<details-table :title="section.title" :columns="section.data[0].map(item => item.key)" :items="section.data" :filter="filters[sectionIndex]" v-if="section.showAs == 'table'">
				<template v-slot:body="{ items }">
					<tr v-for="item in items" :class="item[0].value === 'sql' ? (item[1].is_sql = true) : ''">
						<td v-for="item in item">
							<pretty-print :data="item.value" v-if="!item.is_sql"></pretty-print>

                            <div class="database-query" v-else>
                                <div class="database-query-content">
                                    <highlighted-code language="sql" :code="item.value"></highlighted-code>
                                </div>
                            </div>
						</td>
					</tr>
				</template>
			</details-table>
		</div>
	</div>
</template>

<script>
import DetailsTable from '../UI/DetailsTable'
import PrettyPrint from '../UI/PrettyPrint'

import createFilter from '../../features/filter'
import HighlightedCode from "../UI/HighlightedCode.vue";
import StackTrace from "../UI/StackTrace.vue";

export default {
	name: 'UserTab',
	components: {StackTrace, HighlightedCode, DetailsTable, PrettyPrint },
	props: [ 'active', 'userTab' ],
	data: () => ({
		filters: []
	}),
	watch: {
		userTab: {
			handler(val) {
				this.filters = val.sections.map(section => {
					if (section.showAs == 'table') {
						return createFilter(section.data[0].map(item => ({ tag: item.key })))
					}
				})
			},
			immediate: true
		}
	}
}
</script>
