<div>
    <input id="inputSlides" type="file" accept=".pdf" on:input={readFile}/>
    <input id="inputLecture" type="file" accept=".mp4" on:input={readFile}/>
    <button on:click={() => order = !order}>Toggle order</button>
</div>
<div class="flex flex-{order ? "col" : "row"} flex-wrap" id="pdfDiv">
</div>

<script lang="ts">
    import pdfjs from 'pdfjs-dist';
    import pdfjsWorkerEntry from "pdfjs-dist/build/pdf.worker.entry";
    
    pdfjs.disableWorker = true;

    let slides: File;
    let lecture: File;

    let order = true;

    function readFile(e: Event) {
        let target: HTMLInputElement =  e.currentTarget;
        let files = [...target.files];

        let slidesList = files.filter(file => file.name.includes(".pdf"))
        let lectureList = files.filter(file => file.name.includes(".mp4"));
        
        if (slidesList.length) {
            slides = slidesList[0];

            renderPDF(slides)
        }

        if (lectureList.length) {
            lecture = lectureList[0];
        }
    }

    let pages = [];
    let scale = 1;
    let pdfDiv: HTMLDivElement;

    function draw() {
        if (!pdfDiv) {
            pdfDiv = document.getElementById('pdfDiv');
            
            pdfDiv.addEventListener("wheel", zoom);
        }

        pdfDiv.innerHTML = ''
        
        for (let page of pages) {
            let canvas: HTMLCanvasElement = document.createElement("canvas");
            let context = canvas.getContext("2d");
            
            canvas.width = page.width;
            canvas.height = page.height;
            
            context.putImageData(page, 0, 0);

            pdfDiv.appendChild(canvas);
        }
    }

    async function renderPDF(pdfFile: File) {
        let pdf = await pdfjs.getDocument(new Uint8Array(await pdfFile.arrayBuffer())).promise
        
        pages = []
        
        let pageIndices = [...Array(pdf.numPages).keys()].map(i => i + 1)
        for (let pageIndex of pageIndices) {
            let page = await pdf.getPage(pageIndex) 
            
            let viewport = page.getViewport({ scale: scale });
            let canvas: HTMLCanvasElement = document.createElement('canvas');
            let ctx = canvas.getContext('2d');
            let renderContext = { canvasContext: ctx, viewport: viewport };

            canvas.height = viewport.height;
            canvas.width = viewport.width;

            await page.render(renderContext).promise
            pages.push(ctx.getImageData(0, 0, canvas.width, canvas.height));
        }

        draw()
    }

    async function zoom(e: WheelEvent) {
        if (e.ctrlKey) {
            e.preventDefault();
            scale += e.deltaY * -0.01;

            await renderPDF(slides);
            console.log(scale)
        }
    }
</script>