<script lang="ts">
	import {
		copyText,
		createGradientCanvas,
		getCanvasEventXY,
		hexToHsl,
		hexToRGBA,
		isValidHexColor,
		rgbaToHex
	} from './utils';
	import { onMount, createEventDispatcher } from 'svelte';
	import { browser } from '$app/environment';

	export let color = '#0000ff';
	export let copyString = 'Copié !';

	const dispatch = createEventDispatcher();
	let canvas: HTMLCanvasElement;

	let ctx: CanvasRenderingContext2D;
	let isGradientDragging: boolean = false;
	let rgba: { r: number; g: number; b: number; a: number } = { r: 255, g: 255, b: 255, a: 1 };
	let oldColor: string;
	let isCopied: boolean = false;
	let palettes: string[] = ["#000000","#ffffff","#ff0000","#00ff00","#0000ff","#ffff00","#00ffff","#ff00ff","#6a82fb", "#1abc9c", "#16a085", "#f1c40f", "#f39c12", "#2ecc71", "#27ae60", "#e67e22", "#d35400", "#3498db", "#2980b9", "#e74c3c", "#c0392b", "#9b59b6", "#8e44ad", "#ecf0f1", "#bdc3c7", "#34495e", "#2c3e50", "#95a5a6", "#7f8c8d", "#454fbf"];
	let offscreenGradientCanvas:any;

	const KEY_COLORS = 'easy.colors.values';

	export function save(color: string) {
		if (isValidHexColor(color)) {
			setColor(color);
			saveLocalColor(color);
			selectColor();
		}
	}

	onMount(() => {
		rgba = hexToRGBA(color);
		ctx = canvas.getContext('2d', { willReadFrequently: true }) as any;
		drawColor();
		setColor(color);
	});

	function saveLocalColor(color: string) {
		if (browser && localStorage && isValidHexColor(color)) {
			let colors = getLocalColors() || [];
			colors.unshift(color);
			colors = Array.from(new Set(colors));
			colors = colors.slice(0, 10);
			localStorage.setItem(KEY_COLORS, colors.join(','));
		}
		return getLocalColors();
	}

	function getLocalColors(): string[] {
		let results: string[] = [];
		if (browser && localStorage) {
			let str = localStorage.getItem(KEY_COLORS) || '';
			results = str.split(',').filter((o) => isValidHexColor(o));
		}
		return results;
	}

	function drawColor() {
 		if(!offscreenGradientCanvas){
			offscreenGradientCanvas = createGradientCanvas(canvas.width,canvas.height);
		}
		ctx.drawImage(offscreenGradientCanvas, 0, 0);
	}

	function pickColor(event: any) {
		if (!isGradientDragging && !!!event.color) return;
		let { x, y } = getCanvasEventXY(canvas, event);
		drawColor();
		if (event.color) {
			let { r, g, b } = hexToRGBA(event.color);
			rgba = { ...rgba, ...{ r, g, b } };
		} else {
			const imageData = ctx.getImageData(x, y, 1, 1).data;
			const [r, g, b] = imageData;
			rgba = { ...rgba, ...{ r, g, b } };
		}
		drawColorSelector(x, y);
		selectColor();
	}

	function drawColorSelector(x: number = 0, y: number = 0) {
		if (ctx) {
			ctx.save();
			ctx.shadowColor = 'gray';
			ctx.shadowBlur = 3;
			ctx.shadowOffsetX = 0;
			ctx.shadowOffsetY = 0;
			ctx.strokeStyle = '#ffffff';
			ctx.lineWidth = 4;
			ctx.beginPath();
			ctx.arc(x, y, 6, 0, 2 * Math.PI);
			ctx.stroke();
			ctx.restore();
		}
	}

	function selectColor() {
		let hexColor = rgbaToHex(`rgba(${rgba.r}, ${rgba.g}, ${rgba.b}, ${rgba.a})`);
		if (hexColor !== color) {
			dispatch('color', hexColor);
		}
	}

	function estimateColorPosition(hex: string) {
		const hsl = hexToHsl(hex);
		const rgba = hexToRGBA(hex);
		const x = Math.round((hsl.h / 360) * canvas.width);
		const y = Math.round((1 - hsl.l / 100) * canvas.height);
		return { x, y, a: rgba.a };
	}

	function setColor(color: string) {
		if (color && isValidHexColor(color)) {
			const { x, y } = estimateColorPosition(color);
			pickColor({ offsetX: x, offsetY: y, color });
		} else {
			color = oldColor;
		}
	}

	function handleCopy() {
		isCopied = true;
		copyText(color);
		dispatch('copy', color);
		setTimeout(() => {
			isCopied = false;
		}, 700);
		save(color);
	}

	function handleInputFocus(e: any) {
		if (e.target) {
			setTimeout(() => {
				oldColor = color;
				e.target.select();
			}, 200);
		}
	}

	function handleEnterKey(event: any) {
		if (event.key === 'Enter' || event.keyCode === 13) {
			setColor(color);
		}
	}
