<template>
	<div
		class="vplayer"
		:class="{full:full,audio:audio}"
		tabindex="1"
		@keydown.space.stop.prevent="togglePlay"
		@keydown.left.stop.prevent="seek(-5)"
		@keydown.right.stop.prevent="seek(5)"
	>
		<div
			class="controls"
			:class="{cleanmode:bottomHide}"
			@mouseenter="maskMouseEnter"
			@mouseleave="maskMouseLeave"
			@mousemove="maskMouseMove"
			@click.stop.prevent="togglePlayOnClick"
			@dblclick.stop.prevent="toggleFull"
		>
			<div class="seeking" v-if="video.seeking"></div>
			<div class="errored" v-if="video.error">
				<div class="errmsg">
					<div class="hidebtn" @click.stop="video.error=null">+</div>
					<p>{{video.error.stack||video.error}}</p>
					<p v-if="typeof video.error!='string'">播放出错,请刷新重试</p>
				</div>
			</div>
			<div class="poster" v-if="!video.duration || audio" :style="poster"></div>
			<div class="bottom" :class="{hide:bottomHide}" @click.stop.prevent>
				<div v-if="audio" class="a-title">{{playerInfo.title}}</div>
				<div class="progress-bar">
					<div
						ref="progressLine"
						class="progress-line"
						@click.stop="seekTo"
						@mousemove="showCurrTime"
						@mouseenter="tiptime.show = true"
						@mouseleave="tiptime.show = false"
					>
						<div ref="times" class="times" :style="tipStyle" v-show="tiptime.show" v-text="tiptime.v"></div>
						<canvas ref="loadbar" class="loaded"></canvas>
						<div class="played" :style="playedStyle"></div>
						<div class="dot" :style="dotStyle"></div>
					</div>
				</div>
				<div class="btns">
					<div class="play-pause">
						<svg
							v-if="paused"
							@click.stop.prevent="playVideo"
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 20 20"
						>
							<path
								d="M17.28,9l-12.75-8A1,1,0,0,0,3,1.91V17.85a1,1,0,0,0,1.53.85l12.75-8A1,1,0,0,0,17.28,9Z"
							/>
						</svg>
						<svg
							v-else
							@click.stop.prevent="pauseVideo"
							xmlns="http://www.w3.org/2000/svg"
							viewBox="0 0 20 20"
						>
							<rect x="3" y="1" width="3" height="18" rx="1" ry="1" />
							<rect x="14" y="1" width="3" height="18" rx="1" ry="1" />
						</svg>
					</div>
					<div class="time-duration">
						<span class="curr">{{video.currentTime | timeformat}}</span>
						<span class="padding">/</span>
						<span class="total">{{video.duration | timeformat}}</span>
					</div>

					<div class="full-screen" v-if="!audio">
						<svg v-if="small" @click="toFull" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
							<path d="M3,9A1,1,0,0,1,2,8V4A2,2,0,0,1,4,2H8A1,1,0,0,1,8,4H4V8A1,1,0,0,1,3,9Z" />
							<path d="M17,9a1,1,0,0,1-1-1V4H12a1,1,0,0,1,0-2h4a2,2,0,0,1,2,2V8A1,1,0,0,1,17,9Z" />
							<path d="M8,18H4a2,2,0,0,1-2-2V12a1,1,0,0,1,2,0v4H8a1,1,0,0,1,0,2Z" />
							<path d="M16,18H12a1,1,0,0,1,0-2h4V12a1,1,0,0,1,2,0v4A2,2,0,0,1,16,18Z" />
						</svg>
						<svg v-else @click="exitFull" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
							<path
								d="M7.75,1A1.13,1.13,0,0,1,8.88,2.12v4.5A2.26,2.26,0,0,1,6.62,8.88H2.12a1.13,1.13,0,0,1,0-2.26h4.5V2.12A1.13,1.13,0,0,1,7.75,1Z"
							/>
							<path
								d="M12.25,1a1.13,1.13,0,0,1,1.13,1.12v4.5h4.5a1.13,1.13,0,0,1,0,2.26h-4.5a2.26,2.26,0,0,1-2.26-2.26V2.12A1.13,1.13,0,0,1,12.25,1Z"
							/>
							<path
								d="M2.12,11.12h4.5a2.26,2.26,0,0,1,2.26,2.26v4.5a1.13,1.13,0,0,1-2.26,0v-4.5H2.12a1.13,1.13,0,0,1,0-2.26Z"
							/>
							<path
								d="M13.38,11.12h4.5a1.13,1.13,0,0,1,0,2.26h-4.5v4.5a1.13,1.13,0,0,1-2.26,0v-4.5A2.26,2.26,0,0,1,13.38,11.12Z"
							/>
						</svg>
					</div>
					<div class="fake-full" v-if="!audio">
						<svg @click="full=!full" viewBox="0 0 1024 1024" xmlns="http://www.w3.org/2000/svg">
							<path
								d="M957.44 962.56H61.44c-25.6 0-40.96-20.48-40.96-40.96V128c0-25.6 20.48-40.96 40.96-40.96h896c25.6 0 40.96 20.48 40.96 40.96v788.48c0 25.6-15.36 46.08-40.96 46.08zM107.52 875.52h808.96V174.08H107.52v701.44z"
								p-id="3558"
							/>
							<path
								d="M424.96 783.36H230.4c-20.48 0-35.84-15.36-35.84-35.84v-189.44h76.8v153.6h153.6v71.68zM271.36 491.52H194.56V296.96c0-20.48 15.36-35.84 35.84-35.84h189.44v76.8h-153.6l5.12 153.6zM824.32 486.4h-76.8v-153.6h-148.48V261.12h184.32c20.48 0 35.84 15.36 35.84 35.84l5.12 189.44zM788.48 783.36h-184.32v-76.8h148.48v-153.6h76.8v189.44c-5.12 25.6-20.48 40.96-40.96 40.96z"
							/>
						</svg>
					</div>
					<div class="audio-list" v-if="audio">
						<svg
							@click="$emit('list')"
							xmlns="http://www.w3.org/2000/svg"
							version="1.1"
							viewBox="0 0 22 32"
						>
							<path
								d="M20.8 14.4q0.704 0 1.152 0.48t0.448 1.12-0.48 1.12-1.12 0.48h-19.2q-0.64 0-1.12-0.48t-0.48-1.12 0.448-1.12 1.152-0.48h19.2zM1.6 11.2q-0.64 0-1.12-0.48t-0.48-1.12 0.448-1.12 1.152-0.48h19.2q0.704 0 1.152 0.48t0.448 1.12-0.48 1.12-1.12 0.48h-19.2zM20.8 20.8q0.704 0 1.152 0.48t0.448 1.12-0.48 1.12-1.12 0.48h-19.2q-0.64 0-1.12-0.48t-0.48-1.12 0.448-1.12 1.152-0.48h19.2z"
							/>
						</svg>
					</div>
					<div class="volume-wrap">
						<div class="volbtn">
							<svg v-if="muted" @click="muteoff" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
								<path
									d="M15.9,4.19,14.79,5.85A5,5,0,0,1,16,13a5.21,5.21,0,0,0,1.45,1.4A7,7,0,0,0,15.9,4.19Z"
								/>
								<path d="M8.53,5.2,11,3.14V7.67l2,2V3.14A2,2,0,0,0,9.72,1.6L7.1,3.78Z" />
								<path
									d="M11,13.33v3.53h0L6.43,13H3V7H4.67l-2-2H2A1,1,0,0,0,1,6v8a1,1,0,0,0,1,1H5.7l4,3.38a2,2,0,0,0,2.29.2,2.1,2.1,0,0,0,1-1.85v-1.4Z"
								/>
								<rect
									x="8.97"
									y="-2.23"
									width="2"
									height="23.41"
									rx="1"
									ry="1"
									transform="translate(-3.78 9.83) rotate(-45)"
								/>
							</svg>
							<svg v-else @click="muteon" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
								<path
									d="M11,18.85a2,2,0,0,1-1.28-.47L5.7,15H2a1,1,0,0,1-1-1V6A1,1,0,0,1,2,5H5.64l4-3.34a2.09,2.09,0,0,1,1.94-.44A2,2,0,0,1,13,3.14V16.76a2.09,2.09,0,0,1-1,1.83A2,2,0,0,1,11,18.85ZM3,13H6.43L11,16.86V3.14L6.36,7H3Z"
								/>
								<path d="M15.87,15.83l-1.1-1.66a5,5,0,0,0,0-8.32L15.9,4.19a7,7,0,0,1,0,11.64Z" />
							</svg>
						</div>
						<div class="vol-bar-wrap">
							<div class="vol-bar">
								<div class="vol-total" @click.stop.prevent="volPosTotal"></div>
								<div
									class="vol-num"
									ref="numdiv"
									:style="{height: (muted?0: vol.v)+'px'}"
									@click.stop.prevent="volPosNum"
								></div>
								<div
									class="vol-dot"
									@mousedown.stop.prevent="dotDown"
									:style="{bottom:((muted?0: vol.v)+4.5)+'px'}"
								></div>
							</div>
						</div>
					</div>
					<div class="settings" v-if="!audio">
						<svg viewBox="0 0 1024 1024" xmlns="http://www.w3.org/2000/svg">
							<path
								d="M512.458441 673.454182c-88.570221 0-160.628374-72.057129-160.628374-160.628374 0-88.570221 72.058153-160.627351 160.628374-160.627351 88.571245 0 160.628374 72.057129 160.628374 160.627351C673.086815 601.397053 601.029686 673.454182 512.458441 673.454182zM512.458441 438.157201c-41.173748 0-74.670653 33.496905-74.670653 74.66963s33.496905 74.670653 74.670653 74.670653c41.172725 0 74.670653-33.496905 74.670653-74.670653S553.631166 438.157201 512.458441 438.157201z"
							/>
							<path
								d="M599.87949 921.606958 431.591669 921.606958l-42.97886-42.97886c0 0-0.013303 0.303922 0.01228 0.874927-0.12689-2.835581-2.016937-28.167526-29.203112-44.225247-10.321066-6.094812-25.207102-9.317203-43.049468-9.317203-25.173332 0-47.258327 6.391571-47.470151 6.456039l-48.50369-17.579377-100.624768-153.809061 20.519336-63.637366c0 0-0.358157 0.115634-0.99977 0.405229 3.25309-1.464351 31.754215-15.708773 30.728862-64.262606-0.450255-21.335934-7.045463-38.929638-19.60143-52.292993-10.07445-10.722202-20.62883-15.164375-20.73423-15.208377 0.446162 0.185218 0.688685 0.265036 0.688685 0.265036l-21.355377-63.289442 116.239398-183.535083 59.487859-13.196557-0.232291-0.388856c6.250354 3.885494 26.466792 13.731746 44.718481 13.731746 0.001023 0 0.00307 0 0.004093 0 7.212262 0 12.825097-1.272993 17.666358-4.52506 26.491351-17.793248 40.22412-65.424058 42.450834-77.238129l42.234917-34.894741 164.81779 0 42.97886 42.97886c0 0 0.007163-0.216941-0.00614-0.633427 0.051165 1.644453 1.705851 40.543392 39.98262 63.682391 11.289114 6.824429 22.437012 10.141988 34.079166 10.141988 18.544355 0 32.18912-8.736988 32.32522-8.826016-0.396019 0.257873-0.595564 0.408299-0.595564 0.408299l61.657268 13.166881 109.300358 183.586249-21.3871 62.056358c0 0 0.207731-0.073678 0.593518-0.237407-0.110517 0.047072-11.269671 4.865821-21.73707 15.923668-12.840446 13.5629-19.081591 30.211069-19.081591 50.894134 0 21.596877 5.727445 38.319747 17.022699 49.701982 5.942339 5.989411 12.264325 9.067516 12.31242 9.083889l22.081924 65.220421L800.10572 813.939797l-54.772464 14.009062c0 0 0.138146 0.072655 0.408299 0.201591-0.194428-0.092098-15.521508-7.281847-34.561144-7.281847-10.602476 0-20.354584 2.279926-28.981055 6.77531-25.555026 13.31833-37.669948 47.424102-40.181141 59.442834l0.061398-0.308015L599.87949 921.606958zM465.970664 835.649238l102.82283 0c1.449002-3.434216 3.065825-7.011694 4.864798-10.676154 16.266475-33.137725 40.061414-58.573023 68.814272-73.557296 21.016663-10.952446 44.132126-16.505929 68.706825-16.505929 13.373589 0 25.677822 1.682315 36.227086 3.970428l63.435775-92.207052c-0.106424-0.106424-0.211824-0.212848-0.319272-0.320295-19.144012-19.292392-41.964764-54.080709-41.964764-110.248963 0-44.672432 16.146749-84.142375 46.696532-114.14162 2.033309-1.997494 4.065596-3.88754 6.084579-5.677303l-69.272713-116.354008c-11.212366 3.01466-24.240077 5.158487-38.623669 5.158487-27.381627 0-53.80851-7.582699-78.54796-22.538319-33.208333-20.075221-57.81987-49.625234-71.169923-85.453231-1.040702-2.791579-1.976004-5.52176-2.817162-8.178263l-96.851838 0c-10.69355 30.209022-31.501458 72.193229-69.223594 97.53029-19.132756 12.851703-41.203424 19.126616-65.594951 19.126616-0.004093 0-0.007163 0-0.011256 0-18.886139 0-36.630269-3.780093-51.361785-8.754385l-73.780377 116.615974c1.50426 1.364067 3.016707 2.849908 4.52813 4.332678 20.935822 20.544919 46.145993 57.011458 47.348378 113.917516 1.195222 56.623625-20.643156 92.76987-39.174208 113.110127-0.960885 1.055029-1.925862 2.089591-2.888793 3.085268l62.328557 95.2739c11.501962-1.76418 25.295106-3.157923 40.145325-3.157923 33.683147 0 62.873979 7.154957 86.762039 21.263279C437.326277 781.459035 456.316793 810.048163 465.970664 835.649238z"
							/>
						</svg>
						<div class="settings-content">
							<ul class="settings-list">
								<li class="settings-item">
									<div class="s-value">播放速度</div>
									<ul class="s-opt" :rel="video.playbackRate">
										<li
											v-for="(item,index) in speeds"
											:class="{active:item==video.playbackRate}"
											:key="index"
											@click="playSpeed(item)"
										>{{item}}倍</li>
									</ul>
								</li>
								<li class="settings-item">
									<div class="s-value">循环播放</div>
									<ul class="s-opt">
										<li :class="{active:video.cycle==1}" @click="cyclePlay(1)">开</li>
										<li :class="{active:video.cycle==0}" @click="cyclePlay(0)">关</li>
									</ul>
								</li>
							</ul>
						</div>
					</div>
					<div class="quality-list" v-if="!audio">
						<div class="curr-q" v-text="currq.quality"></div>
						<ul class="qitem-list">
							<li
								v-for="item in qlist"
								class="qitem"
								:class="{active:currq.itag==item.itag}"
								:key="item.itag"
								@click="switchQuality(item,true)"
								v-text="item.quality"
							></li>
						</ul>
					</div>
				</div>
			</div>
		</div>
		<video
			ref="video"
			v-if="v"
			class="video"
			:poster="posterImg"
			webkit-playsinline="true"
			playsinline="true"
			preload="meta"
			x-webkit-airplay="allow"
			x5-video-player-type="h5-page"
		></video>
	</div>
