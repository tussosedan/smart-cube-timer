<template>
	<div>
		<v-btn
			v-if="replay"
			:href="replayUrl"
			color="success"
			target="_blank"
		>
			Replay on alg.cubing.net
		</v-btn>
		<v-layout wrap>
			<v-flex
				v-for="stage in stagesInfo"
				:id="stage.id"
				:key="stage.id"
				xs12
				lg4
				xl3
			>
				<v-card
					:dark="stage.dark"
					:color="stage.color"
					:class="stage.class"
				>
					<v-card-title>
						<div :style="{width: '100%'}">
							<h2 class="display-1 font-weight-bold text-xs-left">
								{{stage.name}}
								<v-chip
									v-for="info in stage.infos"
									:key="info.id"
									:color="info.color.startsWith('#') ? null : info.color"
									:text-color="info.textColor.startsWith('#') ? null : info.textColor"
									:style="{
										backgroundColor: info.color.startsWith('#') ? info.color : '',
										color: info.textColor.startsWith('#') ? info.textColor : '',
									}"
									class="stage-info-chip"
									small
								>
									<v-avatar
										v-if="info.avatar"
										:color="info.color.startsWith('#') ? null : info.color"
										:text-color="info.textColor.startsWith('#') ? null : info.textColor"
										class="darken-3"
									>
										{{info.avatar}}
									</v-avatar>
									{{info.text}}
								</v-chip>
							</h2>
							<v-layout class="stage-info headline ma-0">
								<strong :style="{color: 'inherit'}">
									{{stage.time}}
								</strong>
								<small
									v-if="stage.inspectionTime !== null"
									class="inspection-time"
								>
									<span class="time-info">
										{{stage.inspectionTime}}
									</span>
									<span class="time-spacer">
										/
									</span>
									<span class="time-info">
										{{stage.executionTime}}
									</span>
								</small>
								<v-spacer/>
								<div
									v-if="stage.moveCount !== null"
									class="subheading stage-info-right"
								>
									{{stage.moveCount}} turns
								</div>
								<div
									v-if="stage.speed !== null"
									class="subheading stage-info-right"
								>
									{{stage.speed}} tps
								</div>
							</v-layout>
							<div class="content text-xs-left">
								{{stage.sequenceText}}
							</div>
						</div>
					</v-card-title>
				</v-card>
			</v-flex>
		</v-layout>
	</div>
</template>

<script>
import {
	formatTime,
	getInspectionTime,
	getRelativeFaceFromFaces,
	getRotationNotation,
	getRotationNotationFromFaces,
	idealTextColor,
} from '~/lib/utils.js';
import assert from 'assert';
import config from '~/lib/config.js';
import get from 'lodash/get';
import qs from 'querystring';

