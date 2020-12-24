<script>
    import { createFFmpeg, fetchFile } from "@ffmpeg/ffmpeg";
    const ffmpeg = createFFmpeg({ log: true});
    const ffmpegLoad = ffmpeg.load()

    async function videoToSlides(video) {
        await ffmpegLoad;

        ffmpeg.FS('writeFile', 'lecture.mp4', );

        console.log('what')
        await ffmpeg.run('-i', 'lecture.mp4', '-r', '1', 'frame%04d.jpg');
    }
</script>