</template>

<script>
/**
 * 

18  360p  mp4 音视频
22  720p  mp4 音视频


133 240p  mp4 视频
134 360p  mp4 视频
135 480p  mp4 视频
136 720p  mp4 视频
137 1080p mp4 视频

160 144p  mp4 视频
597 144p  mp4 视频(小)

298 720p60   mp4 视频(avc1.4d4020)
299 1080p60  mp4 视频(avc1.64002a)



394 144p    mp4 视频(av01.0.00M.08)
395 240p    mp4 视频(av01.0.00M.08)
396 360p    mp4 视频(av01.0.00M.08)
397 480p    mp4 视频(av01.0.00M.08)
398 720p60  mp4 视频(av01.0.00M.08)
399 1080p60 mp4 视频(av01.0.00M.08)
400 1440p60 mp4 视频(av01.0.00M.08)
401 2160p60 mp4 视频(av01.0.00M.08)
402 4320p60 mp4 视频(av01.0.17M.08)
571 4320p60 mp4 视频(av01.0.17M.08)

140 tiny  mp4 音频
599 tiny  mp4 音频(小)





242 240p  webm 视频
243 360p  webm 视频
244 480p  webm 视频
247 720p  webm 视频
248 1080p webm 视频
271 1440p webm 视频
313 2160p webm 视频

302 720p60  webm 视频
303 1080p60 webm 视频
308 1440p60 webm 视频
315 2160p60 webm 视频

278 144p  webm 视频
598 144p  webm 视频(小)

249 tiny  webm 音频(小)
250 tiny  webm 音频(中)
251 tiny  webm 音频(大)
600 tiny  webm 音频(特小)


 */

