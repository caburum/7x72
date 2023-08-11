<script>
	// @ts-nocheck

	import { onMount } from 'svelte';
	import Paho from 'paho-mqtt';
	import { v4 as uuidv4 } from 'uuid';

	let /** @type {HTMLCanvasElement} */ canvas,
		/** @type {CanvasRenderingContext2D} */ ctx,
		text = '',
		font = 'Ms Gothic', // Arial, Calibri, MS Gothic
		level = 63,
		yOffset = -2;

	onMount(() => {
		ctx = canvas.getContext('2d', { alpha: false, willReadFrequently: true });
	});

	const clear = new Array(7 * 72).fill(0);
	let pixelArray = clear;

	$: if (ctx) {
		ctx.clearRect(0, 0, canvas.width, canvas.height);
		ctx.imageSmoothingEnabled = false;
		ctx.filter = 'url(#remove-alpha)';
		ctx.textBaseline = 'top';
		ctx.textAlign = 'left';
		ctx.font = `10px ${font}`;
		ctx.fontKerning = 'normal';
		ctx.fillStyle = '#fff';
		ctx.fillText(text, 0, yOffset);
		let data = ctx.getImageData(0, 0, canvas.width, canvas.height);
		for (let i = 0; i < data.data.length; i += 4) {
			pixelArray[Math.floor(i / 4)] = data.data[i] > level && data.data[i + 1] > level && data.data[i + 2] > level ? 1 : 0;
		}
		update();
	}

	function update() {
		if (!ctx) return;
		let data = ctx.createImageData(canvas.width, canvas.height);
		for (let i = 0; i < data.data.length; i += 4) {
			let color = pixelArray[i / 4] ? 255 : 0;
			data.data[i] = color;
			data.data[i + 1] = color;
			data.data[i + 2] = color;
			data.data[i + 3] = 255;
		}
		ctx.putImageData(data, 0, 0);
	}

	// let fonts = [];
	// // prettier-ignore
	// const fontCheck = new Set([
	// 	// Windows 10
	// 	'Arial', 'Arial Black', 'Bahnschrift', 'Calibri', 'Cambria', 'Cambria Math', 'Candara', 'Comic Sans MS', 'Consolas', 'Constantia', 'Corbel', 'Courier New', 'Ebrima', 'Franklin Gothic Medium', 'Gabriola', 'Gadugi', 'Georgia', 'HoloLens MDL2 Assets', 'Impact', 'Ink Free', 'Javanese Text', 'Leelawadee UI', 'Lucida Console', 'Lucida Sans Unicode', 'Malgun Gothic', 'Marlett', 'Microsoft Himalaya', 'Microsoft JhengHei', 'Microsoft New Tai Lue', 'Microsoft PhagsPa', 'Microsoft Sans Serif', 'Microsoft Tai Le', 'Microsoft YaHei', 'Microsoft Yi Baiti', 'MingLiU-ExtB', 'Mongolian Baiti', 'MS Gothic', 'MV Boli', 'Myanmar Text', 'Nirmala UI', 'Palatino Linotype', 'Segoe MDL2 Assets', 'Segoe Print', 'Segoe Script', 'Segoe UI', 'Segoe UI Historic', 'Segoe UI Emoji', 'Segoe UI Symbol', 'SimSun', 'Sitka', 'Sylfaen', 'Symbol', 'Tahoma', 'Times New Roman', 'Trebuchet MS', 'Verdana', 'Webdings', 'Wingdings', 'Yu Gothic',
	// 	// macOS
	// 	'American Typewriter', 'Andale Mono', 'Arial', 'Arial Black', 'Arial Narrow', 'Arial Rounded MT Bold', 'Arial Unicode MS', 'Avenir', 'Avenir Next', 'Avenir Next Condensed', 'Baskerville', 'Big Caslon', 'Bodoni 72', 'Bodoni 72 Oldstyle', 'Bodoni 72 Smallcaps', 'Bradley Hand', 'Brush Script MT', 'Chalkboard', 'Chalkboard SE', 'Chalkduster', 'Charter', 'Cochin', 'Comic Sans MS', 'Copperplate', 'Courier', 'Courier New', 'Didot', 'DIN Alternate', 'DIN Condensed', 'Futura', 'Geneva', 'Georgia', 'Gill Sans', 'Helvetica', 'Helvetica Neue', 'Herculanum', 'Hoefler Text', 'Impact', 'Lucida Grande', 'Luminari', 'Marker Felt', 'Menlo', 'Microsoft Sans Serif', 'Monaco', 'Noteworthy', 'Optima', 'Palatino', 'Papyrus', 'Phosphate', 'Rockwell', 'Savoye LET', 'SignPainter', 'Skia', 'Snell Roundhand', 'Tahoma', 'Times', 'Times New Roman', 'Trattatello', 'Trebuchet MS', 'Verdana', 'Zapfino',
	// ].sort());
	// document.fonts.ready.then(() => {
	// 	for (const font of fontCheck.values()) {
	// 		if (document.fonts.check(`12px "${font}"`)) fonts.push(font);
	// 	}
	// 	fonts = fonts;
	// });

	const clientId = 'app-' + uuidv4();
	const client = new Paho.Client('wss://mqtt.4hcomputers.club/mqtt', clientId);
	const channel = 'sign-draw-1';
	const channel2 = 'sign-draw-2';

	let connected = false;
	function connect() {
		client.connect({
			onSuccess: () => {
				console.log('connected');
				connected = true;
				client.subscribe(channel);
			},
			onFailure: () => {
				console.log('failed');
				connected = false;
				setTimeout(() => {
					connect();
				}, 3000);
			},
			cleanSession: true,
		});
	}

	client.onConnectionLost = (_) => {
		console.log('disconnected');
		connected = false;
		setTimeout(() => {
			connect();
		}, 1000);
	};
	connect();

	let updateIds = [];
	let lastRecieved = '';
	let firstUpdate = false;

	client.onMessageArrived = (message) => {
		if (message.destinationName === channel) {
			const data = JSON.parse(message.payloadString);
			console.log(data);
			if (updateIds.indexOf(data.updateId) === -1 && data.grid && JSON.stringify(data.grid) != JSON.stringify(pixelArray)) {
				console.log('Updating grid');
				pixelArray = [...data.grid];
				update();
				lastRecieved = JSON.stringify(pixelArray);
			}
			if (updateIds.indexOf(data.updateId) > -1) updateIds.splice(updateIds.indexOf(data.updateId), 1);
			firstUpdate = true;
		}
	};
	$: {
		if (connected && (firstUpdate || !(JSON.stringify(pixelArray) == JSON.stringify(clear)))) {
			const updateId = Math.round(Math.random() * 1000000);
			updateIds.push(updateId);
			console.log('Creating new updateId: ' + updateId);
			if (lastRecieved != JSON.stringify(pixelArray)) {
				client.publish(channel, JSON.stringify({ updateId, grid: pixelArray }), 0, true);
				client.publish(channel2, pixelArray.join(''), 0, true);
			}
		}
	}
