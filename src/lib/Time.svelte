<script lang="ts">
    import noUiSlider from 'nouislider';
    // Or the namespace:
    import 'nouislider/dist/nouislider.css';

    import {onMount} from "svelte";

    export let filteredData;
    export let yearMin;
    export let yearMax;
    export let currentYearMin;
    export let currentYearMax;

    let strain;
    let svg;

    onMount(() => {
        let slider = document.getElementById('slider');
        noUiSlider.create(slider, {
            start: [yearMin, yearMax],
            connect: true,
            range: {
                'min': Math.round(yearMin),
                'max': yearMax
            },
            step: 1,
            format: {
                from: function (value) {
                    return parseInt(value);
                },
                to: function (value) {
                    return parseInt(value);
                }
            },
            behaviour: 'tap-drag',
            tooltips: true,
        });
        // }, 1000)


        // slider.noUiSlider.on("update", (e) => {
        slider.noUiSlider.on("change", (e) => {
            [currentYearMin, currentYearMax] = [e[0], e[1]];
        })
    })


</script>

<div id="slider">
</div>

<style>
    #slider {
        /*width: 20%;*/
        /*flex: 30%;*/
        /*margin-top: 25px;*/
        margin-top: 3%;
        width: 200px;
    }
</style>