</script>

<div class="color-picker-container">
	<div>
		<div class="gradient-container">
			<canvas
				bind:this={canvas}
				width="255"
				height="255"
				on:touchmove|stopPropagation|preventDefault={(e) => pickColor(e)}
				on:touchstart|stopPropagation|preventDefault={(e) => (
					(isGradientDragging = true), pickColor(e)
				)}
				on:touchend|stopPropagation|preventDefault={() => (isGradientDragging = false)}
				on:mousemove|stopPropagation|preventDefault={(e) => pickColor(e)}
				on:mousedown|stopPropagation|preventDefault={(e) => (
					(isGradientDragging = true), pickColor(e)
				)}
				on:mouseup|stopPropagation|preventDefault={() => (isGradientDragging = false)}
			/>
		</div>
		<div class="color-container">
			<button
				class="color-preview"
				style:background-color={color}
				title={color}
				on:click={handleCopy}
			>
				{#if isCopied}
					<span>{copyString}</span>
				{:else}
					<input
						name="color"
						id="color-input"
						on:click|stopPropagation|preventDefault
						bind:value={color}
						on:focus={handleInputFocus}
						on:blur={() => setColor(color)}
						on:keydown={handleEnterKey}
					/>
				{/if}
			</button>
			<div class="color-palettes">
				{#each palettes as item}
					<button
						class="btn btn-sm p-0 pallete"
						style:background-color={item}
						on:click={() => setColor(item)}>&nbsp;</button
					>
				{/each}
			</div>
		</div>
	</div>
</div>

<style lang="scss">
	.color-picker-container {
		display: flex;
		align-items: center;
		justify-content: center;
	}

	canvas {
		border: 1px solid #d1d1d1;
		cursor: crosshair;
		border-radius: 5px;
	}

	.color-container {
		display: flex;
		margin-top: 8px;
		justify-content: flex-start;
	}
	.color-preview {
		height: 64px;
		width: 64px;
		border: 1px solid #d1d1d1;
		border-radius: 3px;
		display: flex;
		align-items: center;
		justify-content: center;
		color: #ffffffaa;
		font-size: 10px;
		text-shadow: 1px 1px 1px #000;

		input {
			border: none;
			padding: 0;
			width: 100%;
			color: #fff;
			font-size: 10px;
			text-shadow: 1px 1px 1px #000;
			background-color: transparent;
			outline: none;
			text-align: center;
		}
	}
	.color-palettes {
		flex-grow: 1;
		display: flex;
		flex-wrap: wrap;
		max-width: 184px;
		padding: 0 2px;
		margin-left: 6px;
		.pallete {
			width: 16px;
			margin: 1px;
			height: 16px;
			border: 1px solid #d1d1d1;
			border-radius: 2px;
			cursor: pointer;
		}
		.pallete:hover {
			border: 1px solid #b1b1b1;
		}
	}
</style>
