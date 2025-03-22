<template>
	<div class="gallery-container">
		<div class="mode-switch">
			<button @click="switchBackground">切换背景</button>
			<button @click="toggleViewMode">{{ isListMode ? "3D模式" : "列表模式" }}</button>
			<button @click="toggleControlMode" v-show="!isListMode">{{ isControlEnabled ? "固定视角" : "自由视角" }}</button>
			<button @click="toggleRotation" v-show="!isListMode">{{ isRotating ? "暂停旋转" : "播放旋转" }}</button>
			<div class="color-picker">
				<input type="color" v-model="startColor" />
				<input type="color" v-model="endColor" />
			</div>
		</div>
		<div ref="container" class="scene-container" v-show="!isListMode"></div>
		<div class="list-container" v-show="isListMode">
			<div v-for="(photo, index) in photos" :key="index" class="photo-item" @click="selectPhoto(index)">
				<img :src="photo.url" :alt="photo.title" />
			</div>
		</div>
		<audio ref="audioPlayer" :src="currentTrack.url" @timeupdate="updateProgress" @ended="nextTrack"></audio>
		<div class="music-title">当前播放: {{ currentTrack.title }}</div>
		<div class="music-controls">
			<button @click="prevTrack"><img src="/src/images/prev.svg" alt="上一曲" /></button>
			<button @click="toggleMusic">
				<img :src="isPlaying ? '/src/images/play.svg' : '/src/images/pause.svg'" :alt="isPlaying ? '暂停' : '播放'" />
			</button>
			<button @click="nextTrack"><img src="/src/images/next.svg" alt="下一曲" /></button>
			<input type="range" min="0" max="100" v-model="progress" @input="seek" />
			<div class="color-picker">
				<input type="color" v-model="musicStartColor" />
				<input type="color" v-model="musicEndColor" />
			</div>
		</div>
		<div class="music-visualizer">
			<canvas ref="visualizerCanvas"></canvas>
		</div>
	</div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, computed, watch } from "vue";
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import TWEEN from '@tweenjs/tween.js';

const container = ref(null);
const isListMode = ref(false);
const isControlEnabled = ref(true);
const isRotating = ref(true);
const rotationSpeed = ref(0.0006);  // 背景的旋转速度控制
const isPlaying = ref(true);
const audioPlayer = ref(null);
const startColor = ref('#34d5af'); // 默认红色
const endColor = ref('#a689c8'); // 默认紫色
const musicStartColor = ref('#ff0000'); // 默认红色
const musicEndColor = ref('#800080'); // 默认紫色

const backgroundImages = [
	"/images/33.jpg",
	"/images/22.jpg",
	"/images/11.jpg"
];
let currentBackgroundIndex = ref(0);

// 示例图片数据
const photos = ref([
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
	{ url: "/images/photo1.svg", title: "Photo 1" },
	{ url: "/images/photo2.svg", title: "Photo 2" },
	{ url: "/images/photo3.svg", title: "Photo 3" },
	{ url: "/images/photo4.svg", title: "Photo 4" },
]);

const tracks = ref([
	{ url: "/music/告五人 - 带我去找夜生活.mp3", title: "告五人 - 带我去找夜生活" },
	{ url: "/music/温奕心 - 一路生花.mp3", title: "温奕心 - 一路生花" },
	{ url: "/music/执子之手-宝石Gem&一哩哩一.mp3", title: "执子之手-宝石Gem&一哩哩一" }
]);
const currentTrackIndex = ref(0);
const currentTrack = computed(() => tracks.value[currentTrackIndex.value]);
const progress = ref(0);

// Three.js 相关变量
let scene, camera, renderer, controls;
let photoMeshes = [];
let backgroundSphere;
let raycaster = new THREE.Raycaster();
let mouse = new THREE.Vector2();

// 在 script setup 中定义 textureLoader
const textureLoader = new THREE.TextureLoader();

