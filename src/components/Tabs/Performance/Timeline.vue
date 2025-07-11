<template>
	<div class="timeline" :class="{ 'show-details': showDetails }" ref="timeline">
		<details-table title="Timeline" icon="pie-chart" :columns="columns" :items="presentedEvents" :filter="filter" :no-table-head="! showDetails" filter-example="database query duration:>50" :per-page="100">
			<template v-slot:toolbar="{ filter }">
				<div class="header-group">
                    <label class="header-toggle" title="Hide native Magento 2 timeline events. Only events <5ms will be hidden.">
                        <input type="checkbox" v-model="hideNative">
                        Hide "spam" native events
                    </label>

                    <label class="header-toggle">
						<input type="checkbox" v-model="condense">
						Condense
					</label>
				</div>

				<div class="header-group header-group-icons" v-if="availableTags.length">
					<a v-for="tag in availableTags" href="#" class="header-item" :class="{ 'active': hiddenTags && ! hiddenTags.includes(tag.tag) }" :title="tag.title" @click="toggleTag(tag.tag)">
						<icon :name="tag.icon"></icon>
					</a>

					<div class="header-search">
						<input type="search" v-model="filter.input" placeholder="Search...">
						<icon name="search"></icon>
					</div>

					<a href="#" title="Toggle details" class="header-item" :class="{ 'active': showDetails }" @click.prevent="toggleDetails">
						<icon name="list"></icon>
					</a>
				</div>
			</template>
			<template v-slot:body="{ items }">
				<tr v-for="group, index in items">
					<td class="timeline-chart">
						<div class="chart-event-group popover-container" :style="group.groupStyle" @click="showPopover(index)">
							<div class="group-label" :class="group.labelClass" :style="group.labelStyle">
								<span class="label-tags" v-if="group.tags">
									<span v-for="tag in resolveTags(group.tags)">
										<icon :name="tag.icon" :title="tag.title"></icon>
									</span>
								</span>
								{{ group.name.replace('/ns', '') }}
								<span v-if="! group.condensed">{{formatTiming(group.duration)}}</span>
							</div>

							<div class="group-event" :class="event.eventClass" :style="event.eventStyle" v-for='event, index in group.events'>
								<div class="event-bar">
									<div class="bar-light" :style="section.style" v-for="section in event.childrenSections"></div>
								</div>
							</div>

							<popover class="timeline-popover" ref="popovers">
								<div class="popover-event" :class="event.eventClass" v-for="event in group.events">
									<div class="event-header">
										<h1>{{event.name}}</h1>

										<div class="header-tags">
											<span v-for="tag in resolveTags(event.tags)">
												<icon :name="tag.icon" :title="tag.title"></icon>
											</span>
										</div>
									</div>

									<div class="event-description" v-if="event.description != event.name">
										<highlighted-code v-if="event.tags && event.tags.indexOf('databaseQueries') > -1" language="sql" :code="event.description"></highlighted-code>
										<div v-else>{{event.description}}</div>
									</div>

									<div class="event-timings">
										<div class="timings-timing timing-total">
											<div class="timing-value">
												{{formatTiming(event.duration)}}
											</div>
											<div class="timing-label">
												Total
											</div>
										</div>

										<div class="timings-timing timing-self">
											<div class="timing-value">
												{{formatTiming(event.durationSelf)}}
											</div>
											<div class="timing-label">
												Self
											</div>
										</div>

										<div class="timings-timing timing-children">
											<div class="timing-value">
												{{formatTiming(event.durationChildren, 'ms', '–')}}
											</div>
											<div class="timing-label">
												Children
											</div>
										</div>
									</div>
								</div>
							</popover>
						</div>
					</td>
					<td class="timeline-description">
						<slot name="table-description" :item="group">
							<div class="description-content">
								<span class="description-tags" v-if="group.tags && group.tags.length">
									<span v-for="tag in resolveTags(group.tags)">
										<icon :name="tag.icon" :title="tag.title"></icon>
									</span>
								</span>
								<highlighted-code v-if="group.tags && group.tags.indexOf('databaseQueries') > -1" language="sql" :code="group.description"></highlighted-code>
								<div v-else>{{group.description}}</div>
							</div>
						</slot>
					</td>
					<td class="timeline-timing timing-total">{{group.duration|formatTiming}}</td>
					<td class="timeline-timing">{{group.durationSelf|formatTiming}}</td>
					<td class="timeline-timing">{{group.durationChildren|formatTiming('ms', group.condensed ? '' : '–')}}</td>
				</tr>

				<tr class="timeline-size-monitor">
					<td class="timeline-graph" ref="timelineChart"></td>
					<td class="timeline-description"></td>
					<td class="timeline-timing"></td>
					<td class="timeline-timing"></td>
					<td class="timeline-timing"></td>
				</tr>
			</template>
		</details-table>
	</div>