const types = {
	mp4: {
		video: {
			"144p": [160, 597],
			"240p": [133, 395],
			"360p": [134, 396],
			"480p": [135, 397],
			"720p": [136],
			"720p60": [398],
			"1080p": [137],
			"1080p60": [399],
			"1440p60": [400],
			"2160p60": [401],
			"4320p60":[402,571]
		},
		audio: [140, 599],
	},
	webm: {
		video: {
			"144p": [278, 598, 160, 597],
			"240p": [242, 133, 395],
			"360p": [243, 134, 396],
			"480p": [244, 135, 397],
			"720p": [247, 136, 398],
			"720p60": [302, 398],
			"1080p": [248, 137],
			"1080p60": [303, 399],
			"1440p": [271],
			"1440p60": [308, 400],
			"2160p": [313],
			"2160p60": [315, 401],
			"4320p60":[402,571]
		},
		audio: [249, 250, 251, 600, 140, 599],
	},
};

const keys = [
	"144p",
	"240p",
	"360p",
	"480p",
	"720p",
	"720p60",
	"1080p",
	"1080p60",
	"1440p",
	"1440p60",
	"2160p",
	"2160p60",
	"4320p60" 
];

const def = {
	req: "",
	thread: 2,
	thunk: 1048576,
	start: 0,
	end: 1048576,
};

