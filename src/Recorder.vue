<script setup>
import { ref, reactive, computed, watch, onMounted, shallowReactive} from 'vue';

// 映像サイズ
const captureSizeW = 1280
const captureSizeH = 720
const videoSizeW = 640
const videoSizeH = 360

// download file
const filename_title = "presentationvideomaker"
const filename_ext = "webm"

// utility functions 
const logging = function(obj){
    console.log(obj)
}
const isNumber = function(value) {
  return ((typeof value === 'number') && (isFinite(value)));
};

const hostname = window.location.origin;

const mediaDenied = ref(false);

// states
const nowRecordable = computed(() => ( tracks.screen != null ))
const nowRecording = computed(() => (recording.recorder != null && recording.recorder.state != 'inactive'));
const nowPlayable = computed(() => (recording.blob != null && !nowRecording.value));
const nowPlaying = ref(false);

// variables for 'ref' binding
const previewCanvas = ref(null);
const replayVideo = ref(null);
const control = ref(null);

const videos = ref(null);
const microphones = ref(null);
const speakers = ref(null);
const menu = ref(null);
const selectors = {
    videos: videos,
    microphones: microphones,
    // speakers: speakers,
    menu: menu,
};
const buttons_record = ref(null);
const buttons_pause = ref(null);
const buttons_stop = ref(null);
const buttons_play = ref(null);
const button_download = ref(null);
const button_bgImage = ref(null);

// 録画関係
const tracks = shallowReactive({
    screen: null,
    video: null,
    microphone: null,
    canvas: null,
})
let videoTrackProcessor = null;
let screenTrackProcessor = null;
let lastScreenFrame = null;
let canvasContext = null;
let mimeType = null;
const recording = shallowReactive({
    recorder: null,
    mediaStream: null,
    blob: null,
    objectURL: null,
})

// 仮想背景
let bokeh = ref(10);
let virtualBGImage = ref(null);
let bgSegmenter = null;
const segmenterOnResults = {
    blankBG(results){
        canvasContext.save()
        canvasContext.clearRect(captureSizeW, 0, videoSizeW, videoSizeH);
        canvasContext.drawImage(results.segmentationMask, captureSizeW, 0, videoSizeW, videoSizeH)
        
        canvasContext.globalCompositeOperation = "source-in"
        canvasContext.drawImage(results.image, captureSizeW, 0, videoSizeW, videoSizeH)
        results.segmentationMask.close();
        results.image.close();
        
        canvasContext.restore()

        if(lastScreenFrame){
            try{
                canvasContext.drawImage(lastScreenFrame, 0, 0, captureSizeW, captureSizeH);
            }catch(e){
                console.debug(e)
            }
        }
    },
    virtBG(results){
        canvasContext.save()
        canvasContext.clearRect(captureSizeW, 0, videoSizeW, videoSizeH);
        canvasContext.drawImage(results.segmentationMask, captureSizeW, 0, videoSizeW, videoSizeH)
        
        const thre = 100;
        var id = canvasContext.getImageData(captureSizeW, 0, videoSizeW, videoSizeH)
        for(let i = 3; i < id.data.length; i += 4){
            id.data[i] = (id.data[i] > thre)? 255:0
        }
        canvasContext.putImageData(id, captureSizeW, 0)
        
        canvasContext.globalCompositeOperation = "source-in"
        canvasContext.drawImage(results.image, captureSizeW, 0, videoSizeW, videoSizeH)
        results.segmentationMask.close();
        results.image.close();
        
        canvasContext.globalCompositeOperation = "destination-atop"
        canvasContext.drawImage(virtualBGImage.value, captureSizeW, 0, videoSizeW, videoSizeH)

        canvasContext.restore()

        if(lastScreenFrame){
            try{
                canvasContext.drawImage(lastScreenFrame, 0, 0, captureSizeW, captureSizeH);
            }catch(e){
                console.debug(e)
            }
        }
    },
    blurBG(results){
        canvasContext.save()
        canvasContext.clearRect(captureSizeW, 0, videoSizeW, videoSizeH);
        canvasContext.drawImage(results.segmentationMask, captureSizeW, 0, videoSizeW, videoSizeH)
        
        const thre = 100;
        var id = canvasContext.getImageData(captureSizeW, 0, videoSizeW, videoSizeH)
        for(let i = 3; i < id.data.length; i += 4){
            id.data[i] = (id.data[i] > thre)? 255:0
        }
        canvasContext.putImageData(id, captureSizeW, 0)
        
        canvasContext.globalCompositeOperation = "source-in"
        canvasContext.drawImage(results.image, captureSizeW, 0, videoSizeW, videoSizeH)
        
        canvasContext.globalCompositeOperation = "destination-atop"
        canvasContext.filter = `blur(${bokeh.value}px)`;
        canvasContext.drawImage(results.image, captureSizeW, 0, videoSizeW, videoSizeH)

        results.segmentationMask.close();
        results.image.close();

        canvasContext.restore()

        if(lastScreenFrame){
            try{
                canvasContext.drawImage(lastScreenFrame, 0, 0, captureSizeW, captureSizeH);
            }catch(e){
                console.debug(e)
            }
        }
    },
}

