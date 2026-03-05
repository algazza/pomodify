<script lang="ts">
	import { onDestroy } from 'svelte'

	type Mode = 'focus' | 'break'
	type VisualState = 'focus' | 'break' | 'paused'

	let focusMinutes = 25
	let breakMinutes = 5
	let mode: Mode = 'focus'
	let isRunning = false
	let autoContinueAfterFocus = true
	let remainingSeconds = focusMinutes * 60
	let intervalId: ReturnType<typeof setInterval> | null = null
	let notificationMessage = ''
	let notificationTimeout: ReturnType<typeof setTimeout> | null = null
	let tickAudioContext: AudioContext | null = null

	const formatTime = (seconds: number) => {
		const minutes = Math.floor(seconds / 60)
		const remaining = seconds % 60

		return `${minutes.toString().padStart(2, '0')}:${remaining.toString().padStart(2, '0')}`
	}

	$: timerLabel = formatTime(remainingSeconds)
	$: runningStatus = isRunning ? 'Running' : 'Stopped'
	$: currentModeLabel = mode === 'focus' ? 'Focus' : 'Break'
	$: currentModeDuration = (mode === 'focus' ? focusMinutes : breakMinutes) * 60
	$: isPaused = !isRunning && remainingSeconds !== currentModeDuration
	$: visualState = isRunning ? mode : isPaused ? 'paused' : mode

	const visualConfig: Record<
		VisualState,
		{
			title: string
			icon: string
			themeColor: string
			bgClass: string
			cardBorder: string
			cardBg: string
			pillBorder: string
			pillText: string
			buttonBg: string
			buttonHover: string
			secondaryButton: string
			secondaryHover: string
			rangeAccent: string
			modeActive: string
			modeInactive: string
			footerText: string
		}
	> = {
		focus: {
			title: 'Focus',
			icon: '🍅',
			themeColor: '#7f1d1d',
			bgClass: 'from-red-950 via-red-900 to-zinc-950',
			cardBorder: 'border-red-900/60',
			cardBg: 'bg-red-950/35',
			pillBorder: 'border-red-700/70',
			pillText: 'text-red-200',
			buttonBg: 'bg-red-600',
			buttonHover: 'hover:bg-red-500',
			secondaryButton: 'border-red-700 bg-red-950/30 text-red-100',
			secondaryHover: 'hover:bg-red-900/40',
			rangeAccent: 'accent-red-500',
			modeActive: 'bg-red-600 text-white',
			modeInactive: 'border border-red-800 bg-transparent text-red-200 hover:bg-red-900/40',
			footerText: 'text-red-100'
		},
		break: {
			title: 'Break',
			icon: '☕',
			themeColor: '#134e4a',
			bgClass: 'from-teal-950 via-emerald-900 to-zinc-950',
			cardBorder: 'border-emerald-900/60',
			cardBg: 'bg-emerald-950/35',
			pillBorder: 'border-emerald-700/70',
			pillText: 'text-emerald-200',
			buttonBg: 'bg-emerald-600',
			buttonHover: 'hover:bg-emerald-500',
			secondaryButton: 'border-emerald-700 bg-emerald-950/30 text-emerald-100',
			secondaryHover: 'hover:bg-emerald-900/40',
			rangeAccent: 'accent-emerald-500',
			modeActive: 'bg-emerald-600 text-white',
			modeInactive: 'border border-emerald-800 bg-transparent text-emerald-200 hover:bg-emerald-900/40',
			footerText: 'text-emerald-100'
		},
		paused: {
			title: 'Paused',
			icon: '⏸️',
			themeColor: '#78350f',
			bgClass: 'from-amber-950 via-amber-900 to-zinc-950',
			cardBorder: 'border-amber-900/60',
			cardBg: 'bg-amber-950/35',
			pillBorder: 'border-amber-700/70',
			pillText: 'text-amber-200',
			buttonBg: 'bg-amber-600',
			buttonHover: 'hover:bg-amber-500',
			secondaryButton: 'border-amber-700 bg-amber-950/30 text-amber-100',
			secondaryHover: 'hover:bg-amber-900/40',
			rangeAccent: 'accent-amber-500',
			modeActive: 'bg-amber-600 text-white',
			modeInactive: 'border border-amber-800 bg-transparent text-amber-200 hover:bg-amber-900/40',
			footerText: 'text-amber-100'
		}
	}

	$: activeVisual = visualConfig[visualState]

	$: {
		if (typeof document !== 'undefined') {
			document.title = `${activeVisual.icon} ${timerLabel} • ${activeVisual.title}`
			document.body.style.backgroundColor = activeVisual.themeColor

			let iconLink = document.querySelector<HTMLLinkElement>('link[rel="icon"]')
			if (!iconLink) {
				iconLink = document.createElement('link')
				iconLink.rel = 'icon'
				document.head.appendChild(iconLink)
			}

			const faviconSvg = encodeURIComponent(
				`<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64"><rect width="64" height="64" rx="14" fill="${activeVisual.themeColor}"/><text x="32" y="42" font-size="30" text-anchor="middle">${activeVisual.icon}</text></svg>`
			)
			iconLink.href = `data:image/svg+xml,${faviconSvg}`

			let themeMeta = document.querySelector<HTMLMetaElement>('meta[name="theme-color"]')
			if (!themeMeta) {
				themeMeta = document.createElement('meta')
				themeMeta.name = 'theme-color'
				document.head.appendChild(themeMeta)
			}
			themeMeta.content = activeVisual.themeColor
		}
	}

	const stopInterval = () => {
		if (intervalId) {
			clearInterval(intervalId)
			intervalId = null
		}
	}

	const playTickSound = () => {
		if (typeof window === 'undefined') return

		if (!tickAudioContext || tickAudioContext.state === 'closed') {
			tickAudioContext = new AudioContext()
		}

		const oscillator = tickAudioContext.createOscillator()
		const gain = tickAudioContext.createGain()

		oscillator.type = 'square'
		oscillator.frequency.value = 1200
		gain.gain.value = 0.0001

		oscillator.connect(gain)
		gain.connect(tickAudioContext.destination)

		const now = tickAudioContext.currentTime
		oscillator.start(now)
		gain.gain.exponentialRampToValueAtTime(0.16, now + 0.005)
		gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.08)
		oscillator.stop(now + 0.09)
	}

	const showCompletionNotification = (completedMode: Mode) => {
		const nextMode = completedMode === 'focus' ? 'break' : 'focus'
		notificationMessage = `${completedMode === 'focus' ? 'Focus' : 'Break'} session finished. ${nextMode === 'break' ? 'Break time starts now.' : 'Ready for focus again.'}`

		if (notificationTimeout) {
			clearTimeout(notificationTimeout)
		}
		notificationTimeout = setTimeout(() => {
			notificationMessage = ''
		}, 5000)

		if (typeof window === 'undefined' || !('Notification' in window)) return

		if (Notification.permission === 'granted') {
			new Notification('Pomodify', {
				body: notificationMessage,
				icon: '/favicon.svg'
			})
		}
	}

	const setRemainingFromMode = () => {
		remainingSeconds = (mode === 'focus' ? focusMinutes : breakMinutes) * 60
	}

	const playAlarm = () => {
		const audioContext = new AudioContext()
		const beep = (delay: number) => {
			const oscillator = audioContext.createOscillator()
			const gain = audioContext.createGain()

			oscillator.type = 'sine'
			oscillator.frequency.value = 920
			gain.gain.value = 0.0001

			oscillator.connect(gain)
			gain.connect(audioContext.destination)

			const startAt = audioContext.currentTime + delay
			oscillator.start(startAt)

			gain.gain.exponentialRampToValueAtTime(0.45, startAt + 0.01)
			gain.gain.exponentialRampToValueAtTime(0.0001, startAt + 0.3)
			oscillator.stop(startAt + 0.32)
		}

		beep(0)
		beep(0.4)
		beep(0.8)

		setTimeout(() => {
			void audioContext.close()
		}, 1300)
	}

	const handleTimerComplete = () => {
		const completedMode = mode
		playAlarm()
		showCompletionNotification(completedMode)

		if (mode === 'focus') {
			mode = 'break'
			remainingSeconds = breakMinutes * 60

			if (autoContinueAfterFocus) {
				startTimer()
				return
			}

			isRunning = false
			stopInterval()
			return
		}

		mode = 'focus'
		remainingSeconds = focusMinutes * 60
		isRunning = false
		stopInterval()
	}

	const tick = () => {
		if (remainingSeconds <= 1) {
			handleTimerComplete()
			return
		}

		if (remainingSeconds <= 5) {
			playTickSound()
		}

		remainingSeconds -= 1
	}

	const startTimer = () => {
		if (isRunning) return

		if (typeof window !== 'undefined' && 'Notification' in window && Notification.permission === 'default') {
			void Notification.requestPermission()
		}

		isRunning = true
		stopInterval()
		intervalId = setInterval(tick, 1000)
	}

	const pauseTimer = () => {
		isRunning = false
		stopInterval()
	}

	const toggleTimer = () => {
		if (isRunning) {
			pauseTimer()
			return
		}

		startTimer()
	}

	const resetTimer = () => {
		pauseTimer()
		setRemainingFromMode()
	}

	const switchMode = (nextMode: Mode) => {
		mode = nextMode
		resetTimer()
	}

	const endTimer = () => {
		pauseTimer()
		mode = 'focus'
		remainingSeconds = focusMinutes * 60
	}

	const handleFocusChange = (event: Event) => {
		const target = event.target as HTMLInputElement
		focusMinutes = Number(target.value)

		if (!isRunning && mode === 'focus') {
			remainingSeconds = focusMinutes * 60
		}
	}

	const handleBreakChange = (event: Event) => {
		const target = event.target as HTMLInputElement
		breakMinutes = Number(target.value)

		if (!isRunning && mode === 'break') {
			remainingSeconds = breakMinutes * 60
		}
	}

	onDestroy(() => {
		stopInterval()
		if (notificationTimeout) {
			clearTimeout(notificationTimeout)
		}
		if (tickAudioContext && tickAudioContext.state !== 'closed') {
			void tickAudioContext.close()
		}
	})
