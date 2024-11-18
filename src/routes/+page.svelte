<script>
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';

	let scene,
		camera,
		renderer,
		yeti,
		keys,
		gridHelper,
		mixer,
		walkAction,
		idleAction,
		jumpAction,
		idleJumpAction;

	const moveSpeed = 0.05;
	const jumpHeight = 1; // Maximum jump height
	const jumpSpeed = 0.1; // Speed of jump
	let isJumping = false; // Flag to track jumping state
	let jumpProgress = 0; // Progress of the jump (0 to 1)
	let zoomDistance = 10; // Initial camera distance
	const minZoom = 5; // Minimum zoom distance
	const maxZoom = 30; // Maximum zoom distance

	function init() {
		// Create the scene
		scene = new THREE.Scene();

		// Set up the camera
		camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
		camera.position.set(0, 5, zoomDistance);

		// Renderer setup
		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(window.innerWidth, window.innerHeight);
		document.body.appendChild(renderer.domElement);

		// Add a grid that visually extends far into the distance
		gridHelper = new THREE.GridHelper(1000, 100, 0x888888, 0x444444);
		gridHelper.position.y = 0;
		scene.add(gridHelper);

		// Load the Yeti model
		const loader = new GLTFLoader();
		loader.load(
			'/Yeti.gltf',
			(gltf) => {
				yeti = gltf.scene;
				yeti.scale.set(0.5, 0.5, 0.5);
				yeti.position.set(0, 0, 0);
				yeti.rotation.y = Math.PI;
				scene.add(yeti);

				// Set up the animation mixer and actions
				mixer = new THREE.AnimationMixer(yeti);

				// Locate animations
				const walkAnimation = gltf.animations.find((anim) => anim.name.toLowerCase() === 'walk');
				const idleAnimation = gltf.animations.find((anim) => anim.name.toLowerCase() === 'idle');
				const idleJumpAnimation = gltf.animations.find(
					(anim) => anim.name.toLowerCase() === 'idle_jump'
				);
				const jumpAnimation = gltf.animations.find((anim) => anim.name.toLowerCase() === 'jump');

				// Assign actions
				if (walkAnimation) walkAction = mixer.clipAction(walkAnimation);
				if (idleAnimation) {
					idleAction = mixer.clipAction(idleAnimation);
					idleAction.play(); // Default idle animation
				}
				if (idleJumpAnimation) idleJumpAction = mixer.clipAction(idleJumpAnimation);
				if (jumpAnimation) jumpAction = mixer.clipAction(jumpAnimation);
			},
			undefined,
			(error) => {
				console.error('Error loading Yeti model:', error);
			}
		);

		// Lighting
		const light = new THREE.DirectionalLight(0xffffff, 1);
		light.position.set(5, 5, 5);
		scene.add(light);

		// Event listeners
		window.addEventListener('resize', onWindowResize);
		window.addEventListener('wheel', onScrollZoom);
		window.addEventListener('keydown', onKeyDown);
		window.addEventListener('keyup', onKeyUp);

		// Initialize keyboard input tracking
		keys = {};
	}

	function onWindowResize() {
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);
	}

	function onScrollZoom(event) {
		// Adjust zoom distance based on scroll
		zoomDistance = Math.min(maxZoom, Math.max(minZoom, zoomDistance + event.deltaY * 0.1));
	}

	function onKeyDown(event) {
		keys[event.key] = true;

		// Handle jump on spacebar
		if (event.key === ' ' && !isJumping && idleJumpAction && jumpAction) {
			isJumping = true;
			jumpProgress = 0;
			idleAction?.stop();
			walkAction?.stop();
			idleJumpAction.reset().play();
		}
	}

	function onKeyUp(event) {
		keys[event.key] = false;
	}

	function handleYetiMovement() {
		if (isJumping) {
			// Simulate jump with a sine wave for smooth motion
			jumpProgress += jumpSpeed;
			const jumpValue = Math.sin(jumpProgress * Math.PI) * jumpHeight;
			yeti.position.y = jumpValue;

			// Transition to the Jump animation at the peak of the jump
			if (jumpProgress >= 0.5 && idleJumpAction.isRunning()) {
				idleJumpAction.stop();
				jumpAction.reset().play();
			}

			// End the jump
			if (jumpProgress >= 1) {
				isJumping = false;
				yeti.position.y = 0; // Reset to ground level
				jumpAction.stop();
				idleAction?.play(); // Return to idle
			}
			return; // Skip other movement while jumping
		}

		let isMoving = false;
		let direction = null;

		// Movement logic
		if (yeti) {
			if (keys['w'] && keys['a']) {
				yeti.position.z -= moveSpeed * Math.SQRT1_2;
				yeti.position.x -= moveSpeed * Math.SQRT1_2;
				direction = Math.PI + Math.PI / 4;
				isMoving = true;
			} else if (keys['w'] && keys['d']) {
				yeti.position.z -= moveSpeed * Math.SQRT1_2;
				yeti.position.x += moveSpeed * Math.SQRT1_2;
				direction = Math.PI - Math.PI / 4;
				isMoving = true;
			} else if (keys['s'] && keys['a']) {
				yeti.position.z += moveSpeed * Math.SQRT1_2;
				yeti.position.x -= moveSpeed * Math.SQRT1_2;
				direction = -Math.PI / 4;
				isMoving = true;
			} else if (keys['s'] && keys['d']) {
				yeti.position.z += moveSpeed * Math.SQRT1_2;
				yeti.position.x += moveSpeed * Math.SQRT1_2;
				direction = Math.PI / 4;
				isMoving = true;
			} else if (keys['w']) {
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

			// Rotate the Yeti
			if (direction !== null) {
				yeti.rotation.y = direction;
			}

			// Handle animations
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
