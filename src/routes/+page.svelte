<script lang="ts">
	import { onDestroy, onMount } from 'svelte';

	type Board = number[][];
	type Piece = number[][];

	let countdown = $state<number | null>(null);
	let gameStarted = $state<boolean>(false);

	const CONFIG = {
		COLS: 10,
		ROWS: 20,
		DROP_INTERVAL: 800,
		COLORS: ['#20B2AA', '#9370DB', '#F4A460', '#4682B4', '#CD5C5C', '#3CB371', '#DAA520'],
		PIECES: [
			[
				[
					[0, 0, 0, 0],
					[1, 1, 1, 1],
					[0, 0, 0, 0],
					[0, 0, 0, 0]
				],
				[
					[0, 1, 0, 0],
					[0, 1, 0, 0],
					[0, 1, 0, 0],
					[0, 1, 0, 0]
				]
			],
			[
				[
					[0, 1, 0],
					[1, 1, 1],
					[0, 0, 0]
				],
				[
					[0, 1, 0],
					[0, 1, 1],
					[0, 1, 0]
				],
				[
					[0, 0, 0],
					[1, 1, 1],
					[0, 1, 0]
				],
				[
					[0, 1, 0],
					[1, 1, 0],
					[0, 1, 0]
				]
			],
			[
				[
					[1, 0, 0],
					[1, 1, 1],
					[0, 0, 0]
				],
				[
					[0, 1, 1],
					[0, 1, 0],
					[0, 1, 0]
				],
				[
					[0, 0, 0],
					[1, 1, 1],
					[0, 0, 1]
				],
				[
					[0, 1, 0],
					[0, 1, 0],
					[1, 1, 0]
				]
			],
			[
				[
					[0, 0, 1],
					[1, 1, 1],
					[0, 0, 0]
				],
				[
					[0, 1, 0],
					[0, 1, 0],
					[0, 1, 1]
				],
				[
					[0, 0, 0],
					[1, 1, 1],
					[1, 0, 0]
				],
				[
					[1, 1, 0],
					[0, 1, 0],
					[0, 1, 0]
				]
			],
			[
				[
					[1, 1, 0],
					[0, 1, 1],
					[0, 0, 0]
				],
				[
					[0, 0, 1],
					[0, 1, 1],
					[0, 1, 0]
				]
			],
			[
				[
					[0, 1, 1],
					[1, 1, 0],
					[0, 0, 0]
				],
				[
					[0, 1, 0],
					[0, 1, 1],
					[0, 0, 1]
				]
			],
			[
				[
					[1, 1],
					[1, 1]
				]
			]
		]
	};

	let score = $state<number>(0);
	let gameOver = $state<boolean>(false);
	let paused = $state<boolean>(false);

	let board = $state<Board>(
		Array(CONFIG.ROWS)
			.fill(0)
			.map(() => Array(CONFIG.COLS).fill(0))
	);
	let currentPiece = $state<Piece | null>(null);
	let currentX = $state<number>(0);
	let currentY = $state<number>(0);
	let currentType = $state<number>(0);
	let currentRotation = $state<number>(0);
	let dropInterval: ReturnType<typeof setInterval>;

	let canMove = $derived(!gameOver && !paused && !countdown);

	function isGameOver() {
		return board[0].some((cell) => cell !== 0) || board[1].some((cell) => cell !== 0);
	}

	function startGameWithCountdown() {
		board = Array(CONFIG.ROWS)
			.fill(0)
			.map(() => Array(CONFIG.COLS).fill(0));
		score = 0;
		gameOver = false;
		paused = false;
		gameStarted = false;
		countdown = 3;

		const countdownInterval = setInterval(() => {
			if (countdown && countdown > 1) {
				countdown--;
			} else {
				clearInterval(countdownInterval);
				countdown = null;
				gameStarted = true;
				resetGame();
			}
		}, 1000);
	}

	function newPiece() {
		currentType = Math.floor(Math.random() * CONFIG.PIECES.length);
		currentRotation = 0;
		currentPiece = CONFIG.PIECES[currentType][currentRotation];
		currentX = Math.floor((CONFIG.COLS - currentPiece[0].length) / 2);
		currentY = 0;

		if (!isValid(0, 0, currentPiece) || isGameOver()) {
			gameOver = true;
			clearInterval(dropInterval);
		}
	}

	function rotate() {
		const nextRotation = (currentRotation + 1) % CONFIG.PIECES[currentType].length;
		const nextPiece = CONFIG.PIECES[currentType][nextRotation];
		for (const kick of [0, -1, 1, -2, 2]) {
			if (isValid(kick, 0, nextPiece)) {
				currentPiece = nextPiece;
				currentRotation = nextRotation;
				currentX += kick;
				return;
			}
		}
	}

	function isValid(offsetX: number, offsetY: number, piece: Piece | null) {
		if (!piece) return false;
		for (let y = 0; y < piece.length; y++) {
			for (let x = 0; x < piece[y].length; x++) {
				if (!piece[y][x]) continue;
				const newX = currentX + x + offsetX;
				const newY = currentY + y + offsetY;
				if (newX < 0 || newX >= CONFIG.COLS || newY >= CONFIG.ROWS) return false;
				if (newY < 0) continue;
				if (board[newY][newX]) return false;
			}
		}
		return true;
	}

	function merge() {
		if (!currentPiece) return;
		currentPiece.forEach((row, y) => {
			row.forEach((value, x) => {
				if (value) {
					board[currentY + y][currentX + x] = currentType + 1;
				}
			});
		});
		board = [...board];
	}

	function clearLines() {
		let linesCleared = 0;
		for (let y = CONFIG.ROWS - 1; y >= 0; y--) {
			if (board[y].every((cell) => cell !== 0)) {
				board.splice(y, 1);
				board.unshift(Array(CONFIG.COLS).fill(0));
				linesCleared++;
				y++;
			}
		}
		if (linesCleared > 0) {
			score += [0, 100, 300, 500, 800][linesCleared];
			board = [...board];
		}
	}

	function drop() {
		if (!canMove) return;
		if (!isValid(0, 1, currentPiece)) {
			merge();
			clearLines();
			newPiece();
		} else {
			currentY++;
		}
	}

	function hardDrop() {
		if (!canMove) return;
		while (isValid(0, 1, currentPiece)) {
			currentY++;
			score += 2;
		}
		drop();
	}

	function moveLeft() {
		if (canMove && isValid(-1, 0, currentPiece)) currentX--;
	}

	function moveRight() {
		if (canMove && isValid(1, 0, currentPiece)) currentX++;
	}

	function handleKeydown(e: KeyboardEvent) {
		if (countdown || (!gameStarted && !gameOver)) return;

		if (e.code === 'KeyP') {
			paused = !paused;
			return;
		}

		if (!canMove) return;

		switch (e.code) {
			case 'ArrowLeft':
				moveLeft();
				break;
			case 'ArrowRight':
				moveRight();
				break;
			case 'ArrowDown':
				drop();
				break;
			case 'ArrowUp':
			case 'KeyR':
				rotate();
				break;
			case 'Space':
				hardDrop();
				break;
		}
	}

	function resetGame() {
		clearInterval(dropInterval);
		dropInterval = setInterval(drop, CONFIG.DROP_INTERVAL);
		board = Array(CONFIG.ROWS)
			.fill(0)
			.map(() => Array(CONFIG.COLS).fill(0));
		score = 0;
		gameOver = false;
		paused = false;
		newPiece();
	}

	onMount(() => {
		window.addEventListener('keydown', handleKeydown);
		startGameWithCountdown();
	});

	onDestroy(() => {
		window.removeEventListener('keydown', handleKeydown);
		clearInterval(dropInterval);
	});