const background = reactive({
    state: 'normal',
    states: {normal:'そのまま', virtual:'仮想背景', blur:'ぼかし', blank:'背景抜き', },
    get isNormal(){
        return this.state == 'normal';
    },
    setNormal(){
        this.state = 'normal';
    },
    get isVirtual(){
        return this.state == 'virtual';
    },
    setVirtual(){
        this.state = 'virtual';
    },
    get isBlur(){
        return this.state == 'blur';
    },
    setBlur(){
        this.state = 'blur';
    },
    get isBlank(){
        return this.state == 'blank';
    },
    setBlank(){
        this.state = 'blank';
    },
    getProcessor(){
        switch(this.state){
            case 'virtual':
                if(virtualBGImage.value)
                    return segmenterOnResults.virtBG;
                else
                    return segmenterOnResults.blankBG;
                break;
            case 'blur':
                return segmenterOnResults.blurBG;
                break;
            case 'blank':
                return segmenterOnResults.blankBG;
                break;
            default:
                return null;
        }
    }
})
watch(background, (newstate, oldstate) => {
    if(videoTrackProcessor == null)
        return;
    
    const bgProcessor = background.getProcessor();
    if(bgProcessor){
        if(bgSegmenter == null){
            bgSegmenter = new SelfieSegmentation({locateFile: (file) => {
                return `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`;
            }});
            bgSegmenter.setOptions({
                //   modelSelection: 0, // general model
                modelSelection: 1, // landscape model
            });
        }

        bgSegmenter.onResults(bgProcessor);
    }else if(bgSegmenter){
        bgSegmenter.close();
        bgSegmenter = null;
    }
});
watch(virtualBGImage, (newstate, oldstate) => {
    if(videoTrackProcessor == null)
        return;
    if(!background.isVirtual)
        return;
    if(oldstate != null)
        return;
    if(bgSegmenter == null)
        return;

    bgSegmenter.onResults(background.getProcessor());
});

// parts layout variables
let windowSizeW = ref(window.innerWidth);
let windowSizeH = ref(window.innerHeight);

const previewSizeW = ref(captureSizeW);
const previewSizeH = ref(captureSizeH);
const previewScale = ref(1);
const previewStyle = computed(() => {
    return {
        width: previewSizeW.value + 'px',
        height: previewSizeH.value + 'px',
        scale: previewScale.value, // transform: 'scale(' + previewScale.value + ')' の代わりができるらしい…
    };
});
const previewCanvasStyle = reactive({
    visibility: 'visible',
    get visible(){ return this.visibility == 'visible'; },
    set visible(flag){ this.visibility = flag? 'visible' : 'hidden'; }
});
const replayVideoStyle = reactive({
    visibility: 'hidden',
    get visible(){ return this.visibility == 'visible'; },
    set visible(flag){ this.visibility = flag? 'visible' : 'hidden'; }
});
const controlStyle = reactive({
    _width: NaN,
    get width(){ return isNumber(this._width)? this._width + 'px' : '';},
    _top: NaN,
    get top(){ return isNumber(this._top)? this._top+ 'px' : '';},
    opacity: 1
});

