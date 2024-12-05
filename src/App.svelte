<script>
	import { onMount } from "svelte";
	import { FFmpeg } from '@ffmpeg/ffmpeg';
  import { fetchFile, toBlobURL } from '@ffmpeg/util';

	const setImageSmooth = () => {
		ctx.imageSmoothingEnabled = true;
		ctx.imageSmoothingQuality = "high";
		tempCtx.imageSmoothingEnabled = true;
		tempCtx.imageSmoothingQuality = "high";
		offscreenCanvas.imageSmoothingEnabled = true;
		offscreenCanvas.imageSmoothingQuality = "high";
	}
	
	const DPR = window.devicePixelRatio || 1

	const canvasToBlob = (blobCanvas, format = "image/png", quality = 1.0) =>
		new Promise((resolve, reject) =>
			blobCanvas?.toBlob ? blobCanvas.toBlob(resolve, format, quality) : reject("Invalid canvas.")
		);
	
	const blobURLCache = new Map();

	const loadBlobToCanvas = (blobCtx, blobCanvas, blob) =>
    new Promise((resolve, reject) => {
			if (!blobCanvas || !blob) return reject("Invalid input.");

			// Check if this blob already has an associated URL
			let objectURL = blobURLCache.get(blob);
			if (!objectURL) {
				// Create a new object URL and cache it
				objectURL = URL.createObjectURL(blob);
				blobURLCache.set(blob, objectURL);
			}

			const img = new Image();
			img.onload = () => {
				setImageSmooth();
				blobCtx.drawImage(img, 0, 0);
				resolve();
			};
			img.onerror = () => reject("Failed to load Blob.");
			img.src = objectURL;
    });
	
	let tempCanvas
	let tempCtx
	let thumbnailCanvas
	let thumbnailCtx
	let offscreenCanvas
	let offscreenCtx
	let canvas
	let ctx
	let isDrawing = false

	let timelineRef
	let workspaceRef

  let ffmpeg
  let message = ''

	let debug = ''

	let state = {
		width: 500 * DPR,
		height: 500 * DPR,
		activeShot: null,
		shots: new Set(),
		activeTimeline: {
			row: 0,
			col: 0,
		},
		timeline: Array(1) // 5 rows (layers)
			.fill(null)
			.map(() => Array(100).fill(null)), // 100 columns (frames) per row
		isPlaying: false,
		frameRate: 12,
		workspace: {
      scale: 1,
      translate: { x: 0, y: 0 },
      rotate: 0,
      transformDelta: 0,
      touchAction: '',
      touchCount: 0,
      initialTouchDistance: 0,
      initialTouchAngle: 0,
      initialTouchMidpoint: { x: 0, y: 0 },
      previousScale: 0,
      lastBrushPosition: null,
      lastTimestamp: null,
    },
		onionSkinEnabled: true,
		onionSkinFramesAfter: 2,
		onionSkinFramesBefore: 2
	};

	onMount(async () => {
		ctx = canvas.getContext('2d');

    offscreenCanvas.width = state.width;
    offscreenCanvas.height = state.height;
    offscreenCtx = offscreenCanvas.getContext('2d')

		tempCanvas.width = state.width;
    tempCanvas.height = state.height;
    tempCtx = tempCanvas.getContext('2d');

		// Generate a thumbnail using the dedicated thumbnail canvas
		const thumbnailSize = 100; // Desired thumbnail size
		const scale = thumbnailSize / Math.max(canvas.width, canvas.height);
		thumbnailCanvas = document.createElement('canvas');
		thumbnailCanvas.width = canvas.width * scale;
		thumbnailCanvas.height = canvas.height * scale;
		thumbnailCtx = thumbnailCanvas.getContext('2d');

		loadFFmpg()
		
		const stopAnimationOnVisibilityChange = () => {
			if (document.hidden && state.isPlaying) {
					stopAnimation(); // Pause animation when the tab becomes hidden
			}
		};

		const stopAnimationOnUnload = () => {
				stopAnimation(); // Ensure animation is stopped on page unload
		};

		// Attach both events
		document.addEventListener('visibilitychange', stopAnimationOnVisibilityChange);
		window.addEventListener('beforeunload', stopAnimationOnUnload);

		setTimeout(() => {
			scaleAndCenterCanvas()
		}, 0)
	});
	
	// WORKSPACE

	const scaleAndCenterCanvas = () => {
		const canvasWidth = state.width / DPR
		const canvasHeight = state.height / DPR
    const { clientWidth: workspaceWidth, clientHeight: workspaceHeight } = workspaceRef;
    
		// Define padding value
    const padding = 0 // Adjust this value as needed
  
    // Calculate the available space with padding
    const availableWidth = workspaceWidth - padding * 2
    const availableHeight = workspaceHeight - padding * 2
  
    // Calculate the scale factor to fit the canvas within the available space
    const scaleX = availableWidth / canvasWidth
    const scaleY = availableHeight / canvasHeight
    const newScale = Math.min(scaleX, scaleY)
  
    // Calculate the new translation to center the canvas with padding
    const newTranslateX = (workspaceWidth - canvasWidth * newScale) / 2
    const newTranslateY = (workspaceHeight - canvasHeight * newScale) / 2
  
    state.workspace.scale = newScale
    state.workspace.translate.x = newTranslateX
    state.workspace.translate.y = newTranslateY
  }

	const handleWheel = (e) => {
		e.preventDefault()
	
		const { left, top } = workspaceRef.getBoundingClientRect()
		const offsetX = e.clientX - left
		const offsetY = e.clientY - top
		const { scale, translate, rotate } = state.workspace
	
		if (e.ctrlKey) { // Scaling
			const deltaScale = Math.exp(-e.deltaY / 100)
			const newScale = Math.max(0.2, Math.min(2, scale * deltaScale))
			const newTranslate = {
				x: offsetX - ((offsetX - translate.x) * newScale) / scale,
				y: offsetY - ((offsetY - translate.y) * newScale) / scale,
			}
	
			state.workspace.scale = newScale
			state.workspace.translate = newTranslate
		} else if (e.altKey) { // Rotation logic
      const deltaRotate = e.deltaY * -0.1
      const newRotate = rotate + deltaRotate

      // Calculate the new translation to pivot around the cursor
      const radians = deltaRotate * (Math.PI / 180)
      const sin = Math.sin(radians)
      const cos = Math.cos(radians)

      const dx = offsetX - translate.x
      const dy = offsetY - translate.y

      const newTranslate = {
          x: offsetX - (dx * cos - dy * sin),
          y: offsetY - (dx * sin + dy * cos)
      }

      state.workspace.translate = newTranslate
      state.workspace.rotate = newRotate
    } else if (e.shiftKey) { // Translate logic
			state.workspace.translate = {
				x: translate.x - e.deltaX,
				y: translate.y - e.deltaY,
			}
		}
	}

	// GESTURE / DRAW EVENTS

	const processFrame = async () => {
		// Optionally save or process the canvas (e.g., generate thumbnails)
    const blob = await canvasToBlob(canvas);

    thumbnailCtx.clearRect(0, 0, thumbnailCanvas.width, thumbnailCanvas.height);
    thumbnailCtx.drawImage(canvas, 0, 0, thumbnailCanvas.width, thumbnailCanvas.height);

    // Convert the thumbnail canvas to a blob
    const thumbnailBlob = await canvasToBlob(thumbnailCanvas);
    const newThumbnailURL = URL.createObjectURL(thumbnailBlob);

    // Update state with the new frame and thumbnail
    if (state.activeShot) {
			if (state.activeShot.thumbnailURL) {
				// Revoke the old URL before assigning a new one
				URL.revokeObjectURL(state.activeShot.thumbnailURL);
			}
			state.activeShot.blob = blob;
			state.activeShot.thumbnailURL = newThumbnailURL; // Assign new URL
    } else {
			addShot(
				state.activeTimeline.row,
				state.activeTimeline.col,
				blob,
				newThumbnailURL
			);
    }
	}

	const onPointerStart = (e) => {
		// direct for touch
		// stylus for pencil

		if (state.isPlaying) return
		
		const touchType = e?.changedTouches?.[0]?.touchType || 'stylus'
		debug = touchType

		// if (touchType !== 'stylus') return

		const clientX = e?.clientX || e.touches[0].clientX
		const clientY = e?.clientY || e.touches[0].clientY
    const rect = canvas.getBoundingClientRect()
    const scaleX = canvas.width / rect.width
    const scaleY = canvas.height / rect.height

    const normalizedX = (clientX - rect.left) * scaleX
    const normalizedY = (clientY - rect.top) * scaleY

    isDrawing = true

		// Begin a new path on the canvas
		ctx.beginPath()
		ctx.moveTo(normalizedX, normalizedY)

		// Set line style
		ctx.lineCap = "round" // Round line caps
		ctx.lineJoin = "round" // Round line caps
		ctx.strokeStyle = "black"
		ctx.lineWidth = 15

		// Draw a point immediately on pointer down
		ctx.lineTo(normalizedX, normalizedY)
		ctx.stroke()
	}

	const onPointerMove = (e) => {
		if (state.isPlaying) return
    if (!isDrawing) return

		const clientX = e?.clientX || e.touches[0].clientX
		const clientY = e?.clientY || e.touches[0].clientY
    const rect = canvas.getBoundingClientRect();
    const scaleX = canvas.width / rect.width;
    const scaleY = canvas.height / rect.height;

    const normalizedX = (clientX - rect.left) * scaleX;
    const normalizedY = (clientY - rect.top) * scaleY;

    // Draw a line from the last point to the current point
    ctx.lineTo(normalizedX, normalizedY);
    ctx.stroke();
	}

	const onPointerEnd = async (e) => {
    if (!isDrawing) return
    isDrawing = false
    ctx.closePath()
		await processFrame()
	}
	
	// TIMELINE - EDITING

	const printColumn = async () => {
    offscreenCtx.clearRect(0, 0, offscreenCanvas.width, offscreenCanvas.height)

		// For every layer in this frame...
    for (const row of state.timeline) {
			const cell = row[state.activeTimeline.col]
			if (cell?.blob) {
				tempCtx.clearRect(0, 0, tempCanvas.width, tempCanvas.height)
				await loadBlobToCanvas(tempCtx, tempCanvas, cell.blob) // Convert blob to a temp canvas you can draw
				setImageSmooth()
				offscreenCtx.drawImage(tempCanvas, 0, 0) // draw the temp ctx for this layer onto the off screen canvas
			}
    }

    // Draw the accumulated offscreenCanvas onto the primary canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    setImageSmooth()
		ctx.drawImage(offscreenCanvas, 0, 0)

    if (state.onionSkinEnabled) {
			// Clear onion skin canvases
			const onionCanvases = document.querySelectorAll('[id^="onion-"]')
			onionCanvases.forEach((target) => {
				const ctx = target.getContext('2d')
				if (ctx) ctx.clearRect(0, 0, target.width, target.height)
			})

      await updateOnionSkinCanvases()
    }
	}

	const remove = () => {
		if (state.isPlaying) return

    const row = [...state.timeline[state.activeTimeline.row]] // Create a shallow copy
    const currCol = state.activeTimeline.col

    // Revoke the Object URL of the frame being removed
    const frameToRemove = row[currCol]
    if (frameToRemove) {
			// Revoke the thumbnail URL
			if (frameToRemove.thumbnailURL) {
				URL.revokeObjectURL(frameToRemove.thumbnailURL)
			}
			// Revoke the blob URL (if applicable and stored in blobURLCache)
			const blobObjectURL = blobURLCache.get(frameToRemove.blob)
			if (blobObjectURL) {
				URL.revokeObjectURL(blobObjectURL)
				blobURLCache.delete(frameToRemove.blob) // Remove it from the cache
			}
    }

    // Shift all frames after the current column to the left
    for (let i = currCol; i < row.length - 1; i++) {
      row[i] = row[i + 1]; // Shift values to the left
    }

    // Clear the last frame in the row since it has been shifted
    row[row.length - 1] = null;

    // Update the state with the modified row
    state.timeline[state.activeTimeline.row] = row; // This triggers reactivity

    // Determine the new active frame
    if (currCol < row.length - 1) {
      state.activeTimeline.col = currCol; // If there is still a frame to the right, make it active
    } else if (currCol > 0) {
      state.activeTimeline.col = currCol - 1; // If no frames to the right, move to the previous frame
    } else {
			// If this was the only frame, clear active frame
			state.activeTimeline.col = 0;
			state.activeShot = null;
    }

    // Update the active shot based on the new timeline position
    state.activeShot = state.timeline[state.activeTimeline.row][state.activeTimeline.col];

		

    // Clear the canvas if no active frame exists
    if (!state.activeShot) {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    } else {
      printColumn();
    }
	};

	const insert = () => {
		if (state.isPlaying) return

    const row = [...state.timeline[state.activeTimeline.row]]; // Create a shallow copy
    const currCol = state.activeTimeline.col;

    // Forward shift: Start from the end and move to the current column
    for (let i = row.length - 1; i > currCol; i--) {
        row[i] = row[i - 1]; // Shift values to the right
    }

    // Insert a new empty frame at the current column + 1
    row[currCol + 1] = null;

    // Update the state with the modified row
    state.timeline[state.activeTimeline.row] = row; // This triggers reactivity
		setActiveTimeline(state.activeTimeline.row, state.activeTimeline.col + 1, true)
	};

	const push = (dir) => {
    // Get the selected row
    const row = [...state.timeline[state.activeTimeline.row]]; // Create a shallow copy
    const currCol = state.activeTimeline.col;

    console.log("Before push:", row);

    if (dir > 0) {
			// Forward shift: Start from the end and move to the current column
			for (let i = row.length - 1; i > currCol; i--) {
					row[i] = row[i - 1]; // Shift values to the right
			}
			row[currCol] = null; // Clear the selected column
    } else if (dir < 0) {
			// Backward shift: Start from the current column and move to the end
			if (row[currCol + 1] !== null && currCol + 1 < row.length) {
					return; // Exit the function if a "shot" is immediately to the right
			}
			for (let i = currCol; i < row.length - 1; i++) {
				row[i] = row[i + 1]; // Shift values to the left
			}
			row[row.length - 1] = null; // Clear the last column
    }

    // Update the timeline row with the modified copy
    state.timeline[state.activeTimeline.row] = row; // This triggers reactivity
		printColumn()

    console.log("After push:", row);
	};

	const addShot = (row, col, blobInput, thumbnailURLInput) => {
    const blob = blobInput;
    const thumbnailURL = thumbnailURLInput; // Thumbnail URL
    const newShot = { blob: blob, thumbnailURL: thumbnailURL };

    if (state.timeline[row][col]?.thumbnailURL) {
        // Revoke old thumbnail URL if it exists
        URL.revokeObjectURL(state.timeline[row][col].thumbnailURL);
    }

    state.shots.add(newShot);
    state.activeShot = newShot;
    state.timeline[row][col] = newShot;
    state.activeTimeline.row = row;
    state.activeTimeline.col = col;
	};

	// TIMELINE - ANIMATION

	let animationInterval; // Holds the interval for playing animation
	let prevOnionSkinVal;

	const playAnimation = () => {
    if (state.isPlaying) return; // Prevent multiple play calls
    state.isPlaying = true;
		prevOnionSkinVal = state.onionSkinEnabled
		state.onionSkinEnabled = false

    const frameRate = state.frameRate > 0 ? state.frameRate : 4; // Default to 4 FPS if invalid
    const frameInterval = 1000 / frameRate; // Milliseconds per frame

    let frameIndex = state.activeTimeline.col;

    // Define a function to play one frame
    const playFrame = () => {
			state.activeTimeline.col = frameIndex; // Update the active column
			// frameIndex = (frameIndex + 1) % state.timeline[0].length; // Move to the next frame or loop back to the start
			frameIndex = (frameIndex + 1) % 6; // Move to the next frame or loop back to the start
			setActiveTimeline(state.activeTimeline.row, frameIndex, true)

			// Stop if no frames exist after looping back to the start
			// if (frameIndex === 0 && !state.timeline.some(row => row.some(cell => cell?.blob))) {
			// 	stopAnimation();
			// }
    };
    
    playFrame(); // Immediately play the first frame
    animationInterval = setInterval(playFrame, frameInterval); // Start the interval for subsequent frames
	};

	const stopAnimation = () => {
		if (!state.isPlaying) return; // Prevent multiple stop calls
		state.isPlaying = false;
		state.onionSkinEnabled = prevOnionSkinVal;
		clearInterval(animationInterval);
	};

	const toggleAnimation = () => {
		if (state.isPlaying) {
			stopAnimation();
		} else {
			playAnimation();
		}
	};

	let programmaticScroll = false
	
	const setActiveTimeline = (row, col, trigger) => {
		if (row === state.activeTimeline.row && col === state.activeTimeline.col) return
		// Update active timeline state
		state.activeTimeline.row = row
		state.activeTimeline.col = col
		state.activeShot = state.timeline[row][col]
		printColumn()

		if (trigger) {
			programmaticScroll = true
			timelineRef.scrollTo({
				left: col * 56,
				behavior: 'instant'
			})
			programmaticScroll = false
		}
	};

	const handleTimelineScroll = (e) => {
		e.preventDefault()

		if (programmaticScroll) return

		// Calculate the current column based on scroll position
		const timelineScroll = e.target.scrollLeft
		const step = Math.floor(timelineScroll / 56)

		// Update active timeline only if the column changes
		if (step !== state.activeTimeline.col) {
			setActiveTimeline(state.activeTimeline.row, step)
		}
	};

	// TIMELINE - ONION SKINNING

	const updateOnionSkinCanvases = async () => {
    for (let i = -state.onionSkinFramesBefore; i <= state.onionSkinFramesAfter; i++) {
			if (i === 0) continue; // Skip the current frame

			const frameIndex = state.activeTimeline.col + i;

			if (frameIndex < 0 || frameIndex >= state.timeline[0].length) continue; // Skip invalid frames

			const targetCanvas = document.querySelector(`#onion-${i}-onion`);
			if (!targetCanvas) {
				// console.warn(`Canvas not found: #onion-${i}-onion`);
				continue;
			}

			const targetCtx = targetCanvas.getContext('2d')
			// targetCtx.setTransform(DPR, 0, 0, DPR, 0, 0);
			if (!targetCtx) {
				// console.warn(`Context not found for canvas: #onion-${i}-onion`);
				continue;
			}

			// Clear the canvas
			targetCanvas.width = state.width;
			targetCanvas.height = state.height;
			targetCtx.clearRect(0, 0, targetCanvas.width, targetCanvas.height);

			const cell = state.timeline[state.activeTimeline.row][frameIndex];
			if (cell?.blob) {
				try {
					await loadBlobToCanvas(targetCtx, targetCanvas, cell.blob);
				} catch (error) {
					console.error(`Failed to load blob for frame ${frameIndex}:`, error);
				}
			}
    }
	};

	// EXPORT PROCESSING

	const generateRandomVideo = async (videoName) => {	
		if (state.isPlaying) return

		// Determine the last column with any content
		const lastFrameIndex = state.timeline[0].length - 1;
		let maxColumn = 0;
		for (let col = 0; col <= lastFrameIndex; col++) {
				for (let row = 0; row < state.timeline.length; row++) {
						if (state.timeline[row][col]?.blob) {
								maxColumn = col;
								break; // No need to check further rows for this column
						}
				}
		}

		// Collect all frames from the timeline up to the last used column
		const frames = [];
		for (let col = 0; col <= maxColumn; col++) {
				// Set a solid background color (e.g., white)
				ctx.fillStyle = "white"; // Change to "black" or any color if needed
				ctx.fillRect(0, 0, state.width, state.height); // Fill the entire canvas with the color

				// Layer all rows for the current column
				for (let row = 0; row < state.timeline.length; row++) {
						const cell = state.timeline[row][col];
						if (cell?.blob) {
								// Load the blob onto the offscreen canvas
								offscreenCtx.clearRect(0, 0, state.width, state.height); // Clear the offscreen canvas
								await loadBlobToCanvas(offscreenCtx, offscreenCanvas, cell.blob);

								// Draw the offscreen canvas onto the main canvas
								setImageSmooth()
								ctx.drawImage(offscreenCanvas, 0, 0);
						}
				}

				// Convert the canvas data to a buffer
				const imageData = canvas.toDataURL('image/png').split(',')[1];
				const imageBuffer = Uint8Array.from(atob(imageData), (c) => c.charCodeAt(0));

				frames.push({ index: col, buffer: imageBuffer });
		}


		for (const { index, buffer } of frames) {
			const filename = `frame-${index.toString().padStart(4, '0')}.png`;
			await ffmpeg.writeFile(filename, buffer);
		}

		await ffmpeg.exec([
			'-framerate', `${state.frameRate}`,
			'-i', 'frame-%04d.png',
			'-c:v', 'libx264',
			'-pix_fmt', 'yuv420p',
			videoName
		]);

		const data = await ffmpeg.readFile(videoName);
		
		const blob = new Blob([data.buffer], { type: 'video/mp4' });
		const url = URL.createObjectURL(blob);

		const a = document.createElement('a');
		a.href = url;
		a.download = videoName;
		a.click();

		for (const { index } of frames) {
			const filename = `frame-${index.toString().padStart(4, '0')}.png`;
			await ffmpeg.deleteFile(filename);
		}
		await ffmpeg.deleteFile(videoName);
		URL.revokeObjectURL(url);
	};

	const loadFFmpg = async () => {
		const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm';
		ffmpeg = new FFmpeg();
		ffmpeg.on('log', ({ message: msg }) => {
			message = msg;
			console.log(msg);
		});

		await ffmpeg.load({
			coreURL: `${baseURL}/ffmpeg-core.js`,
			wasmURL: `${baseURL}/ffmpeg-core.wasm`,
		});
	};

	let lastTap = 0; // Tracks the time of the last tap
  const doubleTapThreshold = 300; // Maximum time interval for a double-tap in milliseconds

  const doubleClick = (e) => {
    const currentTime = new Date().getTime();
    const timeSinceLastTap = currentTime - lastTap;
		
		// Double-tap detected
    if (timeSinceLastTap < doubleTapThreshold && timeSinceLastTap > 0) {
      e.preventDefault()
    }

    lastTap = currentTime;
  }

	let isScrolling = false;
	let startTouchX = 0;
	let startTouchY = 0;
	const SCROLL_THRESHOLD = 10; // Adjust the threshold as needed

	const handleTouchStart = (event) => {
		// Record the initial touch position
		startTouchX = event?.clientX || event.touches[0].clientX;
		startTouchY = event?.clientY || event.touches[0].clientY;
		isScrolling = false; // Reset scrolling flag
	}

	const handleTouchMove = (event) => {
		const touchX = event?.clientX || event.touches[0].clientX;
		const touchY = event?.clientY || event.touches[0].clientY;

		// Calculate the movement
		const deltaX = Math.abs(touchX - startTouchX);
		const deltaY = Math.abs(touchY - startTouchY);

		// If movement exceeds threshold, treat as scroll
		if (deltaX > SCROLL_THRESHOLD || deltaY > SCROLL_THRESHOLD) {
			isScrolling = true;
		}
	}

	const handleTouchEnd = (event, cellIndex) => {
		if (isScrolling) return // Touchend ignored due to scroll-like movement

		if (event.target.dataset?.cellindex) {
			const cellIndex = parseInt(event.target.dataset?.cellindex)
			setActiveTimeline(state.activeTimeline.row, cellIndex, true)
		}
	}
