<script lang="ts">
    import * as d3 from "d3";

    import {onMount} from "svelte";
    import {affiliationsFilename, authorsFilename, papersFilename} from "../dataLoader.ts";

    // export let filteredData;

    // let specPah = "../netpanorama-vis/templates/institutionProj.json"
    // let specPath = "../../netpanorama-vis/templates/institutionProj.json"
    let specPath = "/netpanorama-vis/templates/InstitutionProj.json"

    let width: number = 1200;
    let height: number = 600;

    let legendWidth = 200;
    let legendHeight = 350;

    let institutions;
    let links;

    let element: HTMLElement;
    let svg: HTMLElement;
    let svgLegend: SVGElement;
    let colorScale = d3.scaleLinear([0, 10], ["white", "blue"]);
    let nodeToNeighbors = {};

    onMount(() => {
        getNetpanNet().then(() => {
            render(width, height);
        })
    })

    function updateDimensions() {
        const rect = element.getBoundingClientRect();
        width = rect.width - legendWidth - 200;
        height = rect.height;
    }

    async function getNetpanNet() {
        if (width && element) {
            // element.innerHTML = ""

            let viewer = await NetPanoramaTemplateViewer.render(specPath, {
                affiliations: affiliationsFilename,
                width: width,
                height: height
            }, "affiliationNet");

            // institutions = viewer.state.networkAff.nodes
            institutions = viewer.state.network.nodes
            links = viewer.state.network.links

            console.log(333, viewer)
            // console.log(33, links.filter(l => l.source.id == "Korea University"))

            // Fill neighbor map
            institutions.forEach(node => {
                nodeToNeighbors[node.id] = [];
            })
            links.forEach(link => {
                let source = link.source.id;
                let target = link.target.id;

                nodeToNeighbors[source].push(target);
                nodeToNeighbors[target].push(source);
            })
        }
    }

    function render(width, height) {
        if (!svg) return;

        institutions.sort((a, b) => {
            let countryCompare = a.data["Country "].localeCompare(b.data["Country "])
            if (countryCompare == 0) {
                return a.data["Discipline"].localeCompare(b.data["Discipline"])
            }
            return countryCompare
        })

        let currentCountry = institutions[0].data["Country "]
        let pos = 1
        let countryToCount = {};

        for (let inst of institutions) {
            let country = inst.data["Country "]

            if (countryToCount[country]) {
                countryToCount[country] += 1
            } else {
                countryToCount[country] = 1
            }

            if (currentCountry != country) {
                pos += 1;
                currentCountry = country;
            }
            inst.pos = pos;

            pos += 1;
        }

        const radialScale = d3.scaleLinear([0, pos], [0, Math.PI * 2])

        // const countryColorScale = d3.scaleOrdinal(d3.schemeAccent)
        const countryColorScale = d3.scaleOrdinal(d3.schemeSet1)

        const radiusScale = d3.scaleLinear(d3.extent(institutions.map(d => d.degree)), [4, 10]);

        const distance = 300;
        const x = (d) => distance * Math.cos(radialScale(d.pos))
        const y = (d) => {
            return distance * Math.sin(radialScale(d.pos))
        }

        //  Lines
        // const linksMark = d3.select(svg)
        //     .selectAll(".link")
        //     .data(links)
        //     .join("line")
        //     // .attr("d", d => line(d))
        //     .attr("x1", d => x(d.source))
        //     .attr("y1", d => y(d.source))
        //     .attr("x2", d => x(d.target))
        //     .attr("y2", d => y(d.target))
        //     .attr("stroke", "black")
        //     .classed("link", true)

        const linksMark = d3.select(svg)
            .selectAll(".link")
            .data(links)
            .join("path")
            .attr("d", d => {
                let x1 = x(d.source);
                let x2 = x(d.target);
                let y1 = y(d.source);
                let y2 = y(d.target);

                const distance = Math.sqrt((x2 - x1) ** 2 + (y2 - y1) ** 2)

                let xMid = (x2 + x1)/2
                let yMid = (y2 + y1)/2

                let width = (0 - xMid)
                let height = (0 - yMid)

                let xI = 0 - width + width * distance / 300;
                let yI = 0 - height + height * distance / 300;

                return `M ${x1} ${y1} Q ${xI} ${yI} ${x2} ${y2}`
            })
            .attr("stroke", "black")
            .attr("fill", "none")
            .classed("link", true)

        let nodesMark = d3.select(svg)
            .selectAll(".node")
            .data(institutions)
            .join("circle")
            .attr("cx", d => x(d))
            .attr("cy", d => y(d))
            .attr("r", d => radiusScale(d.degree))
            // .attr("r", 7)
            // .attr("fill", d => countryColorScale(d.data["Country "]))
            .attr("fill", d => countryColorScale(d.data["Discipline"]))
            .attr("stroke", "black")
            .classed("node", true)
            .on("mouseover", mouseOver)
            .on("mouseleave", mouseLeave)

        function mouseOver(e, d) {
            d3.selectAll(".node")
                .style("opacity", d2 => {
                    if (!nodeToNeighbors[d.id].includes(d2.id)) {
                        return 0.3;
                    }
                })
                .attr("stroke-width", d2 => {
                    if (nodeToNeighbors[d.id].includes(d2.id)) {
                        return 3;
                    } else if (d2 == d) {
                        return 6;
                    }
                })

            d3.select(this)
                .style("opacity", 1)

            d3.selectAll(".nodeText")
                .attr("display", dtext => {
                    if (d.id == dtext.id) {
                        return ""
                    }
                    return "none"
                })

            // TODO: make it faste by precomputing neighbors
            d3.selectAll(".link")
                .style("opacity", link => {
                    if (link.source.id != d.id && link.target.id != d.id) {
                        return 0.2;
                    }
                    return 1;
                })

            d3.selectAll(".country")
                .style("opacity", d2 => {
                    if (d.data["Country "] == d2.data[0]) {
                        return 1
                    }
                    return 0.5
                })
        }

        function mouseLeave(e, d) {
            d3.selectAll(".node")
                .style("opacity", 1)
                .attr("stroke-width", 1)

            d3.selectAll(".link")
                .style("opacity", link => {
                    return 1
                })
        }

        const pie = d3.pie().padAngle(0.05).sort(null).sortValues(null).startAngle(Math.PI / 2).endAngle(3 * Math.PI).value(d => d[1]);
        const arc = d3.arc().innerRadius(distance + 40).outerRadius(distance + 40 + 50);

        const countries = d3.select(svg)
            .append("g")
            .attr("fill", "#ccc")
            .attr("stroke", "#000")
            .attr("stroke-width", "1px")
            .attr("stroke-linejoin", "round")
            .selectAll(".country")
            .data(pie(Object.entries(countryToCount)))
            .join("path")
            .attr("d", arc.cornerRadius(6))
            .classed("country", true)
            .on("mouseover", mouseOverCountry)
            .on("mouseleave", mouseLeaveCountry)

        d3.select(svg)
            .selectAll("textCountry")
            // .data(pie(Object.values(countryToCount)))
            .data(pie(Object.entries(countryToCount)))
            .join("text")
            .attr("transform", (d, i) => {
                return `translate(${arc.centroid(d)})`
            })
            .attr("x", (d, i) => {
                return 0;
            })
            .attr("y", d => {
                return 0
            })
            .text((d, i) => {
                return Object.keys(countryToCount)[i]
            })
            .attr("text-anchor", "middle")
            .classed("textCountry", true)
            .on("mouseover", mouseOverCountry)
            .on("mouseleave", mouseLeaveCountry)


        function mouseOverCountry(e, d) {
            d3.selectAll(".node")
                .style("opacity", .5)
            // d3.selectAll(".country")
            //     .style("opacity", .5)

            d3.selectAll(".country")
                .style("opacity", d2 => {
                    if (d.index == d2.index) {
                        return 1
                    }
                    return 0.5
                })

            d3.selectAll(".textCountry")
                .style("opacity", dtext => {
                    if (d.index == dtext.index) {
                        return 1
                    }
                    return 0.5
                })

            d3.selectAll(".nodeText")
                .attr("display", dtext => {
                    if (d.id == dtext.id) {
                        return ""
                    }
                    return "none"
                })

            // TODO: make it faste by precomputing neighbors
            d3.selectAll(".link")
                .style("opacity", link => {
                    if (link.source.id != d.id && link.target.id != d.id) {
                        return 0.2;
                    }
                    return 1;
                })
        }

        function mouseLeaveCountry(e, d) {
            d3.selectAll(".node")
                .style("opacity", 1)

            d3.selectAll(".link")
                .style("opacity", link => {
                    return 1
                })
        }



        // Institutions labels
        d3.select(svg)
            .selectAll(".nodeText")
            .data(institutions)
            .join("text")
            .attr("x", d => x(d) + 8)
            .attr("y", d => y(d) - 10)
            .text(d => d.id)
            .classed("nodeText", true)
            .attr("display", "none")
            .on("mouseover", mouseOver)
            .on("mouseleave", mouseLeave)


        renderLegend();
        function renderLegend() {
            // Add one dot in the legend for each name.
            d3.select(svgLegend)
                .selectAll("mydots")
                .data(countryColorScale.domain())
                .join("circle")
                .attr("cx", 100)
                .attr("cy", function (d, i) {
                    return 100 + i * 25
                }) // 100 is where the first dot appears. 25 is the distance between dots
                .attr("r", 7)
                .style("fill", function (d) {
                    return countryColorScale(d)
                })

            // Add one dot in the legend for each name.
            d3.select(svgLegend)
                .selectAll("mylabels")
                .data(countryColorScale.domain())
                .join("text")
                .attr("x", 120)
                .attr("y", function (d, i) {
                    return 100 + i * 25
                }) // 100 is where the first dot appears. 25 is the distance between dots
                .style("fill", function (d) {
                    return countryColorScale(d)
                })
                .text(function (d) {
                    return d
                })
                .attr("text-anchor", "left")
                .style("alignment-baseline", "middle")
        }
    }


    $: render(width, height)
</script>

<div id="circular-div" bind:this={element} on:resize={updateDimensions}>
    <svg bind:this={svg} width={width} height={width} viewBox="{-width / 2}, {-width / 2}, {width}, {width}">
        <!--    <svg bind:this={svg} width={width} height={width}>-->
    </svg>
    <svg bind:this={svgLegend} width={legendWidth} height={legendHeight}>
    </svg>
</div>
<div id="affiliationNet">
</div>


<style>
    #circular-div {
        /*flex-grow: 1;*/
        /*flex: 1;*/
        /*width: 0;*/
        /*width: 1000px;*/
        /*height: 1000px;*/
        justify-content: center;
        align-items: center;
    }

    svg {
        display: inline-block;
        vertical-align: top;
    }
</style>