const selectorHeightMax = ref(0);

// selector variables
const selectorsStyle = {
    videos: reactive({
        _top: 0,
        get top(){ return isNumber(this._top)? this._top+ 'px' : '';},
        _left: 0,
        get left(){ return isNumber(this._left)? this._left+ 'px' : '';},
        _width: 0,
        get width(){ return isNumber(this._width)? this._width+ 'px' : '';},
        _height: 0,
        get height(){ return isNumber(this._height)? this._height+ 'px' : '';},
        visibility: 'hidden',
        set visible(flag){ this.visibility = flag? 'visible' : 'hidden'; },
        get visible(){ return this.visibility == 'visible'; },
    }),
    microphones: reactive({
        _top: 0,
        get top(){ return isNumber(this._top)? this._top+ 'px' : '';},
        _left: 0,
        get left(){ return isNumber(this._left)? this._left+ 'px' : '';},
        _width: 0,
        get width(){ return isNumber(this._width)? this._width+ 'px' : '';},
        _height: 0,
        get height(){ return isNumber(this._height)? this._height+ 'px' : '';},
        visibility: 'hidden',
        set visible(flag){ this.visibility = flag? 'visible' : 'hidden'; },
        get visible(){ return this.visibility == 'visible'; },
    }),
    speakers: reactive({
        _top: 0,
        get top(){ return isNumber(this._top)? this._top+ 'px' : '';},
        _left: 0,
        get left(){ return isNumber(this._left)? this._left+ 'px' : '';},
        _width: 0,
        get width(){ return isNumber(this._width)? this._width+ 'px' : '';},
        _height: 0,
        get height(){ return isNumber(this._height)? this._height+ 'px' : '';},
        visibility: 'hidden',
        set visible(flag){ this.visibility = flag? 'visible' : 'hidden'; },
        get visible(){ return this.visibility == 'visible'; },
    }),
    menu: reactive({
        _top: 0,
        get top(){ return isNumber(this._top)? this._top+ 'px' : '';},
        _left: 0,
        get left(){ return isNumber(this._left)? this._left+ 'px' : '';},
        // _width: 0,
        // get width(){ return isNumber(this._width)? this._width+ 'px' : '';},
        _height: 0,
        get height(){ return isNumber(this._height)? this._height+ 'px' : '';},
        visibility: 'hidden',
        set visible(flag){ this.visibility = flag? 'visible' : 'hidden'; },
        get visible(){ return this.visibility == 'visible'; },
    })
}
const selectedVideo = ref('none');
const selectedMIC = ref('');
const selectedSPK = ref('');

const videoSelections = reactive({ none: 'なし'});
const microphoneSelections = reactive({});
const speakerSelections = reactive({});