// 初始化Three.js场景
const initThreeJS = () => {
	scene = new THREE.Scene();
	camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
	renderer = new THREE.WebGLRenderer({ antialias: true });

	const containerEl = container.value;
	renderer.setSize(window.innerWidth, window.innerHeight);
	renderer.setPixelRatio(window.devicePixelRatio);
	renderer.setClearColor(0x000022);
	containerEl.appendChild(renderer.domElement);

	camera.position.set(0, 0, 4);

	controls = new OrbitControls(camera, renderer.domElement);
	controls.enableDamping = true;
	controls.dampingFactor = 0.05;
	controls.minDistance = 1;
	controls.maxDistance = 10;
	controls.enablePan = false;
	controls.enableZoom = true;

	camera.lookAt(0, 0, 0);

	createPhotoMeshes();

	const sphereGeometry = new THREE.SphereGeometry(30, 64, 64);
	const galaxyTexture = textureLoader.load(
		backgroundImages[currentBackgroundIndex.value],
		(texture) => {
			texture.colorSpace = THREE.SRGBColorSpace;
			backgroundSphere.material.map = texture;
			backgroundSphere.material.needsUpdate = true;
		},
		undefined,
		(error) => {
			console.error("Error loading galaxy texture:", error);
			backgroundSphere.material.map = null;
			backgroundSphere.material.color.set(0x000022);
		}
	);
	const sphereMaterial = new THREE.MeshBasicMaterial({
		map: galaxyTexture,
		side: THREE.BackSide,
	});
	backgroundSphere = new THREE.Mesh(sphereGeometry, sphereMaterial);
	scene.add(backgroundSphere);

	const ambientLight = new THREE.AmbientLight(0xffffff, 0.8);
	scene.add(ambientLight);

	const directionalLight = new THREE.DirectionalLight(0xffffff, 1.0);
	directionalLight.position.set(0, 1, 2);
	scene.add(directionalLight);

	const pointLight1 = new THREE.PointLight(0x4477ff, 1, 20);
	pointLight1.position.set(5, 5, 5);
	scene.add(pointLight1);

	const pointLight2 = new THREE.PointLight(0xff7744, 1, 20);
	pointLight2.position.set(-5, -5, 5);
	scene.add(pointLight2);

	window.addEventListener("resize", onWindowResize);

	animate();
};

// 窗口大小变化处理
const onWindowResize = () => {
	camera.aspect = window.innerWidth / window.innerHeight;
	camera.updateProjectionMatrix();
	renderer.setSize(window.innerWidth, window.innerHeight);
};

// 在 createPhotoMeshes 中应用标准材质
const createPhotoMeshes = () => {
	photos.value.forEach((photo, index) => {
		const textureLoader = new THREE.TextureLoader();
		textureLoader.load(
			photo.url,
			(texture) => {
				texture.colorSpace = THREE.SRGBColorSpace;
				const geometry = new THREE.PlaneGeometry(0.8, 0.6);
				const material = new THREE.MeshStandardMaterial({
					map: texture,
					side: THREE.DoubleSide,
				});
				const mesh = new THREE.Mesh(geometry, material);

				// 使用改进的斐波那契球面分布算法
				const numPhotos = photos.value.length;
				const radius = 4;
				const goldenRatio = (1 + Math.sqrt(5)) / 2;

				const i = index + 1;
				const phi = Math.acos(1 - (2 * i - 0.5) / (numPhotos + 1));
				const theta = (2 * Math.PI * i) / goldenRatio;

				mesh.position.x = radius * Math.sin(phi) * Math.cos(theta);
				mesh.position.y = radius * Math.sin(phi) * Math.sin(theta);
				mesh.position.z = radius * Math.cos(phi);
				mesh.lookAt(new THREE.Vector3(0, 0, 0));

				scene.add(mesh);
				photoMeshes.push(mesh);
			},
			// 加载进度回调
			(xhr) => {
				console.log(`${photo.title}: ${(xhr.loaded / xhr.total) * 100}% loaded`);
			},
			// 错误回调
			(error) => {
				console.error(`Error loading texture for ${photo.title}:`, error);
				// 加载失败时使用默认纹理
				const defaultMaterial = new THREE.MeshStandardMaterial({
					color: 0x888888,
					side: THREE.DoubleSide,
				});
				const geometry = new THREE.PlaneGeometry(0.8, 0.6);
				const mesh = new THREE.Mesh(geometry, defaultMaterial);
				// 使用相同的位置计算逻辑
				const numPhotos = photos.value.length;
				const radius = 4;
				const goldenRatio = (1 + Math.sqrt(5)) / 2;
				const i = index + 1;
				const phi = Math.acos(1 - (2 * i - 0.5) / (numPhotos + 1));
				const theta = (2 * Math.PI * i) / goldenRatio;
				mesh.position.x = radius * Math.sin(phi) * Math.cos(theta);
				mesh.position.y = radius * Math.sin(phi) * Math.sin(theta);
				mesh.position.z = radius * Math.cos(phi);
				mesh.lookAt(new THREE.Vector3(0, 0, 0));
				scene.add(mesh);
				photoMeshes.push(mesh);
			}
		);
	});
};