</template>

<script>
import DetailsTable from '../../UI/DetailsTable'
import HighlightedCode from '../../UI/HighlightedCode'
import Popover from '../../UI/Popover'

import createFilter from '../../../features/filter'

import debounce from 'just-debounce-it'

export default {
	name: 'Timeline',
	components: { DetailsTable, HighlightedCode, Popover },
	props: { name: {}, timeline: {}, tags: { default: () => [] } },
	data: () => ({
		condense: undefined,
		showDetails: false,

		hiddenTags: undefined,
		presentedEvents: [],

		filter: createFilter([
			{ tag: 'duration', type: 'number' }
		], item => item.description),

        hideNative: false
	}),
	computed: {
		availableTags() {
			return this.tags.filter(tag => this.timeline.events.find(item => item.tags && item.tags.includes(tag.tag)))
		},

		columns() {
			return this.showDetails ? [
				{ name: ' ', sortBy: 'start', class: 'timeline-chart' },
				{ name: 'Event', sortBy: 'name', class: 'timeline-description' },
				{ name: 'Total', sortBy: 'duration', class: 'timeline-timing' },
				{ name: 'Self', sortBy: 'durationSelf', class: 'timeline-timing' },
				{ name: 'Child', sortBy: 'durationChild', class: 'timeline-timing' }
			] : []
		}
	},
	methods: {
		toggleTag(tag) {
			if (this.hiddenTags.includes(tag)) {
				this.hiddenTags = this.hiddenTags.filter(t => t != tag)
			} else {
				this.hiddenTags.push(tag)
			}
            this.refreshEvents()
		},

		resolveTags(tags) {
			return tags.map(tag => this.tags.find(t => t.tag == tag)).filter(tag => tag)
		},

		toggleDetails() {
			this.showDetails = ! this.showDetails
		},

		showPopover(index) {
			this.$refs.popovers[index].toggle()
		},

		async refreshEvents(e) {
			if (! this.timeline || ! this.$refs.timelineChart) return

			// wait until all changes are rendered as we depend on current column size
			await this.$nextTick()

			let timelineWidth = this.$refs.timelineChart.offsetWidth - 16

			// don't bother presenting timelines which are not visible
			if (timelineWidth <= 0) return

			let timeline = this.timeline.filter(this.filter, this.hiddenTags)

            if (this.hideNative) {
                timeline.events = timeline.events.filter((event) => {

                    return !event.name.endsWith('/ns');
                });
            }

			if (this.condense) timeline = timeline.condense()

			this.presentedEvents = timeline.present(timelineWidth)
		},

		// observe resizes of the timeline container, track last width to refresh only on width updates as resize
		// observer also triggers on height changes
		observeResizing() {
			let lastWidth = this.$refs.timeline.offsetWidth

			this.resizeObserver = new ResizeObserver(debounce(([ entry ]) => {
				if (lastWidth != entry.contentRect.width) {
					lastWidth = entry.contentRect.width
					this.refreshEvents(entry)
				}
			}, 10))

			this.resizeObserver.observe(this.$refs.timeline)
		},

		formatTiming(value, unit = 'ms', placeholder = '') {
			if (value === null || value === undefined) return placeholder

			return (value <= 0 || value > 1) ? `${Math.round(value)} ${unit}` : `<1 ${unit}`
		}
	},
	watch: {
        hideNative(val, old) {
            this.refreshEvents();
        },

		condense(val, old) {
			// skip initial assignment from settings to avoid multiple refreshes on mounted
			if (old === undefined) return

			this.refreshEvents()

			this.$settings.global.timelineCondensed[this.name] = this.condense
			this.$settings.save()
		},

		hiddenTags(val, old) {
			// skip initial assignment from settings to avoid multiple refreshes on mounted
			if (old === undefined) return

			this.refreshEvents()

			this.$settings.global.timelineHiddenTags[this.name] = this.hiddenTags
			this.$settings.save()
		},

		showDetails() {
			this.refreshEvents()
		},

		timeline() {
			this.refreshEvents()
		}
	},
	async mounted() {
		this.condense = this.$settings.global.timelineCondensed[this.name] || false
		this.hiddenTags = this.$settings.global.timelineHiddenTags[this.name] || []

		await this.refreshEvents()

		this.observeResizing()
	}
}
</script>

