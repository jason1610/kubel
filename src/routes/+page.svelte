<script lang="ts">
	import { gsap } from 'gsap';
	import seed from 'seed-random';
	import { onMount } from 'svelte';
	import Logo from './logo.svelte';
	import { Confetti } from 'svelte-confetti';
	import { Button } from '$lib/components/ui/button';
	import * as Dialog from '$lib/components/ui/dialog';
	import { Application, Graphics, Rectangle } from 'pixi.js';

	type Vector = {
		x: number;
		y: number;
	};

	type Piece = {
		id: string;
		color: (typeof AVAILABLE_PALETTE)[number];
	};

	const AVAILABLE_PALETTE = [
		'#d93030', // 🟥 Red
		'#33FF57', // 🟩 Green
		'#3385FF', // 🟦 Blue
		'#ffdd24', // 🟨 Yellow
		'#A020F0', // 🟪 Purple
		'#8B4513', // 🟫 Brown
		'#808080', // ⬜ Gray
		'#ff9100' // 🟧 Orange
	] as const;

	const RANDOM_EMOJIS = [
		'😎',
		'😎',
		'😎',
		'😎',
		'😎',
		'😁',
		'😁',
		'😁',
		'😃',
		'😃',
		'😃',
		'🌟',
		'🌟',
		'🌟',
		'🌟',
		'✨',
		'✨',
		'✨',
		'⚡',
		'⚡',
		'⚡',
		'⚡',
		'⚡',
		'⚡',
		'🦍', // Ultra range emojis +10 charm
		'🦧'
	];

	const MIN_COLORS = 3;

	const MIN_BOARD_SIZE = 4;
	const MAX_BOARD_SIZE = 7;

	const PIECE_SPEED: number = 0.2;

	const app = new Application();

	let palette: (typeof AVAILABLE_PALETTE)[number][] = $state([]);
	let canvas: HTMLCanvasElement;
	let startBoard: Piece[][]; // The initial starting grid
	let currentBoard: Piece[][]; // The current state of the grid
	let animationBoard: Piece[][]; // The temporary state during animation

	let moveOffset: Vector = $state({ x: 0, y: 0 });
	let isDragging: boolean = $state(false);
	let lockedAxis: 'x' | 'y' | null = null;
	let dragDirectionDelta: Vector = { x: 0, y: 0 };
	let dragStartBoardPosition: Vector = { x: 0, y: 0 };
	let dragStartScreenPosition: Vector = { x: 0, y: 0 };

	let colorCount: number[] = [];
	let timer: string = $state('');
	let moveCount: number = $state(0);
	let copied: boolean = $state(false);
	let solved: boolean = $state(false);
	let openHelp: boolean = $state(false);
	let completedColors: boolean[] = $state([]);

	$effect(() => {
		const tick = () => {
			const now = new Date();
			const nextUTC = new Date(
				Date.UTC(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate() + 1, 0, 0, 0)
			);
			const timeDiff = nextUTC.getTime() - now.getTime();

			const totalSeconds = Math.floor(timeDiff / 1000);
			const hours = Math.floor(totalSeconds / 3600);
			const minutes = Math.floor((totalSeconds % 3600) / 60);
			const seconds = totalSeconds % 60;

			timer = `${hours}h ${minutes}m ${seconds}s`;
			if (hours === 0 && minutes === 0 && seconds === 0) initBoard(true);
		};

		tick();
		const interval = setInterval(tick, 1000);

		return () => clearInterval(interval);
	});

	function getDisplayedCellSize() {
		const currentCanvasWidth = canvas.clientWidth;
		const currentCanvasHeight = canvas.clientHeight;
		const scaleX = currentCanvasWidth / app.canvas.width;
		const scaleY = currentCanvasHeight / app.canvas.height;
		const scale = (scaleX + scaleY) / 2;
		const displayedCellSize = (canvas.width / currentBoard.length) * scale;
		return displayedCellSize;
	}

	function gridToScreen(pos: Vector) {
		const displayedCellSize = getDisplayedCellSize();
		const canvasBounds = canvas.getBoundingClientRect();
		const canvasOffsetX = canvasBounds.left;
		const canvasOffsetY = canvasBounds.top;
		const pixelX = pos.x * displayedCellSize + canvasOffsetX;
		const pixelY = pos.y * displayedCellSize + canvasOffsetY;
		return { x: pixelX, y: pixelY } as Vector;
	}

	function lerp(start: number, end: number) {
		return start + (end - start) * PIECE_SPEED;
	}

	function animate() {
		const pieceSize = canvas.width / animationBoard.length;

		for (let x = 0; x < animationBoard.length; x++) {
			for (let y = 0; y < animationBoard[x].length; y++) {
				const piece = app.stage.getChildByLabel(animationBoard[x][y].id);
				if (!piece) return;

				const newScreenPosition: Vector = {
					x: x * pieceSize,
					y: y * pieceSize
				};

				if (
					isDragging &&
					!lockedAxis &&
					moveOffset.x === 0 &&
					moveOffset.y === 0 &&
					dragStartBoardPosition.x === x &&
					dragStartBoardPosition.y === y &&
					currentBoard[dragStartBoardPosition.x][dragStartBoardPosition.y].id ===
						animationBoard[x][y].id
				) {
					// Animate the selected piece
					newScreenPosition.x += dragDirectionDelta.x * 0.2;
					newScreenPosition.y += dragDirectionDelta.y * 0.2;
					piece.zIndex = 1;
					piece.position.set(
						lerp(piece.position.x, newScreenPosition.x),
						lerp(piece.position.y, newScreenPosition.y)
					);
				} else if (isDragging && lockedAxis === 'y' && x === dragStartBoardPosition.x) {
					// Animate the moving col
					piece.zIndex = 0;
					newScreenPosition.y += dragDirectionDelta.y * PIECE_SPEED;
					piece.position.set(
						lerp(piece.position.x, newScreenPosition.x),
						lerp(piece.position.y, newScreenPosition.y)
					);
				} else if (isDragging && lockedAxis === 'x' && y === dragStartBoardPosition.y) {
					// Animate moving row
					piece.zIndex = 0;
					newScreenPosition.x += dragDirectionDelta.x * PIECE_SPEED;
					piece.position.set(
						lerp(piece.position.x, newScreenPosition.x),
						lerp(piece.position.y, newScreenPosition.y)
					);
				} else {
					// Animate the rest
					piece.zIndex = 0;
					piece.position.set(
						lerp(piece.position.x, newScreenPosition.x),
						lerp(piece.position.y, newScreenPosition.y)
					);
				}
			}
		}
	}

	function floodFill(pos: Vector, targetColor: string) {
		let count = 0;
		let queue = [pos];
		let visited: string[] = []; // Visited IDS

		while (queue.length > 0) {
			let { x, y } = queue.shift()!;
			if (visited.includes(x + '-' + y)) continue;
			visited.push(x + '-' + y);
			count++;

			if (
				x > 0 &&
				currentBoard[x - 1][y] !== null &&
				currentBoard[x - 1][y].color === targetColor
			) {
				queue.push({ x: x - 1, y });
			}
			if (
				x < currentBoard.length - 1 &&
				currentBoard[x + 1][y] !== null &&
				currentBoard[x + 1][y].color === targetColor
			) {
				queue.push({ x: x + 1, y });
			}
			if (
				y > 0 &&
				currentBoard[x][y - 1] !== null &&
				currentBoard[x][y - 1].color === targetColor
			) {
				queue.push({ x, y: y - 1 });
			}
			if (
				y < currentBoard[0].length - 1 &&
				currentBoard[x][y + 1] !== null &&
				currentBoard[x][y + 1].color === targetColor
			) {
				queue.push({ x, y: y + 1 });
			}
		}
		return count;
	}

	function findColorId(color: string) {
		for (let x = 0; x < currentBoard.length; x++) {
			for (let y = 0; y < currentBoard[0].length; y++) {
				if (currentBoard[x][y].color === color) return { x, y } as Vector;
			}
		}
		return { x: 0, y: 0 } as Vector;
	}

	function checkForWin() {
		for (let i = 0; i < palette.length; i++) {
			let { x, y } = findColorId(palette[i]);
			if (floodFill({ x, y }, palette[i]) !== colorCount[i]) {
				completedColors[i] = false;
			} else {
				completedColors[i] = true;
			}
		}

		solved = completedColors.includes(false) ? false : true;
	}

	function animateSelectPiece(piece: Graphics, color: string) {
		const pieceSize = canvas.width / currentBoard.length;
		const border = { radius: 0 };

		gsap.to(border, {
			duration: PIECE_SPEED,
			radius: pieceSize / 2,
			onUpdate: () => {
				piece.clear();
				piece.roundRect(0, 0, pieceSize, pieceSize, border.radius);
				piece.fill(color);
			}
		});
	}

	function animateDeselectedPiece(piece: Graphics, color: string) {
		const pieceSize = canvas.width / currentBoard.length;
		const border = { radius: pieceSize / 2 };

		gsap.to(border, {
			duration: PIECE_SPEED,
			radius: 0,
			onUpdate: () => {
				piece.clear();
				piece.roundRect(0, 0, pieceSize, pieceSize, border.radius);
				piece.fill(color);
			}
		});
	}

	function getIdPosition(id: string) {
		for (let x = 0; x < currentBoard.length; x++) {
			for (let y = 0; y < currentBoard[0].length; y++) {
				if (currentBoard[x][y].id === id) return { x, y };
			}
		}
		return { x: 0, y: 0 } as Vector;
	}

	// Sets all variables for initiating a move
	function handlePointerDown(e: PointerEvent, id: string) {
		if (solved) return;
		isDragging = true;
		lockedAxis = null;

		const piecePosition = getIdPosition(id);
		if (!piecePosition) return;
		dragStartBoardPosition = piecePosition;

		const piecePixelPosition = gridToScreen(piecePosition);
		if (!piecePixelPosition) return;
		const displayedCellSize = getDisplayedCellSize();
		if (!displayedCellSize) return;
		const pieceCenterX = piecePixelPosition.x + displayedCellSize / 2;
		const pieceCenterY = piecePixelPosition.y + displayedCellSize / 2;
		dragStartScreenPosition.x = pieceCenterX;
		dragStartScreenPosition.y = pieceCenterY;

		const dx = e.clientX - dragStartScreenPosition.x;
		const dy = e.clientY - dragStartScreenPosition.y;
		dragDirectionDelta = { x: dx, y: dy };

		const piece = app.stage.getChildByLabel(id);
		if (piece) {
			animateSelectPiece(piece as Graphics, currentBoard[piecePosition.x][piecePosition.y].color);
		}
	}

	// Continuously calculate the current move and update the animated board
	function handlePointerMove(e: PointerEvent) {
		if (!isDragging) return;
		const dx = e.clientX - dragStartScreenPosition.x;
		const dy = e.clientY - dragStartScreenPosition.y;
		const displayedCellSize = getDisplayedCellSize();
		const hoveredGridOffset: Vector = {
			x: Math.round(dx / displayedCellSize),
			y: Math.round(dy / displayedCellSize)
		};

		if (hoveredGridOffset.x === 0 && hoveredGridOffset.y === 0) {
			lockedAxis = null;
		} else if (!lockedAxis) {
			lockedAxis = Math.abs(dx) > Math.abs(dy) ? 'x' : 'y';
		}

		moveOffset = {
			x: lockedAxis === 'x' ? hoveredGridOffset.x : 0,
			y: lockedAxis === 'y' ? hoveredGridOffset.y : 0
		};

		dragDirectionDelta = {
			x:
				(((Math.abs(dx) + displayedCellSize / 2) % displayedCellSize) - displayedCellSize / 2) *
				(dx === 0 ? 1 : Math.sign(dx)),
			y:
				(((Math.abs(dy) + displayedCellSize / 2) % displayedCellSize) - displayedCellSize / 2) *
				(dy === 0 ? 1 : Math.sign(dy))
		};

		for (let x = 0; x < animationBoard.length; x++) {
			for (let y = 0; y < animationBoard[x].length; y++) {
				if (
					(lockedAxis === 'x' && y === dragStartBoardPosition.y) ||
					(lockedAxis === 'y' && x === dragStartBoardPosition.x)
				) {
					const newX =
						(((x + moveOffset.x + animationBoard.length) % animationBoard.length) +
							animationBoard.length) %
						animationBoard.length;
					const newY =
						(((y + moveOffset.y + animationBoard.length) % animationBoard.length) +
							animationBoard.length) %
						animationBoard.length;
					animationBoard[newX][newY] = { ...currentBoard[x][y] };
				} else {
					animationBoard[x][y] = { ...currentBoard[x][y] };
				}
			}
		}
	}

	// Take the state of the animation board and apply to the current board
	function handlePointerUp() {
		if (!isDragging) return;
		if (moveOffset.x !== 0 || moveOffset.y !== 0) moveCount++;

		isDragging = false;
		lockedAxis = null;

		moveOffset.x = 0;
		moveOffset.y = 0;

		const piece = app.stage.getChildByLabel(
			currentBoard[dragStartBoardPosition.x][dragStartBoardPosition.y].id
		);
		if (piece) {
			animateDeselectedPiece(
				piece as Graphics,
				currentBoard[dragStartBoardPosition.x][dragStartBoardPosition.y].color
			);
		}

		currentBoard = structuredClone(animationBoard);
		checkForWin();
	}

	async function initBoard(restart: boolean = false) {
		// Reset all stats
		moveCount = 0;
		solved = false;

		// First calc what the initial map looks like
		const now = new Date().toISOString().split('T')[0];
		// const now = new Date().toISOString(); // For debugging, gives a new board every second

		const currentSize =
			MIN_BOARD_SIZE + Math.floor((MAX_BOARD_SIZE - MIN_BOARD_SIZE + 1) * seed(now + 'size')());

		const shuffledPalette = [...AVAILABLE_PALETTE]
			.map((color, i) => ({ color, rand: seed(now + i.toString())() }))
			.sort((a, b) => a.rand - b.rand)
			.map((item) => item.color);
		const numberOfColors =
			MIN_COLORS + Math.floor((AVAILABLE_PALETTE.length - MIN_COLORS + 1) * seed(now + 'colors')());

		startBoard = Array.from({ length: currentSize }, () => Array(currentSize).fill(null));
		palette = shuffledPalette.slice(0, numberOfColors);
		colorCount = new Array(palette.length).fill(0);
		completedColors = new Array(colorCount.length).fill(false);

		const allPositions = [];
		for (let x = 0; x < currentSize; x++) {
			for (let y = 0; y < currentSize; y++) {
				allPositions.push({ x, y });
			}
		}

		const shuffledPositions = allPositions.sort(() => seed(now + 'shuffle')() - 0.5);

		for (let i = 0; i < palette.length; i++) {
			const { x, y } = shuffledPositions[i];
			startBoard[x][y] = { id: `${x}-${y}`, color: palette[i] };
			colorCount[i]++;
		}

		for (let x = 0; x < currentSize; x++) {
			for (let y = 0; y < currentSize; y++) {
				if (!startBoard[x][y]) {
					const randomColorIndex = Math.floor(seed(now + `${x}-${y}`)() * palette.length);
					startBoard[x][y] = { id: `${x}-${y}`, color: palette[randomColorIndex] };
					colorCount[randomColorIndex]++;
				}
			}
		}

		if (colorCount.includes(0)) console.error('Board missing color from palette 💀🎨');

		currentBoard = structuredClone(startBoard);
		animationBoard = structuredClone(startBoard);

		// Then init the scene
		if (!restart) {
			await app.init({
				canvas,
				backgroundAlpha: 0,
				width: 800,
				height: 800
			});
		}

		const pieceSize = canvas.width / currentBoard.length;

		if (!restart) {
			for (let x = 0; x < currentBoard.length; x++) {
				for (let y = 0; y < currentBoard[x].length; y++) {
					const piece = new Graphics();
					piece.rect(0, 0, pieceSize, pieceSize);
					piece.hitArea = new Rectangle(0, 0, pieceSize, pieceSize);
					piece.fill(startBoard[x][y].color);
					piece.position.set(x * pieceSize, y * pieceSize);
					piece.label = currentBoard[x][y].id;
					piece.eventMode = 'static';
					piece.on('pointerdown', (e) => {
						handlePointerDown(e, piece.label);
					});
					app.stage.addChild(piece);
				}
			}
		}

		if (!restart) {
			app.ticker.add(animate);
		}
	}

	function getEmojiForColor(color: (typeof AVAILABLE_PALETTE)[number]): string {
		const colorMap: Record<(typeof AVAILABLE_PALETTE)[number], string> = {
			'#d93030': '🟥',
			'#33FF57': '🟩',
			'#3385FF': '🟦',
			'#ffdd24': '🟨',
			'#A020F0': '🟪',
			'#8B4513': '🟫',
			'#808080': '⬜',
			'#ff9100': '🟧'
		};
		return colorMap[color] || '⬛';
	}

	function copyEmojiBoard() {
		const daysSinceStart = Math.floor(
			(Date.now() - new Date('2025-02-05').getTime()) / (1000 * 60 * 60 * 24)
		);
		const randomEmoji = RANDOM_EMOJIS[Math.floor(Math.random() * RANDOM_EMOJIS.length)];
		const rotatedBoard = currentBoard[0].map((_, colIndex) =>
			currentBoard.map((row) => row[colIndex])
		);
		const boardString = rotatedBoard
			.map((row) => row.map((piece) => getEmojiForColor(piece.color)).join(''))
			.join('\n');

		navigator.clipboard.writeText(
			`Kubel #${daysSinceStart} in ${moveCount} moves ${randomEmoji}\n${boardString}`
		);

		copied = true;
		setTimeout(() => {
			copied = false;
		}, 1000);
	}

	onMount(() => {
		initBoard();

		if (!localStorage.getItem('hasPlayed')) {
			localStorage.setItem('hasPlayed', 'true');
			openHelp = true;
		}
	});