watch(selectedVideo, (newSelection) => {
    if(bgSegmenter){
        bgSegmenter.close();
        bgSegmenter = null;
    }
    if(newSelection == 'none'){
        previewSizeW.value = captureSizeW;

        if(tracks.video != null){
            tracks.video.forEach((track) => track.stop());
            tracks.video = null;
        }
        if(videoTrackProcessor != null){
            if(! videoTrackProcessor.readable.locked) videoTrackProcessor.readable.cancel();
            videoTrackProcessor = null;
        }
        tracks.video = null;
    }
    else{
        previewSizeW.value = captureSizeW + videoSizeW;

        const bgProcessor = background.getProcessor();
        if(bgProcessor){
            bgSegmenter = new SelfieSegmentation({locateFile: (file) => {
                return `https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file}`;
            }});
            bgSegmenter.setOptions({
                //   modelSelection: 0, // general model
                modelSelection: 1, // landscape model
            });
            bgSegmenter.onResults(bgProcessor);
        }

        navigator.mediaDevices.getUserMedia({
            audio: false,
            video: {
                width: videoSizeW,
                height: videoSizeH,
                deviceId: newSelection,
            }
        }).then((video) => {
            console.log(video);
            if(tracks.video != null){
                tracks.video.forEach((track) => track.stop());
            }
            tracks.video = video.getTracks();

            if(canvasContext == null){
                canvasContext = previewCanvas.value.getContext('2d');
            }
            if(videoTrackProcessor != null && !videoTrackProcessor.readable.locked){
                videoTrackProcessor.readable.cancel();
            }
            videoTrackProcessor = new MediaStreamTrackProcessor({ track: tracks.video[0] });
            videoTrackProcessor.readable.pipeTo(new WritableStream({
                start() {
                    console.log('writable stream for video started');
                },
                write(videoFrame){
                    if(bgSegmenter){
                        createImageBitmap(videoFrame).then((bitmap) => {
                            bgSegmenter.send({image: bitmap}).then(()=>{
                                videoFrame.close();
                                bitmap.close()
                            });
                        });
                    }else{
                        if(lastScreenFrame){
                            canvasContext.drawImage(lastScreenFrame, 0, 0, captureSizeW, captureSizeH);
                        }

                        canvasContext.save();
                        //canvasContext.clearRect(captureSizeW, 0, videoSizeW, videoSizeH);

                        // canvasContext.scale(-1, 1);
                        // canvasContext.drawImage(videoFrame, -previewSizeW.value, 0, videoSizeW, videoSizeH);
                        canvasContext.drawImage(videoFrame, captureSizeW, 0, videoSizeW, videoSizeH);

                        canvasContext.restore();
                        videoFrame.close();
                    }
                },
                close(){
                    console.log('writable stream for video was closed');
                    if(tracks.video != null){
                        tracks.video.forEach((track) => track.stop());
                        tracks.video = null;
                    }
                    if(bgSegmenter){
                        bgSegmenter.close();
                        bgSegmenter = null;
                    }
                    if(lastScreenFrame)
                        canvasContext.drawImage(lastScreenFrame, 0, 0, captureSizeW, captureSizeH);
                },
                abort(err){
                    console.log('writable stream for video was aborted:', err);
                    this.close()
                },
            }));
        })
    }
})

function _loadMicrophone(){
    console.log('selected microphone:');

    if(tracks.microphone != null){
        tracks.microphone.forEach((track) => track.stop());
        tracks.microphone = null;
    }
    navigator.mediaDevices.getUserMedia({
        video: false,
        audio: {
            deviceId: selectedMIC.value,
            echoCancellation: false,
            noiseSuppression: true,
        },
    }).then((micMedia) => {
        tracks.microphone = micMedia.getTracks()
    });
}
watch(selectedMIC, _loadMicrophone);

function updatePartsSize([prevSizeW, prevSizeH]){
    const winSizeW = windowSizeW.value;
    const winSizeH = windowSizeH.value; 
    const scaleW = winSizeW / prevSizeW;
    const scaleH = winSizeH / prevSizeH;

    // preview の大きさ調整 ＝ 全体が見えるように scale を調整する
    let scale = scaleW;
    if(scaleW > scaleH)
        scale = scaleH;

    if(scale > 1)
        scale = 1;

    previewScale.value = scale;

    /*  control の位置調整
        preview は scale で縮小する
        けど、HTML要素の配置計算では、次の兄弟要素の位置は縮小する前の大きさで計算される
        次の要素の div の位置を、↑の位置からの相対位置で移動させる必要がある
        ので、top の値は負値になる
    */
    if(control.value){
        controlStyle._width = (prevSizeW * scale)
        let height = prevSizeH * scale + control.value.offsetHeight;
        //console.log(height, prevSizeH, scale, control.value.offsetHeight)
        if(height > winSizeH){
            controlStyle._top = (prevSizeH * (scale - 1) - (height - winSizeH));
            controlStyle.opacity = 0.2;
            selectorHeightMax.value = winSizeH - (height - winSizeH);
        }
        else{
            controlStyle._top = (prevSizeH * (scale - 1));
            controlStyle.opacity = 1;

            selectorHeightMax.value = prevSizeH * scale;
        }
    }
}