</script>

<main
	class="flex flex-col h-full" 
	on:touchstart={doubleClick}
	on:touchend={(e) => e.preventDefault()}
>
	<div class="flex justify-between absolute z-10 w-full">
		<button
			class="p-5 flex items-center justify-center"
			on:mouseup="{() => {
				
			}}"
			on:touchend="{() => {
				
			}}"
		>
			<img class="icon" width="25" height="25" src="/img/back.svg"/>
		</button>
		<div class="flex justify-center relative w-[200px] overflow-hidden">
			<!-- <button on:mouseup={() => { remove() }} on:touchend={() => { remove() }} class="px-3 w-full max-w-[200px] flex items-center justify-center">
				<img class="icon" width="25" height="25" src="/img/delete.svg"/>
			</button> -->
			<!-- <div class="w-[1px] bg-[purple] absolute h-[1000px] left-[calc(50%-1px)]"></div> -->
			<!-- <button on:mouseup={() => { insert() }} on:touchend={() => { insert() }} class="px-3 w-full max-w-[200px] flex items-center justify-center">
				<img class="icon" width="25" height="25" src="/img/insert.svg"/>
			</button> -->
		</div>
		<button
			class="p-5 flex items-center justify-center"
			on:mouseup="{() => {
				generateRandomVideo('test.mp4')
			}}"
			on:touchend="{() => {
				generateRandomVideo('test.mp4')
			}}"
		>
			<img class="icon" width="25" height="25" src="/img/download.svg"/>
		</button>
	</div>
	<!-- <div class=" flex justify-center items-center flex-1 bg-[rgb(37,37,37)] hide-scroll"> -->
	
	<div
		class="w-full h-full overflow-hidden relative touch-none"
		on:touchstart={onPointerStart}
		on:touchmove={onPointerMove}
		on:touchend={onPointerEnd}
		on:mousedown={onPointerStart}
		on:mousemove={onPointerMove}
		on:mouseup={onPointerEnd}
		on:wheel={handleWheel}
		bind:this={workspaceRef}
	>
		<!-- <div class="absolute top-0 left-0">{debug}</div> -->
		<div
			style="
				transform: translate({state.workspace.translate.x}px, {state.workspace.translate.y}px) scale({state.workspace.scale}) rotate({state.workspace.rotate}deg);
				width: {state.width / DPR}px;
				height: {state.height / DPR }px;"
			class="origin-top-left touch-none flex justify-center items-center"
		>
			<div class="relative bg-white w-full h-full">
				<canvas
					bind:this={offscreenCanvas}
					class="hidden absolute left-[100%] w-full h-full pointer-events-none bg-[green]"
					style="width: {0}px; height: {state.height / DPR}px;"
					width={state.width}
					height={state.height}
				/>
				<canvas
					bind:this={tempCanvas}
					class="hidden absolute left-[-100%] w-full h-full pointer-events-none bg-[blue]"
					style="width: {0}px; height: {state.height / DPR}px;"
					width={state.width}
					height={state.height}
				/>
				{#if state.onionSkinEnabled}
					{#each Array(state.onionSkinFramesAfter).fill(0) as _, index}
						<canvas
							id="onion-{0 - index - 1}-onion"
							class="absolute top-0 left-0 w-full h-full pointer-events-none bg-transparent"
							style="width: {state.width / DPR}px; height: {state.height / DPR}px; opacity: {0.5 - index * 0.1};"
							width={state.width}
							height={state.height}
						/>
					{/each}
				{/if}
				<canvas
					class="absolute top-0 left-0 w-full h-full"
					style="width: {state.width / DPR}px; height: {state.height / DPR}px;"
					bind:this={canvas}
					width={state.width}
					height={state.height}
				/>
				{#if state.onionSkinEnabled}
					{#each Array(state.onionSkinFramesAfter).fill(0) as _, index}
						<canvas
							id="onion-{index + 1}-onion"
							class="absolute top-0 left-0 w-full h-full pointer-events-none bg-transparent"
							style="width: {state.width / DPR}px; height: {state.height / DPR}px; opacity: {0.5 - index * 0.1};"
							width={state.width}
							height={state.height}
						/>
					{/each}
				{/if}
			</div>
		</div>
	</div>

	<div
		on:touchstart={handleTouchStart}
		on:touchmove={handleTouchMove}
		on:touchend={handleTouchEnd}
		on:mousedown={handleTouchStart}
		on:mousemove={handleTouchMove}
		on:mouseup={handleTouchEnd}
		class="absolute w-full bottom-0 mb-9"
	>
		<div class="flex flex-col relative">
			<!-- <div class="text-white min-w-[100px] max-w-[100px] flex">
				<p class="flex items-center pl-3 text-[11px]"><b>Layer 1</b></p>
			</div> -->
			<!-- svelte-ignore a11y-click-events-have-key-events -->
			<div class="relative flex">
				<div class="flex justify-start flex-1">
					
				</div>
				<div class="flex justify-center">
					<button on:touchend={toggleAnimation} on:mouseup={toggleAnimation} class="px-8 text-white flex items-center justify-center">
						{#if !state.isPlaying}
							<img class="icon min-w-[25px]" width="25" height="25" src="/img/play.svg"/>
						{:else}
							<img class="icon min-w-[25px]" width="25" height="25" src="/img/pause.svg"/>
						{/if}
					</button>
				</div>
				<div class="flex justify-start flex-1">
					<div class="flex justify-start flex-1">
						<button on:mouseup={() => { remove() }} on:touchend={() => { remove() }} class="py-4 px-4 flex items-center text-white gap-2">
							<img class="icon min-w-[25px]" width="25" height="25" src="/img/delete.svg"/>
							<!-- <p class="text-[14px]">Delete</p> -->
						</button>
						<!-- <div class="w-[1px] bg-[purple] absolute h-[1000px] left-[calc(50%-1px)]"></div> -->
						<button on:mouseup={() => { insert() }} on:touchend={() => { insert() }} class="px-4 flex items-center text-white gap-2">
							<img class="icon min-w-[25px]" width="25" height="25" src="/img/insert.svg"/>
							<!-- <p class="text-[14px]">Insert</p> -->
						</button>
					</div>
				</div>
			</div>
			
			<div
				bind:this={timelineRef}
				on:scroll={handleTimelineScroll}
				class="hide-scroll overflow-x-scroll overflow-y-visible relative {state.isPlaying ? 'pointer-events-none' : ''}">
				<!-- blue line -->
				<!-- <div style="left: 50%" class="z-50 w-[2px] h-[56px] bg-[rgb(52,152,219)] fixed pointer-events-none" /> -->
				<!-- button row -->
				
				<div class="relative min-w-max pl-[calc(50%+1px)] pr-[calc(50%)]">
					<div class="flex flex-col gap-[1px]">
						{#each state.timeline as row, rowIndex}
							<div class="flex flex-nowrap">
								{#each row as cell, cellIndex}
									<button data-cellindex={cellIndex} class="min-w-14 h-14 flex justify-center rounded-[6px] pr-[2px] relative">		
										<div
											style="box-shadow: inset 0px 0px 0px 0px rgba(255,255,255,0.8), inset 0px 0px 0px 1px rgba(0,0,0,0.8);"
											class="{cell !== null ? 'bg-white' : 'bg-[rgba(0,0,0,0.25)]'} w-[calc(100%-0px)] h-full pointer-events-none relative rounded-[6px] overflow-hidden
											{state.activeTimeline.col === cellIndex ? 'border-[1px] border-[rgb(52,152,219)]' : 'border-[1px] border-[#fff]'}"
										>
										<p class="
											{state.activeTimeline.col === cellIndex ? 'bg-[rgb(52,152,219)]' : 'bg-[rgba(0,0,0,1)]'} 
											absolute top-0 left-0 z-20 text-[9px] px-1 py-[1px] text-left text-[rgba(255,255,255,1)] rounded-br-[2px]">{cellIndex + 1}</p>
										{#if cell?.thumbnailURL}
											<img
												src="{cell.thumbnailURL}"
												class=" absolute top-0 left-0 w-full h-full object-cover "
												alt="Thumbnail"
											/>
										{/if}
										</div>
									</button>
								{/each}
							</div>
						{/each}
					</div>
				</div>
			</div>
			<!-- <div class="flex min-w-[100px] max-w-[100px]">
				<button on:touchend={toggleAnimation} on:mouseup={toggleAnimation} class="w-full text-white flex items-center justify-center">
					{#if !state.isPlaying}
						<img class="icon" width="25" height="25" src="/img/play.svg"/>
					{:else}
						<img class="icon" width="25" height="25" src="/img/pause.svg"/>
					{/if}
				</button>
			</div> -->
		</div>
	</div>
</main>

<style>
	.icon {
		pointer-events: none;
	}

	.bg-darkgray {
		background: var(--dark-gray);
	}

	.bg-mediumgray {
		background: var(--medium-gray);
	}

	main {
		height: 100%;
		background: var(--dark);
	}

	canvas {
		width: 500px;
		height: 500px;
	}

	.hide-scroll::-webkit-scrollbar {
    display: none; /* Hide scrollbar */
	}
</style>