export default {
	props: [
		'replay',
		'stages',
		'mode',
		'time',
		'cross',
		'isXcross',
		'oll',
		'isOll2Look',
		'pll',
		'pllLooks',
		'cll',
		'rouxBlock',
		'scrambleText',
	],
	data() {
		return {
		};
	},
	computed: {
		stagesInfo() {
			return config.stagesData[this.mode].map(({id, name, color, dark, showInspection}, index, stages) => {
				const stage = this.stages[id] || {time: null};
				const previousStage = index === 0 ? null : this.stages[stages[index - 1].id];
				const previousTime = previousStage ? previousStage.time : null;
				const deltaTime = (stage.time || this.time) - (previousTime === null ? 0 : previousTime);

				const isStageFinished = stage.time !== null && stage.sequence.length !== 0;

				const moveCount = isStageFinished ? stage.sequence.length : null;
				const speed = isStageFinished ? (moveCount / (deltaTime / 1000)).toFixed(2) : null;

				const downFace = (this.mode === 'roux' && previousStage && previousStage.orientation) ? previousStage.orientation.down : this.cross;
				const {inspection, execution} = (isStageFinished && showInspection)
					? getInspectionTime({stage, cross: downFace, previousTime})
					: {inspection: null, execution: null};

				const infos = [];
				if (id === 'unknown') {
					if (this.cross) {
						infos.push({
							text: `${config.faceColors[this.cross].name} Cross`,
							color: config.faceColors[this.cross].color,
							textColor: idealTextColor(config.faceColors[this.cross].color),
						});
					}

					if (this.isXcross) {
						infos.push({
							text: 'XCross',
							color: '#4A148C',
							textColor: idealTextColor('#4A148C'),
						});
					}
				}

				if (id === 'oll') {
					if (this.oll) {
						infos.push({
							text: this.oll.name,
							color: '#f5f5f5',
							textColor: idealTextColor('#f5f5f5'),
						});
					}
					if (this.isOll2Look) {
						infos.push({
							avatar: '2',
							text: 'Look',
							color: 'green',
							textColor: 'white',
						});
					}
				}

				if (id === 'pll') {
					if (this.pll) {
						infos.push({
							text: this.pll.name,
							color: '#FFEE58',
							textColor: idealTextColor('#FFEE58'),
						});
					}
					if (this.pllLooks.length > 1) {
						infos.push({
							avatar: this.pllLooks.length.toString(),
							text: 'Look',
							color: 'green',
							textColor: 'white',
						});
					}
				}

				if (id === 'cll') {
					if (this.cll) {
						infos.push({
							text: this.cll.name,
							color: '#FFEE58',
							textColor: idealTextColor('#FFEE58'),
						});
					}
				}

				let sequenceText = '--';

				if (stage.sequence) {
					if (stage.sequence.length === 0) {
						if (stage.time !== null && stage.sequence.length === 0) {
							sequenceText = '(Skipped)';
						}
					} else if (this.cross !== null) {
						sequenceText = stage.sequence.toString({cross: this.cross});

						if (id === 'unknown') {
							const rotationNotation = getRotationNotation({from: this.cross, to: 'D'});
							if (rotationNotation !== '') {
								sequenceText = `${rotationNotation} ${sequenceText}`;
							}
						}
						// eslint-disable-next-line no-negated-condition
					} else if (this.rouxBlock !== null) {
						if (id === 'unknown') {
							sequenceText = stage.sequence.toString({
								rouxBlock: {
									side: stage.orientation.left,
									bottomDirection: stage.orientation.down,
								},
								fixDirection: false,
							});
							const rotationNotation = getRotationNotationFromFaces({
								from: [stage.orientation.left, stage.orientation.down],
								to: ['L', 'D'],
							});
							if (rotationNotation !== '') {
								sequenceText = `${rotationNotation} ${sequenceText}`;
							}
						} else {
							const result = stage.sequence.toText({
								rouxBlock: {
									side: previousStage.orientation.left,
									bottomDirection: previousStage.orientation.down,
								},
								fixDirection: true,
							});

							sequenceText = result.text;

							if (stage.orientation !== null) {
								const [
									relativeDownFrom,
									relativeLeftFrom,
									relativeDownTo,
									relativeLeftTo,
								] = [
									result.orientation.down,
									result.orientation.left,
									stage.orientation.down,
									stage.orientation.left,
								].map((face) => (
									getRelativeFaceFromFaces(face, {
										from: [previousStage.orientation.left, previousStage.orientation.down],
										to: ['L', 'D'],
									})
								));

								assert(relativeLeftFrom === 'L');
								assert(relativeLeftTo === 'L');

								const rotationNotation = getRotationNotationFromFaces({
									from: ['L', relativeDownFrom],
									to: ['L', relativeDownTo],
								});

								if (rotationNotation !== '') {
									sequenceText += ` ${rotationNotation}`;
								}
							}
						}
					} else {
						sequenceText = stage.sequence.toString();
					}
				}

				return {
					id,
					name,
					infos,
					color,
					dark,
					sequenceText,
					time: formatTime(deltaTime),
					moveCount,
					speed,
					inspectionTime: inspection && formatTime(inspection),
					executionTime: execution && formatTime(execution),
				};
			});
		},
		replayUrl() {
			const lines = this.stagesInfo.map((stageInfo) => {
				const stage = this.stages[stageInfo.id];
				if (!stage.sequence || stage.sequence.length === 0) {
					return null;
				}
				const stageDatum = get(config.stagesData, [this.mode], []).find(({id}) => stage.id === id);
				const stageName = stageDatum ? ` // ${stageDatum.name}` : '';
				const line = `${stageInfo.sequenceText}${stageName}`;
				return line;
			}).filter((line) => line !== null);

			const alg = lines.join('\n');
			const setup = this.scrambleText;
			return `https://alg.cubing.net/?${qs.encode({alg, setup})}`;
		},
	},
};
</script>

<style>
	.stage-info {
		line-height: 1 !important;
	}

	.stage-info-chip {
		z-index: 0;
	}

	.inspection-time {
		font-size: 70%;
		opacity: 0.7;
		display: flex;
		line-height: 1.5em;
		margin-left: 0.5em;
	}

	.stage-info-right {
		margin-left: 0.6rem;
	}

	.time-info {
		position: relative;
		align-self: flex-end;
	}

	.time-spacer {
		width: 0.8em;
		text-align: center;
		align-self: flex-end;
	}
</style>
