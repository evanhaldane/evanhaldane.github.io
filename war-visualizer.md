---
layout: default
title:  "Career WAR visualizer"
---
What happens to baseball players whose career WARs have followed a certain trajectory? Projection systems give you an expectation for the upcoming year. I made a little tool to vizualize the recent past and near future for players similar to a comparison player of your choice.

<style>
    .baseline {
        fill: none;
        stroke: red;
        stroke-linejoin: round;
        stroke-linecap: round;
        stroke-width: 4.5;
    }
    .similar {
        fill: none;
        stroke: #999999;
        stroke-linejoin: round;
        stroke-linecap: round;
        stroke-width: 2;
    }

    .svg-container{
         display: inline-block;
         position: relative;
         width: 100%;
         vertical-align: middle; /* top | middle | bottom ... do what you want */
    }

    .my-svg{ /* svg into : object, img or inline */
        display: block;
        position: absolute;
        top: 0;
        left: 0;
        width: 100%; /* only required for <img /> */
    }
    form {
        display:inline;
    }
    text {
        font-size:20px;
    }
</style>

<div>Comparison Player: 
<form id="playerForm">
    <input type="text" id="inputPlayerName">
</form>
</div>
<div class="svg-container">
    <svg preserveAspectRatio="xMinYMin meet" viewBox="0 0 960 500">
    </svg>
</div>