import { timeDuration, addEventListenerOnce, webp } from "@/utils";
import delayer from "@/utils/delayer";
import { imgSrc, videoBaseURL } from "@/service";

let loader, timer, delay;

const canplay = (t) => {
	if (t && t.len) {
		return (
			Object.keys(t.initRange).length + Object.keys(t.indexRange).length
		);
	}
	return false;
};

const format = (item, playerInfo) => {
	const uri = `/${playerInfo.id}/${item.itag}.${
		/webm/.test(item.type) ? "webm" : "mp4"
	}`;
	const mirrors = videoBaseURL
		.split(";")
		.filter((v) => v)
		.map((v) => v + uri);
	return {
		itag: item.itag,
		quality: item.quality,
		req: mirrors[0],
		init: {
			start: Number(item.initRange.start),
			end: Number(item.initRange.end),
		},
		index: {
			start: Number(item.indexRange.start),
			end: Number(item.indexRange.end),
		},
		mimeCodec: item.type,
		len: Number(item.len),
		duration: Number(playerInfo.duration),
		meta: `${playerInfo.id}:${item.itag}`,
		mirrors,
	};
};

const getvideo = (s, maparr) => {
	for (let k of keys) {
		const itags = maparr[k];
		for (let i of itags) {
			if (canplay(s[i])) {
				return s[i];
			}
		}
	}
};

const getaudio = (s, itags) => {
	for (let i of itags) {
		if (canplay(s[i])) {
			return s[i];
		}
	}
};

