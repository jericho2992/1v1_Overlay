<script>
    export let player;
    export let wins=0;
    export let goals=0;
    import { tweened } from "svelte/motion";
    const bTween = tweened(player.boost,
    {
        duration: 100,
    });
    $: bTween.set(player.boost);
</script>
<div class="main">
    {#if player.team===1}
        <div class="score-2">
            <h1>{goals}</h1>
            <div class="win-ticks">
                {#each Array(wins) as _}
                    <div style="background: rgb(10, 10, 10); min-height: 5px; min-width: 5px; max-width: 5px; margin-left: 5px;"></div>
                {/each}
            </div>
        </div>
    {/if}
    <div class="name-plate">
        {#if player.team===0}
            <div class="p1-boost">
                <div style="background: white; min-height: 0.5rem; min-width: {$bTween/100*22}vw"></div>               
            </div>
            <div class="p1-name-div">
                <h1>{player.name}</h1>
            </div>
        {:else}
            <div class="p2-boost">
                <div style="background: white; min-height: 0.5rem; min-width: {$bTween/100*22}vw"></div>
            </div>
            <div class="p2-name-div">            
                <h1>{player.name}</h1>
            </div>
        {/if}
    </div>
    {#if player.team===0}
        <div class="score-1">
            <h1>{goals}</h1>
            <div class="win-ticks">
                {#each Array(wins) as _}
                    <div style="background: rgb(10, 10, 10); min-height: 5px; min-width: 5px; max-width: 5px; margin-left: 5px;"></div>
                {/each}
            </div>
        </div>
    {/if}
</div>
<style>
    .main {
        display: flex;
        flex-direction: row;
        flex-wrap: nowrap;
        margin:0;
        
    }
    .name-plate {
        width: 22vw;
        min-width: 22vw;
    }
    h1{
        margin: 0;
        font-size: 2.5rem;
        padding: 0.5em;
    }
    .p1-name-div {
        color: white;
        background-image: linear-gradient(-45deg, rgb(0, 0, 255), rgb(0, 0, 100));
    }
    .p2-name-div {
        color: white;
        background-image: linear-gradient(-45deg, rgb(210, 140, 0), rgb(111, 54, 0));
    }
    .p1-boost {
        display: flex;
        flex-direction: row;
        justify-content: flex-end;
        min-width: 22vw;
        min-height: 0.5rem;
        background-color: rgb(40, 40, 40);
    }
    .p2-boost {
        display: flex;
        flex-direction: row;
        justify-content: flex-start;
        min-width: 22vw;
        min-height: 0.5rem;
        background-color: rgb(40, 40, 40);
    }
    .score-1 {
        display: flex;
        flex-direction: column;
        flex-wrap: nowrap;
        background-image: linear-gradient(rgb(0, 100, 255), rgb(0, 200, 255));
    }
    .score-2 {
        display: flex;
        flex-direction: column;
        flex-wrap: nowrap;
        background-image: linear-gradient(rgb(180, 120, 0), rgb(255, 220, 0))
    }
    .win-ticks {
        display: flex;
        flex-direction: row;
    }
</style>