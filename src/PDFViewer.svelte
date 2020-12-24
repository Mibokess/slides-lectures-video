<canvas id="canvas" bind:this={canvas}/>
<video id='video'/>
<div>
    <input id="inputSlides" type="file" accept=".pdf" on:input={readFile}/>
    <input id="inputLecture" type="file" accept=".mp4" on:input={readFile}/>
    <button on:click={() => order = !order}>{ order ? "Column" : "Row" }</button>
    <button on:click={() => getImagesInVideoParts(getSlidesImages(), lecture)}>Process</button>
    <div>
        <p>{stage} of {numberOfStages}</p>
        <progress value={$progress}></progress>
    </div>
</div>
<div class="flex items-center flex-{order ? "col" : "row"} flex-wrap" id="pdfDiv">
    {#each pages as { page, scale } (page.pageNumber)}
        <PageDiv {page} {scale} {lecturePart} />
    {/each}
</div>

<script lang="ts">
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';

	const progress = tweened(0, {
		duration: 400,
		easing: cubicOut
	});

    import PageDiv from './PageDiv.svelte';
    
    import pdfjslib from 'pdfjs-dist';
    import pdfjsWorkerEntry from "pdfjs-dist/build/pdf.worker.entry";

    import pixelmatch from 'pixelmatch';
    import cropImageData from 'crop-image-data';

    import { createFFmpeg, fetchFile } from "@ffmpeg/ffmpeg";
    const ffmpeg = createFFmpeg({ log: true });
    const ffmpegLoad = ffmpeg.load()

    let stage = 1;
    let numberOfStages = 3;

    let slides: File;
    let lecture: File;
    let lecturePart;

    let pages = [];
    let pdf;
    let pageNumbers: number[];
    let scale = 1;

    let order = true;
    let canvas: HTMLCanvasElement;

    async function readFile(e: Event) {
        let target: HTMLInputElement =  e.currentTarget;
        let files = [...target.files];

        let slidesList = files.filter(file => file.name.includes(".pdf"))
        let lectureList = files.filter(file => file.name.includes(".mp4"));
        
        if (slidesList.length) {
            slides = slidesList[0];

            getPages(slides);
        }

        if (lectureList.length) {
            lecture = lectureList[0];
        }
    }

    async function getPages(pdfFile: File) {
        pdf = await pdfjslib.getDocument(new Uint8Array(await pdfFile.arrayBuffer())).promise
        
        pages = []
        
        pageNumbers = [...Array(pdf.numPages).keys()].map(i => i + 1)
        for (let pageNumber of pageNumbers) {
            let page = await pdf.getPage(pageNumber) 

            pages.push({
                page: page,
                scale: scale
            });
        }

        pages = pages
    }

    function getSlidesImages(): ImageData[] {
        let slidesImages: ImageData[] = [];

        for (let pageNumber of pageNumbers) {
            let canvasPage: HTMLCanvasElement = document.getElementById("canvasPage" + pageNumber);
            let context = canvasPage.getContext("2d");

            let imageData = context.getImageData(0, 0, canvasPage.width, canvasPage.height);
            slidesImages.push(imageData);
        }

        return slidesImages;
    }

    interface frame {
        imageData: ImageData,
        frameNumber: number
    }

    interface segment {
        start: number,
        end: number
    }

    async function getImagesInVideoParts(images: ImageData[], video: File) {
        let videoLength = await getVideoLength(video);
        
        let videoFrames = await getVideoFrames(video, videoLength, 30);
        stage++;
        progress.set(0);

        let videoSegments = await getVideoSegments(images, videoFrames, videoLength);
        stage++;
        progress.set(0);

        let videoFiles = await getVideoFiles(video, videoSegments);
        stage++;
        
        return videoFiles;
    }

    async function getVideoFrames(video: File, videoLength:number, secondsPerFrame: number): Promise<frame[]> {
        await ffmpegLoad;
        ffmpeg.FS('writeFile', 'lecture.mp4', await fetchFile(video));
        
        let frames: frame[] = [];

        let viewPort = pages[0].page.getViewport({ scale: scale });
        let width = viewPort.width;
        let height = viewPort.height;

        let maxIndex = Math.floor(videoLength / secondsPerFrame) + 1;
        for (let i = 0; i < 10; i++) {
            await ffmpeg.run(
                '-accurate_seek', 
                '-ss', 
                `${i * secondsPerFrame}`,
                '-i', 
                'lecture.mp4', 
                '-vf',
                `format=rgba,scale=${width}:${height}`,
                '-frames:v', '1', 
                '-f', 'rawvideo', 
                `frame${i}.bin`
            );
            
            let bin = await ffmpeg.FS('readFile', `frame${i}.bin`);
            let imageData: ImageData = new ImageData(new Uint8ClampedArray(bin), width, height);

            frames.push({ imageData: imageData, frameNumber: i * secondsPerFrame });
            
            progress.set(i / (maxIndex - 1));
        }

        return frames;
    }

    async function getVideoLength(video: File): Promise<number> {
        let videoElement: HTMLVideoElement = document.createElement('video');
        videoElement.preload = 'metadata';

        let durationPromise: Promise<number> = new Promise(function(resolve, reject) {
            videoElement.onloadedmetadata = function() {
                resolve(videoElement.duration);
            } 
        });

        videoElement.src = URL.createObjectURL(video);

        return durationPromise;
    }

    async function getVideoSegments(images: ImageData[], frames: frame[], videoLength: number): Promise<segment[]> {
        let segments: segment[] = [];
        let lastUpdatedSegmentIndex = 0;
        let currentFrame = 0;

        for (; currentFrame < frames.length; currentFrame++) {
            if (compareImages(images[0], frames[currentFrame].imageData)) {
                segments.push({ start: frames[currentFrame].frameNumber, end: null });
                currentFrame++;
                
                break;
            }
        }

        progress.set(1 / images.length);
        
        for (let [imageIndex, image] of images.slice(1).entries()) {
            let matched = false;
            for (let frameIndex = currentFrame; frameIndex < frames.length; frameIndex++) {
                if (compareImages(image, frames[frameIndex].imageData)) {
                    segments[lastUpdatedSegmentIndex].end = frames[frameIndex].frameNumber;

                    segments.push({ start: frames[frameIndex].frameNumber, end: null });
                    
                    currentFrame = frameIndex + 1;
                    matched = true;
                    lastUpdatedSegmentIndex = segments.length - 1;

                    break;
                }
            }

            if (image == images[images.length - 1] && segments.length == images.length && !segments[segments.length - 1].end) {
                segments[segments.length - 1].end = videoLength;
            } else if (!matched) {
                segments.push({ start: null, end: null })
            }

            progress.set(imageIndex / images.length);
        }
        
        return segments;
    }

    function compareImages(firstImage: ImageData, secondImage: ImageData): boolean {
        let cropXPercentage = 15;
        let cropYPercentage = 35;

        let cropX = Math.round(firstImage.width * cropXPercentage / 100 / 2);
        let cropY = Math.round(firstImage.height * cropYPercentage / 100 / 2);

        let cropOptions = { top: cropY, right: cropX, bottom: cropY, left: cropX }
        
        let firstImageCropped = cropImageData(firstImage, cropOptions);
        let secondImageCropped = cropImageData(secondImage, cropOptions);

        const numDiffPixels = pixelmatch(firstImageCropped.data, secondImageCropped.data, null, firstImageCropped.width, firstImageCropped.height, { threshold: 0.1});
        
        let cutoffPercentage = 15;

        return (numDiffPixels / (firstImageCropped.width * firstImageCropped.height) * 100) < cutoffPercentage;
    }

    async function getVideoFiles(video: File, videoSegments: segment[]): Promise<File[]> {
        await ffmpegLoad;
        ffmpeg.FS('writeFile', 'video.mp4', await fetchFile(video));

        let frameRate = 25;
        let videoSegmentFiles: File[] = [];
        for (let [segmentIndex, segment] of videoSegments.entries()) { 
            if (segment.start && segment.end) {
                await ffmpeg.run('-ss', `${segment.start / frameRate}`, '-to', `${segment.end / frameRate}`, '-i', 'video.mp4', `segment${segmentIndex}.mp4`);

                let videoSegment = await ffmpeg.FS('readFile', `segment${segmentIndex}.mp4`);
                console.log('videoSegment')
                console.log(videoSegment)
            } else {
                videoSegmentFiles.push(null)
            } 
        }

        return null;
    }   


    async function zoom(e: WheelEvent) {
        if (e.ctrlKey) {
            e.preventDefault();
            scale += e.deltaY * -0.01;

            await getPages(slides);
            console.log(scale)
        }
    }

    
</script>