export default {
	props: {
		playerInfo: {
			type: Object,
			required: true,
		},
		autoplay: {
			type: Boolean,
			default: true,
		},
		audio: {
			type: Boolean,
			default: false,
		},
	},
	data() {
		return {
			firstItag: localStorage.getItem("itag"),
			muted: false,
			paused: true,
			small: true,
			tiptime: {
				v: "",
				show: false,
				left: "",
			},
			v: false,
			video: {
				duration: 0,
				currentTime: 0,
				played: 0,
				seeking: false,
				error: "",
				playbackRate: sessionStorage.getItem("playbackRate") || 1,
				cycle: sessionStorage.getItem("cycle") || 0,
			},
			speeds: [0.5, 0.75, 1, 1.25, 1.5, 1.75, 2],
			vol: {
				v: 60,
			},
			bottomHide: false,
			full: false,
			currq: {
				quality: "",
				itag: 0,
			},
		};
	},
	computed: {
		playedStyle() {
			return {
				width: `${this.video.played * 100}%`,
			};
		},
		dotStyle() {
			return {
				left: `calc(${this.video.played * 100}% - 5px)`,
			};
		},
		tipStyle() {
			return {
				left: this.tiptime.left,
			};
		},
		posterImg() {
			return imgSrc(this.playerInfo.id);
		},
		poster() {
			return {
				backgroundImage: `url("${this.posterImg}")`,
			};
		},
		mp4() {
			const r = [];
			const s = this.playerInfo.streams;
			if (!this.audio) {
				const firstItag = this.firstItag;
				if (canplay(s[firstItag])) {
					r.push(format(s[firstItag], this.playerInfo));
				} else {
					const v = getvideo(s, types.mp4.video);
					if (v) {
						r.push(format(v, this.playerInfo));
					}
				}
			}

			const a = getaudio(s, types.mp4.audio);
			if (a) {
				r.push(format(a, this.playerInfo));
			}
			return r;
		},
		webm() {
			const r = [];
			const s = this.playerInfo.streams;
			if (!this.audio) {
				const firstItag = this.firstItag;
				if (canplay(s[firstItag])) {
					r.push(format(s[firstItag], this.playerInfo));
				} else {
					const v = getvideo(s, types.webm.video);
					if (v) {
						r.push(format(v, this.playerInfo));
					}
				}
			}

			const a = getaudio(s, types.webm.audio);
			if (a) {
				r.push(format(a, this.playerInfo));
			}
			return r;
		},
		qlist() {
			const r = [];
			const s = this.playerInfo.streams;
			const videos = webp ? types.webm.video : types.mp4.video;
			for (let q of keys) {
				const itags = videos[q];
				if(!itags){
					continue;
				}
				for (let i of itags) {
					if (canplay(s[i])) {
						r.push({
							quality: q,
							itag: i,
						});
						break;
					}
				}
			}
			return r.reverse();
		},
	},
	created() {},
	mounted() {
		this.init();
		delay = new delayer(
			() => {
				if (!this.audio) {
					this.bottomHide = true;
				}
			},
			() => {
				this.bottomHide = false;
			},
			2000
		);
	},
	beforeDestroy() {
		this.destroy();
	},
	methods: {
		destroy() {
			if (loader) {
				loader.destroy();
			}
			if (this.$refs.video) {
				this.$refs.video.pause();
			}
		},
		init() {
			this.$emit("init");
			if (this.$refs.video) {
				this.$refs.video.pause();
			}
			if (loader) {
				loader.destroy();
			}
			this.v = false;
			this.$nextTick(() => {
				this.v = true;
				this.$nextTick(() => {
					const loadItem = webp ? this.webm : this.mp4;
					if (!loadItem.length) {
						this.video.error = "资源不存在或不支持";
						return;
					}
					const v = this.$refs.video;
					v.addEventListener("seeking", () => {
						this.video.seeking = true;
					});
					v.addEventListener("seeked", () => {
						this.video.seeking = false;
					});
					v.addEventListener("pause", () => {
						this.paused = true;
					});
					v.addEventListener("play", () => {
						this.paused = false;
					});
					v.addEventListener("durationchange", () => {
						this.video.duration = v.duration;
					});
					v.addEventListener("progress", () => {
						this.updateLoadBar(v.buffered, v.duration);
					});
					v.addEventListener("timeupdate", () => {
						requestAnimationFrame(() => {
							const played = v.currentTime / v.duration;
							this.video.played = played;
							this.video.currentTime = v.currentTime;
						});
					});
					v.addEventListener("ended", () => {
						if (this.video.cycle == 1) {
							setTimeout(() => {
								v.play().catch((err) => console.info(err));
							}, 1000);
						} else {
							if (this.audio) {
								this.$emit("ended", this.playerInfo);
							}
						}
					});
					v.addEventListener("loadedmetadata", () => {
						this.playSpeed();
						if (this.autoplay) {
							v.play().catch((err) => console.info(err));
						}
					});

					loader = new fastloadjs(def);
					loader.listen("ready", (loaders, dispatchs, initdatas) => {
						this.$emit(
							"loadersready",
							loaders,
							dispatchs,
							initdatas
						);
						// 移动端没有progress事件,只能用这个更新
						loaders.forEach((loader) => {
							let i = 0;
							loader.listen("res.done", () => {
								i++;
								setTimeout(() => {
									this.updateLoadBar(v.buffered, v.duration);
								}, 100 * i);
							});
						});
					});
					loader.listen("error", (err) => {
						this.video.error = err;
						loader && loader.pause();
					});
					loader.attach(v, loadItem);
					if (!this.audio) {
						const loadItemVideo = loadItem[0];
						this.switchQuality({
							itag: loadItemVideo.itag,
							quality: loadItemVideo.quality,
						});
					}

					this.reset();
					this.$emit("load", loadItem);
				});
			});
		},
		reset() {
			this.video = {
				duration: 0,
				currentTime: 0,
				played: 0,
				seeking: false,
				error: "",
				playbackRate: sessionStorage.getItem("playbackRate") || 1,
				cycle: sessionStorage.getItem("cycle") || 0,
			};
			this.vol = {
				v: 60,
			};
		},
		getTime(dst) {
			let time = dst * this.$refs.video.duration;
			if (time < 0) {
				time = 0;
			} else if (time > this.$refs.video.duration) {
				time = this.$refs.video.duration;
			}
			return time;
		},
		seek(n) {
			const t = this.$refs.video.duration;
			if (!t || !isFinite(t)) {
				return;
			}
			const s = this.$refs.video.currentTime + n;
			if (s > 0 && s < t) {
				this.video.currentTime = s;
				this.$refs.video.currentTime = s;
				loader.seekTo(s);
			}
			delay.reset();
			delay.delay();
		},
		seekTo(e) {
			let dx;
			const target = e.target;
			if (target.classList.contains("dot")) {
				dx = target.offsetLeft + e.offsetX;
			} else {
				dx = e.offsetX;
			}
			const w = this.$refs.progressLine.clientWidth;
			const dst = dx / w;
			const time = this.getTime(dst);
			if (!isFinite(time)) {
				return;
			}
			this.video.currentTime = time;
			this.$refs.video.currentTime = time;
			this.tiptime.v = timeDuration(time);
			this.tiptime.show = true;
			loader.seekTo(time);
		},
		showCurrTime(e) {
			let dx;
			const target = e.target;
			if (target.classList.contains("dot")) {
				dx = target.offsetLeft + e.offsetX;
			} else {
				dx = e.offsetX;
			}
			const w = this.$refs.progressLine.clientWidth;
			const dst = dx / w;
			const time = this.getTime(dst);
			const sw = this.$refs.times.clientWidth;
			let left = dx - sw / 2;
			if (left < 0) {
				left = 0;
			}
			if (left > w - sw) {
				left = w - sw;
			}
			this.tiptime.v = timeDuration(time);
			this.tiptime.left = `${left}px`;
		},
		togglePlayOnClick() {
			if (!this.audio) {
				this.togglePlay();
			}
		},
		togglePlay() {
			if (this.$refs.video.paused) {
				this.playVideo();
			} else {
				this.pauseVideo();
			}
		},
		toggleFull() {
			if (!this.audio) {
				this.full = !this.full;
			}
		},
		pauseVideo() {
			this.$refs.video.pause();
		},
		playVideo() {
			this.$refs.video.play().catch((err) => console.info(err));
		},
		muteoff() {
			this.$refs.video.muted = false;
			this.muted = false;
		},
		muteon() {
			this.$refs.video.muted = true;
			this.muted = true;
		},
		toFull() {
			try {
				this.$el.requestFullscreen();
				this.small = false;
			} catch (e) {
				this.small = true;
			}
		},
		exitFull() {
			try {
				document.exitFullscreen();
				this.small = true;
			} catch (e) {
				this.small = false;
			}
		},
		volPosTotal(e) {
			const n = 60 - e.offsetY;
			this.setVol(n);
		},
		volPosNum(e) {
			const n = e.target.clientHeight - e.offsetY;
			this.setVol(n);
		},
		dotDown(e) {
			let y, vol;
			const onVolMoveProgress = (e) => {
				const w = 60;
				const dx = y - e.pageY;
				let ox = vol + dx;
				if (ox < 0) {
					ox = 0;
				}
				if (ox > w) {
					ox = w;
				}
				this.setVol(ox);
			};
			y = e.pageY;
			vol = this.$refs.numdiv.clientHeight;
			document.addEventListener("mousemove", onVolMoveProgress);
			addEventListenerOnce(document, "mouseup", () => {
				document.removeEventListener("mousemove", onVolMoveProgress);
			});
		},
		setVol(n) {
			this.vol.v = n;
			this.$refs.video.volume = Math.min(1, n / 60);
			this.muted = !n;
			this.$refs.video.muted = this.muted;
		},
		maskMouseEnter() {
			delay.reset();
			requestAnimationFrame(() => {
				this.$el.focus();
			});
		},
		maskMouseLeave() {
			delay.do();
		},
		maskMouseMove() {
			delay.reset();
			delay.delay();
		},
		switchQuality(item, reload) {
			this.currq = item;
			this.firstItag = item.itag;
			if (reload) {
				localStorage.setItem("itag", item.itag);
				this.pauseVideo();
				this.init(); // 当前未实现无缝质量切换.
			}
		},
		playSpeed(v) {
			if (!v) {
				v = sessionStorage.getItem("playbackRate") || 1;
			}
			if (!this.$refs.video) {
				return;
			}
			this.$refs.video.playbackRate = v;
			this.video.playbackRate = v;
			sessionStorage.setItem("playbackRate", v);
		},
		cyclePlay(v) {
			sessionStorage.setItem("cycle", v);
			this.video.cycle = v;
		},
		updateLoadBar(buffered, duration) {
			if (!buffered || !duration) {
				return;
			}
			const c = this.$refs.loadbar;
			if (!c) {
				return;
			}
			const ctx = c.getContext("2d");
			ctx.clearRect(0, 0, c.width, c.height);
			ctx.fillStyle = "#ddd";
			var inc = c.width / duration;
			// display TimeRanges
			for (let i = 0; i < buffered.length; i++) {
				var startX = buffered.start(i) * inc;
				var endX = buffered.end(i) * inc;
				var width = endX - startX;
				ctx.fillRect(startX, 0, width, c.height);
			}
		},
	},
	watch: {
		"playerInfo.id"(v) {
			if (v) {
				this.init();
			}
		},
	},
	filters: {
		timeformat(t) {
			return timeDuration(t);
		},
	},
};
</script>

