<script>
    import Filter from "./filter.svelte";
    import Rem from "./rem.svelte";
</script>

<div id="container">
    <h1>CSS Tools</h1>
    <Filter />
    <Rem />
</div>

<style>
    @font-face {
        font-family: "JetBrains Mono";
        src: url("/fonts/JetBrainsMono-Regular.woff2") format("woff2");
        font-weight: normal;
        font-style: normal;
    }
    @font-face {
        font-family: "JetBrains Mono";
        src: url("/fonts/JetBrainsMono-Light.woff2") format("woff2");
        font-weight: 100;
        font-style: normal;
    }

    :global(*) {
        font-family: "JetBrains Mono", sans-serif;
    }
    :global(.container) {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 2rem;
        margin: 1rem;
    }
    :global(input:not([type="color"])) {
        padding: 0.5rem;
    }

    :root {
        background-color: #292E29;
        color: white;
    }

    #container {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        width: 100%;
        height: 100%;
        gap: 3rem;
    }
</style>