watch([previewSizeW, previewSizeH], updatePartsSize);

// selector の大きさ調整
const resetSelectorHeight = (ySpace) => {
    Object.keys(selectors).forEach((target) => {
        const selector = selectors[target];

        let optgheight = selector.value.firstChild.offsetHeight;
        if(optgheight > ySpace)
            selectorsStyle[target]._height = ySpace;
        else
            selectorsStyle[target]._height = optgheight;
    });
};
watch(selectorHeightMax, resetSelectorHeight);
watch(microphoneSelections, () => {
    let count = 10;
    const cancelInterval = setInterval(()=>{
        if(microphones.value.firstChild.children.length != Object.keys(microphoneSelections).length){
            if(--count) return;
            clearInterval(cancelInterval);
            console.log('faild to reset selection menu height')
            return;
        }

        resetSelectorHeight(selectorHeightMax.value);
        clearInterval(cancelInterval);
    }, 1000);
});

(async function setupSelections(){
    try{ 
        const media = await navigator.mediaDevices.getUserMedia({video:true, audio:true});
    }catch(err){
        // console.log({ error:err })
        if(err.name == 'NotAllowedError')
            mediaDenied.value = true;
    }
    // デバイスリストを取得して、そのうちカメラデバイスのみを選択リストに追加
    const devices = await navigator.mediaDevices.enumerateDevices();
    devices.forEach((device) => {
        if(device.kind == 'videoinput'){
            videoSelections[device.deviceId] = device.label
            if(device.deviceId == 'default')
                selectedVideo.value = 'default'
        }
        if(device.kind == 'audioinput'){
            microphoneSelections[device.deviceId] = device.label
            if(device.deviceId == 'default')
                selectedMIC.value = 'default'
        }
        if(device.kind == 'audiooutput'){
            speakerSelections[device.deviceId] = device.label
            if(device.deviceId == 'default')
                selectedSPK.value = 'default'
        }
    });
})();

function onWindowResize(){
    windowSizeW.value = window.innerWidth;
    windowSizeH.value = window.innerHeight;

    updatePartsSize([previewSizeW.value, previewSizeH.value]);
}

onMounted(()=>{
    onWindowResize();
    window.addEventListener('resize', onWindowResize);

    const types = ['video/webm;codecs=h264,opus', 'video/webm;codecs=h264,aac', 'video/webm;codecs=h264', 'video/webm;codecs=opus', 'video/webm'];
    for(let t of types){
        if(MediaRecorder.isTypeSupported(t) ){
            mimeType = t;
            break;
        }
    }
    console.log("mimeType is", mimeType)
})

// UI parts event processors
function onSelectorClick(button, target){
    const selector = selectors[target].value;
    const style = selectorsStyle[target]
    const br_button = button.getBoundingClientRect();
    const br_selector = selector.getBoundingClientRect();
    style._top = (br_button.top - br_selector.height);
    style._height = br_selector.height;
    const left = br_button.right - br_selector.width;
    if(left < 0)
        style._left = 0;
    else
        style._left = left;
    // style._width = br_selector.width;
    Object.keys(selectorsStyle).forEach((selector) => {
        selectorsStyle[selector].visible = (selector == target && !selectorsStyle[selector].visible )
    });
}
function closeSelectors(){
    Object.keys(selectorsStyle).forEach((selector) => {
        selectorsStyle[selector].visible = false;
    });
}

