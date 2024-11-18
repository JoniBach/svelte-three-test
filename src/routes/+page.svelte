<script>
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';

	let scene, camera, renderer, yeti, keys, gridHelper, mixer;
	let walkAction, idleAction, jumpIdleAction, jumpLandAction;

	const moveSpeed = 0.05;
	const jumpHeight = 2; // Maximum height of the jump
	const jumpSpeed = 0.1; // Speed of jump progression
	let isJumping = false; // Is the Yeti in the air?
	let jumpProgress = 0; // Progress of the jump (0 to 1)
	let zoomDistance = 10; // Camera zoom distance
	const minZoom = 5; // Min zoom distance
	const maxZoom = 30; // Max zoom distance

	function init() {
		scene = new THREE.Scene();

		camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
		camera.position.set(0, 5, zoomDistance);

		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);
		document.body.appendChild(renderer.domElement);

		gridHelper = new THREE.GridHelper(1000, 100, 0x888888, 0x444444);
		gridHelper.position.y = 0;
		scene.add(gridHelper);

		const loader = new GLTFLoader();
		loader.load(
			'/Yeti.gltf',
			(gltf) => {
				yeti = gltf.scene;
				yeti.scale.set(0.5, 0.5, 0.5);
				yeti.position.set(0, 0, 0);
				scene.add(yeti);

				mixer = new THREE.AnimationMixer(yeti);

				const animations = gltf.animations;
				console.log(
					'Available animations:',
					animations.map((anim) => anim.name)
				);

				// Assign animations
				idleAction = mixer.clipAction(animations.find((a) => a.name === 'Idle'));
				walkAction = mixer.clipAction(animations.find((a) => a.name === 'Walk'));
				jumpIdleAction = mixer.clipAction(animations.find((a) => a.name === 'Jump_Idle'));
				jumpLandAction = mixer.clipAction(animations.find((a) => a.name === 'Jump_Land'));

				idleAction?.play();
			},
			undefined,
			(error) => {
				console.error('Error loading GLTF:', error);
			}
		);

		const light = new THREE.DirectionalLight(0xffffff, 1);
		light.position.set(5, 5, 5);
		scene.add(light);

		window.addEventListener('resize', onWindowResize);
		window.addEventListener('wheel', onScrollZoom);
		window.addEventListener('keydown', onKeyDown);
		window.addEventListener('keyup', onKeyUp);

		keys = {};
	}

	function onWindowResize() {
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);
	}

	function onScrollZoom(event) {
		zoomDistance = Math.min(maxZoom, Math.max(minZoom, zoomDistance + event.deltaY * 0.1));
	}

	function onKeyDown(event) {
		keys[event.key] = true;

		if (event.key === ' ') {
			event.preventDefault();
			if (!isJumping) startJump();
		}
	}

	function onKeyUp(event) {
		keys[event.key] = false;
	}

	function startJump() {
		if (!jumpIdleAction || !jumpLandAction) return;

		isJumping = true;
		jumpProgress = 0;

		// Stop other animations
		idleAction?.stop();
		walkAction?.stop();

		// Play Jump_Idle animation
		jumpIdleAction.reset().play();
		console.log('Jump started');
	}

	function handleYetiMovement() {
		if (isJumping) {
			// Update jump progress
			jumpProgress += jumpSpeed;

			// Calculate jump height using a sine wave
			const jumpValue = Math.sin(jumpProgress * Math.PI) * jumpHeight;
			yeti.position.y = jumpValue;

			// Transition to Jump_Land at the descent
			if (jumpProgress >= 0.5 && jumpIdleAction.isRunning()) {
				jumpIdleAction.stop();
				jumpLandAction.reset().play();
			}

			// End the jump
			if (jumpProgress >= 1) {
				isJumping = false;
				yeti.position.y = 0; // Reset to ground level
				jumpLandAction.stop();
				idleAction?.play();
				console.log('Jump ended');
			}

			return; // Skip walking logic while jumping
		}

		// Walking logic remains unchanged
		let isMoving = false;
		let direction = null;

		if (yeti) {
			if (keys['w']) {
				yeti.position.z -= moveSpeed;
				direction = Math.PI;
				isMoving = true;
			} else if (keys['s']) {
				yeti.position.z += moveSpeed;
				direction = 0;
				isMoving = true;
			} else if (keys['a']) {
				yeti.position.x -= moveSpeed;
				direction = -Math.PI / 2;
				isMoving = true;
			} else if (keys['d']) {
				yeti.position.x += moveSpeed;
				direction = Math.PI / 2;
				isMoving = true;
			}

			if (direction !== null) yeti.rotation.y = direction;

			// Animation transitions
			if (walkAction && idleAction) {
				if (isMoving) {
					if (!walkAction.isRunning()) {
						idleAction.stop();
						walkAction.play();
					}
				} else {
					if (!idleAction.isRunning()) {
						walkAction.stop();
						idleAction.play();
					}
				}
			}
		}
	}

	function updateCameraPosition() {
		if (yeti) {
			const offset = new THREE.Vector3(0, 5, zoomDistance);
			const yetiPosition = yeti.position.clone();
			const cameraTargetPosition = yetiPosition.add(offset);

			camera.position.lerp(cameraTargetPosition, 0.1);
			camera.lookAt(yeti.position);
		}
	}

	function animate() {
		requestAnimationFrame(animate);

		handleYetiMovement();
		updateCameraPosition();

		if (mixer) mixer.update(0.016);

		renderer.render(scene, camera);
	}

	onMount(() => {
		init();
		animate();
	});
</script>

<style>
	:global(body) {
		margin: 0;
		overflow: hidden;
	}
</style>
