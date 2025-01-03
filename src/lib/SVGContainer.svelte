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
        width: 80vmin;
        border-style: dashed;
        border-width: 1px;
    }

    @media (max-width: 600px) {
        .svg-container {
            width: 95vmin;
        }
    }
</style>