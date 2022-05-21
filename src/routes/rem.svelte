<script>
    let px;
    let rem;

    const setPx = () => rem = px / 16;
    const setRem = () => px = rem * 16;

    $: setPx(px);
    $: setRem(rem);
</script>

<div class="container">
    <h3>PX to REM</h3>

    <div class="input-group">
        <label for="px">PX:</label>
        <input name="px" type="number" bind:value={px}/>
    </div>
    <div class="input-group">
        <label for="rem">REM:</label>
        <input name="rem" type="number" bind:value={rem}/>
    </div>
</div>
