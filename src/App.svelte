<script>
  import PlayerPanel from "./PlayerPanel.svelte";
  import { fade } from "svelte/transition";
    import { identity } from "svelte/internal";
  let players=[
    {
        name: "Blue",
        team: 0,
        boost: 33,
        score: 0,
    },
    {
        name: "Orange",
        team: 1,
        boost: 33,
        score: 0,
    }
  ];
  let teams = [
    {score: 0},
    {score: 0},
  ];
  let time=300;
  let OT=false;
  let focus;
  let t1Wins=0;
  let t2Wins=0;
  let stats = false;
  $: seconds = time % 60;
  $: minutes = Math.floor(time/60);
  const WsSubscribers = {
    __subscribers: {},
    websocket: undefined,
    webSocketConnected: false,
    registerQueue: [],
    init: function(port, debug, debugFilters) {
        port = port || 49322;
        debug = debug || false;
        if (debug) {
            if (debugFilters !== undefined) {
                console.warn("WebSocket Debug Mode enabled with filtering. Only events not in the filter list will be dumped");
            } else {
                console.warn("WebSocket Debug Mode enabled without filters applied. All events will be dumped to console");
                console.warn("To use filters, pass in an array of 'channel:event' strings to the second parameter of the init function");
            }
        }
        WsSubscribers.webSocket = new WebSocket("ws://localhost:" + port);
        WsSubscribers.webSocket.onmessage = function (event) {
            let jEvent = JSON.parse(event.data);
            if (!jEvent.hasOwnProperty('event')) {
                return;
            }
            let eventSplit = jEvent.event.split(':');
            let channel = eventSplit[0];
            let event_event = eventSplit[1];
            if (debug) {
                if (!debugFilters) {
                    console.log(channel, event_event, jEvent);
                } else if (debugFilters && debugFilters.indexOf(jEvent.event) < 0) {
                    console.log(channel, event_event, jEvent);
                }
            }
            WsSubscribers.triggerSubscribers(channel, event_event, jEvent.data);
        };
        WsSubscribers.webSocket.onopen = function () {
            WsSubscribers.triggerSubscribers("ws", "open");
            WsSubscribers.webSocketConnected = true;
            WsSubscribers.registerQueue.forEach((r) => {
                WsSubscribers.send("wsRelay", "register", r);
            });
            WsSubscribers.registerQueue = [];
        };
        WsSubscribers.webSocket.onerror = function () {
            WsSubscribers.triggerSubscribers("ws", "error");
            WsSubscribers.webSocketConnected = false;
        };
        WsSubscribers.webSocket.onclose = function () {
            WsSubscribers.triggerSubscribers("ws", "close");
            WsSubscribers.webSocketConnected = false;
        };
    },
    /**
     * Add callbacks for when certain events are thrown
     * Execution is guaranteed to be in First In First Out order
     * @param channels
     * @param events
     * @param callback
     */
    subscribe: function(channels, events, callback) {
        if (typeof channels === "string") {
            let channel = channels;
            channels = [];
            channels.push(channel);
        }
        if (typeof events === "string") {
            let event = events;
            events = [];
            events.push(event);
        }
        channels.forEach(function(c) {
            events.forEach(function (e) {
                if (!WsSubscribers.__subscribers.hasOwnProperty(c)) {
                    WsSubscribers.__subscribers[c] = {};
                }
                if (!WsSubscribers.__subscribers[c].hasOwnProperty(e)) {
                    WsSubscribers.__subscribers[c][e] = [];
                    if (WsSubscribers.webSocketConnected) {
                        WsSubscribers.send("wsRelay", "register", `${c}:${e}`);
                    } else {
                        WsSubscribers.registerQueue.push(`${c}:${e}`);
                    }
                }
                WsSubscribers.__subscribers[c][e].push(callback);
            });
        })
    },
    clearEventCallbacks: function (channel, event) {
        if (WsSubscribers.__subscribers.hasOwnProperty(channel) && WsSubscribers.__subscribers[channel].hasOwnProperty(event)) {
            WsSubscribers.__subscribers[channel] = {};
        }
    },
    triggerSubscribers: function (channel, event, data) {
        if (WsSubscribers.__subscribers.hasOwnProperty(channel) && WsSubscribers.__subscribers[channel].hasOwnProperty(event)) {
            WsSubscribers.__subscribers[channel][event].forEach(function(callback) {
                if (callback instanceof Function) {
                    callback(data);
                }
            });
        }
    },
    send: function (channel, event, data) {
        if (typeof channel !== 'string') {
            console.error("Channel must be a string");
            return;
        }
        if (typeof event !== 'string') {
            console.error("Event must be a string");
            return;
        }
        if (channel === 'local') {
            this.triggerSubscribers(channel, event, data);
        } else {
            let cEvent = channel + ":" + event;
            WsSubscribers.webSocket.send(JSON.stringify({
                'event': cEvent,
                'data': data
            }));
        }
    }
  };
  WsSubscribers.init(49322, false);
  WsSubscribers.subscribe('game', 'update_state', (d)=>{
    const num = Object.values(d.players).length;
    if(num>1 && num<3) {
        players=Object.values(d.players); 
    }  
    teams=d.game.teams;
    time=d.game.time_seconds;
    OT=d.game.isOT;
    focus=d.game.target;
});
WsSubscribers.subscribe('game', 'match_ended', (d)=>{
    console.log(d);
    if(d.winner_team_num===0) {
        t1Wins = t1Wins+1;
    } else {
        t2Wins = t2Wins+1;
    }
    stats = true;
});

