<script lang="ts">
	import { onMount } from 'svelte';
	import { createExtrusion } from 'extrude-js';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

	let canvas: HTMLCanvasElement;

	const defaults = {
		size: 20,
		length: 100,
		centerHoleDiameter: 4.19,
		slotWidth: 5.26,
		slotDepth: 1.5,
		innerWidth: 11.99,
		trapezoidBaseFromCenter: 6.34,
		cornerRadius: 1.0
	};

	let profile = $state({ ...defaults });

	type Key = keyof typeof defaults;
	const fields: { key: Key; label: string }[] = [
		{ key: 'size', label: 'Size (mm)' },
		{ key: 'length', label: 'Length (mm)' },
		{ key: 'centerHoleDiameter', label: 'Center Hole Diameter (mm)' },
		{ key: 'slotWidth', label: 'Slot Width (mm)' },
		{ key: 'slotDepth', label: 'Slot Depth (mm)' },
		{ key: 'innerWidth', label: 'Inner Width (mm)' },
		{ key: 'trapezoidBaseFromCenter', label: 'Trapezoid Base From Center (mm)' },
		{ key: 'cornerRadius', label: 'Corner Radius (mm)' }
	];

	function buildGeometry(p: typeof defaults) {
		const geometry = createExtrusion(p);
		const positions: number[] = [];
		const polys = (geometry as { polygons: { vertices: number[][] }[] }).polygons;
		for (const poly of polys) {
			const verts = poly.vertices;
			for (let i = 1; i < verts.length - 1; i++) {
				positions.push(...verts[0], ...verts[i], ...verts[i + 1]);
			}
		}
		const geo = new THREE.BufferGeometry();
		geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
		geo.computeVertexNormals();
		geo.center();
		return geo;
	}

	let mesh: THREE.Mesh | null = null;
	let edges: THREE.LineSegments | null = null;

	let applied = $state({ ...defaults });
	const dirty = $derived((Object.keys(profile) as Key[]).some((k) => profile[k] !== applied[k]));

	function apply() {
		if (!mesh || !edges) return;
		const geo = buildGeometry(profile);
		mesh.geometry.dispose();
		mesh.geometry = geo;
		edges.geometry.dispose();
		edges.geometry = new THREE.EdgesGeometry(geo, 15);
		applied = { ...profile };
	}

	function clear() {
		profile = { ...applied };
	}

	onMount(() => {
		const renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
		renderer.setPixelRatio(devicePixelRatio);

		const scene = new THREE.Scene();
		scene.background = new THREE.Color(0x1a1a1a);

		const camera = new THREE.PerspectiveCamera(
			45,
			canvas.clientWidth / canvas.clientHeight,
			0.1,
			1000
		);
		camera.position.set(80, 60, 120);
		camera.lookAt(0, 0, 0);

		const ro = new ResizeObserver(() => {
			const w = canvas.clientWidth;
			const h = canvas.clientHeight;
			renderer.setSize(w, h, false);
			camera.aspect = w / h;
			camera.updateProjectionMatrix();
		});
		ro.observe(canvas);

		const controls = new OrbitControls(camera, canvas);
		controls.enableDamping = true;

		scene.add(new THREE.AmbientLight(0xffffff, 0.4));

		const keyLight = new THREE.DirectionalLight(0xffffff, 1.5);
		keyLight.position.set(100, 100, 100);
		scene.add(keyLight);

		const fillLight = new THREE.DirectionalLight(0xffffff, 0.8);
		fillLight.position.set(-100, 50, -50);
		scene.add(fillLight);

		const rimLight = new THREE.DirectionalLight(0xffffff, 1.0);
		rimLight.position.set(0, -100, -100);
		scene.add(rimLight);

		const geo = buildGeometry(profile);
		const group = new THREE.Group();

		const mat = new THREE.MeshStandardMaterial({ color: 0x888888, metalness: 0.3, roughness: 0.5 });
		mesh = new THREE.Mesh(geo, mat);
		group.add(mesh);

		edges = new THREE.LineSegments(
			new THREE.EdgesGeometry(geo, 15),
			new THREE.LineBasicMaterial({ color: 0x000000, transparent: true, opacity: 0.3 })
		);
		group.add(edges);

		scene.add(group);

		mesh.geometry.computeBoundingBox();
		const size = new THREE.Vector3();
		mesh.geometry.boundingBox!.getSize(size);
		const dist = Math.max(size.x, size.y, size.z) * 1.8;
		controls.target.set(0, 0, 0);
		camera.position.set(dist * 0.5, dist * 0.4, dist);
		camera.lookAt(0, 0, 0);
		controls.update();

		let raf: number;
		function animate() {
			raf = requestAnimationFrame(animate);
			controls.update();
			renderer.render(scene, camera);
		}
		animate();

		return () => {
			cancelAnimationFrame(raf);
			ro.disconnect();
			renderer.dispose();
			mesh?.geometry.dispose();
			edges?.geometry.dispose();
			mesh = null;
			edges = null;
		};
	});
</script>

<svelte:head>
	<style>
		html,
		body {
			margin: 0;
			padding: 0;
			overflow: hidden;
			height: 100%;
		}
		.layout {
			display: flex;
			width: 100vw;
			height: 100vh;
			overflow: hidden;
			flex-direction: row;
		}
		.panel {
			width: 240px;
			flex-shrink: 0;
			display: flex;
			flex-direction: column;
			overflow: hidden;
			box-sizing: border-box;
			border-right: 1px solid #ccc;
		}
		.panel-scroll {
			flex: 1;
			overflow-y: auto;
			padding: 1rem;
		}
		.panel-footer {
			padding: 0.5rem 1rem;
			border-top: 1px solid #ccc;
			background: inherit;
		}
		.field {
			display: block;
			margin-bottom: 0.75rem;
		}
		.field input {
			width: 100%;
			box-sizing: border-box;
		}
		.buttons {
			display: flex;
			gap: 0.5rem;
		}
		.buttons button {
			flex: 1;
		}
		@media (max-width: 600px) {
			.layout {
				flex-direction: column-reverse;
			}
			.panel {
				width: 100%;
				height: 50vh;
				border-right: none;
				border-top: 1px solid #ccc;
			}
		}
	</style>
</svelte:head>

<div class="layout">
	<aside class="panel">
		<div class="panel-scroll">
			<h2 style="margin-top:0">Profile</h2>
			{#each fields as { key, label } (key)}
				<label class="field">
					{label}<br />
					<input
						type="number"
						step="0.01"
						value={profile[key]}
						oninput={(e) =>
							(profile = { ...profile, [key]: parseFloat((e.target as HTMLInputElement).value) })}
						style="outline: 2px solid {profile[key] !== applied[key] ? '#f59e0b' : 'grey'}"
					/>
				</label>
			{/each}
		</div>
		<div class="panel-footer">
			<div class="buttons">
				<button onclick={apply} disabled={!dirty}>Apply</button>
				<button onclick={clear} disabled={!dirty}>Clear</button>
			</div>
		</div>
	</aside>
	<canvas bind:this={canvas} style="flex:1;display:block;width:100%;height:100%;"></canvas>
</div>
