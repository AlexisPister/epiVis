<script lang="ts">
    import * as Plot from "@observablehq/plot";
    import * as d3 from "d3"

    import {data, MODEL, parseModels, PUBYEAR} from "../dataLoader.ts";
    import {onMount} from "svelte";
    import {MODELS} from "../dataLoader.js";

    let element: HTMLElement;
    let width;
    let height;
    let overlap = 1;

    function updateDimensions() {
        const rect = element.getBoundingClientRect();
        width = rect.width;
        height = rect.height;
    }

    let modelData = []
    data.forEach(d => {
        let models = parseModels(d[MODEL])
        let time = d[PUBYEAR]
        models.forEach(m => {
            modelData.push({PUBYEAR: time, MODEL: m, name: MODELS[m]})
        })
    })

    onMount(() => {
        updateDimensions()
        let dataFiltered = data.filter(d => d[MODEL] && d[PUBYEAR])

        let group = d3.flatGroup(
            dataFiltered,
            ...[MODEL, PUBYEAR].map(k => d => d[k]) // instead of d => d.Sex and d => d.Eye etc
        )

        group = group.map(d => {
            d.push({v: d[2].length})
            // return {value: d[2].length, year: Number(d[1]), model: d[0]}
            return {value: d[2].length, year: d[1], model: d[0]}
        })

        // let modelGroup = d3.groups(group, d => d[0]);
        let modelGroup = d3.groups(group, d => d.model);

        let ridgePlot = Plot.plot({
            height: 40 + new Set(dataFiltered.map(d => d[MODEL])).size * 50,
            width,
            marginBottom: 1,
            marginLeft: 120,
            x: {axis: "top"},
            y: {axis: null, range: [2.5 * 17 - 2, (2.5 - overlap) * 17 - 2]},
            fy: {label: null, domain: dataFiltered.map(d => d[MODEL])}, // preserve input order
            marks: [
                modelGroup.map(([, values]) => [
                    Plot.areaY(values, {
                        x: "year",
                        y: "value",
                        fy: "model",
                        curve: "basis",
                        sort: "year",
                        fill: "rgba(128,197,227,0.6)"
                    }),
                    Plot.lineY(values, {
                        x: "year",
                        y: "value",
                        fy: "model",
                        curve: "basis",
                        sort: "year",
                        strokeWidth: 1
                    })
                ])
            ]
        })

        // Remove for now
        // document.querySelector("#ridgeplot").append(ridgePlot)


        let heatmap = Plot.plot({
            marginLeft: 250,
            // marginBottom: 80,
            width: width,
            height: height - 30,
            x: {label: null},
            y: {label: "Model Type", },
            color: {label: "Count", legend: true, scheme: "YlGnBu"},
            marks: [
                Plot.cell(modelData, Plot.group({fill: "count"}, {x: "PUBYEAR", y: "name"}))
                // Plot.cell(data, Plot.group({fill: "count"}, {x: PUBYEAR, y: MODEL}))
            ]
        })

        document.querySelector("#heatmap").append(heatmap)

        d3.select("#heatmap").selectAll("svg > g > text").style("font-size", "13px");
    })

</script>

<!--<div id="ridgeplot">-->
<!--</div>-->

<div id="heatmap" bind:this={element}>
</div>

<style>
    #heatmap {
        height: 100%;
    }
</style>