// 恢复原始的 animate 函数逻辑
const animate = () => {
	requestAnimationFrame(animate);

	// 更新射线
	raycaster.setFromCamera(mouse, camera);

	// 计算物体与射线的交集
	const intersects = raycaster.intersectObjects(photoMeshes);

	// 重置所有物体的状态
	photoMeshes.forEach((mesh) => {
		new TWEEN.Tween(mesh.scale)
			.to({ x: 1, y: 1, z: 1 }, 300)
			.easing(TWEEN.Easing.Quadratic.Out)
			.start();
		mesh.material.emissive.setHex(0x000000); // 移除发光
	});

	// 对于每个相交的物体，设置放大和发光效果
	for (let i = 0; i < intersects.length; i++) {
		new TWEEN.Tween(intersects[i].object.scale)
			.to({ x: 1.5, y: 1.5, z: 1.5 }, 300)
			.easing(TWEEN.Easing.Quadratic.Out)
			.start();
		intersects[i].object.material.emissive.setHex(0x444444); // 设置发光
		intersects[i].object.material.color.setHex(0xffffff); // 设置为白色以模拟边框阴影
	}
	// 自动旋转背景球体
	if (isRotating.value) {
		backgroundSphere.rotation.y += rotationSpeed.value; // 背景球体自转
		// 自动旋转图片围绕球体的Y轴
		photoMeshes.forEach((mesh) => {
			mesh.position.applyAxisAngle(new THREE.Vector3(0, 1, 0), rotationSpeed.value);
			mesh.lookAt(new THREE.Vector3(0, 0, 0)); // 确保图片始终面向球体中心
		});
	}

	TWEEN.update();
	controls.update();
	renderer.render(scene, camera);
};

// 切换显示模式
const toggleViewMode = () => {
	isListMode.value = !isListMode.value;
};

// 切换控制模式
const toggleControlMode = () => {
	isControlEnabled.value = !isControlEnabled.value;
	if (controls) {
		controls.enabled = isControlEnabled.value;
		if (!isControlEnabled.value) {
			// 固定视角模式：固定垂直角度，只允许水平旋转
			controls.minPolarAngle = Math.PI / 2; // 90度
			controls.maxPolarAngle = Math.PI / 2;
			controls.enablePan = false;
			controls.enableRotate = true;
			controls.enableZoom = false;

			// 调整相机位置到圆形正上方
			const radius = 3; // 与createPhotoMeshes中的radius保持一致
			camera.position.set(0, 0, 2);
			controls.target.set(0, 0, 0);
		} else {
			// 自由视角模式：恢复默认设置
			controls.minPolarAngle = 0;
			controls.maxPolarAngle = Math.PI;
			controls.enablePan = true;
			controls.enableRotate = true;
			controls.enableZoom = true;
		}
	}
};

// 切换旋转状态
const toggleRotation = () => {
	isRotating.value = !isRotating.value;
};

