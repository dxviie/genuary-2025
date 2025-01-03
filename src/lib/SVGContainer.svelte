<script lang="ts">
	const { children } = $props();
</script>

<div class="svg-container">
	{@render children()}
</div>

<style>
    .svg-container {
        display: flex;
        flex-direction: column;
        align-items: center;
        width: 95vmin;
        object-fit: contain;
    }
</style>