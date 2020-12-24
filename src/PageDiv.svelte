<div class="flex-initial">
    <canvas id='canvasPage{page.pageNumber}' bind:this={canvas}>
        <div id='textDivPage{page.pageNumber}' bind:this={div}></div>
    </canvas>
</div>

<script lang="ts">
    export let page;
    export let scale;
    export let lecturePart;

    import { TextLayerBuilder } from './text-layer/text_layer_builder.js';
    import { onMount } from 'svelte';
    import { EventBus } from './text-layer/ui_utils.js';
    
    let canvas: HTMLCanvasElement;
    let div: HTMLDivElement;
    let eventBus = new EventBus();

    onMount(async () => {
        let viewPort = page.getViewport({ scale: scale })
        let context = canvas.getContext("2d");
    
        canvas.width = viewPort.width;
        canvas.height = viewPort.height;
    
        page.render({
            canvasContext: context,
            viewport: viewPort
        });

        let textContent = await page.getTextContent();
        let textLayer = new TextLayerBuilder({
            textLayerDiv: div,
            eventBus: eventBus,
            pageIndex: page.pageNumber,
            viewport: viewPort
        });

        textLayer.setTextContent(textContent);
        textLayer.render();
    });

</script>