<script>
	export let title;
	export let x;
	export let y;
</script>

<div
	class="absolute rounded-md bg-tgray px-3 py-2 font-warownia text-xxs capitalize text-tbeige"
	style="
		top: {y + 10}px;
		left: {x + 15}px;"
>
	{title}
</div>