</script>

<div class="game-container">
	<div class="game-area">
		<div class="score">Score: {score}</div>
		<div class="board">
			{#each board as row, y}
				{#each row as cell, x}
					<div
						class="cell"
						style="background-color: {cell
							? CONFIG.COLORS[cell - 1]
							: currentPiece &&
								  currentPiece[y - currentY]?.[x - currentX] &&
								  y >= currentY &&
								  x >= currentX &&
								  y < currentY + currentPiece.length &&
								  x < currentX + currentPiece[0].length
								? CONFIG.COLORS[currentType]
								: ''}"
					></div>
				{/each}
			{/each}
		</div>
		{#if countdown}
			<div class="overlay">
				<div class="countdown">{countdown}</div>
			</div>
		{/if}
		{#if gameOver}
			<div class="overlay">
				<div class="game-over">Game Over!</div>
				<button onclick={startGameWithCountdown}>New Game</button>
			</div>
		{/if}
	</div>
	<div class="controls">
		<h2>Controls</h2>
		<p><span>Rotate</span> <span class="key">R</span></p>
		<p><span>Left</span> <span class="key">←</span></p>
		<p><span>Right</span> <span class="key">→</span></p>
		<p><span>Soft Drop</span> <span class="key">↓</span></p>
		<p><span>Hard Drop</span> <span class="key">Space</span></p>
		<p><span>Pause</span> <span class="key">P</span></p>
	</div>
</div>

<style>
	.game-container {
		display: flex;
		flex-direction: row;
		align-items: flex-start;
		gap: 30px;
		padding: 15px;
		font-family: Arial, sans-serif;
		background-color: #1a1a1a;
		color: white;
		justify-content: center;
		height: 90%;
		position: relative;
	}

	.board {
		display: grid;
		grid-template-rows: repeat(20, 18px);
		grid-template-columns: repeat(10, 18px);
		gap: 1px;
		background-color: #333;
		border: 2px solid #444;
		padding: 5px;
		border-radius: 4px;
	}

	.cell {
		background-color: #1a1a1a;
		border-radius: 2px;
		transition: background-color 0.1s;
	}

	.score {
		font-size: 24px;
		font-weight: bold;
		color: #20b2aa;
		text-shadow: 0 0 5px rgba(32, 178, 170, 0.3);
		margin-bottom: 20px;
	}

	.controls {
		text-align: left;
		color: #aaa;
		background-color: #222;
		padding: 20px;
		border-radius: 8px;
		min-width: 180px;
	}

	.controls p {
		margin: 12px 0;
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.key {
		display: inline-block;
		background-color: #2a2a2a;
		padding: 4px 8px;
		border-radius: 4px;
		font-family: monospace;
		min-width: 20px;
		text-align: center;
		border: 1px solid #444;
	}

	button {
		padding: 12px 24px;
		font-size: 18px;
		cursor: pointer;
		background: #20b2aa;
		color: white;
		border: none;
		border-radius: 4px;
		transition: all 0.2s;
		box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
	}

	button:hover {
		background: #1a9090;
		transform: translateY(-1px);
	}
	button:active {
		transform: translateY(1px);
	}

	.overlay {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.85);
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		gap: 20px;
	}

	.countdown {
		font-size: 72px;
		font-weight: bold;
		color: #20b2aa;
		text-shadow: 0 0 20px rgba(32, 178, 170, 0.5);
		animation: pulse 1s infinite;
	}

	.game-over {
		font-size: 48px;
		font-weight: bold;
		color: #cd5c5c;
		text-shadow: 0 0 20px rgba(205, 92, 92, 0.5);
		animation: shake 0.5s;
	}

	@keyframes pulse {
		0% {
			transform: scale(1);
		}
		50% {
			transform: scale(1.2);
		}
		100% {
			transform: scale(1);
		}
	}

	@keyframes shake {
		0% {
			transform: translateX(0);
		}
		25% {
			transform: translateX(-10px);
		}
		50% {
			transform: translateX(10px);
		}
		75% {
			transform: translateX(-10px);
		}
		100% {
			transform: translateX(0);
		}
	}
</style>