<style lang="scss">
@use '../../../mixins' as *;
@use 'sass:map';

$timeline-colors-light: (
	"blue":   ( normal: hsl(212deg 89% 55%), alternative: hsl(212deg 88% 70%) ),
	"red":    ( normal: hsl(359deg 57% 55%), alternative: hsl(359deg 57% 70%) ),
	"green":  ( normal: hsl(109deg 52% 45%), alternative: hsl(109deg 52% 60%) ),
	"purple": ( normal: hsl(273deg 57% 55%), alternative: hsl(273deg 57% 70%) ),
	"grey":   ( normal: hsl(240deg 5% 30%),  alternative: hsl(240deg 5% 65%) ),
    "pinkp":  ( normal: hsl(319deg 52% 60%), alternative: hsl(319deg 52% 60%) )
);

$timeline-colors-dark: (
	"blue":   ( normal: hsl(212deg 83% 60%), alternative: hsl(212deg 84% 45%) ),
	"red":    ( normal: hsl(359deg 52% 60%), alternative: hsl(359deg 52% 45%) ),
	"green":  ( normal: hsl(109deg 47% 50%), alternative: hsl(109deg 47% 35%) ),
	"purple": ( normal: hsl(273deg 52% 60%), alternative: hsl(273deg 52% 45%) ),
	"grey":   ( normal: hsl(240deg 5% 50%),  alternative: hsl(240deg 5% 30%) ),
    "pinkp":  ( normal: hsl(319deg 52% 60%), alternative: hsl(319deg 52% 60%) )
);

