<script>
	const { children } = $props();
</script>

<main>
	<h1>Genuary 2025</h1>
	<p>
		Here you can find <a href="https://www.d17e.dev">d17e</a>'s entries for <a href="https://genuary.art">Genuary 2025</a>.
	</p>
	{@render children()}
</main>

<style>
    :global(body) {
        font-family: "Courier New", serif;
        line-height: 1.4rem;
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
        padding: 1rem .5rem;
    }
</style>