function onScreenButtonClick(){
    closeSelectors();
    navigator.mediaDevices.getDisplayMedia({
        //audio: true,
        audio: false,
        video: true,
    }).then((media) => {
        console.log(media);

        if(tracks.screen){
            tracks.screen.forEach((track) => track.stop());
        }
        tracks.screen = media.getTracks()
        const videos = media.getVideoTracks();
        videos.forEach((track) => {
            track.addEventListener('ended', (ev) => {
                console.log(ev)
                tracks.screen = null;
            });
        })

        const video = videos[0];
        video.applyConstraints({width: captureSizeW, height: captureSizeH});

        // ToDo: recording 中なら audio track を recorder に飛ばす!

        if(canvasContext == null){
            canvasContext = previewCanvas.value.getContext('2d');
        }
        if(screenTrackProcessor != null && !screenTrackProcessor.readable.locked){
            screenTrackProcessor.readable.cancel();
        }
        screenTrackProcessor = new MediaStreamTrackProcessor({ track: video });

        screenTrackProcessor.readable.pipeTo(new WritableStream({
            start() {
                console.log('writable stream for screen started');
            },
            write(screenFrame){
                if(lastScreenFrame)
                    lastScreenFrame.close();
                if( !videoTrackProcessor )
                    canvasContext.drawImage(screenFrame, 0, 0, captureSizeW, captureSizeH);
                lastScreenFrame = screenFrame;
            },
            close(){
                if(lastScreenFrame){
                    lastScreenFrame.close();
                    lastScreenFrame = null;
                }
                canvasContext.clearRect(0, 0, captureSizeW, captureSizeH);
                console.log('writable stream for screen was closed');
            },
            abort(err){
                console.log('writable stream for screen was aborted:', err);
            },
        }));
    })
}

function onStopButtonClick(event){
    if(nowRecording.value){
        recording.recorder.stop();
    }
    else if(nowPlaying.value){
        previewCanvasStyle.visible = true;
        replayVideoStyle.visible = false;

        replayVideo.value.pause();
        nowPlaying.value = false;
    }
}

function onPauseButtonClick(event){
    if(nowRecording.value){
        if(recording.recorder.state == 'recording')
            recording.recorder.pause();
        else if(recording.recorder.state == 'paused')
            recording.recorder.resume();
    }else if(nowPlaying.value){
        replayVideo.value.pause();
    }
}

function onRecordButtonClick(event){
    if(recording.blob == null){
        // nop
    }else if(recording.blob.constructor.name == 'Array' &&  recording.blob.length > 0){
        if(! confirm('録画データがあります。削除して録画を開始してよろしいですか。')){
            return;
        }
    }
    recording.blob = new Array();
    if(recording.objectURL != null){
        URL.revokeObjectURL(recording.objectURL);
    }

    const recordingTracks = Array();

    if(tracks.microphone == null){
        if(selectedMIC.value == null){
            alert('使用するマイクを選択してください')
            return false;
        } else
            _loadMicrophone();
    }else{
        tracks.microphone.forEach((track) => recordingTracks.push(track));
    }
    
    const canvasMedia = previewCanvas.value.captureStream();
    tracks.canvas = canvasMedia.getTracks();

    tracks.canvas.forEach((track) => recordingTracks.push(track));

    // screen の音声トラックを追加
    //tracks.screen.forEach((track) => {if(track.kind == 'audio') recordingTracks.push(track)});

    console.log('recording tracks:', recordingTracks);
    recording.mediaStream = new MediaStream(recordingTracks);
    console.log('recording media:', recording.mediaStream);
    recording.recorder = new MediaRecorder(recording.mediaStream, {
        audioBitsPerSecond : 128000,
        videoBitsPerSecond : 4000000,
        'mimeType' : mimeType,
    });
    console.log('recorder:', recording.recorder);
    recording.recorder.ondataavailable = function(event){
        recording.blob.push(event.data);
        console.log(recording.blob);
    }
    recording.recorder.onstop = function(event){
        console.log(event);
        if(tracks.canvas != null){
            tracks.canvas = null;
        }
        if(tracks.microphone.constructor.name == 'Array'){
            console.log('on recorder stop, stop microphone capture');
            tracks.microphone.forEach((track) => track.stop())
            tracks.microphone = null;
        }

        recording.objectURL = URL.createObjectURL(new Blob(recording.blob, { type: mimeType }));
        buttons_pause.value.style.animationName = '';
        recording.recorder = null;
    }
    recording.recorder.onerror = function(event){
        console.log(event);
        alert('録画中にエラーが発生しました:' + event.error);

        recording.recorder.onstop(event);
    }
    recording.recorder.onpause = function(event){
        console.log(event);
        buttons_pause.value.style.animationName = 'emphasis';
    }
    recording.recorder.onresume = function(event){
        console.log(event);
        buttons_pause.value.style.animationName = '';
    }

    recording.recorder.start();
}