WsSubscribers.subscribe('game', 'match_destroyed', (d)=>{
    stats = false;
});

//score stats styling
$: if(players[0].score>players[1].score) {
    const p1 = document.getElementById("p1-score");
    const p2 = document.getElementById("p2-score");
    if(p1 && p2) {
        p1.style.color = "red";
        p2.style.color = "black";
    }
} else if(players[1].score>players[0].score) {
    const p1 = document.getElementById("p1-score");
    const p2 = document.getElementById("p2-score");
    if(p1 && p2) {
        p2.style.color = "red";
        p1.style.color = "black";
    }
}

//shots stats styling
$: if(players[0].shots>players[1].shots) {
    const p1 = document.getElementById("p1-shots");
    const p2 = document.getElementById("p2-shots");
    if(p1 && p2) {
        p1.style.color = "red";
        p2.style.color = "black";
    }
} else if(players[1].shots>players[0].shots) {
    const p1 = document.getElementById("p1-shots");
    const p2 = document.getElementById("p2-shots");
    if(p1 && p2) {
        p2.style.color = "red";
        p1.style.color = "black";
    }
}

//saves stats styling
$: if(players[0].saves>players[1].saves) {
    const p1 = document.getElementById("p1-saves");
    const p2 = document.getElementById("p2-saves");
    if(p1 && p2) {
        p1.style.color = "red";
        p2.style.color = "black";
    }
} else if(players[1].saves>players[0].saves) {
    const p1 = document.getElementById("p1-saves");
    const p2 = document.getElementById("p2-saves");
    if(p1 && p2) {
        p2.style.color = "red";
        p1.style.color = "black";
    }
}

//demos stats styling
$: if(players[0].demos>players[1].demos) {
    const p1 = document.getElementById("p1-demos");
    const p2 = document.getElementById("p2-demos");
    if(p1 && p2) {
        p1.style.color = "red";
        p2.style.color = "black";
    }
} else if(players[1].demos>players[0].demos) {
    const p1 = document.getElementById("p1-demos");
    const p2 = document.getElementById("p2-demos");
    if(p1 && p2) {
        p2.style.color = "red";
        p1.style.color = "black";
    }
}

</script>

