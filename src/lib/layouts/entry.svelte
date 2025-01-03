<script lang="ts">
	const { children } = $props();
</script>

<main>
	<a href="/">Back to overview</a>
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

    :global(h1) {
        text-align: center;
    }

    @media (max-width: 600px) {
        :global(h1) {
            font-size: 1.5rem;
        }
    }

    :global(button) {
        padding: 0.5rem 1rem;
        font-size: 1rem;
        font-family: "Courier New", serif;
        font-weight: bold;
        background-color: #fff;
        color: #000;
        border-radius: 1rem;
        cursor: pointer;
        border-style: dashed;
        border-color: #000;
        border-width: 1px;
    }

    :global(button:active) {
        transform: translate(0, 1px);
    }

    :global(svg) {
        width: 100%;
        height: auto;
        object-fit: contain;
    }
</style>