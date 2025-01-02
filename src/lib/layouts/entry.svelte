<script>
	const { children } = $props();
</script>

<main>
	<a href="/">Back to overview</a>
	{@render children()}
</main>

<style>
    :global(body) {
        font-family: "Courier New", serif;
    }

    :global(a) {
        color: #000;
        text-decoration: none;
        border-bottom-style: dashed;
        border-bottom-width: 1px;
    }

    :global(a:hover) {
        color: darkorange;
    }

    :global(main) {
        display: flex;
        flex-direction: column;
        align-items: center;
    }
</style>