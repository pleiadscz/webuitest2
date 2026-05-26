<script lang="ts">
	// UserAvatar component: displays user initials on a hardcoded yellow circle SVG background.
	// Does NOT fetch profile image from the server.
	export let name: string = '';
	export let className: string = 'size-7';

	function getInitials(fullName: string): string {
		const sanitized = (fullName ?? '').trim();
		if (!sanitized) return '?';
		const parts = sanitized.split(' ');
		const first = sanitized[0];
		const last = parts.length > 1 ? parts[parts.length - 1][0] : '';
		return (first + last).toUpperCase();
	}

	$: initials = getInitials(name);
</script>

<div class="{className} relative rounded-full overflow-hidden flex items-center justify-center select-none">
	<!-- Hardcoded yellow circle SVG background -->
	<svg
		xmlns="http://www.w3.org/2000/svg"
		width="320"
		height="320"
		viewBox="0 0 320 320"
		class="absolute inset-0 w-full h-full"
		aria-hidden="true"
	>
		<circle cx="160" cy="160" r="148" fill="#F0C020" />
	</svg>
	<!-- Initials overlay -->
	<span
		class="relative z-10 font-semibold text-white leading-none"
		style="font-size: clamp(0.55rem, 35%, 1.1rem);"
	>
		{initials}
	</span>
</div>