// 选择照片
const selectPhoto = (index) => {
	if (isListMode.value) {
		isListMode.value = false;
		// 等待DOM更新后聚焦到选中的照片
		setTimeout(() => {
			if (photoMeshes[index]) {
				const position = photoMeshes[index].position;
				console.log('Selected photo position:', position);
				// 使用 TWEEN 平滑移动相机位置
				new TWEEN.Tween(camera.position)
					.to({
						x: position.x * 1.2, // 调整倍率以确保图片占据大部分屏幕
						y: position.y * 1.2,
						z: position.z + 1.5 // 调整z轴位置以靠近图片
					}, 1000) // 1秒内完成移动
					.easing(TWEEN.Easing.Quadratic.Out)
					.start();
				// 使用 TWEEN 平滑移动相机目标
				new TWEEN.Tween(controls.target)
					.to({
						x: position.x,
						y: position.y,
						z: position.z
					}, 1000)
					.easing(TWEEN.Easing.Quadratic.Out)
					.onUpdate(() => controls.update())
					.start();
			}
		}, 100);
	}
};

// 处理窗口大小变化
const handleResize = () => {
	if (!container.value || isListMode.value) return;

	const containerEl = container.value;
	camera.aspect = containerEl.clientWidth / containerEl.clientHeight;
	camera.updateProjectionMatrix();
	renderer.setSize(containerEl.clientWidth, containerEl.clientHeight);
};

// 切换音乐状态
const toggleMusic = () => {
	if (audioPlayer.value) {
		if (isPlaying.value) {
			audioPlayer.value.pause();
		} else {
			audioPlayer.value.play().catch(() => {
				// 如果自动播放失败，等待用户交互后再播放
				document.addEventListener('click', playCurrentTrack, { once: true });
			});
		}
		isPlaying.value = !isPlaying.value;
	}
};

const prevTrack = () => {
	currentTrackIndex.value = (currentTrackIndex.value - 1 + tracks.value.length) % tracks.value.length;
	playCurrentTrack();
};

const nextTrack = () => {
	currentTrackIndex.value = (currentTrackIndex.value + 1) % tracks.value.length;
	playCurrentTrack();
};

const playCurrentTrack = () => {
	if (audioPlayer.value) {
		// 先暂停并重置当前音频
		audioPlayer.value.pause();
		audioPlayer.value.currentTime = 0;

		// 设置新的音频源
		audioPlayer.value.src = currentTrack.value.url;

		// 等待音频加载完成后再播放
		audioPlayer.value.load();
		audioPlayer.value.addEventListener('canplaythrough', () => {
			audioPlayer.value.play().catch((error) => {
				console.error('Error playing audio:', error);
			});
			isPlaying.value = true;
		}, { once: true });
	}
};

const updateProgress = () => {
	if (audioPlayer.value) {
		progress.value = (audioPlayer.value.currentTime / audioPlayer.value.duration) * 100;
	}
};

const seek = (event) => {
	if (audioPlayer.value) {
		const seekTime = (event.target.value / 100) * audioPlayer.value.duration;
		audioPlayer.value.currentTime = seekTime;
	}
};

const audioContext = new (window.AudioContext || window.webkitAudioContext)();
const analyser = audioContext.createAnalyser();
analyser.fftSize = 128;
const bufferLength = analyser.frequencyBinCount;
const dataArray = new Uint8Array(bufferLength);

const visualizerCanvas = ref(null);

let autoSwitchInterval;