function onPlayButtonClick(event){
    if(recording.blob == null){
        alert('録画データがありません');
        return;
    }

    if(! replayVideoStyle.visible){
        replayVideo.value.src = recording.objectURL;

        previewCanvasStyle.visible = false;
        replayVideoStyle.visible = true;
    }

    replayVideo.value.play();
    nowPlaying.value = true;
}

function onDownloadButtonClick(event){
    closeSelectors();
    if(recording.objectURL != null){
        event.target.href = recording.objectURL;
        event.target.download = filename_title + "." + (new Date()).toISOString() + "." + filename_ext; 
    }
}

function onBackgroundImageFileSelected(event){
    createImageBitmap(event.target.files[0]).then((img) => virtualBGImage.value = img);
    background.setVirtual();
}
</script>

<template>
<div class="origin-top-left bg-orange-50 relative" :style="previewStyle">
<canvas ref="previewCanvas" class="absolute top-0 left-0 w-full h-full" :style="previewCanvasStyle" :width="previewSizeW" :height="previewSizeH"></canvas>
<video  ref="replayVideo"   class="absolute top-0 left-0 w-full h-full" :style="replayVideoStyle" controls></video>
</div>
<div ref="control" class="relative text-red-500 bg-slate-800 text-6xl font-sans px-2 bottom-0 hover:!opacity-100" :style="controlStyle">
    <!-- <button :disabled="recording" class="disabled:text-red-100 disabled:bg-slate-300 pb-2 px-2">◉</button>-->
    <span class="text-blue-500 absolute right-2 font-serif pt-1">
        <button type="button" name="screen"     :disabled="nowPlaying" class="disabled:opacity-25 px-2" @click="onScreenButtonClick">&#x1F4BB;&#xFE0E;</button>
        <button type="button" name="video"      :disabled="nowRecording | nowPlaying" class="disabled:opacity-25 px-2"  @click="onSelectorClick($event.target, 'videos')">&#x1F3A5;&#xFE0E;</button>
        <button type="button" name="microphone" :disabled="nowRecording | nowPlaying" class="disabled:opacity-25 px-2"  @click="onSelectorClick($event.target, 'microphones')">&#x1F3A4;&#xFE0E;</button>
        <button type="button" name="speaker"    :disabled="nowRecording" class="disabled:opacity-25 px-2 rotate-180 hidden" @click="onSelectorClick($event.target, 'speakers')">&#x1F56A;&#xFE0E;</button>
        <a ref="button_download" class="px-2 cursor-pointer"  :class="{ 'opacity-25': !nowPlayable }" @click="onDownloadButtonClick">&#x1F4E5;&#xFE0E;</a>
        <button type="button" id="menu" class="disabled:opacity-25 text-white px-2" @click="onSelectorClick($event.target, 'menu')">☰</button>
    </span>
    <button type="button" ref="buttons_record" :disabled="!(nowRecordable & !nowRecording & !nowPlaying)"  class="disabled:opacity-25 py-1 px-2" @click="onRecordButtonClick">◉</button>
    <button type="button" ref="buttons_pause"  :disabled="!(nowRecording | nowPlaying)" class="disabled:opacity-25 pb-2 px-1 rotate-90 emphasis" @click="onPauseButtonClick">〓</button>
    <button type="button" ref="buttons_stop"   :disabled="!(nowRecording | nowPlaying)" class="disabled:opacity-25 pb-2 px-2" @click="onStopButtonClick">◼</button>
    <button type="button" ref="buttons_play"   :disabled="!(nowPlayable)"  class="disabled:opacity-25 pb-2 px-2" @click="onPlayButtonClick">▶</button>