</script>

<main class={`min-h-screen bg-linear-to-b text-red-50 transition-colors duration-400 ${activeVisual.bgClass}`}>
	<div class="mx-auto flex w-full max-w-5xl flex-col gap-8 px-4 py-8 sm:px-6 lg:px-8">
		<header class={`rounded-2xl border p-5 shadow-2xl shadow-black/30 backdrop-blur ${activeVisual.cardBorder} ${activeVisual.cardBg}`}>
			<div class="flex flex-wrap items-center justify-between gap-3">
				<h1 class="text-xl font-semibold tracking-tight sm:text-2xl">Pomodify</h1>
				<span class={`rounded-full border px-3 py-1 text-sm font-medium ${activeVisual.pillBorder} ${activeVisual.pillText}`}>
					{runningStatus}
				</span>
			</div>
			<div class="mt-4 flex items-end justify-between gap-4">
				<div>
					<p class="text-sm text-red-200/80">Current session</p>
					<p class="text-lg font-medium">{currentModeLabel}</p>
				</div>
				<p class="font-mono text-4xl font-semibold tracking-widest sm:text-5xl">{timerLabel}</p>
			</div>
		</header>

		<section class={`rounded-3xl border p-8 shadow-xl shadow-black/30 sm:p-10 ${activeVisual.cardBorder} ${activeVisual.cardBg}`}>
			{#if notificationMessage}
				<div class={`mb-6 rounded-xl border px-4 py-3 text-sm font-medium ${activeVisual.cardBorder} ${activeVisual.footerText} bg-black/25`}>
					{notificationMessage}
				</div>
			{/if}

			<div class="mb-8 flex items-center justify-center rounded-2xl border border-white/10 bg-black/20 p-8 sm:p-10">
				<p class="font-mono text-7xl font-semibold tracking-widest sm:text-8xl md:text-9xl">{timerLabel}</p>
			</div>

			<div class="grid gap-5 sm:grid-cols-2">
				<label class="flex flex-col gap-2">
					<span class="text-sm font-medium text-red-200">Focus duration: {focusMinutes} min</span>
					<input
						type="range"
						min="25"
						max="60"
						step="1"
						value={focusMinutes}
						on:input={handleFocusChange}
						disabled={isRunning}
						class={activeVisual.rangeAccent}
					/>
				</label>

				<label class="flex flex-col gap-2">
					<span class="text-sm font-medium text-red-200">Break duration: {breakMinutes} min</span>
					<input
						type="range"
						min="5"
						max="15"
						step="1"
						value={breakMinutes}
						on:input={handleBreakChange}
						disabled={isRunning}
						class={activeVisual.rangeAccent}
					/>
				</label>
			</div>

			<div class="mt-6 flex flex-wrap gap-3 justify-center">
				<button
					type="button"
					class={`rounded-xl px-12 py-4 font-medium text-white transition text-xl ${activeVisual.buttonBg} ${activeVisual.buttonHover}`}
					on:click={toggleTimer}
				>
					{isRunning ? 'Pause' : 'Start'}
				</button>
				{#if isPaused}
					<button
						type="button"
						class={`rounded-xl border px-12 py-4 text-xl font-medium transition ${activeVisual.secondaryButton} ${activeVisual.secondaryHover}`}
						on:click={endTimer}
					>
						End timer
					</button>
				{/if}
				<button
					type="button"
					class={`rounded-xl border px-12 py-4 text-xl font-medium transition ${activeVisual.secondaryButton} ${activeVisual.secondaryHover}`}
					on:click={resetTimer}
				>
					Reset
				</button>
			</div>

			<div class={`mt-6 flex flex-col gap-4 rounded-xl border p-4 sm:flex-row sm:items-center sm:justify-between ${activeVisual.cardBorder} ${activeVisual.cardBg}`}>
				<div class="flex gap-2">
					<button
						type="button"
						class={`rounded-lg px-3 py-2 text-sm font-medium transition ${
							mode === 'focus'
								? activeVisual.modeActive
								: activeVisual.modeInactive
						}`}
						on:click={() => switchMode('focus')}
					>
						Focus mode
					</button>
					<button
						type="button"
						class={`rounded-lg px-3 py-2 text-sm font-medium transition ${
							mode === 'break'
								? activeVisual.modeActive
								: activeVisual.modeInactive
						}`}
						on:click={() => switchMode('break')}
					>
						Break mode
					</button>
				</div>

				<label class={`flex items-center gap-3 text-sm ${activeVisual.footerText}`}>
					<input type="checkbox" bind:checked={autoContinueAfterFocus} class={`h-4 w-4 ${activeVisual.rangeAccent}`} />
					Auto-start break after focus ends
				</label>
			</div>
		</section>
	</div>
</main>