.timeline {
	.table-header {
		padding-bottom: 10px;
	}

	table {
		table-layout: fixed;
	}

	.timeline-timing, .timeline-description {
		display: none;
	}

	.timeline-description {
		.description-content {
			align-items: center;
			display: flex;
			word-break: break-word;
		}

		.description-tags {
			font-size: 95%;
			margin-right: 5px;
			opacity: 0.7;
		}
	}

	.timeline-timing {
		padding-right: 10px !important;
		text-align: right;
		width: 80px;

		&.timing-total {
			font-weight: 600;
		}
	}

	.timeline-size-monitor {
		td {
			padding-bottom: 0;
			padding-top: 0;
		}
	}

	.timeline-chart {
		padding: 8px !important;

		.chart-event-group {
			cursor: pointer;
			height: 16px;
			position: relative;

			.group-label {
				color: hsl(206deg 30% 30%);
				font-size: 12px;
				line-height: 16px;
				overflow: hidden;
				padding: 0 6px;
				position: absolute;
				text-overflow: ellipsis;
				white-space: nowrap;
				z-index: 100;

				.label-tags {
					font-size: 95%;
					opacity: 0.8;
				}

				&.inside {
					color: #fff !important;
					text-align: center;
					width: 100% !important;
				}

				&.before {
					right: 100%;
					text-align: right;
				}

				&.after {
					left: 100%;
				}

				@each $color, $values in $timeline-colors-light {
					&.#{$color} { color: map.get($values, 'normal'); }
				}

				@include dark {
					@each $color, $values in $timeline-colors-dark {
						&.#{$color} { color: map.get($values, 'normal'); }
					}
				}
			}
		}

		.group-event {
			height: 100%;
			position: absolute;

			.event-bar {
				background: rgb(66, 149, 197);
				border-radius: 3px;
				height: 100%;
				left: 0;
				overflow: hidden;
				position: absolute;
				top: 0;
				width: 100%;

				@include dark {
					background: rgb(100, 157, 202);
				}

				.bar-light {
					height: 100%;
					position: absolute;
				}
			}

			@each $color, $values in $timeline-colors-light {
				&.#{$color} {
					.event-bar { background: map.get($values, 'normal'); }
					.event-bar .bar-light { background: map.get($values, 'alternative'); }
				}
			}

			@include dark {
				@each $color, $values in $timeline-colors-dark {
					&.#{$color} {
						.event-bar { background: map.get($values, 'normal'); }
						.event-bar .bar-light { background: map.get($values, 'alternative'); }
					}
				}
			}
		}
	}

	.timeline-popover {
		.popover-content {
			padding-bottom: 0;
		}

		.popover-event {
			border-bottom: 1px solid rgba(#333, 0.1);
			margin-bottom: 5px;

			@include dark { border-color: rgba(#eee, 0.1); }

			&:last-child {
				border-bottom: 0;
				margin-bottom: 0;
			}

			.event-header {
				display: flex;
				padding: 12px 12px 14px;

				h1 {
					flex: 1;
					font-size: 110%;
					margin: 0;
					text-align: left;
				}

				.header-tags {
					color: #777;
				}
			}

			.event-description {
				background: rgba(#333, 0.03);
				border-top: 1px solid rgba(#333, 0.1);
				padding: 12px;
				text-align: left;

				@include dark {
					background: rgba(#ddd, 0.03);
					border-color: rgba(#eee, 0.1);
				}
			}

			.event-timings {
				border-top: 1px solid rgba(#333, 0.1);
				display: flex;

				@include dark { border-color: rgba(#eee, 0.1); }

				.timings-timing {
					border-right: 1px solid rgba(#333, 0.1);
					flex: 1;
					padding: 10px 0;

					@include dark { border-color: rgba(#eee, 0.1); }

					&:last-child { border-right: 0; }

					.timing-value {
						font-size: 110%;
					}

					.timing-label {
						color: #777;
						margin-top: 5px;
						font-size: 95%;

						&:before {
							content: '';
							display: inline-block;
							border-radius: 5px;
							background: rgb(66, 149, 197);
							width: 14px;
							height: 8px;
							vertical-align: 0px;

							@include dark { background: rgb(100, 157, 202); }
						}
					}

					&.timing-total {
						.timing-value {
							font-weight: 600;
						}

						.timing-label:before {
							background: linear-gradient(to right, rgb(66, 149, 197) 50%, rgb(120, 177, 222) 50%);

							@include dark { background: linear-gradient(to right, rgb(100, 157, 202) 50%, rgb(46, 129, 177) 50%); }
						}
					}

					&.timing-children {
						.timing-label:before {
							background: rgb(120, 177, 222);

							@include dark { background: rgb(46, 129, 177); }
						}
					}
				}
			}

			@each $color, $values in $timeline-colors-light {
				&.#{$color} {
					.timings-timing {
						.timing-label:before { background: map.get($values, 'normal'); }
						&.timing-total .timing-label:before { background: linear-gradient(to right, map.get($values, 'normal') 50%, map.get($values, 'alternative') 50%); }
						&.timing-children .timing-label:before { background: map.get($values, 'alternative'); }
					}
				}
			}

			@include dark {
				@each $color, $values in $timeline-colors-dark {
					&.#{$color} {
						.timings-timing {
							.timing-label:before { background: map.get($values, 'normal'); }
							&.timing-total .timing-label:before { background: linear-gradient(to right, map.get($values, 'normal') 50%, map.get($values, 'alternative') 50%); }
							&.timing-children .timing-label:before { background: map.get($values, 'alternative'); }
						}
					}
				}
			}
		}
	}

	&.show-details {
		.timeline-timing, .timeline-description {
			display: table-cell;
		}

		.timeline-chart {
			width: 20%;

			.chart-event-group {
				.group-label {
					display: none;
				}
			}
		}
	}
}
</style>