</script>

<canvas
	bind:this={canvas}
	height="7"
	width="72"
	on:click={(e) => {
		// toggle a pixel is on or off on a canvas click
		const rect = canvas.getBoundingClientRect();
		const x = Math.floor(((e.clientX - rect.left) / (rect.right - rect.left)) * canvas.width);
		const y = Math.floor(((e.clientY - rect.top) / (rect.bottom - rect.top)) * canvas.height);

		const index = x + y * canvas.width;
		pixelArray[index] = pixelArray[index] ? 0 : 1;
		update();
	}}
/>

<input bind:value={text} type="text" />

<button
	on:click={() => {
		pixelArray = new Array(canvas.width * canvas.height).fill(false);
		update();
	}}>Clear</button
>

<button
	on:click={() => {
		localStorage.setItem('pixelArray', JSON.stringify(pixelArray));
	}}>Save to LS</button
>
<button
	on:click={() => {
		const saved = localStorage.getItem('pixelArray');
		if (saved) {
			pixelArray = JSON.parse(saved);
			update();
		}
	}}>Load from LS</button
>

<!-- <select bind:value={font}>
	{#each fonts as font}
		<option value={font}>{font}</option>
	{/each}
</select> -->

<!-- <input bind:value={level} type="range" min="0" max="255" step="5" />
{level} -->

<svg width="0" height="0" style="position: absolute; z-index: -1;">
	<defs>
		<filter id="filter" x="0" y="0" width="100%" height="100%" color-interpolation-filters="sRGB">
			<feGaussianBlur in="SourceGraphic" stdDeviation="1" result="blur" />
			<feColorMatrix
				in="blur"
				type="matrix"
				values="1 0 0 0 0
					0 1 0 0 0
					0 0 1 0 0
					0 0 0 29 -1"
				result="goo"
			/>
			<feComposite in="SourceGraphic" in2="goo" operator="atop" />

			<feComponentTransfer>
				<feFuncR type="table" tableValues="0 1" />
				<feFuncG type="table" tableValues="0 1" />
				<feFuncB type="table" tableValues="0 1" />
				<feFuncA type="table" tableValues="0 1" />
			</feComponentTransfer>

			<!-- <feGaussianBlur in="SourceGraphic" stdDeviation="0" result="blur" />
			<feColorMatrix
				in="blur"
				mode="matrix"
				values="1 0 0 0 0
					0 1 0 0 0
					1 0 1 0 0
					0 0 0 15 -8"
				result="goo"
			/>
			<feComposite in="SourceGraphic" in2="goo" operator="atop" />
			<feGaussianBlur stdDeviation="1" />

			<feColorMatrix
				type="matrix"
				values="0 0 0 0 0
							0 0 0 0 0
							0 0 0 0 0
							0 0 0 1 0"
			/> -->
		</filter>
	</defs>
</svg>

<style>
	canvas {
		image-rendering: pixelated;
		image-rendering: crisp-edges;
		width: 100%;
	}
</style>