</script>

<svelte:window onpointermove={handlePointerMove} onpointerup={handlePointerUp} />

<main class="flex min-h-svh flex-col overflow-x-hidden">
	<div class="mx-auto w-fit select-none font-mono sm:pt-12 {isDragging ? 'cursor-grabbing' : ''}">
		<!-- Header -->
		<div class="flex justify-center gap-2 px-4 py-4 sm:px-0">
			<div class="flex flex-1 select-auto items-end">
				<p>Kubel.io</p>
			</div>
			<div
				class="relative flex h-14 min-w-14 shrink-0 items-center justify-center border bg-muted p-4 text-4xl font-bold
			{!moveCount ? 'text-muted-foreground' : 'text-foreground'} 
			{solved ? 'bg-green-500 text-white' : ''}"
			>
				{moveCount}
				{#if (isDragging && moveOffset.x !== 0) || moveOffset.y !== 0}
					<span
						class="absolute bottom-0.5 right-0.5 text-xs font-thin leading-none text-muted-foreground"
						>+1</span
					>
				{/if}
			</div>

			<div class="flex flex-1 items-end justify-end gap-2">
				{#if moveCount > 0 && !solved}
					<Button
						size="sm"
						variant="outline"
						onclick={() => initBoard(true)}
						class="font-semibold text-muted-foreground"
					>
						Reset
					</Button>
				{/if}
				<Dialog.Root bind:open={openHelp}>
					<Dialog.Trigger>
						{#snippet child({ props })}
							<Button
								size="sm"
								{...props}
								variant="outline"
								class="font-semibold text-muted-foreground"
							>
								?
							</Button>
						{/snippet}
					</Dialog.Trigger>
					<Dialog.Content class="font-mono sm:max-w-80">
						<Logo class="mx-auto size-8" />
						<Dialog.Header class="sm:text-center">
							<Dialog.Title>Kubel</Dialog.Title>
							<Dialog.Description>The daily 2D Rubik's cube challenge</Dialog.Description>
						</Dialog.Header>
						<div class="text-center">
							<p class="font-sans">
								Click and drag the pieces to move them. Win the game by moving all the pieces of the
								same color next to each other. Use as few moves as possible.
							</p>
							<br />
							<p class="font-sans">New puzzle every day!</p>
						</div>
						<Button class="w-full font-bold" onclick={() => (openHelp = false)}>PLAY</Button>
					</Dialog.Content>
				</Dialog.Root>
			</div>
		</div>

		<!-- Board -->
		<div
			class="mx-auto aspect-square w-full max-w-sm sm:size-96
			{!isDragging && !solved ? 'cursor-grab' : ''}"
		>
			<canvas
				bind:this={canvas}
				class="size-full select-none"
				oncontextmenu={(e) => e.preventDefault()}
			></canvas>
		</div>

		<!-- Solved -->
		{#if solved}
			<div class="relative mt-4 bg-green-500 px-4 py-2 sm:px-0">
				<div class="absolute left-0 top-0 hidden sm:block">
					<Confetti colorArray={palette} delay={[100, 2000]} cone x={[-1, -2.5]} y={[0.25, 0.75]} />
				</div>
				<div class="absolute right-0 top-0 hidden sm:block">
					<Confetti colorArray={palette} delay={[100, 2000]} cone x={[1, 2.5]} y={[0.25, 0.75]} />
				</div>
				<p class="text-center text-2xl font-bold text-white">Solved!</p>
			</div>
			<div class="mt-4 flex">
				<Button
					size="sm"
					onclick={() => initBoard(true)}
					variant="outline"
					class="font-semibold focus-visible:z-10"
				>
					Play Again
				</Button>
				{#if navigator.clipboard}
					<Button
						size="sm"
						class="grow font-semibold focus-visible:z-10"
						variant="secondary"
						onclick={copyEmojiBoard}
					>
						{#if copied}
							Copied
						{:else}
							Copy Emoji Board
						{/if}
					</Button>
				{/if}
			</div>
		{/if}

		<!-- Footer -->
		<div class="flex w-full flex-wrap-reverse justify-between gap-2 px-4 py-4 sm:px-0">
			<div class="flex items-start justify-start truncate">
				<p class="truncate text-xs text-muted-foreground">New puzzle in {timer}</p>
			</div>

			<div class="flex shrink-0 items-start justify-end gap-1">
				{#each completedColors as color, i}
					<div
						style="background-color: {color ? palette[i] : 'transparent'};"
						class="size-4 border"
					></div>
				{/each}
			</div>
		</div>
	</div>

	<!-- Links -->
	<div class="mt-auto flex w-full justify-end gap-4 p-4 pt-0">
		<Button
			size="sm"
			target="_blank"
			variant="outline"
			class="h-7 text-muted-foreground"
			href="https://github.com/jason1610/kubel">GitHub</Button
		>
		<Button
			size="sm"
			target="_blank"
			variant="outline"
			class="h-7 text-muted-foreground"
			href="https://bsky.app/profile/1610.io">BlueSky</Button
		>
	</div>
</main>