onMounted(() => {
	initThreeJS();
	window.addEventListener("resize", handleResize);
	audioPlayer.value = document.querySelector('audio');

	// 初始化 CSS 变量
	document.documentElement.style.setProperty('--start-color', startColor.value);
	document.documentElement.style.setProperty('--end-color', endColor.value);
	document.documentElement.style.setProperty('--music-start-color', musicStartColor.value);
	document.documentElement.style.setProperty('--music-end-color', musicEndColor.value);

	// 尝试播放音乐
	audioPlayer.value.play().catch(() => {
		// 如果自动播放失败，等待用户交互后再播放
		document.addEventListener('click', playCurrentTrack, { once: true });
	});

	const audioElement = audioPlayer.value;
	const source = audioContext.createMediaElementSource(audioElement);
	source.connect(analyser);
	analyser.connect(audioContext.destination);

	const canvas = visualizerCanvas.value;
	const canvasCtx = canvas.getContext("2d");

	const draw = () => {
		requestAnimationFrame(draw);

		analyser.getByteFrequencyData(dataArray);

		// 清除画布并设置为透明
		canvasCtx.clearRect(0, 0, canvas.width, canvas.height);

		const barWidth = (canvas.width / bufferLength) * 2.5;
		let barHeight;
		let x = 0;

		for (let i = 0; i < bufferLength; i++) {
			barHeight = dataArray[i];

			const r = barHeight + 25 * (i / bufferLength);
			const g = 250 * (i / bufferLength);
			const b = 50;

			canvasCtx.fillStyle = `rgb(${r},${g},${b})`;
			canvasCtx.fillRect(x, canvas.height - barHeight / 2, barWidth, barHeight / 2);

			x += barWidth + 1;
		}
	};

	draw();

	window.addEventListener('mousemove', (event) => {
		// 将鼠标位置转换为标准化设备坐标 (-1 to +1)
		mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
		mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
	});

	document.addEventListener('click', () => {
		if (audioContext.state === 'suspended') {
			audioContext.resume();
		}
	}, { once: true });

	// 设置每隔30秒自动切换背景
	autoSwitchInterval = setInterval(() => {
		switchBackground();
	}, 30000); // 30000毫秒 = 30秒
});

onBeforeUnmount(() => {
	window.removeEventListener("resize", handleResize);
	if (renderer) {
		renderer.dispose();
	}
	audioContext.close();

	// 清除定时器
	clearInterval(autoSwitchInterval);
});

watch(startColor, (newColor) => {
	document.documentElement.style.setProperty('--start-color', newColor);
});

watch(endColor, (newColor) => {
	document.documentElement.style.setProperty('--end-color', newColor);
});

watch(musicStartColor, (newColor) => {
	document.documentElement.style.setProperty('--music-start-color', newColor);
});

watch(musicEndColor, (newColor) => {
	document.documentElement.style.setProperty('--music-end-color', newColor);
});

// 切换背景函数
const switchBackground = () => {
	currentBackgroundIndex.value = (currentBackgroundIndex.value + 1) % backgroundImages.length;

	// 创建一个新的材质用于过渡
	const transitionMaterial = new THREE.MeshBasicMaterial({
		map: backgroundSphere.material.map,
		side: THREE.BackSide,
		transparent: true,
		opacity: 1.0
	});

	// 创建一个新的球体用于过渡
	const transitionSphere = new THREE.Mesh(backgroundSphere.geometry, transitionMaterial);
	scene.add(transitionSphere);

	// 加载新的背景纹理
	const newTexture = textureLoader.load(
		backgroundImages[currentBackgroundIndex.value],
		(texture) => {
			texture.colorSpace = THREE.SRGBColorSpace;
			backgroundSphere.material.map = texture;
			backgroundSphere.material.needsUpdate = true;

			// 使用 TWEEN 库实现渐变效果
			new TWEEN.Tween(transitionMaterial)
				.to({ opacity: 0 }, 1000) // 1秒内完成渐变
				.easing(TWEEN.Easing.Quadratic.Out)
				.onComplete(() => {
					scene.remove(transitionSphere); // 渐变完成后移除过渡球体
				})
				.start();
		},
		undefined,
		(error) => {
			console.error("Error loading new background texture:", error);
			backgroundSphere.material.map = null;
			backgroundSphere.material.color.set(0x000022);
		}
	);
};
</script>

<style scoped>
.gallery-container {
	width: 100vw;
	height: 100vh;
	position: relative;
	overflow: auto;
	background: #000;
}

.mode-switch {
	position: fixed;
	top: 20px;
	right: 20px;
	z-index: 100;
	display: flex;
	gap: 10px;
	padding: 10px;
	background: rgba(0, 0, 0, 0.3);
	border-radius: 8px;
	backdrop-filter: blur(8px);
}

.mode-switch button {
	padding: 8px 16px;
	border: none;
	border-radius: 4px;
	background: linear-gradient(to right, var(--start-color), var(--end-color));
	color: #fff;
	cursor: pointer;
	font-size: 14px;
	transition: all 0.3s ease;
}

.mode-switch button:hover {
	background: linear-gradient(to right, var(--end-color), var(--start-color));
	transform: translateY(-2px);
}

