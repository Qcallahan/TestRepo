<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    
    let tempData = [];
    let selectedCharacters = new Set();

    onMount(async () => {
        const res = await fetch('global.csv'); 
        const csv = await res.text();
        tempData = d3.csvParse(csv, d3.autoType);

        createScatterPlot();
    });

    function createScatterPlot() {
        const margin = { top: 40, right: 40, bottom: 50, left: 60 };
        const width = 600 - margin.left - margin.right;
        const height = 400 - margin.top - margin.bottom;

        const svg = d3.select('main')
            .append('svg')
            .attr('width', width + margin.left + margin.right)
            .attr('height', height + margin.top + margin.bottom)
            .append('g')
            .attr('transform', `translate(${margin.left},${margin.top})`);

        const yScale = d3.scaleLinear()
            .domain([d3.min(tempData, d => d.wins) - 100, d3.max(tempData, d => d.wins) + 100])
            .range([height, 0]);

        const xScale = d3.scaleLinear()
            .domain([d3.min(tempData, d => d.losses) - 100, d3.max(tempData, d => d.losses) + 100])
            .range([0, width]);

        // Define color scale with 21 unique colors
        const colorScale = d3.scaleOrdinal()
            .domain(tempData.map(d => d.most_played_character))
            .range(["#1f77b4", "#aec7e8", "#ff7f0e", "#ffbb78", "#2ca02c", "#98df8a", "#d62728", "#ff9896", "#9467bd", "#c5b0d5", "#8c564b", "#c49c94", "#e377c2", "#f7b6d2", "#7f7f7f", "#c7c7c7", "#bcbd22", "#dbdb8d", "#17becf", "#9edae5", "#1a55FF"]); // Add your 21 colors here

        svg.append('g')
            .attr('transform', `translate(0,${height})`)
            .call(d3.axisBottom(xScale))
            .append('text')
            .attr('x', width / 2)
            .attr('y', margin.bottom - 10)
            .attr('fill', '#000')
            .attr('text-anchor', 'middle')
            .text('Losses');

        svg.append('g')
            .call(d3.axisLeft(yScale))
            .append('text')
            .attr('transform', 'rotate(-90)')
            .attr('x', -height / 2)
            .attr('y', -margin.left + 20)
            .attr('fill', '#000')
            .attr('text-anchor', 'middle')
            .text('Wins');

        const circles = svg.selectAll('circle')
            .data(tempData)
            .enter()
            .append('circle')
            .attr('cx', d => xScale(d.losses))
            .attr('cy', d => yScale(d.wins))
            .attr('r', 5)
            .style('fill', d => colorScale(d.most_played_character))
            .style('opacity', d => selectedCharacters.has(d.most_played_character) ? 1 : 0.1)
            .on('mouseover', handleMouseOver)
            .on('mouseout', handleMouseOut);


        // Create buttons for each character option
        const characterOptions = Array.from(new Set(tempData.map(d => d.most_played_character)));
        const buttons = d3.select('main')
            .insert('div', 'svg')
            .attr('class', 'buttons-container')
            .selectAll('button')
            .data(characterOptions)
            .enter()
            .append('button')
            .text(d => d)
            .on('click', toggleCharacter)
            .style('background-color', d => selectedCharacters.has(d) ? '#66bb6a' : '');


        // Select All button
        d3.select('main')
            .insert('button', 'svg')
            .text('Select All')
            .on('click', selectAll)
            .style('background-color', '');

        // Deselect All button
        d3.select('main')
            .insert('button', 'svg')
            .text('Deselect All')
            .on('click', deselectAll)
            .style('background-color', '');


        function toggleCharacter(event) {
            const character = event.target.textContent;
            if (selectedCharacters.has(character)) {
                selectedCharacters.delete(character);
                d3.select(this).style('background-color', '');
            } else {
                selectedCharacters.add(character);
                d3.select(this).style('background-color', '#66bb6a');
            }

            svg.selectAll('circle')
                .style('opacity', d => selectedCharacters.has(d.most_played_character) ? 1 : 0.05);

            circles.sort((a, b) => {
                if (selectedCharacters.size === 0) return 0; // No selected characters, maintain order
                const aSelected = selectedCharacters.has(a.most_played_character);
                const bSelected = selectedCharacters.has(b.most_played_character);
                if (aSelected && !bSelected) return 1; // A is selected character and B is not, move A to front
                if (!aSelected && bSelected) return -1; // B is selected character and A is not, move B to front
                if (aSelected && bSelected) {
                    // Both A and B are selected characters, prioritize the most recently selected character
                    if (a.most_played_character === character) return 1; // A is the most recently selected character, move to front
                    if (b.most_played_character === character) return -1; // B is the most recently selected character, move to front
                }
                return 0; // Other cases, maintain order
            });
        }

        function selectAll() {
            characterOptions.forEach(character => selectedCharacters.add(character));

            svg.selectAll('circle')
                .style('opacity', 1);

            buttons.style('background-color', '#66bb6a');
        }

        function deselectAll() {
            selectedCharacters.clear();

            svg.selectAll('circle')
                .style('opacity', 0.05);

            buttons.style('background-color', '');
        }

        const tooltip = d3.select('body').append('div')
            .attr('class', 'tooltip')
            .style('opacity', 0);

        function handleMouseOver(event, d) {
            const topRole = d.topRole;
            const rank = d.rank;
            const mostPlayedCharacter = d.most_played_character;
            const username = d.username;

            d3.select(this)
                .style('stroke', 'black')
                .style('stroke-width', 2);

            const tooltip = d3.select('main').append('div')
                .attr('class', 'tooltip')
                .style('opacity', 0)
                .html(`<strong>Username:</strong> ${username}<br><strong>Top Role:</strong> ${topRole}<br><strong>Rank:</strong> ${rank}<br><strong>Most Played Character:</strong> ${mostPlayedCharacter}`);

            const tooltipWidth = parseInt(tooltip.style('width'));
            const tooltipHeight = parseInt(tooltip.style('height'));
            console.log(event.pageX);
            tooltip.style('left', (event.pageX - tooltipWidth / 2) + 'px')
                .style('top', (event.pageY - tooltipHeight - 10) + 'px')
                .transition()
                .duration(200)
                .style('opacity', 1);
        }

        function handleMouseOut() {
            d3.select(this)
                .style('stroke', 'none');

            d3.select('.tooltip')
                .style('opacity', 0)
                .remove();
        }


    }
</script>

<main>
</main>

<style>
    .tooltip {
        position: absolute;
        text-align: center;
        width: 120px;
        height: auto;
        padding: 6px;
        font: 12px sans-serif;
        background: #ddd;
        border: 0px;
        border-radius: 8px;
        pointer-events: none;
    }
</style>