<main>
    {#if players[0].team===0}
     <PlayerPanel bind:player={players[0]} bind:wins={t1Wins} bind:goals={teams[0].score}></PlayerPanel>
     <div class='clock'>
        {#if !OT}
            {#if seconds<10}
               <h1>{minutes}:0{seconds}</h1>
            {:else}
               <h1>{minutes}:{seconds}</h1>
            {/if}
        {:else}
            {#if seconds<10}
              <h1>+{minutes}:0{seconds}</h1>
           {:else}
              <h1>+{minutes}:{seconds}</h1>
           {/if}
        {/if}
    </div>
     <PlayerPanel bind:player={players[1]} bind:wins={t2Wins} bind:goals={teams[1].score}></PlayerPanel>
    {:else}
     <PlayerPanel bind:player={players[1]} bind:wins={t1Wins} bind:goals={teams[0].score}></PlayerPanel>
     <div class='clock'>
        {#if !OT}
            {#if seconds<10}
               <h1>{minutes}:0{seconds}</h1>
            {:else}
               <h1>{minutes}:{seconds}</h1>
            {/if}
        {:else}
            {#if seconds<10}
              <h1>+{minutes}:0{seconds}</h1>
           {:else}
              <h1>+{minutes}:{seconds}</h1>
           {/if}
        {/if}
    </div>
     <PlayerPanel bind:player={players[0]} bind:wins={t2Wins} bind:goals={teams[1].score}></PlayerPanel>
    {/if}
    {#if stats && players[0].team===0}
        <div class='stats-contain' transition:fade="{{duration: 5000}}">
            <table>
                <tr><th></th><th>{players[0].name}</th><th>{players[1].name}</th></tr>
                <tr><td id="p1-score">{players[0].score}</td><td>Score</td><td id="p2-score">{players[1].score}</td></tr>
                <tr><td id="p1-shots">{players[0].shots}</td><td>Shots</td><td id="p2-shots">{players[1].shots}</td></tr>
                <tr><td id="p1-saves">{players[0].saves}</td><td>Saves</td><td id="p2-saves">{players[1].saves}</td></tr>
                <tr><td id="p1-demos">{players[0].demos}</td><td>Demos</td><td id="p2-demos">{players[1].demos}</td></tr>
            </table>
        </div>
    {/if}
    {#if stats && players[0].team===1}
        <div class='stats-contain' transition:fade="{{duration: 5000}}">
            <table>
                <tr><th></th><th>{players[1].name}</th><th>{players[0].name}</th></tr>
                <tr><td id="p2-score">{players[1].score}</td><td>Score</td><td id="p1-score">{players[0].score}</td></tr>
                <tr><td id="p2-shots">{players[1].shots}</td><td>Shots</td><td id="p1-shots">{players[0].shots}</td></tr>
                <tr><td id="p2-saves">{players[1].saves}</td><td>Saves</td><td id="p1-saves">{players[0].saves}</td></tr>
                <tr><td id="p2-demos">{players[1].demos}</td><td>Demos</td><td id="p1-demos">{players[0].demos}</td></tr>
            </table>
        </div>
    {/if}
</main>

<style>
  main {
    display: flex;
    flex-direction: row;
    flex-wrap: nowrap;
    align-items: flex-start;
    justify-content: center;
    background: none;
  }
  .clock {
    background-color: black;
    color: white;
  }
  h1{
        margin: 0;
        font-size: 2.5rem;
        padding: 0.5em 0.35em;
    }
    .stats-contain {
        display: flex;
        flex-direction: column;
        justify-content: flex-start;
        align-items: center;
        position: fixed;
        min-height: 200vh;
        min-width: 100vw;
        background-image: linear-gradient(to right, blue, black, orange);
        z-index: -1;
    }
    table, th, td {
        font-size: 1.5em;
        padding: 0.7em;
    }
    table {
        margin-top: 30vh;
        background-image: linear-gradient(-45deg, rgb(150, 150, 150), rgb(50, 50, 50));
        border-collapse: collapse;
        border-radius: 0.5em;
    }
    th {
        padding: 0.7em;
    }
</style>