</div>
<div class="static">
    <select size=2
            ref="videos" id="videos"
            class="absolute text-xl border border-blue-900 shadow min-w-max"
            :style="selectorsStyle.videos"
            @click="selectorsStyle.videos.visibility = 'hidden'"
            v-model="selectedVideo"
    >
        <optgroup label="カメラを選択" class="bg-blue-900 text-white" onchange="console.log('fuga')">
            <option v-for="(value, key) in videoSelections" :value="key" class="bg-blue-400 checked:bg-blue-50">{{ value }}</option>
        </optgroup>
    </select>
    <select size=2
            ref="microphones" id="microphones"
            class="absolute text-xl border border-slate-900 shadow min-w-max"
            :style="selectorsStyle.microphones"
            @click="selectorsStyle.microphones.visibility = 'hidden'"
            v-model="selectedMIC"
    >
        <optgroup label="マイクを選択" class="bg-blue-900 text-white">
            <option v-for="(value, key) in microphoneSelections" :value="key" class="bg-blue-400 checked:bg-blue-50">{{ value }}</option>
        </optgroup>
    </select>
    <select size=2
            ref="speakers" id="speakers"
            class="absolute text-xl border border-slate-900 shadow min-w-max"
            :style="selectorsStyle.speakers"
            @click="selectorsStyle.speakers.visibility = 'hidden'"
            v-model="selectedSPK"
    >
        <optgroup label="音声出力を選択" class="bg-blue-900 text-white">
            <option v-for="(value, key) in speakerSelections" :value="key" class="bg-blue-400 checked:bg-blue-50">{{ value }}</option>
        </optgroup>
    </select>
    <div ref="menu"
         id="menu"
         class="absolute w-fit text-2xl"
         :style="selectorsStyle.menu"
         
    >
        <div class="rounded-t-xl bg-blue-900 text-white">
            <p class="px-4 py-1">背景</p> 
            <div class="pl-10 pr-5 py-2 bg-blue-400 checked:bg-blue-50">
                <label class="block my-1">
                    <input type="radio" value="normal"  v-model="background.state" />
                    そのまま
                </label>
                <label class="block my-1">
                    <input type="radio" value="virtual" v-model="background.state" />
                    仮想背景
                    <label class="px-3 my-1 bg-blue-900 ">
                        <input type="file" class="hidden" accept="image/gif, image/jpeg, image/png" @change="onBackgroundImageFileSelected"/>
                        <span :class="{hidden: virtualBGImage == null}">&#x1F5F9;&#xFE0E;</span>
                        <span :class="{hidden: virtualBGImage != null}">&#x1F4C4;&#xFE0E;</span>
                        背景画像を選択
                    </label>
                </label>
                <label class="block my-1">
                    <input type="radio" value="blur"    v-model="background.state" />
                    ぼかし
                    <label class="ml-10 px-3 my-1 bg-blue-900 ">
                        度合
                        <input type="range" min="3" max="10" step="1" v-model="bokeh"/>
                    </label>
                </label>
                <label class="block my-1">
                    <input type="radio" value="blank"   v-model="background.state" />
                    背景抜き
                </label>
            </div>
            
        </div>
    </div>
    <div  class="absolute w-full top-1/2 -translate-y-1/2 text-center text-red-700 font-bold text-4xl bg-white py-10" :style="{display: mediaDenied? 'block':'none'}">
        カメラとマイクへのアクセスが拒否されました。<br/>
        利用するには、ブラウザの設定から<br/>
        <span class="text-blue-700">{{ hostname }}</span><br/>
        へカメラとマイクへのアクセスを許可したうえで、リロードしてください。
    </div>
</div>
</template>

<style>
.emphasis {
    animation-duration: 1500ms;
    animation-timing-function: ease-in-out;
    animation-iteration-count: infinite;
}
@keyframes emphasis {
    0%, 100% {
        transform: scale(1) rotate(90deg);
    }
    50% {
        transform: scale(1.2) rotate(90deg);
    }
}
</style>