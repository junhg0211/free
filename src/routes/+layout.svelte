<script lang="ts">
	import favicon from '#lib/assets/favicon.svg';
	import type { LayoutProps } from './$types';

	let { children }: LayoutProps = $props();
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<style>
		@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css');

		:root {
			color-scheme: light dark;

			--background: #f7f7f9;
			--surface: #ffffff;
			--foreground: #19191e;
			--muted: #71717a;
			--border: #e4e4e7;

			--available: #a6d9c1;
			--busy: #d17c7c;
			--unknown: #d4d4d8;

			--radius: 4px;
		}

		@media (prefers-color-scheme: dark) {
			:root {
				--background: #19191e;
				--surface: #242429;
				--foreground: #f7f7f9;
				--muted: #a1a1aa;
				--border: #38383f;

				--available: #387c60;
				--busy: #a55050;
				--unknown: #48484e;
			}
		}

		* {
			box-sizing: border-box;
		}

		body {
			margin: 0;
			background: var(--background);
			color: var(--foreground);
			font-family: Pretendard, sans-serif;
			font-variant-numeric: tabular-nums;
		}

		button {
			font: inherit;
			color: inherit;
		}

		.container {
			max-width: 600px;
			margin: auto;
			padding: 24px 16px;
		}

		.date {
			background: var(--surface);
			border: 1px solid var(--border);
			border-radius: var(--radius);
			transition:
				background 150ms,
				border-color 150ms;
			cursor: pointer;
		}

		.date:hover:not(:disabled) {
			border-color: var(--muted);
		}

		.date:disabled {
			opacity: 0.4;
			cursor: default;
		}

		.calendar {
			display: grid;
			grid-template-columns: repeat(7, minmax(0, 1fr));
			gap: 4px;
		}

		.timeline {
			background: var(--available);
		}

		.busy {
			background: var(--busy);
		}

		button {
			background-color: transparent;
			border: none;
			cursor: pointer;
		}
	</style>
</svelte:head>

{@render children()}
