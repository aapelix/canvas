<script lang="ts">
	import { onMount } from 'svelte';
	import { getStroke, type Vec2 } from 'perfect-freehand';

	import { fly } from 'svelte/transition';

	let open = $state(false);

	type Point = [number, number, number];

	type Stroke = {
		id: string;
		points: Point[];
		fillColor: string;
		strokeColor: string;
		stroke: number;
		size: number;
	};

	let strokes: Stroke[] = [];
	let undone: Stroke[] = [];

	let current: Stroke | null = null;

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;

	let offsetX = 0;
	let offsetY = 0;
	let scale = 1;

	let panning = false;
	let lastX = 0;
	let lastY = 0;

	let currentFillColor = $state('#000');
	let currentStrokeColor = $state('#ff0000');
	let currentSize = $state(16);
	let currentStroke = $state(4);

	function toWorld(x: number, y: number) {
		return [(x - offsetX) / scale, (y - offsetY) / scale] as const;
	}

	function renderStroke(points: Point[], stroke: Stroke) {
		if (points.length === 1) {
			const [x, y] = points[0];

			const r = stroke.size / 2;

			return new Path2D(`
			M ${x} ${y}
			m -${r}, 0
			a ${r},${r} 0 1,0 ${r * 2},0
			a ${r},${r} 0 1,0 -${r * 2},0
		`);
		}

		const outline = getStroke(points, {
			size: stroke.size,
			thinning: 0.5,
			smoothing: 0.5,
			streamline: 0.5
		});

		return new Path2D(getFlatSvgPathFromStroke(outline));
	}

	function getSvgPath(points: number[][]) {
		if (points.length < 2) return '';

		const avg = (a: number, b: number) => (a + b) / 2;

		let d = `M${points[0][0]},${points[0][1]} Q${points[1][0]},${points[1][1]} ${avg(points[1][0], points[2][0])},${avg(points[1][1], points[2][1])} T`;

		for (let i = 2; i < points.length - 1; i++) {
			const a = points[i];
			const b = points[i + 1];
			d += `${avg(a[0], b[0])},${avg(a[1], b[1])} `;
		}

		return d + 'Z';
	}

	import polygonClipping from 'polygon-clipping';

	function getFlatSvgPathFromStroke(stroke: Vec2[]) {
		const faces = polygonClipping.union([stroke]);

		const d: string[] = [];

		faces.forEach((face) =>
			face.forEach((points) => {
				d.push(getSvgPath(points));
			})
		);

		return d.join(' ');
	}

	function resize() {
		canvas.width = window.innerWidth;
		canvas.height = window.innerHeight;
	}

	function draw() {
		ctx.setTransform(scale, 0, 0, scale, offsetX, offsetY);

		ctx.clearRect(-offsetX / scale, -offsetY / scale, canvas.width / scale, canvas.height / scale);

		for (const stroke of strokes) {
			const path = renderStroke(stroke.points, stroke);

			ctx.fillStyle = stroke.fillColor;

			ctx.fill(path);

			ctx.strokeStyle = stroke.strokeColor;
			ctx.lineWidth = stroke.stroke;
			ctx.stroke(path);
		}
	}

	function undo() {
		const last = strokes.pop();
		if (!last) return;

		undone.push(last);
		strokes = [...strokes];
		draw();
	}

	function redo() {
		const stroke = undone.pop();
		if (!stroke) return;

		strokes.push(stroke);
		strokes = [...strokes];
		draw();
	}

	function keydown(e: KeyboardEvent) {
		if (e.metaKey && e.key === 'z') {
			if (e.shiftKey) redo();
			else undo();
		}
	}

	function down(e: PointerEvent) {
		if (e.button === 1) {
			panning = true;
			lastX = e.clientX;
			lastY = e.clientY;

			return;
		}

		const [x, y] = toWorld(e.clientX, e.clientY);

		current = {
			id: crypto.randomUUID(),
			points: [[x, y, e.pressure || 0.5]],
			fillColor: currentFillColor,
			strokeColor: currentStrokeColor,
			stroke: currentStroke,
			size: currentSize
		};

		strokes = [...strokes, current];
		undone = [];

		draw();
	}

	function move(e: PointerEvent) {
		if (panning) {
			offsetX += e.clientX - lastX;
			offsetY += e.clientY - lastY;

			lastX = e.clientX;
			lastY = e.clientY;

			draw();

			canvas.style.cursor = 'grabbing';
			return;
		}

		canvas.style.cursor = 'crosshair';

		if (!current) return;

		const [x, y] = toWorld(e.clientX, e.clientY);

		current.points = [...current.points, [x, y, e.pressure || 0.5]];

		draw();
	}

	function up() {
		current = null;
		panning = false;
	}

	function wheel(e: WheelEvent) {
		e.preventDefault();

		const zoomIntensity = 0.001;
		const delta = -e.deltaY;

		const mouseX = e.clientX;
		const mouseY = e.clientY;

		const before = toWorld(mouseX, mouseY);

		const newScale = Math.min(20, Math.max(0.05, scale * (1 + delta * zoomIntensity)));
		scale = newScale;

		const afterX = (mouseX - offsetX) / scale;
		const afterY = (mouseY - offsetY) / scale;

		offsetX += (afterX - before[0]) * scale;
		offsetY += (afterY - before[1]) * scale;

		draw();
	}

	onMount(() => {
		resize();
		ctx = canvas.getContext('2d')!;
		window.addEventListener('resize', resize);
	});