<link rel="stylesheet" href="{{"/public/css/auto-complete.css" | relative_url }}">
<script src="https://d3js.org/d3.v4.min.js"></script>
<script src="{{"/public/js/auto-complete.min.js" | relative_url }}"></script>
<script type="text/javascript">
    var svg = d3.select("svg"),
    margin = {top: 20, right: 20, bottom: 30, left: 50},
    width = 960 - margin.left - margin.right,
    height = 500 - margin.top - margin.bottom,
    g = svg.append("g").attr("transform", "translate(" + margin.left + "," + margin.top + ")"),
    players;
    
    var x = d3.scaleLinear()
        .domain([28, 33])
        .rangeRound([0, width]);
    var y = d3.scaleLinear()
            .domain([-2, 10])
            .rangeRound([height, 0]);

    g.append("g")
        .attr("class", "xaxis")
        .attr("transform", "translate(0," + height + ")")
        .call(d3.axisBottom(x).ticks(5).tickFormat(d3.format(".0f")));
    
    g.append("g")
        .call(d3.axisLeft(y))
        .append("text")
        .attr("fill", "#000")
        .attr("transform", "rotate(-90)")
        .attr("y", 6)
        .attr("dy", ".71em")
        .attr("text-anchor", "end")
        .text("WAR600");


    function compareTo(baseId){
        // update for future years
        const comparisonYear = 2017;
        // could be changed via UI
        const yearsToConsider = 3;

        var baselinePlayer = players.find((d) => d.key == baseId);

        // look back number of years requested
        const minAge = (baselinePlayer.values[0].Age - baselinePlayer.values[0].Season +
                         comparisonYear - yearsToConsider + 1);
        // two years past comparison year
        const maxAge = minAge + yearsToConsider+1;
        // only want to compare to players with an appropriate future
        comparable = players.filter((player)=> player.values.some((obj)=>obj.Age == maxAge));
        // add in original player
        comparable.push(baselinePlayer)
        // calculate difference for comparables and add to object
        comparable = comparable.map(function(player){
            player.difference = difference(baselinePlayer, player, minAge, maxAge);
            return player
        });
        
        // take twenty most similar players
        var similar = comparable.sort((player1,player2) => player1.difference-player2.difference).slice(0, 21);
        
        // sort by Age for graphing
        similar = similar.map((player) => {
            player.values = player.values.sort((year1, year2) => year1.Age - year2.Age);
            return player;
            });
        
        // create x scale from minimum age to maximum age
        x = d3.scaleLinear()
            .domain([minAge, maxAge])
            .rangeRound([0, width]);

        // line function based on defined x and y scales
        line = d3.line()
            .x((d) => x(d.Age))
            .y((d) => y(d.WAR600));

        // update axis
        g.select(".xaxis")
            .call(d3.axisBottom(x).ticks(yearsToConsider+2).tickFormat(d3.format(".0f")));

        // update selection for similar players
        var similarSelection = g.selectAll("path.similar")
            .data(similar.slice(1,21));
        
        // update selection for baseline player
        var baselineSelection = g.selectAll("path.baseline")
            .data(similar.slice(0,1));

        // remove outgoing data
        similarSelection.exit().remove();
        baselineSelection.exit().remove();

        // for new similar players, draw lines
        // opacity depends exponentially on similarity
        similarSelection
            .enter()
            .append("path")
            .attr("class", "similar")
            .merge(similarSelection)
            .attr("d", (d) => line(d.values.filter((d)=>d.Age >= minAge && d.Age <= maxAge)))
            .style("stroke-opacity", (d)=>1.54*Math.exp(-d.difference*1.74));

        // draw new baseline player
        baselineSelection
            .enter()
            .append("path")
            .attr("class", "baseline")
            .merge(baselineSelection)
            .attr("d", (d) => line(d.values.filter((d)=>d.Age >= minAge && d.Age <= maxAge)));
    }
    
    function difference(baselinePlayer, comparisonPlayer, minAge, maxAge){
        const mostRecentAge = (maxAge - 2);
        totalDiff = baselinePlayer.values.reduce((runningTotal, currentObj) => {
            baselineAge = currentObj.Age;
            if (baselineAge >= minAge && baselineAge <= maxAge){
                correspondingPlayerYear = comparisonPlayer.values.find((d)=>d.Age == baselineAge);
                var currentDiff = Math.abs(+currentObj.WAR600 - 
                    (correspondingPlayerYear ? +correspondingPlayerYear.WAR600 : 0));
                if (baselineAge == mostRecentAge){
                    currentDiff = currentDiff * 0.6;
                } else if (baselineAge == mostRecentAge - 1){
                    currentDiff = currentDiff * 0.3;
                } else if (baselineAge == mostRecentAge - 2){
                    currentDiff = currentDiff * 0.1;
                }
            } else {
                currentDiff = 0;
            }
            
            return runningTotal + currentDiff;
        }, 0);

        return totalDiff;
    }

    document.addEventListener("DOMContentLoaded", function(){
            // Handler when the DOM is fully loaded

            d3.csv("seasons.csv", function(data){
                // create array of players indexed by playerid
            players = d3.nest()
                        .key((d)=>d.playerid)
                        .entries(data);
            // calculate WAR600 for each player-year
            players = players.map((playerObject) => {
                playerObject.values = playerObject.values.map((year) => {
                    year.WAR600 = (+year.WAR / + year.PA) * 600;
                    return year;
                })
                return playerObject;
            });

            // players who have a recent season can be chosen for comparison
            var recentPlayers = players.filter((player) => player.values.some((d)=>d.Season == "2017"));
            // get name out of one of the seasons for player
            recentPlayers = recentPlayers.map((d) => ({id: d.key, Name: d.values[0].Name}));
            
            // create autoComplete based on player names
            new autoComplete({
                selector: '#inputPlayerName',
                minChars: 2,
                source: function(term, suggest){
                    term = term.toLowerCase();
                    var matches = [];
                    for (i=0; i<recentPlayers.length; i++) {
                        if (~recentPlayers[i].Name.toLowerCase().indexOf(term)) {
                            matches.push(recentPlayers[i]);
                        }
                    }
                    suggest(matches);
                },
                // render options in autocomplete with player name and player id as attribute
                renderItem: function (item, search){
                        search = search.replace(/[-\/\\^$*+?.()|[\]{}]/g, '\\$&');
                        var re = new RegExp("(" + search.split(' ').join('|') + ")", "gi");
                        return '<div class="autocomplete-suggestion" data-id="' +
                                item.id + '" data-name="' + item.Name + '" data-val="' + 
                                search + '">'+item.Name.replace(re, "<b>$1</b>")+'</div>';
                    },
                // upon selection, get player name and call function to compare others to player
                onSelect: function(e, term, item){
                        playerid = +item.getAttribute('data-id');
                        document.getElementById("inputPlayerName").value = item.textContent;
                        compareTo(playerid);
                    }
            });
        });

    });

</script>

Notes: 

- Here we visualize WAR600, or WAR per 600 plate appearances

- "Similarity" is defined as the sum of the absolute distance between the player's WAR over the previous three seasons compared to other players at the same ages.

- Only position players are included here.
