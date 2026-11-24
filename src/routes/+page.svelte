<script lang="ts">
	import { onMount } from 'svelte';

	const API_URL =
		'https://script.google.com/macros/s/AKfycbzTt0gOthKckwqmW9W0MpeLaa43zRfCBfiFKPbRrUx2lrSo9CO250ljePJ7B6ZMPzft/exec';

	let today = new Date();
	let year = $state(today.getFullYear());
	let month = $state(today.getMonth());

	let firstDay = $derived(new Date(year, month, 1).getDay());

	let lastDate = $derived(new Date(year, month + 1, 0).getDate());

	let dates = $derived(
		Array.from({ length: Math.ceil((firstDay + lastDate) / 7) * 7 }, (_, i) => {
			const date = i - firstDay + 1;
			return date > 0 && date <= lastDate ? date : null;
		})
	);

	function moveMonth(offset: number) {
		const next = new Date(year, month + offset, 1);

		if (
			next.getFullYear() < today.getFullYear() ||
			(next.getFullYear() === today.getFullYear() && next.getMonth() < today.getMonth())
		) {
			return;
		}

		year = next.getFullYear();
		month = next.getMonth();
	}

	type BusyPeriod = {
		start: string;
		end: string;
	};

	let busy = $state<BusyPeriod[]>([]);
	let loading = $state(true);
	let error = $state('');
	let updatedAt = $state('');

	function getBusyPeriods(year: number, month: number, date: number) {
		const dayStart = getDayStart(year, month, date);
		const dayEnd = dayStart + RANGE_MS;

		return busy
			.map((period) => ({
				start: Math.max(new Date(period.start).getTime(), dayStart),
				end: Math.min(new Date(period.end).getTime(), dayEnd)
			}))
			.filter((period) => period.start < period.end)
			.map((period) => ({
				left: ((period.start - dayStart) / RANGE_MS) * 100,
				width: ((period.end - period.start) / RANGE_MS) * 100
			}));
	}

	function getFreePeriods(year: number, month: number, date: number) {
		const periods = getBusyPeriods(year, month, date).sort((a, b) => a.left - b.left);

		const free: { left: number; width: number }[] = [];
		let cursor = getPastPercentage(year, month, date);

		for (const period of periods) {
			if (period.left > cursor) {
				free.push({
					left: cursor,
					width: period.left - cursor
				});
			}

			cursor = Math.max(cursor, period.left + period.width);
		}

		if (cursor < 100) {
			free.push({
				left: cursor,
				width: 100 - cursor
			});
		}

		return free;
	}

	let selected = $state<{
		year: number;
		month: number;
		date: number;
	} | null>(null);

	let selectedFree = $derived(
		selected ? getFreePeriods(selected.year, selected.month, selected.date) : []
	);

	function formatTime(percent: number) {
		const minutes = Math.round(START_HOUR * 60 + ((percent / 100) * RANGE_MS) / 60000);

		const h = Math.floor(minutes / 60);
		const m = minutes % 60;

		return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}`;
	}

	let now = $state(Date.now());

	function getPastPercentage(year: number, month: number, date: number) {
		const dayStart = getDayStart(year, month, date);

		return Math.min(100, Math.max(0, ((now - dayStart) / RANGE_MS) * 100));
	}

	function formatDuration(percent: number) {
		const minutes = Math.round(((percent / 100) * RANGE_MS) / 60000);

		const h = Math.floor(minutes / 60);
		const m = minutes % 60;

		return [h ? `${h}h` : '', m ? `${m}m` : ''].filter(Boolean).join(' ') || '0h';
	}

	let totalFree = $derived(selectedFree.reduce((sum, p) => sum + p.width, 0));

	const START_HOUR = 9;
	const END_HOUR = 24;

	const HOUR_MS = 3600000;
	const RANGE_MS = (END_HOUR - START_HOUR) * HOUR_MS;

	function getDayStart(year: number, month: number, date: number) {
		return Date.UTC(year, month, date) + (START_HOUR - 9) * HOUR_MS;
	}

	onMount(async () => {
		const timer = setInterval(() => {
			now = Date.now();
		}, 1000);

		try {
			const response = await fetch(API_URL);
			if (!response.ok) throw new Error('HTTP Error');

			const data = await response.json();

			if (data.error || !Array.isArray(data.busy)) {
				throw new Error('Invalid API response');
			}

			busy = data.busy;
			updatedAt = data.updatedAt;
		} catch (e) {
			error = '일정을 불러오지 못했습니다.';
		} finally {
			loading = false;
		}

		return () => clearInterval(timer);
	});
</script>

<div class="container">
	<div class="navigation">
		<button onclick={() => moveMonth(-1)}>‹</button>
		<strong>{year}. {month + 1}.</strong>
		<button onclick={() => moveMonth(1)}>›</button>
	</div>

	<div class="calendar">
		{#each ['SUN', 'MON', 'TUE', 'WED', 'THU', 'FRI', 'SAT'] as day}
			<div class="weekday">{day}</div>
		{/each}

		{#each dates as date}
			<button
				class="date"
				disabled={date === null || getPastPercentage(year, month, date) >= 100}
				onclick={() => {
					if (date !== null) {
						selected = { year, month, date };
					}
				}}
			>
				{#if date !== null}
					<div>{date}</div>

					{#if !loading && !error}
						<div class="timeline">
							{#each getBusyPeriods(year, month, date) as period}
								<div class="busy" style:left="{period.left}%" style:width="{period.width}%"></div>
							{/each}
							<div class="past" style:width="{getPastPercentage(year, month, date)}%"></div>
						</div>
					{/if}
				{/if}
			</button>
		{/each}
	</div>
	{#if selected && !loading && !error}
		<section class="details">
			<header class="details-header">
				<div>
					<small>
						{new Date(selected.year, selected.month, selected.date).toLocaleDateString('en-US', {
							weekday: 'long'
						})}
					</small>

					<h2>
						{selected.year}.
						{selected.month + 1}.
						{selected.date}.
					</h2>
				</div>

				<div class="summary">
					<strong>{formatDuration(totalFree)}</strong>
					<small>Available hours</small>
				</div>
			</header>

			<div class="timeline large">
				{#each getBusyPeriods(selected.year, selected.month, selected.date) as period}
					<div class="busy" style:left="{period.left}%" style:width="{period.width}%"></div>
				{/each}

				<div
					class="past"
					style:width="{getPastPercentage(selected.year, selected.month, selected.date)}%"
				></div>
			</div>

			<div class="time-axis">
				{#each ['09:00', '12:00', '15:00', '18:00', '21:00', '24:00'] as time}
					<span>{time}</span>
				{/each}
			</div>

			<h3>Available hours</h3>

			<div class="periods">
				{#each selectedFree as period}
					<div class="period">
						<span class="dot"></span>
						<span>
							{formatTime(period.left)}
							–
							{formatTime(period.left + period.width)}
						</span>
						<span class="duration">
							{formatDuration(period.width)}
						</span>
					</div>
				{:else}
					<small>No available hours.</small>
				{/each}
			</div>

			<!--
			<button
				class="copy"
				onclick={() =>
					navigator.clipboard.writeText(
						`${selected.year}. ${selected.month + 1}. ${selected.date}. (KST)\n` +
							selectedFree
								.map((p) => `${formatTime(p.left)} – ${formatTime(p.left + p.width)}`)
								.join('\n')
					)}
			>
				Copy availability
			</button>
      -->
		</section>
	{/if}
</div>

<style>
	.container {
		max-width: 600px;
		margin: auto;
	}

	.navigation {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 1rem;
	}

	.calendar {
		display: grid;
		grid-template-columns: repeat(7, 1fr);
	}

	.weekday {
		text-align: center;
		padding: 0.5rem;
		font-size: 0.75rem;
	}

	.date {
		aspect-ratio: 1;
		padding: 0.5rem;
		border: 1px solid #8882;
	}

	.date {
		display: flex;
		flex-direction: column;
		justify-content: space-between;
	}

	.timeline {
		position: relative;
		width: 100%;
		height: 8px;
		background: #a8d5b5;
		border-radius: var(--radius);
		overflow: hidden;
	}

	.busy {
		position: absolute;
		top: 0;
		height: 100%;
		background: #d87979;
	}

	.past {
		position: absolute;
		top: 0;
		left: 0;
		height: 100%;
		background: var(--unknown, #bdbdc5);
		z-index: 1;
	}

	.details {
		margin-top: 24px;
		padding: 24px;
		background: var(--surface);
		border: 1px solid var(--border);
		border-radius: var(--radius);
	}

	.details-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 12px;
		margin-bottom: 24px;
	}

	.details h2 {
		margin: 4px 0 0;
		font-size: 1.4rem;
		letter-spacing: -0.04em;
	}

	.details h3 {
		margin-top: 24px;
		font-size: 0.9rem;
		font-weight: 600;
	}

	.details small {
		color: var(--muted);
		font-size: 0.75rem;
	}

	.summary {
		text-align: right;
	}

	.summary strong {
		display: block;
		font-size: 1.5rem;
		color: #529b76;
		letter-spacing: -0.04em;
	}

	.timeline.large {
		height: 24px;
		border-radius: var(--radius);
	}

	.time-axis {
		display: flex;
		justify-content: space-between;
		margin-top: 8px;
		font-size: 0.65rem;
		color: var(--muted);
	}

	.periods {
		display: flex;
		flex-direction: column;
		gap: 6px;
	}

	.period {
		display: flex;
		align-items: center;
		gap: 12px;
		padding: 14px;
		background: var(--background);
		border-radius: var(--radius);
		font-size: 0.9rem;
	}

	.dot {
		width: 8px;
		height: 8px;
		flex-shrink: 0;
		background: var(--available);
		border-radius: 50%;
	}

	.duration {
		margin-left: auto;
		color: var(--muted);
		font-size: 0.75rem;
	}

	.copy {
		width: 100%;
		padding: 12px;
		margin-top: 16px;
		background: transparent;
		border: 1px solid var(--border);
		border-radius: var(--radius);
		cursor: pointer;
		transition: background 150ms;
	}

	.copy:hover {
		background: var(--background);
	}

	@media (max-width: 480px) {
		.details {
			padding: 16px;
		}
	}
</style>