</script>

<svelte:window on:keydown={keydown} />
<canvas
	bind:this={canvas}
	onpointerdown={down}
	onpointermove={move}
	onpointerup={up}
	onwheel={wheel}
	class="fixed inset-0 touch-none"
></canvas>

<button
	onclick={() => (open = !open)}
	class="fixed top-3 right-3 z-10 flex h-10 w-10 cursor-pointer items-center justify-center rounded-xl border border-gray-200 bg-white/90 shadow hover:bg-white"
	aria-label="Settings"
>
	<svg
		xmlns="http://www.w3.org/2000/svg"
		width="24"
		height="24"
		viewBox="0 0 24 24"
		fill="none"
		stroke="currentColor"
		stroke-width="2"
		stroke-linecap="round"
		stroke-linejoin="round"
		class="lucide lucide-settings-icon lucide-settings"
		><path
			d="M9.671 4.136a2.34 2.34 0 0 1 4.659 0 2.34 2.34 0 0 0 3.319 1.915 2.34 2.34 0 0 1 2.33 4.033 2.34 2.34 0 0 0 0 3.831 2.34 2.34 0 0 1-2.33 4.033 2.34 2.34 0 0 0-3.319 1.915 2.34 2.34 0 0 1-4.659 0 2.34 2.34 0 0 0-3.32-1.915 2.34 2.34 0 0 1-2.33-4.033 2.34 2.34 0 0 0 0-3.831A2.34 2.34 0 0 1 6.35 6.051a2.34 2.34 0 0 0 3.319-1.915"
		/><circle cx="12" cy="12" r="3" /></svg
	>
</button>

{#if open}
	<div
		transition:fly={{ x: 300, duration: 200 }}
		class="fixed top-16 right-3 z-10 flex w-56 flex-col gap-3 rounded-xl border border-gray-200 bg-white/95 p-3 shadow-lg"
	>
		<div class="font-medium">Brush</div>

		<label class="flex items-center justify-between text-sm">
			Fill
			<input
				type="color"
				bind:value={currentFillColor}
				class="aspect-square h-9 w-8 rounded-full border-none"
			/>
		</label>

		<label class="flex items-center justify-between text-sm">
			Stroke
			<input
				type="color"
				bind:value={currentStrokeColor}
				class="aspect-square h-9 w-8 rounded-full border-none"
			/>
		</label>

		<label class="flex flex-col gap-1 text-sm">
			Size
			<input type="range" min="1" max="100" bind:value={currentSize} />
		</label>

		<label class="flex flex-col gap-1 text-sm">
			Line width
			<input type="range" min="1" max="20" bind:value={currentStroke} />
		</label>

		<button
			onclick={() => {
				strokes = [];
				undone = [];
				draw();
			}}
			class="cursor-pointer rounded-full bg-red-600 px-3 py-1 text-sm text-white shadow hover:bg-red-700"
		>
			Clear
		</button>
	</div>
{/if}

<style>
	canvas {
		display: block;
	}
</style>