<style lang="less">
.clearfix() {
	*zoom: 1; // for ie6 ie7
	&:before,
	&:after {
		content: " "; // 1
		display: table; // 2
	}
	&:after {
		clear: both;
	}
}
.vplayer {
	width: 100%;
	min-height: 400px;
	// height: 100%; // 组件模式需使用100%
	position: relative;
	overflow: hidden;
	user-select: none;
	font-family: -apple-system-font, "Helvetica Neue", sans-serif;
	font-size: 14px;
	background: #282828;
	outline: none;
	&.audio {
		min-height: 100px;
		height: initial;
		.controls {
			background: #333;
			.bottom {
				background: #333;
				left: 170px;
				height: 30px;
				.progress-line,
				.loaded,
				.played {
					height: 2px;
				}
				.dot {
					width: 6px;
					height: 6px;
					top: -2px;
				}
				&:hover {
					.progress-line,
					.loaded,
					.played {
						height: 2px;
					}
					.dot {
						width: 6px;
						height: 6px;
					}
				}
				.progress-line {
					.times {
						font-size: 12px;
					}
				}
				.play-pause {
					padding: 6px 0 0;
					svg {
						width: 14px;
						height: 14px;
					}
				}
				.time-duration {
					padding: 4px 0 4px 10px;
					font-size: 12px;
				}
				.volume-wrap {
					padding: 7px 5px 0 0;
					svg {
						width: 14px;
						height: 14px;
					}
					.vol-bar-wrap {
						top: -70px;
						height: 78px;
						left: 6px;
						.vol-bar {
							border-radius: 6px;
						}
					}
				}
				.a-title {
					position: absolute;
					width: 100%;
					top: -24px;
					text-align: center;
					color: #fff;
					text-shadow: 0 0 1px #888;
					white-space: nowrap;
					overflow: hidden;
					text-overflow: ellipsis;
					padding: 0 12px;
				}
				.audio-list {
					float: right;
					padding: 7px 10px 0 10px;
					margin-right: 5px;
					svg {
						width: 16px;
						height: 16px;
						fill: #e8e8e8;
						cursor: pointer;
						&:hover {
							fill: #fff;
						}
					}
				}
			}
			.errored {
				bottom: 0;
				z-index: 230;
				.errmsg {
					padding: 0 30px;
					font-size: 12px;
					p {
						margin: 0;
					}
				}
			}
			.seeking {
				left: 0;
				width: 170px;
				position: absolute;
				z-index: 12;
			}
			.poster {
				width: 170px;
				height: 100px;
				background-size: cover;
			}
		}
	}
	&.full {
		position: fixed;
		left: 0;
		right: 0;
		top: 0;
		bottom: 0;
		max-height: 100%;
		z-index: 9999;
	}
	.video,
	.controls,
	.poster {
		position: absolute;
		left: 0;
		right: 0;
		top: 0;
		bottom: 0;
		width: 100%;
		height: 100%;
		outline: none;
	}

	.controls {
		z-index: 66;
		&.cleanmode {
			cursor: none;
		}
		.errored {
			display: flex;
			justify-content: center;
			align-items: center;
			z-index: 8;
			position: absolute;
			left: 0;
			right: 0;
			top: 0;
			bottom: 60px;
			width: 100%;
			.errmsg {
				background: #222;
				color: #d00;
				padding: 30px;
				box-shadow: 0 0 10px #444;
				position: relative;
				.hidebtn {
					z-index: 78;
					position: absolute;
					top: 5px;
					right: 0px;
					line-height: 0;
					font-size: 20px;
					transform: rotate(45deg);
					cursor: pointer;
				}
			}
		}
		.seeking {
			height: 100%;
			background: url("https://assets.suconghou.cn/load.gif") no-repeat
				center;
			background-size: 40px;
		}
		.poster {
			background-size: contain;
			background-position: center center;
			background-repeat: no-repeat;
		}
		.bottom {
			box-sizing: content-box;
			position: absolute;
			z-index: 100;
			bottom: 0;
			height: 40px;
			left: 0;
			right: 0;
			background: linear-gradient(
				to top,
				rgba(0, 0, 0, 0.6),
				rgba(0, 0, 0, 0)
			);
			padding-top: 20px;
			transition: all 0.2s ease;
			&.hide {
				visibility: hidden;
				opacity: 0;
			}
			&:hover {
				.progress-line,
				.loaded,
				.played {
					height: 5px;
				}
				.dot {
					width: 11px;
					height: 11px;
				}
			}
		}
		.progress-bar {
			height: 3px;
		}
		.progress-line {
			width: 100%;
			height: 3px;
			position: relative;
			cursor: pointer;
			background: rgba(255, 255, 255, 0.3);
			.times {
				position: absolute;
				height: 20px;
				top: -26px;
				line-height: 20px;
				border-radius: 1px;
				color: #fff;
				padding: 0 5px;
				display: inline-block;
				left: 0;
				background: rgba(1, 1, 1, 0.8);
				font-size: 14px;
				box-shadow: 0 0 4px rgba(1, 1, 1, 0.5);
				transition: visibility 0.01s ease;
				&.hide {
					visibility: hidden;
				}
			}
			.dot {
				width: 9px;
				height: 9px;
				border-radius: 50%;
				position: absolute;
				background: #fefefe;
				top: -3px;
				left: 0;
			}
			.loaded {
				position: absolute;
				height: 3px;
				width: 100%;
			}
			.played {
				position: absolute;
				height: 3px;
				background: #45a9ff;
				width: 0%;
			}
		}
		.btns {
			.clearfix();
			.play-pause {
				margin-left: 18px;
				float: left;
				padding: 8px 0;
				svg {
					width: 18px;
					height: 18px;
					cursor: pointer;
					fill: #ddd;
					&:hover {
						fill: #fff;
					}
				}
			}
		}
		.time-duration {
			pointer-events: none;
			float: left;
			min-width: 20px;
			max-width: 200px;
			overflow: hidden;
			height: 20px;
			padding: 8px 0 8px 14px;
			color: #ddd;
			font-size: 14px;
			line-height: 20px;
			box-sizing: content-box;
			.curr,
			.total {
				display: inline-block;
			}
			.padding {
				padding: 0 5px;
			}
		}
		.volume-wrap {
			float: right;
			margin-right: 5px;
			text-align: right;
			width: 30px;
			height: 20px;
			padding: 8px 0;
			color: #ddd;
			line-height: 20px;
			position: relative;
			box-sizing: content-box;
			svg {
				width: 20px;
				height: 20px;
				cursor: pointer;
				fill: #ddd;
				&:hover {
					fill: #fff;
				}
			}
			&:hover {
				.vol-bar-wrap {
					display: block;
				}
			}
			.vol-bar-wrap {
				display: none;
				width: 30px;
				height: 92px;
				position: absolute;
				top: -86px;
				left: 4px;
				z-index: 88;
			}
			.vol-bar {
				position: relative;
				background: rgba(1, 1, 1, 0.8);
				border-radius: 1px;
				width: 30px;
				height: 78px;
				box-shadow: 0 0 2px rgba(1, 1, 1, 0.5);
				.vol-total {
					height: 60px;
					width: 3px;
					background: #919191;
					position: absolute;
					left: 14px;
					bottom: 9px;
					cursor: pointer;
				}
				.vol-dot {
					height: 9px;
					width: 9px;
					border-radius: 50%;
					background: #fff;
					position: absolute;
					left: 11px;
					cursor: pointer;
					bottom: 64.5px;
				}
				.vol-num {
					height: 60px;
					width: 3px;
					background: #45a9ff;
					position: absolute;
					left: 14px;
					bottom: 9px;
					cursor: pointer;
				}
			}
		}
		&.hide {
			display: none;
		}
		.playbutton {
			position: absolute;
			top: 50%;
			left: 50%;
			padding: 9px;
			border: 2px solid #e8e8e8;
			border-radius: 50%;
			cursor: pointer;
			transform: translate(-50%, -50%);
			display: flex;
			justify-content: center;
			align-items: center;
			z-index: 196;
			background: rgba(1, 1, 1, 0.5);
			svg {
				width: 20px;
				height: 20px;
				cursor: pointer;
				fill: #e8e8e8;
				position: relative;
				left: 1px;
			}
			&.hide {
				visibility: hidden;
			}
			&.loading {
				border: none;
				background: url("https://assets.suconghou.cn/loading.gif")
					no-repeat center;
				svg {
					display: none;
				}
				background-size: contain;
				width: 14px;
				height: 14px;
			}
			&:hover {
				border-color: #fff;
				svg {
					fill: #fff;
				}
			}
		}
		.full-screen,
		.fake-full {
			float: right;
			margin-right: 18px;
			text-align: right;
			width: 40px;
			height: 20px;
			padding: 8px 0;
			color: #ddd;
			line-height: 20px;
			box-sizing: content-box;
			svg {
				width: 20px;
				height: 20px;
				cursor: pointer;
				fill: #ddd;
				&:hover {
					fill: #fff;
				}
			}
		}
		.fake-full {
			margin-right: 2px;
		}
		.quality-list {
			float: right;
			margin-right: 10px;
			text-align: right;
			color: #ddd;
			line-height: 20px;
			position: relative;
			&:hover {
				.qitem-list {
					display: block;
				}
			}
			.qitem-list {
				position: absolute;
				bottom: 30px;
				padding: 0 0 10px 0;
				margin: 0;
				display: none;
				white-space: nowrap;
			}
			.curr-q {
				text-transform: uppercase;
				height: 20px;
				padding: 8px 0 8px 14px;
				font-size: 14px;
				line-height: 20px;
				box-sizing: content-box;
			}
			.qitem {
				background: rgba(1, 1, 1, 0.6);
				display: block;
				text-transform: uppercase;
				padding: 2px 10px;
				cursor: pointer;
				&:hover,
				&.active {
					background: rgba(1, 1, 1, 0.8);
				}
			}
		}
		.settings {
			float: right;
			padding-right: 10px;
			padding-left: 3px;
			text-align: right;
			color: #ddd;
			line-height: 20px;
			box-sizing: content-box;
			padding-top: 7px;
			position: relative;
			svg {
				width: 22px;
				height: 22px;
				cursor: pointer;
				fill: #ddd;
				&:hover {
					fill: #fff;
				}
			}
			&:hover {
				.settings-content {
					display: block;
				}
			}
			&-content {
				position: absolute;
				bottom: 24px;
				color: #fff;
				width: 76px;
				text-align: left;
				text-indent: 10px;
				padding-bottom: 10px;
				left: -4px;
				display: none;
			}
			&-list {
				padding: 0;
				margin: 4px 0;
				background: rgba(1, 1, 1, 0.6);
			}
			&-item {
				list-style: none;
				position: relative;
				&:hover {
					cursor: pointer;
					background: rgba(1, 1, 1, 0.8);
					.s-opt {
						display: block;
					}
				}
				.s-opt,
				ul,
				li {
					list-style: none;
					padding: 4px 0;
					display: block;
				}
				li:hover {
					background: rgba(1, 1, 1, 0.8);
				}
				.s-value {
					padding: 4px 0;
				}
				.s-opt {
					display: none;
					width: 60px;
					background: rgba(1, 1, 1, 0.6);
					left: 76px;
					cursor: pointer;
					bottom: 0;
					position: absolute;
					padding: 0;
					li.active {
						color: #2196f3;
					}
				}
			}
		}
	}
}
@media screen and (max-width: 900px) {
	.vplayer {
		.controls {
			.quality-list {
				margin-right: 0;
			}
			.volume-wrap,
			.settings {
				display: none;
			}
		}
	}
}
</style>