.scene-container {
	width: 100%;
	height: 100%;
	position: absolute;
	top: 0;
	left: 0;
}

.list-container {
	display: grid;
	grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
	gap: 20px;
	padding: 20px;
	margin: 0 auto;
	overflow-y: auto;
}

.photo-item {
	aspect-ratio: 4/3;
	overflow: hidden;
	border-radius: 8px;
	cursor: pointer;
	transition: transform 0.3s ease;
}

.photo-item:hover {
	transform: scale(1.05);
	box-shadow: 0 0 10px rgba(255, 255, 255, 0.8);
}

.photo-item img {
	width: 100%;
	height: 100%;
	object-fit: cover;
}

@media (max-width: 768px) {
	.list-container {
		grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
		gap: 10px;
		padding: 10px;
	}

	.mode-switch {
		top: 10px;
		right: 10px;
	}
}

.music-title {
	position: fixed;
	bottom: 88px;
	right: 20px;
	z-index: 100;
	color: #fff;
	background: rgba(0, 0, 0, 0.3);
	padding: 5px 10px;
	border-radius: 4px;
	backdrop-filter: blur(8px);
}

.music-controls {
	position: fixed;
	bottom: 20px;
	right: 20px;
	z-index: 100;
	display: flex;
	gap: 10px;
	padding: 10px;
	background: linear-gradient(to right, var(--music-start-color), var(--music-end-color));
	border-radius: 8px;
	backdrop-filter: blur(8px);
	align-items: center;
}

.music-controls button {
	padding: 8px 16px;
	border: none;
	border-radius: 4px;
	background: rgba(255, 255, 255, 0.2);
	color: #fff;
	cursor: pointer;
	font-size: 14px;
	transition: all 0.3s ease;
}

.music-controls button:hover {
	background: rgba(255, 255, 255, 0.3);
	transform: translateY(-2px);
}

.music-controls img {
	width: 25px;
	height: 25px;
}

.music-controls input[type="range"] {
	width: 100px;
	cursor: pointer;
	background: linear-gradient(to right, var(--music-start-color), var(--music-end-color));
	border-radius: 5px;
	height: 5px;
	-webkit-appearance: none;
	appearance: none;
}

.music-controls input[type="range"]::-webkit-slider-thumb {
	-webkit-appearance: none;
	appearance: none;
	width: 15px;
	height: 15px;
	background: #fff;
	border-radius: 50%;
	cursor: pointer;
}

.music-controls input[type="range"]::-moz-range-thumb {
	width: 15px;
	height: 15px;
	background: #fff;
	border-radius: 50%;
	cursor: pointer;
}

.color-picker input[type="color"] {
	width: 30px;
	height: 30px;
	border: none;
	cursor: pointer;
}

:root {
	--start-color: #34d5af;
	/* 浅绿色 */
	--end-color: #8f4ddc;
	/* 浅黄绿色 */
	--music-start-color: #6fefc0;
	/* 浅绿色 */
	--music-end-color: #ddfda9;
	/* 浅黄绿色 */
}

.music-visualizer {
	position: fixed;
	bottom: 20px;
	left: 20px;
	z-index: 100;
	width: 200px;
	/* 根据需要调整大小 */
	height: 100px;
	/* 根据需要调整大小 */
	background: transparent;
	/* 设置背景为透明 */
	border-radius: 8px;
	backdrop-filter: blur(8px);
}

.music-visualizer canvas {
	width: 100%;
	height: 100%;
	background: transparent;
	/* 确保 canvas 背景透明 */
}

.switch-background-button {
	position: fixed;
	bottom: 20px;
	right: 20px;
	z-index: 200;
	/* 确保按钮在其他元素之上 */
	padding: 10px 20px;
	background: linear-gradient(to right, var(--start-color), var(--end-color));
	color: #fff;
	border: none;
	border-radius: 8px;
	cursor: pointer;
	font-size: 16px;
	transition: all 0.3s ease;
}

.switch-background-button:hover {
	background: linear-gradient(to right, var(--end-color), var(--start-color));
	transform: translateY(-2px);
}
</style>
